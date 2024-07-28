---
title: 'PyTorch 的 Dataloader (Multiple Process)'
date: 2020-07-18
permalink: /posts/2020/07/pytorch-dataloader-multi-process/
tags:
  - PyTorch
---

> 上一篇讲 Dataloader 的文章中分析了单进程的 Dataloader 工作的情况，本文分析多进程情况下 Dataloader 的工作流程。


上篇中的单进程 Dataloader，**iter()** 返回的 `Iterable` 实例是 `_SingleProcessDataLoaderIter`，这里由于分析多进程的 Dataloader，实际分析的就是 `MultiProcessDataLoaderIter` 实例不断 **next** 取值的过程。`MultiProcessDataLoaderIter` 这个类的注释中有一个数据流的示意图：

                main process                              ||
                     |                                    ||
               {index_queue}                              ||
                     |                                    ||
              worker processes                            ||     DATA
                     |                                    ||
            {worker_result_queue}                         ||     FLOW
                     |                                    ||
      pin_memory_thread of main process                   ||   DIRECTION
                     |                                    ||
               {data_queue}                               ||
                     |                                    ||
                data output                               \/

这个图已经点明了 Dataloader 取数据的主要流程：主进程生成 `index_queue`，worker 进程读取数据放入 `worker_result_queue`，如果 `pin_memory` 为 `True`，主进程还有一个 `pin_memory_thread` 线程，处理得到的数据放入 `data_queue`，输出得到结果。

源码注释主要解释了如何 **gracefully exit**，这里只是想弄清楚 Dataloader 工作的流程，且还是假设数据读到内存而不是显存，所以只分析 `pin_memory` 为 `False` 的情况下读取数据的代码。

# 初始化

首先是检查 `num_workers` 并给 `multiprocessing_context` 赋值：
```python
def __init__(self, loader):
    super(_MultiProcessingDataLoaderIter, self).__init__(loader)

    assert self._num_workers > 0

    if loader.multiprocessing_context is None:
        multiprocessing_context = multiprocessing
    else:
        multiprocessing_context = loader.multiprocessing_context
```
`multiprocessing_context` 是初始化参数 `loader` 的一个属性，字面意思是可以指定多进程的上下文环境，可以指定为已经开启的进程。默认情况下，上下文会被指定成 `multiprocessing`，但这并不是 Python 自带的 `multiprocessing` 包，因为开头的 import 部分是这么写的：

```python
import multiprocessing as python_multiprocessing
import torch.multiprocessing as multiprocessing
```

所以默认的上下文是 PyTorch 的 multiprocessing，这是一个在原生 multiprocessing 上稍作了一些包装的模块。

随后是一些属性的赋值：

```python
    self._worker_init_fn = loader.worker_init_fn
    self._worker_queue_idx_cycle = itertools.cycle(range(self._num_workers))
    self._worker_result_queue = multiprocessing_context.Queue()
    self._worker_pids_set = False
    self._shutdown = False
    self._send_idx = 0  # idx of the next task to be sent to workers
    self._rcvd_idx = 0  # idx of the next task to be returned in __next__
    # information about data not yet yielded, i.e., tasks w/ indices in range [rcvd_idx, send_idx).
    # map: task idx => - (worker_id,)        if data isn't fetched (outstanding)
    #                  \ (worker_id, data)   if data is already fetched (out-of-order)
    self._task_info = {}
    self._tasks_outstanding = 0  # always equal to count(v for v in task_info.values() if len(v) == 1)
    self._workers_done_event = multiprocessing_context.Event()
```
相应的属性含义在名字和注释中写得比较清楚了，注意 `self._task_info` 中放的信息是取数据的 task 与各个 worker 的对应关系。
接着初始化 worker：
```python
    self._index_queues = []
    self._workers = []
    # A list of booleans representing whether each worker still has work to
    # do, i.e., not having exhausted its iterable dataset object. It always
    # contains all `True`s if not using an iterable-style dataset
    # (i.e., if kind != Iterable).
    self._workers_status = []
    for i in range(self._num_workers):
        index_queue = multiprocessing_context.Queue()
        # index_queue.cancel_join_thread()
        w = multiprocessing_context.Process(
            target=_utils.worker._worker_loop,
            args=(self._dataset_kind, self._dataset, index_queue,
                  self._worker_result_queue, self._workers_done_event,
                  self._auto_collation, self._collate_fn, self._drop_last,
                  self._base_seed + i, self._worker_init_fn, i, self._num_workers))
        w.daemon = True
        # NB: Process.start() actually take some time as it needs to
        #     start a process and pass the arguments over via a pipe.
        #     Therefore, we only add a worker to self._workers list after
        #     it started, so that we do not call .join() if program dies
        #     before it starts, and __del__ tries to join but will get:
        #     AssertionError: can only join a started process.
        w.start()
        self._index_queues.append(index_queue)
        self._workers.append(w)
        self._workers_status.append(True)
```
主要是开启了 `num_workers` 个取数据的 worker 进程和放 index 的 `index_queue` 并放到相应的列表中。
```python
    if self._pin_memory:
        # ...
    else:
        self._data_queue = self._worker_result_queue

    _utils.signal_handling._set_worker_pids(id(self), tuple(w.pid for w in self._workers))
    _utils.signal_handling._set_SIGCHLD_handler()
    self._worker_pids_set = True

    # prime the prefetch loop
    for _ in range(2 * self._num_workers):
        self._try_put_index()
```

最后一块：

* 处理 `pin_memory` 相关的初始化，由于 `pin_memory` 为 `False`，`data_queue` 即返回的数据队列就是 `worker_result_queue` 即 worker 进程取到的数据所存放的队列
* 设置信号以及进程号信息
* 进行 2 * `num_workers` 次 prefetch，具体而言就是执行 `_try_put_index` 函数



至此，初始化完成，其中有两个仍需分析的地方：worker 进程和 `_try_put_index` 的具体逻辑。接下来逐个具体分析。

# worker 子进程

上面 fork worker 子进程的代码显示 worker 执行的是 `_utils.worker._worker_loop` 函数：

```python
def _worker_loop(dataset_kind, dataset, index_queue, data_queue, done_event,
                 auto_collation, collate_fn, drop_last, seed, init_fn, worker_id,
                 num_workers):
    # See NOTE [ Data Loader Multiprocessing Shutdown Logic ] for details on the
    # logic of this function.

    try:
        # Intialize C side signal handlers for SIGBUS and SIGSEGV. ...
        signal_handling._set_worker_signal_handlers()

        torch.set_num_threads(1)
        random.seed(seed)
        torch.manual_seed(seed)

        global _worker_info
        _worker_info = WorkerInfo(id=worker_id, num_workers=num_workers,
                                  seed=seed, dataset=dataset)

        from torch.utils.data import _DatasetKind

        init_exception = None

        try:
            if init_fn is not None:
                init_fn(worker_id)

            fetcher = _DatasetKind.create_fetcher(dataset_kind, dataset, auto_collation, collate_fn, drop_last)
        except Exception:
            init_exception = ExceptionWrapper(
                where="in DataLoader worker process {}".format(worker_id))

        # When using Iterable mode, some worker can exit earlier than others ...
        iteration_end = False

        watchdog = ManagerWatchdog()

        while watchdog.is_alive():
            try:
                r = index_queue.get(timeout=MP_STATUS_CHECK_INTERVAL)
            except queue.Empty:
                continue
            if r is None:
                # Received the final signal
                assert done_event.is_set() or iteration_end
                break
            elif done_event.is_set() or iteration_end:
                # `done_event` is set. But I haven't received the final signal
                # (None) yet. I will keep continuing until get it, and skip the
                # processing steps.
                continue
            idx, index = r
            if init_exception is not None:
                data = init_exception
                init_exception = None
            else:
                try:
                    data = fetcher.fetch(index)
                except Exception as e:
                    # ...
                    pass
            data_queue.put((idx, data))
            del data, idx, index, r  # save memory
    except KeyboardInterrupt:
        # Main process will raise KeyboardInterrupt anyways.
        pass
    if done_event.is_set():
        data_queue.cancel_join_thread()
        data_queue.close()
```

这里省略了一些解释和 `_DatasetKind.Iterable` 相关的以及处理异常的注释和代码，核心逻辑就是 `while watchdog.is_alive()` 的循环：不断从 `index_queue` 中取值，得到的是 `(idx, index)` 这样的 `tuple`，其中 `index` 就是前一篇中看到的 `fetcher` 用来取数据的索引，取到数据后将 `(idx, data)` 这样的 `tuple` 放入 `data_queue` 中，开启进程的代码 `args=(self._dataset_kind, self._dataset, index_queue, self._worker_result_queue,` 显示 `data_queue` 就是 `worker_result_queue` ，所以取到的数据实际上是送到了 `worker_result_queue` 中去。

结合注释中数据流向的那张图可以发现符合图中的示意，`index_queue` 中取到的 `(idx, index)` 是主进程放进去的，这里的 `watchdog` 就是一个不断检查主进程是否还活着的“看门狗”，只要主进程不死亡，worker 进程就不跳出这个循环，不断尝试取出 `index_queue` 中的 `(idx, index)` 送给 `fetcher` 取数据。

# _try_put_index 函数

```python
def _try_put_index(self):
    assert self._tasks_outstanding < 2 * self._num_workers
    try:
        index = self._next_index()
    except StopIteration:
        return
    for _ in range(self._num_workers):  # find the next active worker, if any
        worker_queue_idx = next(self._worker_queue_idx_cycle)
        if self._workers_status[worker_queue_idx]:
            break
    else:
        # not found (i.e., didn't break)
        return

    self._index_queues[worker_queue_idx].put((self._send_idx, index))
    self._task_info[self._send_idx] = (worker_queue_idx,)
    self._tasks_outstanding += 1
    self._send_idx += 1
```

前面初始化的代码注释中有解释到 `tasks_outstanding` 实际上对应的是已有的还没有取出数据的任务个数，当这个数少于 worker 数的两倍时，就取出下一批 `index` (对应过程见上一篇文章)，然后通过不断 next 并检查存活状态找到下一个存活的 worker，在对应的 `index_queue` 中放入 `(send_idx, index)` 供 worker 进程取，对应的 `task_info` 也相应地设置成 `(worker_queue_idx,)` 这样的 `tuple` 。

`try_put_index` 就是这样，轮转着给各个 worker 的 `index_queue` 中放入 `(send_idx, index)` 信息，从这里也可以看出多进程 Dataloader 的逻辑中，将 `index` 送到 `index_queue` 这个过程叫 `send`，那么自然，从 worker 读得的数据存放的 `worker_result_queue` 中取出结果这个过程叫 `receive` 了。

初始化完成后，就要不断调用`_MultiProcessingDataLoaderIter` 的 `next` 操作来取数据了，所以接下来分析 `__next__` 的实现。

# \_\_next\_\_

```python
def __next__(self):
    while True:
        # If the worker responsible for `self._rcvd_idx` has already ended ...
        while self._rcvd_idx < self._send_idx:
            info = self._task_info[self._rcvd_idx]
            worker_id = info[0]
            if len(info) == 2 or self._workers_status[worker_id]:  # has data or is still active
                break
            del self._task_info[self._rcvd_idx]
            self._rcvd_idx += 1
        else:
            # no valid `self._rcvd_idx` is found (i.e., didn't break)
            self._shutdown_workers()
            raise StopIteration

        # Now `self._rcvd_idx` is the batch index we want to fetch

        # Check if the next sample has already been generated
        if len(self._task_info[self._rcvd_idx]) == 2:
            data = self._task_info.pop(self._rcvd_idx)[1]
            return self._process_data(data)

        assert not self._shutdown and self._tasks_outstanding > 0
        idx, data = self._get_data()
        self._tasks_outstanding -= 1

        if self._dataset_kind == _DatasetKind.Iterable:
            # ...
            pass
        if idx != self._rcvd_idx:
            # store out-of-order samples
            self._task_info[idx] += (data,)
        else:
            del self._task_info[idx]
            return self._process_data(data)
```

总体逻辑还是比较简单的。首先，由于总体的逻辑是先由主进程 send index，worker 子进程取出数据再由主进程 receive data，所以有效的 `rcvd_idx` 应当总是小于 `send_idx`，在这个条件下不断检查下一个 `rcvd_idx` 是否符合要求直到找到符合要求的即还未从 `worker_result_queue` 中取出数据的 `rcvd_idx`。

此时对应的 `task_info` 应当是 `(worker_queue_idx,)` 所以 `len(self._task_info[self._rcvd_idx])` 为 1，执行 `self._get_data()` 来取出形式为 `idx, data` 的数据。

最后修改对应的信息，如果发现对应的 `send_idx` 和 `rcvd_idx` 不匹配即收到乱序数据，就往 `task_info` 中的 `tuple` 加入 `data`，对应前面 `len(self._task_info[self._rcvd_idx]) == 2` 的情况，返回的数据还要经过 `process_data` 函数处理一下。

那么 `self._get_data` 和 `self._process_data` 分别是什么逻辑呢？

```python
def _try_get_data(self, timeout=_utils.MP_STATUS_CHECK_INTERVAL):
    # Tries to fetch data from `self._data_queue` once for a given timeout ...
    # Returns a 2-tuple:
    #   (bool: whether successfully get data, any: data if successful else None)
    try:
        data = self._data_queue.get(timeout=timeout)
        return (True, data)
    except Exception as e:
        # ...
        pass

def _get_data(self):
    # Fetches data from `self._data_queue` ...
    if self._timeout > 0:
        success, data = self._try_get_data(self._timeout)
        if success:
            return data
        else:
            raise RuntimeError('DataLoader timed out after {} seconds'.format(self._timeout))
    elif self._pin_memory:
        # ...
        pass
    else:
        while True:
            success, data = self._try_get_data()
            if success:
                return data
```

`get_data` 逻辑非常简单，就是在 `try_get_data` 上包了一层，从 `data_queue` 中取出 worker 进程放进去的数据。

```python
def _process_data(self, data):
    self._rcvd_idx += 1
    self._try_put_index()
    if isinstance(data, ExceptionWrapper):
        data.reraise()
    return data
```

同样 `process_data` 也很简单，保持 `rcvd_idx` 一致并执行 `try_put_index` 函数，完成这些扫尾工作后返回数据。由此可见，主进程取出 worker 子进程放进去的数据后取下一批数据的 `index` 索引放到 `index_queue` 中，这样每读完一个 batch 就放入了下一个 batch 的 `index`，保证 `worker` 子进程不断在工作。



至此，PyTorch 的 Dataloader 多进程版本基本工作流程分析结束，现在再看源码注释中那张数据流向的图便会更加清晰：

* 数据由 worker 子进程读出放入 `worker_result_queue` 中，主进程从队列中取出数据并将 `index` 送到 `index_queue` 中供 worker 取用
* 整个过程由 `send_idx`、`rcvd_idx`、`task_info` 等变量控制 task 到 worker 的分配、数据读取的乱序等问题
* fetch 取数据操作是 fork 出的 worker 子进程执行的
* 初始化一次性执行了 2 * num_workers 次 `try_put_index` 操作，可以想象，worker 子进程读数据是耗时操作，主进程取数据和其他后处理耗时少一些，这样几乎可以一直保持各个 worker 在读数据的同时还有一个 `(idx, index)` 的任务等待执行

