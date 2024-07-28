---
title: 'ShadowSocks Guidance'
date: 2019-12-03
permalink: /posts/2019/12/shadowsocks-guidance/
tags:
  - tools
---

> ShadowSocks 使用总结

## 前言

### ShadowSocks 基本认识

本文介绍用`ShadowSocks`这个工具进行科学上网的配置方法，不知道啥是科学上网或者为啥要科学上网的童鞋可以不用再看下去了，相信你也没这个需求。。。

至于为啥选择`ShadowSocks`呢？笔者表示当年用了很长时间的`Lantern`突然被封，其它VPN找了半天，免费的速度太慢，收费的又比较贵，对比之下`ShadowSocks`以稳定著名，价格也能接受，还可以自己租服务器搭服务，于是果断ss。。。

既然用的工具是`ShadowSocks`，那么对它还是要有一些基本的认识：[ShadowSocks](https://shadowsocks.org/en/index.html) 本身是一套基于 Socks5 代理方式加密的在网络上传输数据的技术，常被简称为`ss`。既然要实现网络数据传输，服务端和客户端总是少不了的。简单来说，通过`ss`科学上网，就是将传输的数据通过服务端和客户端的加密，通过 GFW 的检测。

比如你要访问 [Google](https://www.google.com)，那么发出请求的过程中数据是这样传输的：

本地应用 -> ss 客户端 -> ss 服务端 -> Google主机

其中ss客户端安装在你的电脑上，而ss服务端安装在境外(或者其他能够访问 Google 的)服务器上，数据在它们之间的传输是用 Socks5 加密的。

注意，`ss`本身只是一项技术，不要把它和提供`ss`服务器的公司、以及科学上网这个概念搞混了。

### v2ray

**[2019.11 更新]** 这一段时间 `Shadowsocks` 通道频繁被封，很不稳定，一种新的叫做 `v2ray` 技术逐渐成为更流行、更坚挺的科学上网手段，使用逻辑和 `Shadowsocks` 非常像，也是客户端 -- 服务器模式。用 `ss` 搭梯子的同学们可以转战 `v2ray` 了。

----

## 服务端

ss服务器可以通过这些方式获得：

* 购买第三方服务商(如 [shadowsocks.to](https://order.shadowsocks.se/))的ss服务

* 自己租VPS，在服务器上搭建`ss`服务

* 在网上搜免费的`ss`服务器，白嫖

下面就分别介绍一下如何操作。

### 购买第三方 ss 服务

**优点**：省心，不需要自己配置服务端的密码、加密方式等信息，如果哪天被和谐了也可以很快更换服务器

**缺点**：通常有连接数和使用流量限制，不过最初级的服务器和流量也够大多数个人用户使用了

这里推荐笔者自己使用的 [shadowsocks.to](https://order.shadowsocks.se/) 这个提供商提供的`ss`服务，效果不错。注意，提供商的网址修改了多次，因此前面放上来的链接可能已经失效了。

对于个人而言，选择“入门版”已经够用了，具体如何购买这里就不阐述了，网站上很容易找到入口。

付完款后，还需要将配置文件下载下来，里面包含了各个服务器节点的域名、端口、密码、加密方式等信息，等下配置客户端的时候要用到。

![配置文件下载位置](shadowsocks_to_config_download.png)

另外，各个服务器节点旁边的二维码在配置移动端(Android、iOS)的`ss`客户端时也要用到。

### 自己搭建ss服务

**优点**：配置自由；流量不受限制；可以给多人使用

**缺点**：有一定技术门槛；如果 IP 被封，需要换服务器

[ShadowSocks](https://shadowsocks.org/en/index.html) 官网上有关于如何[搭建服务器](https://shadowsocks.org/en/download/servers.html)和[配置信息](https://shadowsocks.org/en/config/quick-guide.html)的说明，按照上面的教程就可以搭好服务器，下面简单地介绍一下基础的步骤：

#### 准备服务器

一台能够访问外网的 VPS 主机是必要的，可以从典型的服务商如 [Vultr](https://www.vultr.com/) 处购买，然后通过`ssh`登录主机，安装`ss`服务。

#### 用pip安装ss服务

一般的主机上都默认安装有Python2和`pip`，连接 VPS 主机，打开终端检查一下：

```
root@xxn:~# python --version
Python 2.7.12
```

```
root@xxn:~# pip --version
pip 19.0.3 from /usr/local/lib/python2.7/dist-packages/pip (python 2.7)
```

接着就可以用`pip`去安装`ShadowSocks`了：

```bash
pip install shadowsocks
```

#### 编辑配置文件

编辑一个配置文件，里面写好配置信息，用于启动`ss`服务。文件的位置可以随便放，只要你知道在哪里，这里以`/etc/shadowsocks.json`为例，新建`/etc/shadowsocks.json`文件，写入内容：

```json
{
  "server_port": 45630,
  "password": "Tgc6zqgeIA",
  "method": "aes-256-cfb",
  "timeout": 300,
  "mode": "tcp_and_udp"
}
```

对应参数的意义：

- `server_port`: `ss`服务的端口，可以随意取值，一般在 1024~65536，不和常用其他端口冲突即可
- `password`: 等下客户端连接时使用的密码，可以随意设置，稍复杂一些为好
- `method`: 加密方式，可选的方式有不少，官网推荐使用`"chacha20-ietf-poly1305"`或者`"aes-256-gcm"`
- `timeout`: 超时时间(秒)
- `mode`: `ss`服务模式，`"tcp_and_udp"`表示既支持 tcp 也支持 udp

上面的参数是我服务器上的设置，各个参数都可以改，只要等下客户端参数保持一致即可。

更加高级的配置信息可见[官网](https://shadowsocks.org/en/config/advanced.html)。

#### 启动ss

打开终端，启动`ss`进程：

```bash
ssserver -c /etc/shadowsocks.json -d start
```

其中，-c后面接配置文件的路径，-d表示`daemon`模式，即守护进程模式运行，启动后不占据终端。

停止ss：

```bash
ssserver -c /etc/shadowsocks.json -d stop
```

如果懂一些 Linux 服务技术，可以将`ss`设置成服务，用`systemctl`设置开机自动启动等等，不赘述了，和其他服务并无二异。

### 找免费 ss 服务

**优点**：顾名思义，省钱

**缺点**：速度不稳定，公开的服务器地址，容易被封

这就没什么好讲的了，自己到网上找公开免费的`ss`服务器，下载地址、密码等信息。不推荐这种做法，因为确实不稳定。

----

## 客户端

### 通用逻辑

不论是哪个平台上的`ss`客户端，配置都是一样的思路：将`ss`服务器的配置信息(至少包括服务器地址、端口、加密方式、密码)添加到客户端即可。简单来说，保证客户端和服务端的配置信息一致，就可以连接。

如果是自己在 VPS 上搭的服务，那么配置信息都是自己输入的，相信不会有什么问题。如果是在提供商处购买，可以直接下载配置文件，或者扫描二维码，即可添加配置信息，不用手动一个个输入。

另外，客户端一般都可以设置`“全局模式”`和`“pac模式”`，顾名思义，全局模式就是所有流量均走代理，pac 模式会根据 GWF List 判断哪些 ip 需要走代理，只在访问这些网站的时候走代理，从而加速国内网站的访问速度。一般不推荐全局模式，因为所有流量都走代理，容易被封。

有些应用可能不那么“智能”，比如 Android Studio 在该走代理连接 Google 更新的时候不走代理，那么可以找找应用里有没有 http/socks5 代理的设置，host设置成 127.0.0.1，端口设置成客户端暴露出来的 http/socks5 端口。如果应用本身不支持 http/socks5 代理，再开全局模式强行让它走代理。

### Windows

在[GitHub官网](https://github.com/shadowsocks/shadowsocks-windows/releases)上下载最新的Windows客户端，解压zip文件得到exe可执行文件，将它和配置文件放在一起，双击打开即可使用。

### Linux

#### shadowsocks-qt5

以 Ubuntu 为例，在[GitHub官网](https://github.com/shadowsocks/shadowsocks-qt5/releases)上下载最新客户端，安装之后运行，找到导入服务器配置文件的选项，导入下载好的配置文件即可。

#### 终端开启

直接从终端也可以开启ss，和前面在 VPS 上安装`ss`服务一样，首先在本地安装`ss`：

```bash
pip install shadowsocks
```

然后编辑配置文件，注意和服务器配置信息保持一致：

```json
{
    "server": "你的ss服务器地址(ip或者域名)",
    "server_port": 45630,
    "password": "Tgc6zqgeIA",
    "method": "chacha20-ietf-poly1305",
    "remarks": "aes-256-cfb",
    "timeout": 300,
    "local_port": 1080
}
```

最后一项`local_port`是在本机上暴露的 socks5 端口，供其他应用走代理用。

启动`ss`进程(以配置文件放在`/etc/shadowsocks.json`为例)：

```bash
sslocal -c /etc/shadowsocks.json -d start
```

#### 终端走代理

上面的设置已经够浏览器科学上网了，然而有时在终端上用 pip/conda/docker 进行 pull/push 时，如果不将源换成国内源，那么速度简直如同便秘一般让人难受，搞不好快要下完时突然连接中断，简直和吃屎一样难受，于是终端走代理的需求应运而生。

终端走代理的设置很简单，只要有暴露出来的 http 端口，就可以连接了，暴露的 socks5 端口也可以用其他工具映射到 http 端口上。

以上面从终端开启`ss`为例，配置显示暴露了 1080 作为 socks5 端口，于是用`privoxy`这个工具映射到 http 端口上：

安装`privoxy`:

```bash
sudo apt-get update
sudo apt-get install privoxy
```

修改配置文件，打开配置文件`/etc/privoxy/config`，取消这一行的注释：

```bash
#listen-address localhost:8118
```

在底部加上：

```bash
forward-socks5 / localhost:1080 .
```

不要忘了最后的点号，8118 是 http 代理的窗口，1080 是 socks5 端口。

然后重启`privoxy`：

```bash
sudo service privoxy restart
```

以后开机之后要启动`privoxy`，将上面的 restart 改成 start 即可。

最后配置两个 bash 变量`http_proxy`和`https_proxy`就能实现终端走代理：

```bash
export http_proxy="http://localhost:8118"
export https_proxy="http://localhost:8118"
```

笔者则是直接在`~/.bashrc`中加入了两个函数控制是否走代理：

```bash
function proxy_on() {
    export http_proxy="localhost:8118"
    export https_proxy="localhost:8118"
    echo -e "Proxy started"
}

function proxy_off(){
    unset http_proxy
    unset https_proxy
    echo -e "Proxy stopped"
}
```

另外，除了`privoxy`，`proxychains`也是一个不错的选择，不过笔者并没有用过。

### Mac

Mac 上的客户端和 Windows、Linux 并没有什么明显的区别，同样在[GitHub官网](https://github.com/shadowsocks/ShadowsocksX-NG/releases)上下载最新客户端，导入配置文件即可。

当然，在 Mac 上也可以打开终端，从终端启动`ss`，和 Linux 一样，不再赘述。

#### 终端走代理

和 Linux 一样，Mac 上也有终端走代理的需求，也可以用类似的方法也实现，即通过`privoxy`这个工具将 socks5 端口映射到 http 上。

如果是`ShadowsocksX-NG`客户端打开的，那么它已经有暴露的 http 端口了，直接设置`http_proxy`和`https_proxy`这两个 bash 变量即可。

例如，笔者 mac 上的`ShadowsocksX-NG`的设置是这样的：

![mac客户端监听http端口](shadowsocsX-NG-http.png)

可见监听的是 http 的 1087 端口，于是

```bash
export http_proxy="http://localhost:1087"
export https_proxy="http://localhost:1087"
```

即可实现终端走代理

<!--安装`privoxy`：-->

<!--```bash-->
<!--brew install privoxy-->
<!--```-->

<!--修改配置文件，在配置文件`/usr/local/etc/privoxy/config`底部加入 socks5 转发：-->

<!--```bash-->
<!--listen-address 0.0.0.0:8118-->
<!--forward-socks5 / localhost:1080 .-->
<!--```-->

<!--和 Linux 一样，要打开客户端的偏好设置查看本机 socks5 监听地址，对应修改一下。-->

<!--启动`privoxy`：-->

<!--```bash-->
<!--sudo /usr/local/sbin/privoxy /usr/local/etc/privoxy/config-->
<!--```-->

<!--最后和 Linux 上一样，还需要在终端设置才能使用：-->

<!--```bash-->
<!--export http_proxy="http://localhost:8118"-->
<!--export https_proxy="http://localhost:8118"-->
<!--```-->

### Android

Android移动端的设置就比较简单了，同样在[GitHub官网](https://github.com/shadowsocks/shadowsocks-android/releases)上直接下载 apk，安装到手机，扫二维码导入服务器配置或者手动输入配置信息即可。

### iOS

iOS由于不开源，客户端没那么好找。笔者找到的还不错的客户端是：`Potatso Lite`，可惜的是在国区 App Store 上找不到。简单的办法是，到某宝上买个美区的 App Store 账号，下载即可。下载完之后扫描二维码导入或者手动输入配置信息，系列操作和 Android 一样。

---

## Reference 

1. https://crifan.github.io/scientific_network_summary/website/
2. https://gist.github.com/alexniver/9a4f1791fe4305b0750a
3. https://tech.jandou.com/to-accelerate-the-terminal.html

