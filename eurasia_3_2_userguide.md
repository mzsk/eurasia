

| | | |
|:|:|:|
|  |  |
|  |

# 安装 #

[下载](http://eurasia.googlecode.com/svn/branches/3.2/eurasia.py) [eurasia.py](http://eurasia.googlecode.com/svn/branches/3.2/eurasia.py) 即可，无需安装。

```
$ svn co https://eurasia.googlecode.com/svn/branches/3.2/
```

  * 需要 unix/linux 系统
  * 需要 libev、libeio 库和 greenlet 模块的支持
  * 在 python2.7 下测试通过（不支持 python3、pypy 不稳定）

## 安装依赖 ##

安装 libev 库（将 libev.so 与 eurasia.py 放在同一目录即可）：

```
$ wget https://eurasia-dl.googlecode.com/files/libev-1.125.tar.bz2
$ tar xjf libev-1.125.tar.bz2
$ cd libev-1.125
$ chmod +x autogen.sh
$ ./autogen.sh
$ ./configure
$ make
$ cp .libs/libev.so /PATH/TO/EURASIA_PY
```

安装 libeio 库（将 libeio.so 与 eurasia.py 放在同一目录即可）：

```
$ wget https://eurasia-dl.googlecode.com/files/libeio-1.424.tar.bz2
$ tar xjf libeio-1.424.tar.bz2
$ cd libeio-1.424
$ chmod +x autogen.sh
$ ./autogen.sh
$ ./configure
$ make
$ cp .libs/libeio.so /PATH/TO/EURASIA_PY
```

安装 greenlet 模块（将 greenlet.so 与 eurasia.py 放在同一目录即可）：

```
$ wget http://pypi.python.org/packages/source/g/greenlet/greenlet-0.4.0.zip
$ unzip greenlet-0.4.0.zip
$ cd greenlet-0.4.0
$ /PATH/TO/PYTHON setup.py build_ext --inplace
$ cp greenlet.so /PATH/TO/EURASIA_PY
```

| | | |
|:|:|:|
|  |  |
|  |

# 快速开始 #

```
from eurasia import httpserver
def handler(http):
    http.start_response('200 OK', [('Content-Type', 'text/html')])
    http.write('<html>Hello world</html>')
    http.close()

server = httpserver('0.0.0.0:8080', handler)
server.serve_forever()
```

执行脚本，使用浏览器访问 http://127.0.0.1:8080/ 即可。

## HTTP 服务器 ##

| eurasia.httpserver(_**addr**_, _**handler**_) |
|:----------------------------------------------|

httpserver 允许多种形式的 addr 参数，例如字符串格式的 addr ：

```
server = httpserver('127.0.0.1:8080', handler)  # IPv4 8080 端口
```

"(family, addr)" 格式的 addr ：

```
server = httpserver((socket.AF_UNIX, '/run/demo.sock'), handler)  # UNIX SOCK 地址
server = httpserver((socket.AF_AF_INET6, ('::', 8080)), handler)  # IPv6 8080 端口
```

| | | |
|:|:|:|
|  |  |
|  |

# HTTP 对象 #

| http.environ`[`_**headername**_`]` | 与 cgi/1.1 协议适配的环境变量 |
|:-----------------------------------|:----------------------------------------|
| http.start\_response (_**status**_, _**headers**_, _**timeout=-1**_) | 与 wsgi 协议适配的响应发起函数 |
| http.read(_**size**_, _**timeout**=-1_) | 读取 http 请求内容 |
| http.readline(_**size**_, _**timeout**=-1_) | 读取 http 请求内容 |
| http.write(_**data**_, _**timeout**=-1_) | 发送 http 响应内容 |
| http.flush() | 将写缓冲区中的数据立即发送 |
| http.close(_**keep\_alive**_=300, _**timeout**=-1_) | 完成 http 响应 |

  * http.write() 必须在 http.start\_response() 后使用
  * 必须使用 http.close() 结束当前这一轮 http 请求

## http.environ ##

可以通过 http.environ`[`_**envname**_`]` 获取的环境变量示范如下：

| 环境变量： | 变量描述： | example： |
|:----------------|:----------------|:-----------|
| REQUEST\_METHOD | 请求的方法 | "GET"、"POST" |
| SERVER\_PROTOCOL | 请求协议及版本 | "HTTP/1.1" |
| REMOTE\_ADDR | 连入客户端的地址 | "192.168.0.2" |
| REMOTE\_PORT | 连入客户端的端口 | 5566 |
| REQUEST\_URI | 完整 uri | "/login?username=tom&passwd=**"**|
| PATH\_INFO | 页面地址 | "/login" |
| QUERY\_STRING | 请求参数 | "username=tom&passwd=**"**|
| CONTENT\_TYPE | POST 等报文类型 | "application/x-www-form-urlencoded" |
| CONTENT\_LENGTH | POST 等报文长度 | 999 |
| HTTP`_`_**HEADERNAME**_ | 普通头部 | HTTP\_REFER："http://www.google.com/" |

## http.close() 与长连接 ##

| http.close(_**keep\_alive**_=300, _**timeout**=-1_) |
|:----------------------------------------------------|

  * 如果浏览器支持 keep-alive 特性，服务器将以 keep-alive 方式处理所有求
  * 默认情况下 http.close() 可用于完成当次请求，并维持当前的（socket）连接
  * 使用 http.close(0) 强制断开（socket）连接（无论是否完成）
  * http.close() 的 `keep_alive` 参数用于指定等待下次 http 请求所需的时间
  * http.close() 的 `timeout` 参数用于设定结束当次 http 请求所需的超时时间

| | | |
|:|:|:|
|  |  |
|  |

# sleep() & idle() #

eurasia 提供 sleep() 和 idle() 这两种简单的调度工具。

| eurasia.sleep(_**seconds**_) |
|:-----------------------------|

  * 在 eurasia 服务器中请使用 eurasia.sleep() 代替 time.sleep()

| eurasia.idle() |
|:---------------|

  * 临时切出当前栈（“greenlet.getcurrent()”）系统空闲时返回

| | | |
|:|:|:|
|  |  |
|  |

# 输入输出 #

eurasia 是基于协程的单进程、单线程服务器，涉及 “文件”、“数据库”、“管道”、“socket” 的阻塞操作会卡住整个服务器。

所有输入输出操作必须使用 eurasia 所提供的 “线程池”、“管道”、“socket” 接口之一来完成。

  * 本节所涉 `apply_`、popen3、socket 等接口必须在服务器 handler 中使用
  * 请勿在 eurasia 服务器中使用非 eurasia 提供的 io 接口

## 线程池 ##

| eurasia.`apply_`(_**func**_, _**args**=()_, _**kwargs**={}_) |
|:-------------------------------------------------------------|

线程池以 `apply_`() 函数的形式提供，尤适用于阻塞的文件及数据库操作。

```
from eurasia import apply_

def handler(http):
    ...

    file = apply_(open, ('filename', 'r'))
    data = apply_(file.read, (200, ))

    ...
```

  * 需要注意的是，通过 `apply_` 提交的任务没有超时，也不能撤销
  * `apply_` 返回所提交任务的返回值，或抛出其异常
  * 通过 `apply_` 提交的任务，必须是能够阻塞线程的（eurasia 提供的接口都不能在 apply`_` 中使用）

## 管道 ##

| eurasia.popen3(`*`_**args**_) |
|:------------------------------|

可以使用 popen3() 函数来操作协程管道，eurasia.popen3() 在操作上与 os.popen3() 类似。

```
from eurasia import popen3

def handler(http):
    ...

    stdin, stdout, stderr = popen3('ls', '-al')
    data = stdin.readline(1024, 10.) # 超时设为 10 秒
    stdout.write(data, 10.)

    ...
```

## SOCKET ##

| eurasia.socket(_**family**=`AF_INET`_) |
|:---------------------------------------|

eurasia 提供了 socket 对象的协程版本，可用于创建 tcp 连接。

```
from eurasia import socket
from socket import AF_INET

def handler(...):
    ...

    s = socket(AF_INET)
    s.connect(('127.0.0.1', 8080), 10.) # 10 秒超时
    s.write('GET / HTTP/1.0\r\n\r\n')
    print s.readline(1024)

    ...
```

  * socket 对象使用详见 “tcp 服务器” 一节

| | | |
|:|:|:|
|  |  |
|  |

# TCP 服务器 #

| eurasia.tcpserver(_**addr**_, _**handler**_) |
|:---------------------------------------------|

tcpserver 允许多种形式的 addr 参数（参见 httpserver）。

| def handler(_**sock**_, _**addr**_): return None |
|:-------------------------------------------------|

tcp handler 接受一个 socket 对象和一个客户端 addr 作为参数。

```
# echo 服务
from eurasia import tcpserver

def handler(sock, addr):
    while 1:
        data = sock.readline(1024, 60.)
        if 'quit' == data.strip():
            sock.close()
            break
        sock.write(data)

server = tcpserver('0.0.0.0:8080', handler)
server.serve_forever()
```

## SOCKET 对象 ##

| socket.connect(_**addr**_, _**timeout**_) | 连接到 addr |
|:------------------------------------------|:---------------|
| socket.recv(_**size**_, _**timeout**_) | 读取（不超过 size 大小的）数据 |
| socket.send(_**data**_, _**timeout**_) | 发送数据，返回发送的字节数 |
| socket.read(_**size**_, _**timeout**=-1_) | 读取（连接断开前）指定 size 大小的数据 |
| socket.readline(_**size**_, _**timeout**=-1_) | 读取（连接断开前）指定 size 大小的数据，直至 `\n` |
| socket.write(_**data**_, _**timeout**=-1_) | 向连接完整发送 data |
| socket.close() | 断开连接 |

  * recv() 不会操作读缓存，不能和 read() / readline() 混用
  * send() 不会操作写缓存，不能和 write() 混用
  * 同一时间只能执行一个读函数（“recv()”、“read()” 或 “readline()”），对同一 socket 对象不能并发读
  * 同一时间只能执行一个写函数（“send()” 或 “write()”）}，对同一 socket 对象不能并发写
  * 在读取过程中连接发生断开 read() / readline() 可能会返回小于 size 的数据，但不会抛出异常（如同收到 EOF）
  * 在发送过程中连接发生断开 write() 会抛出异常 “socket.error(EPIPE, 'Broken pipe')”

## socket.sendfile() ##

| socket.sendfile(_**`in_fd`**_, _**offset**_, _**count**_, _**timeout**=-1_) |
|:----------------------------------------------------------------------------|

socket.sendfile() 能够更加高效地发送静态文件。

```
def handler(sock, addr):
    size = apply_(os.stat, (filename, )).st_size
    in_fd = os.open(filename, os.O_RDONLY | os.O_NONBLOCK)
    sock.sendfile(in_fd, 0, size, 600.)
    sock.close()
```

  * 使用 socket.sendfile() 之前请先阅读 linux/unix sendfile 接口的文档及注意事项
  * in\_fd 需是以非阻方式打开的文件句柄（或 “fcntl(in\_fd, F\_SETFL, O\_NONBLOCK)”）

| | | |
|:|:|:|
|  |  |
|  |

# 直接发送 HTTP 报文 #

| http.socket 对象 |
|:-------------------|

有时我们需要直接通过底层 tcp 连接来发送 http 报文，这时就需要用到 http 对象的 socket 属性了。

| http.socket.send(_**data**_, _**timeout**_=-1) |
|:-----------------------------------------------|
| http.socket.write(_**data**_, _**timeout**=-1_) |

  * 需要注意的是读报文仍然需要使用 http 对象的读取接口

```
# 直接发送 HTTP 报文
from eurasia import httpserver

def handler(http):
    http.socket.write('HTTP/1.1 304 Not Modified\r\n\r\n')
    http.close()

server = httpserver('0.0.0.0:8080', handler)
server.serve_forever()
```

  * 需要注意的是完成 http 请求仍然需要使用 http.close()

一旦使用 http.socket 发送报文，就不应再使用 http.start\_response()、http.write() 。

合理使用 http.socket.write() 能有效提高服务器性能。

### 一览表 ###

| 模式：| 读取报文头：| 读取报文：| 发送报文头：| 发送报文：| 完成响应：|
|:---------|:------------------|:---------------|:------------------|:---------------|:---------------|
| 自动模式 | http.environ | http.read()等 | http.start\_response() | http.write() | http.close() |
| 手动模式 | http.environ | http.read()等 | sock.write() | sock.write() | http.close() |

| 模式：| start\_response() | http.read() | http.write() | http.flush() | http.close() | sock.read() | sock.write() |
|:---------|:------------------|:------------|:-------------|:-------------|:-------------|:------------|:-------------|
| 自动模式 | ○ | ○ | ○ | ○ | ○ | × | × |
| 手动模式 | × | ○ | × | ○ | ○ | × | ○ |

  * sock 指 http.socket 对象
  * http.read() 包括 http.read() `/` readline() 等接口
  * sock.read() 包括 http.socket.read() `/` readline() `/` recv() 等接口
  * sock.write() 包括 http.socket.write() `/` send() 等接口

| | | |
|:|:|:|
|  |  |
|  |

# 超时 #

eurasia 中的多数读写接口可以通过 timeout 参数设置超时，默认 -1 意为无超时。

超时将抛出 eurasia.Timeout 异常。

```
from eurasia import httpserver, Timeout
def handler(http):
    http.start_response('200 OK', [])
    try:
        http.write(data, 10.) # 10 秒超时
    except Timeout as e:
        pass

server = httpserver('0.0.0.0:8080', handler)
server.serve_forever()
```

  * 对 http 对象而言，通常情况下一旦超时发生，应停止处理该 http 请求并丢弃该 http 对象

## 从超时恢复 ##

```
from eurasia import tcpserver, Timeout

def handler(sock):
    while 1:
        try:
            data = sock.recv(1024, 10.)
        except Timeout:
            continue # tcp 的 recv() 超时可以直接重试
        break
    while 1:
        try:
            data = sock.read(10, 10.)
        except Timeout:
            continue # tcp 的 read() 超时可以直接重试
        break
    while 1:
        try:
            num_sent = sock.send('hello', 10.)
        except Timeout:
            continue # tcp 的 send() 超时可以直接重试
        break

    # 当 tcp write() 发生超时时，可以通过 Timeout 异常的
    # num_sent 属性得到已经发送的字节数
    try:
        sock.write('hello', 10.)
    except Timeout as e:
        print 'sent:', e.num_sent
```

  * 对 tcp socket 而言 connect()、recv()、send()、read()、readline() 等方法的超时是可以直接重试的
  * tcp socket 的 write() 发生超时时需要读取 Timeout.num\_sent 以获得已发送字符数

| | | |
|:|:|:|
|  |  |
|  |

# 协程调度 #

  * 阅读本节需要有关于 greenlet 模块的完整知识
  * 本节所涉的底层调度接口并不会经常用到，如不能理解不会影响 eurasia 的使用

## 基本原理 ##

eurasia 使用 greenlet 模块提供的栈切换功能，并在此基础上实现了极其简单的协程调度。

eurasia 中并不存在复杂的协程调度器，使用 eurasia 的底层调度接口，切出当前栈时一般只要切换到上一级的父栈即可。

这是 eurasia 协程栈调度的重要约定（违反该约定，易造成栈泄漏）。

```
# 在 eurasia 中切出当前栈的编程约定

def handler(...):
    back_ = getcurrent()  # 总是回到当前栈
    goto_ = back_.parent  # 总是切换到父栈
    goto_.switch()
```

  * 仅演示用，具体见下面范例

## eurasia.idle\_switch() ##

| eurasia.idle\_switch(_**back`_`**_, _**goto`_`**_, _**args**=()_) |
|:------------------------------------------------------------------|

idle\_switch() 用来临时切出当前栈（即 “greenlet.getcurrent()”）到 goto`_`
（这一步可以直译为 ` goto_.switch(*args) `）。

当系统空闲时，再切回 back`_`。

```
# 使用 idle_switch() 实现 eurasia.idle() —— 当然 eurasia.idle() 确实是这样实现的

from eurasia import idle_switch
from greenlet import getcurrent

def idle():
    back_ = getcurrent()  # 返回到当前栈
    goto_ = back_.parent  # 通常都是切换到上一级的父栈
    idle_switch(back_, goto_)  # goto_.switch() 空闲时切回 back_
```

## eurasia.timer\_switch() ##

| eurasia.timer\_switch(_**back`_`**_, _**goto`_`**_, _**seconds**_, _**args**=()_) |
|:----------------------------------------------------------------------------------|

切出当前栈到 goto`_`，在指定时间（seconds）后再切回 back`_`。

```
# 使用 timer_switch() 实现 eurasia.sleep() —— 当然 eurasia.sleep() 确实是这样实现的

from eurasia import idle_switch
from greenlet import getcurrent

def sleep(seconds):
    back_ = getcurrent()  # 返回到当前栈
    goto_ = back_.parent  # 通常都是切换到上一级的父栈
    timer_switch(back_, goto_, seconds) # seconds 后切回 back_
```

## eurasia.timer\_raise() ##

| eurasia.timer\_raise(_**back`_`**_, _**goto`_`**_, _**seconds**_, _**args**=()_) |
|:---------------------------------------------------------------------------------|

timer\_raise() 与 timer\_switch() 的区别在于 timer\_raise() 是以抛出 eurasia.Timeout 异常结束的。

```
from greenlet import getcurrent
from eurasia import idle_switch, Timeout

def sleep(seconds):
    back_ = getcurrent()  # 返回到当前栈
    goto_ = back_.parent  # 通常都是切换到上一级的父栈
    try:
        timer_raise(back_, goto_, seconds) # seconds 后切回 back_
    except Timeout:
        return
```

| | | |
|:|:|:|
|  |  |
|  |


# 部署 #

## 多服务器 ##

| server.`serve_forever`() | 启动服务器，并进入主循环（会阻塞程序） |
|:-------------------------|:----------------------------------------------------------|
| server.start() | 启动服务器（不会进入主循环，不会阻塞程序） |
| server.stop() | 暂停/停止服务器（不会退出主循环） |

| eurasia.run() | 进入主循环（会阻塞程序） |
|:--------------|:-------------------------------------|
| eurasia.`break_()` | 退出主循环 |

需要在一个线程中启动多个服务器的时候，使用 server.start()、eurasia.run() 来代替 server.serve\_forever() 。

```
from eurasia import httpserver, run, break_

    ...

server1 = eurasia.httpserver('0.0.0.0:8080', handler1)
server1.start()

server2 = eurasia.httpserver('0.0.0.0:8090', handler2)
server2.start()

run() # 进入主循环
```

  * httpserver 与 tcpserver 都支持 stop()、start()、serve\_forever()
  * server.stop() 和 server.start() 是非阻塞的，server.`serve_forever`() 和 eurasia.run() 将使程序进入主循环
  * 可以通过 server.stop() 来暂停服务器，暂停的服务器可以通过 server.start() 再次启动
  * 可以调用 eurasia.`break_()` 退出主循环（server.{{serve\_forever}}}() / eurasia.run() 会退出）

## 部署工具 ##

| eurasia.cpu\_count() | 得到 cpu 及 cpu 核心的总数 |
|:---------------------|:-----------------------------------|
| eurasia.setuid(_**user**_) | 修改程序的执行用户 |
| eurasia.setprocname(_**procname**_) | 修改进程名 |

  * 注意必须把 setprocname() 放在整个程序的头部执行（因为 setprocname() 会重启整个程序）。

## 多核支持 ##

推荐在多个端口上启动多个独立的 eurasia 服务器进程，数量与 cpu 核心相应，这样就可以充分使用 cpu 核心了。

可以通过 iptables（redirect）或者 lvs 对 eurasia 服务器做端口负载均衡。

  * 请勿对 eurasia 服务器做反向代理（比如在 eurasia 上再架一层 nginx）这会限制 eurasia 的并发能力。

## 后台执行 ##

推荐使用 [daemon](http://www.libslack.org/daemon/) 命令启动服务器，使 eurasia 服务器运行于后台。

在多数 unix/linux 发行版上都可以非常方便地安装 daemon 命令。

| | | |
|:|:|:|
|  |  |
|  |

# 开发者 #

## 协程服务器 ##

eurasia 是一种基于协程的服务器框架，协程服务器能让开发者以（较为人性的）同步逻辑来开发单线程的异步程序。

以下是几种典型的服务器框架，便于开发者了解协程服务器的特点：

| 服务器框架：| 协程：| 异步：| 多线程（核心）：| 多线程（应用层）：| 开发模式：| 理论性能：| 并发：|
|:------------------|:---------|:---------|:------------------------|:---------------------------|:---------------|:---------------|:---------|
| paste | × | × | ○ | ○ | 同步 | 中 | 低 |
| twisted | × | ○ | × | × | 异步 | 高 | 高 |
| nodejs | × | ○ | × | × | 异步 | 高 | 高 |
| ①tornado | × | ○ | × | ○ | 同步 | 高 | 中 |
| ②tornado | × | ○ | × | × | 异步 | 高 | 高 |
| fapws3 | × | ○ | × | ○ | 同步 | 高 | 中 |
| gevent | ○ | ○ | × | × | 同步 | 高 | 高 |

  * “同步逻辑” 和 “异步逻辑” 在开发效率和维护成本上区别较大
  * eurasia 和 gevent 是好基友！

## 性能参考 ##

框架性能主要和框架的是否完全异步，及具体实现有关，与是否是协程服务器无关。
诸如 eurasia、gevent 这样的协程服务器与纯异步实现的 twisted、nodejs
他们在性能上（理论上）并没有太大的差别，在并发能力上差别则更小。

最近的一项测试（官网 “Hello, world” 程序）表明，在同等条件下 eurasia 3.2 每秒能够处理的请求数相当于
tornado 4.2 的 140% ，以 tornado 为基准可以得到 eurasia 在各项测试中的参考值。

  * 性能评测仅作参考，应以实际使用情况为准

## 源码阅读及调试向导 ##

在源码文件 “eurasia.py” 中有大量实现是由代码模板生成的，代码模板通常以多行字符串的形式出现。

在源码阅读中，如果发现整段 base64 编码的字符串，须以 base64 将其解码以后用 zlib 解压，即可得到原始的代码模板。

```
# 如何从 base64 得到原始的代码模板
import zlib, base64
print zlib.decompress(base64.decodestring('''\
eNrtWG1v2zYQ/u5fwaEILCGOEPfTaswB1sTDigV20TjrB88QZJluuKikQdHWsl8/HknJpN7sGM4y
    ...
6Ku67qUGv4Y1gzNpMz/zlGfZ4kr6jwc8OwDwrBXw7DUAz14P8H8ApS6xVQ=='''))
```

  * 将得到的代码模板创建为新文件有助于调试中获得正确的代码行号（因为代码模板的执行位置是 `“<string>”` 临时模块）。

## 提交 BUG 及建议 ##

可以通过 [eurasia 用户组](https://groups.google.com/group/eurasia-users) 联系我们。