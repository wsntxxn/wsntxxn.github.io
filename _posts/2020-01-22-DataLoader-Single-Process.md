---
title: 'PyTorch 的 Dataloader (Single Process)'
date: 2020-01-22
permalink: /posts/2020/01/pytorch-dataloader-single-process/
tags:
  - PyTorch
---

> Dataloader 是 PyTorch 提供的根据需求载入数据的接口，但许多地方看文档会觉得不清不楚，阅读 PyTorch 源码，有助于我们了解框架背后具体做了什么，更灵活地写符合需求的代码。

# 前言

第一次看学长的 PyTorch 训练 pipeline 时看到 dataloader 部分可以说是一脸懵逼，无数个问号划过我的脑海，主要是关于 `collate_fn` 这个传给 Dataloader 的参数：啥是 collate_fn? 为啥要传这个参数? 它接收的参数是啥? Dataloader 又是怎么工作的，怎么给主程序 load 数据的? 不过当时这些疑问过也就过去了，代码跑的没问题就不管它。后来终于遇到了自己希望不同类型的数据进行不同的 collate 方式需求，于是需要看 Dataloader 传给 collate_fn 的究竟是啥，也就干脆把 Dataloader 的源码都看了一遍，感觉还不错，记录下来，以备日后查阅。本篇先介绍一下单进程 (single process) 下 Dataloader 的工作流程。

# Dataset

分析 Dataloader 当然要从 **Dataset** 开始谈起，因为 Dataset 是数据的来源。Dataset 非常简单，只是个需要实现 `__getitem__` 和 `__len__` 的类：

```python
class Dataset(object):
      r"""
      All datasets that represent a map from keys to data samples should subclass
      it. All subclasses should overrite :meth:`__getitem__`, supporting fetching a
      data sample for a given key. Subclasses could also optionally overwrite
      :meth:`__len__`, which is expected to return the size of the dataset by many
      :class:`~torch.utils.data.Sampler` implementations and the default options
      of :class:`~torch.utils.data.DataLoader`.
      """
   
      def __getitem__(self, index):
          raise NotImplementedError
            
      def __add__(self, other):
          return ConcatDataset([self, other])
```

文档中写得很清楚：

> All datasets that represent a map from keys to data samples should subclass it. All subclasses should overrite `__getitem__`, supporting fetching a data sample for a given key. Subclasses could also optionally overwrite `__len__`.

这就是一个需要自己实现的 map-style 的根据 key 取 data 的类，`__len__` 可以实现也可以不实现。这里假设我们实验中的 Dataset 是这么定义的：

```python
class dataset(torch.utils.data.Dataset):
    
    def __init__(self, h5file, label_df, colname=("h5index", "label")):
        self._h5file = h5file
        self._label_df = label_df
        self._colname = colname
        
    def __getitem__(self, index):
        if self._dataset is None:
            self._dataset = h5py.File(self._h5file, "r")
        h5index, label = self._label_df.iloc[index].reindex(self._colname).values
        data = self._dataset[h5index][()]
        return torch.as_tensor(data), torch.as_tensor(label)
    
    def __len__(self):
        return len(self._label_df)
```

`h5file` 是存特征的 hdf5 文件，`label_df` 是处理过的记录标签的 `pandas.DataFrame`，`colname` 是 `label_df`中记录 hdf5 文件的索引和标签的那两列列名。

以这个实现为例，我们看看 `Dataloader` 读取数据经过了怎样的一个过程。

# Dataloader 初始化

Dataloader 的参数很多，有些是互斥的，这里还是以最简单的情况为例分析数据读取流程：

```python
ds = dataset(h5file, label_df)
dl = torch.utils.data.Dataloader(ds)
```

那么其他参数都是 Dataloader 的默认参数了，需要注意的是 **num_workers** 此时默认为 0，也就是单进程读取。

```python
class DataLoader(object):
    
    __initialized = False
    
    def __init__(self, dataset, batch_size=1, shuffle=False, sampler=None,
                 batch_sampler=None, num_workers=0, collate_fn=None,
                 pin_memory=False, drop_last=False, timeout=0,
                 worker_init_fn=None, multiprocessing_context=None):
        torch._C._log_api_usage_once("python.data_loader")

        if num_workers < 0:
            raise ValueError('num_workers option should be non-negative; '
                             'use num_workers=0 to disable multiprocessing.')

        if timeout < 0:
            raise ValueError('timeout option should be non-negative')

            self.dataset = dataset
            self.num_workers = num_workers
            self.pin_memory = pin_memory
            self.timeout = timeout
            self.worker_init_fn = worker_init_fn
            self.multiprocessing_context = multiprocessing_context

```

首先检查参数并将初始化参数设置一下。

```python
if isinstance(dataset, IterableDataset):
    # ...
    pass
else:                                                                             
    self._dataset_kind = _DatasetKind.Map

```

这里根据 dataset 的种类来检查其他的参数。`_DatasetKind`中定义了`Map`和`Iterable`两种dataset，很好理解：`Map`就是我们前面定义的那样，按 map 风格用 key 取 value 的 dataset；`Iterable`就是只能迭代取值无法随机读的 dataset，适合 stream data。所以这里`_dataset_kind`就设置成了`Map`，`Iterable`的分支代码在此就省略，让思路更清晰。

```python
class _DatasetKind(object):
    Map = 0    
    Iterable = 1

    @staticmethod
    def create_fetcher(kind, dataset, auto_collation, collate_fn, drop_last):
        # ...
        pass
```

接下来要确定`sampler`或者`batch_sampler`，这是用来生成要读取数据的 index 的采样器，既可以传入自己实现的按自己需求取数据的`sampler`，也可以传入直接返回 index batch 的`batch_sampler`。

```python
if sampler is not None and shuffle:
    # ...
    pass

if batch_sampler is not None:
    # ...
    pass
elif batch_size is None:
    # ...
    pass

if sampler is None:  # give default samplers
    if self._dataset_kind == _DatasetKind.Iterable:   
        sampler = _InfiniteConstantSampler()
    else:  # map-style
        if shuffle:
            sampler = RandomSampler(dataset)
        else:
            sampler = SequentialSampler(dataset)

        if batch_size is not None and batch_sampler is None:
            # auto_collation without custom batch_sampler
            batch_sampler = BatchSampler(sampler, batch_size, drop_last)

self.batch_size = batch_size
self.drop_last = drop_last
self.sampler = sampler
self.batch_sampler = batch_sampler
```

默认的 `sample` 和 `batch_sampler` 是 `None`，`batch_size`是 1，`shuffle` 是 `False` 所以 `sampler` 设置成了 `SequentialSampler`，这个类的详细实现见源码，非常简单，就是一个顺序生成 index 的 `Iterable`；如果 `shuffle` 是 `True`，`sampler` 就是 `RandomSampler`，也是一个很简单的实现，只是将全体 index 先打乱，再顺序生成。

接着 `batch_sampler` 就是默认给定的 `BatchSampler`，它的实现也很简单：

```python
def __iter__(self):
    batch = []
    for idx in self.sampler:
        batch.append(idx)
        if len(batch) == self.batch_size:
            yield batch
            batch = []
    if len(batch) > 0 and not self.drop_last:
        yield batch
```

迭代过程就是不断往 `batch` 中添加 `sampler` 生成的 index 直到达到 `batch_size`，返回这个 index 组成的 batch。

```python
if collate_fn is None:
    if self._auto_collation:
        collate_fn = _utils.collate.default_collate
    else:
        collate_fn = _utils.collate.default_convert
           
self.collate_fn = collate_fn
self.__initialized = True
```

最后是初始化 `collate_fn`，这个参数表示将根据 index 取出的 samples 整合成需要的输入的函数，比如对于变长输入就需要 pad 到长度最长的 sample。这里是根据 `_auto_collation` 这个参数确定的。

```python
@property                                                                             
def _auto_collation(self):
    return self.batch_sampler is not None
```

那么很明显，`batch_sampler` 不是 `None`，于是 `collate_fn` 就是 `default_collate`。

# 数据读取

循环读取数据就是对 **iter()** 得到的 `Iterable` 结果不断 **next** 取值，于是：

```python
def __iter__(self): 
    if self.num_workers == 0:
        return _SingleProcessDataLoaderIter(self)
    else:
        return _MultiProcessingDataLoaderIter(self)
```

由于是单线程，返回的 `Iterable` 就是 `_SingleProcessDataLoaderIter`。

```python
class _SingleProcessDataLoaderIter(_BaseDataLoaderIter):
      def __init__(self, loader):
          super(_SingleProcessDataLoaderIter, self).__init__(loader)
          assert self._timeout == 0
          assert self._num_workers == 0
        
          self._dataset_fetcher = _DatasetKind.create_fetcher(
              self._dataset_kind, self._dataset, self._auto_collation, self._collate_fn, self._drop_last)
       
      def __next__(self):
          index = self._next_index()  # may raise StopIteration
          data = self._dataset_fetcher.fetch(index)  # may raise StopIteration
          if self._pin_memory:
              data = _utils.pin_memory.pin_memory(data)
          return data
       
      next = __next__  # Python 2 compatibility

```

我们来看 `__next__` 的过程：总体思路很简单，先生成 index，然后根据 index 来 fectch data，如果有 pin_memory 的要求再进行相应的操作，这里我们设定数据只读到内存的情况，不需要 pin_memory，所以就直接返回 data。

## 生成 index

生成 index 是通过 `_next_index` 完成的, 定义在基类 `_BaseDataLoaderIter` 中：

```python
class _BaseDataLoaderIter(object):
    def __init__(self, loader):
        self._dataset = loader.dataset
        self._dataset_kind = loader._dataset_kind
        self._auto_collation = loader._auto_collation
        self._drop_last = loader.drop_last
        self._index_sampler = loader._index_sampler
        self._num_workers = loader.num_workers
        self._pin_memory = loader.pin_memory and torch.cuda.is_available()
        self._timeout = loader.timeout
        self._collate_fn = loader.collate_fn
        self._sampler_iter = iter(self._index_sampler)
        self._base_seed = torch.empty((), dtype=torch.int64).random_().item()

    def __iter__(self):
        return self

    def _next_index(self):
        return next(self._sampler_iter)  # may raise StopIteration
```

可以看到，指向的是 `_sampler_iter`，这又是 `iter(self._index_sampler)`，即 `loader._index_sampler`，定义在 `Dataloader` 的参数中： 

```python
@property
def _index_sampler(self):
    if self._auto_collation:
        return self.batch_sampler
    else:
        return self.sampler
```

前面已分析过，`_auto_collation` 是 `True`，所以 `index_sampler` 就是 `self.batch_sampler`，于是生成的 index 就是前面分析过的迭代过程：不断往`batch`中添加`sampler`生成的 index 直到达到`batch_size`，返回这个 index 组成的 batch。

## fetch data

fetch data 的操作就是这一句：

```python
data = self._dataset_fetcher.fetch(index)
```

那么 `_dataset_fetcher` 是怎么定义的呢？

```python
self._dataset_fetcher = _DatasetKind.create_fetcher(
    self._dataset_kind, self._dataset, self._auto_collation, self._collate_fn, self._drop_last)
```

是 `_DatasetKind` 中定义的：

```python
@staticmethod
def create_fetcher(kind, dataset, auto_collation, collate_fn, drop_last):  
    if kind == _DatasetKind.Map:
        return _utils.fetch._MapDatasetFetcher(dataset, auto_collation, collate_fn, drop_last)
    else:  
        return _utils.fetch._IterableDatasetFetcher(dataset, auto_collation, collate_fn, drop_last)
```

显然这里用的是 `_MapDatasetFetcher`：

```python
class _BaseDatasetFetcher(object):
    def __init__(self, dataset, auto_collation, collate_fn, drop_last):
        self.dataset = dataset
        self.auto_collation = auto_collation
        self.collate_fn = collate_fn
        self.drop_last = drop_last

    def fetch(self, possibly_batched_index):
        raise NotImplementedError()

class _MapDatasetFetcher(_BaseDatasetFetcher):
    def __init__(self, dataset, auto_collation, collate_fn, drop_last):
        super(_MapDatasetFetcher, self).__init__(dataset, auto_collation, collate_fn, drop_last)

    def fetch(self, possibly_batched_index):
        if self.auto_collation:
            data = [self.dataset[idx] for idx in possibly_batched_index]
        else:
            data = self.dataset[possibly_batched_index]
        return self.collate_fn(data)
```

所以这里取数据的过程就很清楚了：`auto_collation` 就是 `_SingleProcessDataLoaderIter` 的 `_auto_collation`，值为 `True`，所以 `fetch` 函数先将 index batch 中每个 index 对应 dataset 中的数据取出来放到一个 list 中，然后返回经过 `collate_fn` 整合处理之后的结果。

这里的 `collate_fn` 是 `default_collate`，就不分析代码了，过程也很简单，就是将各个单独的数据整合成一个总的 `Tensor`，多出第一维即 batch_size。



总结一下，`Dataset` 是 `Map` 风格的或者 `Iterable` 的生成数据的数据源；`Dataloader` 接受各种参数，迭代过程中不断生成数据 batch；每次迭代过程 (Single Process) 是：由 `batch_sampler` 迭代取出下一批数据的 index -> `fetch` 函数将数据 batch 放到一个 list 中 -> 将这个 list 送给 `collate_fn` (可以自己定义) 得到整合之后的 data batch，一般第一维是 batch_size。
