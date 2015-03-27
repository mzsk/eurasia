

| | | |
|:|:|:|
|  |  |
|  |


# 安装 #

需要 unix/linux 系统，在 python2.6、python2.7 下经过测试。

  * 暂不支持 windows 系统和 python3.x
  * python2.5 未经充分测试

## 安装过程 ##

在此 [在此](http://eurasia.googlecode.com/files/eurasia-3.1-snapshot.tar.bz2) 下载集成安装包：

```
$ tar -xjf eurasia-3.1-snapshot.tar.bz2
$ cd eurasia-3.1-snapshot
$ sudo /PATH/TO/DEST/bin/python2.6 setup.py install
```

  * 使用哪个 python 执行 setup.py 就安装到哪个 python
  * 安装到系统 python 一般需要管理权限

也可以安装到指定目录：

```
$ /PATH/TO/PYTHON/bin/python2.6 setup.py install /home/foo/MyProject
```

  * 将会安装到 `/home/foo/MyProject/eurasia`
  * 你指定的 python 解释器将被用于编译过程，兼容此 python 版本
  * 需要 `/home/foo/MyProject` 目录权限，方便用户权限部署
  * 方便随项目部署


| | | |
|:|:|:|
|  |  |
|  |


# 快速开始 #

我们将从最简单的 hello world 开始，通过范例快速掌握 eurasia 。

这首先是一个 web 程序。


## hello world ##

```
# 文件名: test.py
from eurasia.web import httpserver, mainloop
def handler(httpfile):
    httpfile['Content-Type'] = 'text/html'
    httpfile.write('<html>hello world!</html>')
    httpfile.close()

httpd = httpserver(('', 8080), handler)
httpd.start()
mainloop()
```

把 handler 绑定到 8080 端口，浏览器访问该端口会得到 hello world 。

执行脚本，启动服务器。

```
$/usr/bin/python2 test.py
```

  * 一次可以通过 httpserver 创建多个 http 服务器
  * 服务器通过 start() 启动 / 以 stop() 暂停
  * 最后执行 mainloop() 主循环


## httpfile 对象 ##

操作 httpfile 对象，获取请求，并完成响应。

request 操作:

| 属性 | httpfile.method | 通常是“GET”或“POST” |
|:-------|:----------------|:--------------------------------|
| 属性 | httpfile.path\_info | 请求的资源位置（路径） |
| 属性 | httpfile.query\_string | 路径“?”之后的部分 |
| 属性 | httpfile.cookie | 得到客户端 cookie（Cookie.SimpleCookie 对象）无则返回 None |
| 字典（取出） | httpfile`[`_**headername**_`]` | 取得指定头部 |

response 操作:

| 属性 | httpfile.status | 响应状态，默认 200 |
|:-------|:----------------|:--------------------------|
| 属性 | httpfile.cookie | 向客户端写入 Cookie（Cookie.SimpleCookie 对象） |
| 字典（写入） | httpfile`[`_**headername**_`]` | 设置头部 |
| 成员函数 | httpfile.write(_**data**_, _**timeout**=-1_) | 发送内容，默认无超时 |
| 成员函数 | httpfile.close(_**keep\_alive**_=-1, _**timeout**=-1_) | 完成本次响应，默认不设置 keep-alive 时间 |
| 成员函数 | httpfile.raw\_close(_**data**_, _**keep\_alive**_=-1, _**timeout**_=-1) | 发送数据同时完成本次请求，data 应包含完成头部 |

下面将是一个完整的例子（文件服务器）：

```
#-*- coding: utf-8 -*-
import os.path
from eurasia.web import httpserver, mainloop
def handler(httpfile):
    # 注意，直接组合路径不安全
    filename = '/var/www' + httpfile.path_info
    if not os.path.exists(filename):
        httpfile.status = 404 # 文件未找到
        httpfile.write('<h1>Not Found</h1>')
        return httpfile.close()
    if os.path.isdir(filename):
        httpfile.status = 403 # 不支持列出目录
        httpfile.write('<h1>Forbidden</h1>')
        return httpfile.close()
    httpfile['Content-Type'] = 'application/octet-stream'
    data = open(filename).read()
    httpfile.write(data)
    httpfile.close()

httpd = httpserver(('', 8080), handler)
httpd.start()
mainloop()
```

  * 虽然 httpfile 形似 dict，但是读取接口仅用于读取 request，写入接口仅用于写入 response
    1. 这意味着从 httpfile`[`...`]` 获取的内容和写入 httpfile`[`...`]` 的内容_**并不一致**_
    1. httpfile.cookie 亦然

  * 这里直接对路径进行相加_**是不安全的**_，用于演示
  * 这里对磁盘文件的读取操作是低效的，应使用框架带有的文件 IO 接口（见“文件”一节）
  * 可以通过 httpfile.sockfile 得到原始 socket（见“文件 -> 文件对象接口列表”一节）


| | | |
|:|:|:|
|  |  |
|  |


# 标准 web 服务器 #

框架的关键应用之一便是 web 。首先通过 httpserver(_**addr**_, _**handler**_) 创建标准 http 服务器。

## 地址格式 ##

接口 httpserver 允许多种形式的 addr 参数。

```
httpserver('127.0.0.1:8080', handler)  # IPv4 地址，端口 8080
httpserver('[::1]:8080', handler)  # IPv6 地址
httpserver(':8080', handler)  # 本机 8080 端口
httpserver(('127.0.0.1', 8080), handler)  # IPv4
httpserver(('::1', 8080), handler)  # IPv6

# 同时指定地址及 family：
from socket import AF_INET6, AF_UNIX
from eurasia.web import httpserver
httpserver((('::1', 8080), AF_INET6), handler)
httpserver(('/var/httpd.sock', AF_UNIX), handler)  # unix socket

# 绑定到 fileno 为 0 的资源描述符，默认 family 是 AF_INET：
httpserver(0, handler)

# 绑定到 fileno 为 0 的描述符，指定 family 为 AF_INET6：
from socket import AF_INET6
from eurasia.web import httpserver
httpserver((AF_INET6, 0), handler)
```

## 启动和暂停服务器 ##

通过 httpserver.start() 和 httpserver.stop() 启动和暂停服务器，暂停的服务器可以通过 httpserver.start() 重新启动。

```
from eurasia.web import httpserver, mianloop

# 工作服务器，绑定到 8080
def handler(httpfile):
    httpfile.write('hello world!')
    httpfile.close()

httpd = httpserver(':8080', handler)
httpd.start()

# 管理服务器，绑定到 8090
def manager(httpfile):
    # 如果请求地址是 /start 则启动工作服务器
    if httpfile.path_info == '/start':
        httpd.start()
   # 如果请求地址是 /pause 则停止工作服务器
    elif httpfile.path_info == '/pause':
        httpd.stop()

man = httpserver('8090', manager)
man.start()
mainloop() # 开始调度
```

注意区分 httpserver.start() 和 mainloop()：
  * mainloop() 是整个程序的调度器
  * httpserver.start() 是启动服务器（开始监听）
  * httpserver 可以有多个而 mainloop() 只有一个

## CGI 规范适配 ##

httpfile 对象是服务器关键接口，其设计在很大程度上与 CGI/1.1 规范适配，以下是一些对应关系：

| 接口类型： | 接口描述： | CGI 对应： |
|:----------------|:----------------|:--------------|
| 字典（读取） | httpfile.environ`[`_**envname**_`]` | 环境变量（environ） |
| 成员函数 | httpfile.read(_**size**=-1_, _**timeout**=-1_) | 标准输入（stdin） |
| 成员函数 | httpfile.readline(_**size**=-1_, _**timeout**=-1_) | 标准输入（stdin） |
| 成员函数 | httpfile.write(_**data**_, _**timeout**=-1_) | 标准输出（stdout） |

可以通过 httpfile.environ`[`_**envname**_`]` 获取的环境变量：

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
| HTTP`_`_**HEADERNAME**_ | 请求头部 | HTTP\_REFER："http://www.google.com/" |

也可以使用 httpfile.request\_uri、httpfile.path\_info、httpfile.query\_string 操作环境变量：

```
# 操作 httpfile.request_uri 会同时影响到 PATH_INFO 和 QUERY_STRING
httpfile.request_uri = '/login?username=tom&passwd=***'
print httpfile.path_info, httpfile.query_string  # 分别是 "/login" 和 "username=tom&passwd=***"
print httpfile.environ['PATH_INFO'], httpfile.environ['QUERY_STRING']  # 同上

# 操作 httpfile.path_info 会影响到 REQUEST_URI
httpfile.path_info = '/check'
print httpfile.request_uri  # uri 变成 "/check?username=tom&passwd=***"，query_string 不变
print httpfile.environ['REQUEST_URI']  # 同上

# 操作 httpfile.query_string 会影响到 REQUEST_URI
httpfile.query_string = 'username=jerry&passwd=***'
print httpfile.request_uri  # uri 变成 "/check?username=jerry&passwd=***"，path_info 不变
```

  * 修改 httpfile.request\_uri、httpfile.path\_info、httpfile.query\_string 时，这三个值会联动
  * 设置 httpfile.environ 这三个值_**不会**_发生联动，一般避免通过 httpfile.environ 直接操作这三个变量

## 解读请求 ##

通过 httpfile.read(_**size**=-1_, _**timeout**=-1_) 和 httpfile.readline(_**size**=-1_, _**timeout**=-1_) 读取报文体。

eurasia.cgietc 模块提供了解析 POST 形式表单的工具 form(_**httpfile**_, _**max\_size**=1048576_, _**timeout**=-1_)：
  * 需要指定一个 httpfile 对象
  * 使用 max\_size 限定允许用户提交表单的最大字节数，超过限制会抛出 ValueError 并立即杀死客户端，默认最多传输 1M 数据
  * 使用 timeout 指定读取表单的超时时间
  * 使用 dict 返回解析的表单
    * 正常情况下每个表单项的取值都是 str
    * 如果提交了多个同名的表单项，那么该表单项取值就是一个 list，在其中保存多个 str 值

```
# -*- conding: utf-8 -*-
from eurasia.cgietc import form
from eurasia.web import httpserver, mainloop

# 返回表单内容
def handler(httpfile):
    form1 = form(httpfile)
    # 假定提交的表单：a=hello&b=world&c=1&c=2&c=3
    # 则输出 {'a': 'hello', 'b': 'world', 'c': ['1', '2', '3']}
    httpfile.write(repr(form1))
    httpfile.close()
```

  * form() 会同时处理 QUERY\_STRING 和 POST 报文体
  * 注意，_**不能**_处理 multipart 报文，比如文件上传
  * 注意，httpfile.read() / httpfile.readline() _**不能**_与 form() 混用，会导内容混乱
  * form() 处理完请求以后将结果以新的 dict 对象返回，不增加 httpfile 的引用

## 完成响应 ##

我们提供了接口 httpfile.start\_response(_**status**_, _**response\_headers**_, _**timeout**=-1_) 用来简化 httpfile`[`...`]`和 httpfile.status 的操作：

```
def handler(httpfile):
    response_status = '200 OK'
    response_headers = [('Content-Type', 'text/html')]
    httpfile.start_response(response_status, response_headers)
    httpfile.sendall('hello world!')
    httpfile.close()
```

httpfile.start\_response() 发送指定的 status 和 headers 报文头，这和 wsgi 规范（[pep333](http://www.python.org/dev/peps/pep-0333/)）中 start\_response() 接口的定义比较接近。

通过 httpfile.start\_response() 完成报文头部以后，就可以通过 httpfile.sendall(_**data**_, _**timeout**=-1_) 发送报文体了。

httpfile.write() 与 httpfile.sendall() 的功能相当接近，都用于发送报文体，他们的区别是：
  * httpfile.write() 会判断头部如果没有发送，会首先发送 httpfile.status 和 httpfile`[`...`]` 的头部设置，再发送报文
  * httpfile.sendall() _**不会**_理会头部是否已发送，而直接发送报文，在没有调用 httpfile.start\_response() 的情况下这会导致严重问题

一般情况下，httpfile.status、httpfile`[`...`]` 与 httpfile.write() 配合使用；httpfile.start\_response() 与 httpfile.sendall() 是一对
  * 需要注意 httpfile.start\_response() 会覆盖 httpfile`[`...`]` 和 htttpfile.status 的设定
  * 严格说 httpfile.start\_response() 的头部内容会追加在 httpfile`[`...`]` 的设定之后

## cookie ##

通过 httpfile.cookie 属性配合标准库 [Cookie](http://docs.python.org/library/cookie.html) 读取和设置 cookie。

```
#!/opt/unladen-2.6.4/bin/python2.6
# -*- coding: utf-8 -*-
from uuid import uuid4
from Cookie import SimpleCookie
from time import gmtime, strftime, time
from eurasia.web import httpserver, mainloop

def handler(httpfile):
    cookie = httpfile.cookie
    if cookie is None:  # 客户端没有 cookie 则设置一个
        cookie = SimpleCookie()
        cookie['uid'] = uuid4().hex
        cookie['uid']['path'] = '/'
        cookie['uid']['expires'] = strftime(  # 设置过期时间是 3600 秒
            '%a, %d-%b-%Y %H:%M:%S GMT', gmtime(time() + 3600))
        httpfile.cookie = cookie # 设置到 httpfile.cookie
        # 通过 start_response()，cookie 随头部一起发送
        httpfile.start_response('200 OK', [('Content-Type', 'text/html; charset=utf-8')])
        httpfile.sendall('<p>创建 cookie 内容：<br><br>%s</p>'%cookie.output(header=''))
        httpfile.sendall('<p><b>该 cookie 已发送至浏览器</b></p>')
        httpfile.close()
    else:  # 客户端已有 cookie，删除该 cookie
        cookie_from_request = httpfile.cookie
        cookie_to_response  = SimpleCookie()
        cookie_to_response['uid'] = ''
        cookie_to_response['uid']['path'] = '/'
        cookie_to_response['uid']['expires'] = 'Mon, 26 Jul 1997 05:00:00 GMT'  # 使过期删除

        httpfile.cookie = cookie_to_response  # 在头部发送前设置好 cookie
        httpfile.start_response('200 OK', [('Content-Type', 'text/html')])
        httpfile.sendall('<p>收到 cookie 内容：<br><br>%s</p>'%cookie_from_request.output(header=''))
        httpfile.sendall('<p><b>该 cookie 已经删除</b></p>')
        httpfile.close()

httpserver(':8080', handler).start()
mainloop()
```

  * 读取 httpfile.cookie 仅用于读取 request 头部，设置 httpfile.cookie 仅用于写入 response 头部
    * 这意味着从 httpfile.cookie 获取的内容和写入 httpfile.cookie 的内容_**不能保证一致**_

  * httpfile.cookie 的设置通过 httpfile.start\_response() 或者 httpfile.write() 发送给浏览器
  * 注意，写入 httpfile.cookie 的内容与从 httpfile.cookie 读取的内容_**不能保证一致**_
  * httpfile.cookie 的所有功能都可以直接通过操作 headers`[`Cookie`]` 和 headers`[`Set-Cookie`]` 来完成
  * 出于安全性考虑，应仅使用 Cookie.SimpleCookie 对象来存取 cookie

## 长连接 ##

请求将被一直保持，直到你调用 `httpfile.close()` 在此期间你可以在任何时候向用户发送内容。

```
# -*- coding: utf-8 -*-
from time import strftime
from eurasia.socket2 import error
from eurasia.web import httpserver, mainloop
httpfiles = set()
def handler(current_httpfile):
    response_headers = [('Content-Type', 'text/html; charset=utf-8')]
    current_httpfile.start_response('200 OK', response_headers)
    current_httpfile.sendall(strftime('[%a, %d-%b-%Y %H:%M:%S GMT] 我加入了<br/>'))
    # 通知其他在线用户，有新人加入
    disconnected_httpfile = []
    for httpfile in httpfiles:
        try:
            # 告诉其他在线用户
            httpfile.sendall(strftime('[%a, %d-%b-%Y %H:%M:%S GMT] 又有新人加入<br/>'))
        except error:
            # 连接已断开
            disconnected_httpfile.append(httpfile)
    # 移除断开的 httpfile
    for httpfile in disconnected_httpfile:
        httpfiles.remove(httpfile)
    # 将当前客户端加入全局列表中
    httpfiles.add(current_httpfile)
httpserver(':8080', handler).start()
mainloop()
```

  * 请使用 firefox 浏览器观看此样例，因为缓存原因一些浏览器可能不能展现即时效果
  * handler 结束时，如果引用为零 httpfile 对象会自动销毁，并同时断开用户连接
  * httpfile.start\_response()、httpfile.sendall() 与 httpfile.write() 都将立即发送数据_**没有缓存**_

## javascript rpc ##

通过 eurasia.cgietc 提供的 browser(_**httpfile**_, _**domain**=None_, _**timeout**=-1_) 接口，服务器端可以在任何时候即时调用客户端的 javascript 函数。

在下面这个例子中，每当有新用户访问站点，所有在线用户都将收到一条 javascript 的提醒。

我们首先定义一个包含有 javascript 的 html 页面, 并与服务器建立一条长连接。 其中的 "my\_alert" 函数就是我们即将要用到的:

```
<html>
<head>
<script language="JavaScript">
function my_alert(stuff) { alert(stuff); };
</script>
</head>
<body>
<!-- 与 /remotecall 位置的服务器脚本建立长连接 -->
<iframe src="/remotecall" style="display: none;"></iframe>
</body>
</html>
```

下面是完整的处理脚本:

```
#-*- coding: utf-8 -*-
from eurasia.socket2 import error
from eurasia.cgietc import browser
from eurasia.web import httpserver, mainloop

page = '''\
<html>
<head>
<script language="JavaScript">
function my_alert(stuff) { alert(stuff); };
</script>
</head>
<body>
<!-- 与 /remotecall 位置的服务器脚本建立长连接 -->
<iframe src="/remotecall" style="display: none;"></iframe>
</body>
</html>
'''
# 保存了所有在线用户的全局列表
global_browsers = set()

def handler(httpfile):
    if httpfile.path_info != '/remotecall':
        httpfile.start_response('200 OK', [('Content-Type', 'text/html')])
        httpfile.sendall(page) # 输出前面定义的上传表单
        return httpfile.close()
    # 创建 Browser 对象
    # 调用客户端名为 "my_alert()" 的 js 函数
    current_browser = browser(httpfile)
    current_browser.my_alert(u'我加入啦!')
    # 通知其他在线用户, 有新人加入
    disconnected = []
    for ibrowser in global_browsers:
        try:
            # 调用在线用户的 my_alert 函数
            ibrowser.my_alert(u'又有新人加入啦!')
        except error: # 已经断开连接
            disconnected.append(ibrowser)
    # 移除已经断开连接的浏览器
    for ibrowser in disconnected:
        global_browsers.remove(ibrowser)
    # 将当前浏览器添加到全局浏览器对象列表中
    global_browsers.add(current_browser)

httpd = httpserver(':8080', handler)
httpd.start()
mainloop()
```

  * browser 接口的 domain 参数用于指定 javascript 域（document.domain）方便跨域调用
  * browser 接口的 timeout 参数用于指定建立 js rpc 连接的超时时间
  * browser 会增加一个 httpfile 对象的引用，在没有引用时，与用户的连接会自动断开

## keep-alive ##

如果浏览器支持 keep-alive 特性，服务器将会自动以 keep-alive 方式处理浏览器请求。

  * 服务器以 keep-alive 方式处理每一个请求
  * 默认情况下使用 httpfile.close() 接口完成本次请求，并继续保持连接
  * 使用 httpfile.shutdown() 强制断开连接（无论是否完成）

## 超时处理 ##

多数 web 接口可以通过 timeout 参数设置超时，timeout 是以秒为单位的浮点数（float），默认 -1 意思是无超时。

```
# -*- coding: utf-8 -*-
from eurasia.cgietc import form
from eurasia.core import timeout  # 超时异常
from eurasia.web import httpserver, mainloop

def handler(httpfile):
    try:
        form1 = form(httpfile, max_size=10240, timeout=30.)  # 30 秒超时
    except timeout:
        httpfile.shutdown()  # 中断连接
    try:
        httpfile.write(repr(form1), timeout=10.)  # 10 秒超时
    except timeout:
        httpfile.shutdown()
    httpfile.close()  # 正常结束请求

httpd = httpserver(':8080', handler)
httpd.start()
mainloop()
```

  * 发生超时，会抛出 eurasia.core.timeout 异常，但是连接并不会随之中断
  * 用户仍然可以恢复因为超时中断的工作，这可能会导致一些潜在问题，因此建议断开连接并重试
  * 和 web 服务器不同，socket 服务器可以有效恢复因 timeout 而中断的工作，这将在后面的章节详解
  * 如果不捕获 timeout，异常将会传递到顶层，自动终止本次请求，断开连接

## wsgiserver ##

使用 wsgiserver(_**addr**_, _**app**_) 创建标准的 wsgi 服务器。

```
from eurasia.web import wsgiserver, mainloop
def app(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return ['hello world!']

httpd = wsgiserver(':8080', app)
httpd.start()
mainloop()
```

  * wsgiserver 使用和 httpserver 相同的地址描述格式
  * wsgiserver 中应使用框架自带的文件 IO 接口（见“文件”一节）

或者使用 wsgi(_**app**_) 将 app 转换为普通的 http handler。

```
from eurasia.web import httpserver, wsgi, mainloop
def app(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return ['hello world!']

def handler(httpfile):
    wsgihandler = wsgi(app)
    wsgihandler(httpfile)

httpd = httpserver(':8080', handler)
httpd.start()
mainloop()
```


| | | |
|:|:|:|
|  |  |
|  |


# 使用标准模板 #

eurasia.template 模块是 Mako 的简化版，带有大部分 Mako 模板的功能，和相同的标签语法。同时也是编译型模板。

template 包涵以下标签语法。

  * 表达式替换

```
this is x: ${x}
```

> 进一步，取值表达式可以嵌入 python 代码，并替换为 python 表达式的运算结果：

```
${int(a) + int(b)}
```

  * 控制结构

> 我们可以在模板中使用条件表达式及叠代循环表达式。

> 这里是条件表达式：

```
%if x==1:
    x is ${x}
%elif x==2:
    x is ${x}
%else:
    x is ${x}
%endif
```

> 这里是循环：

```
%for a in ['one', 'two', 'three', 'four', 'five']:
    %if a[0] == 't':
        its two or three
    %elif a[0] == 'f':
        four/five
    %else:
        one
    %endif
%endfor
```

  * python 代码

> 可以在模板中运行 python 代码：

```
<% # 这里是 Python 代码
a = 1
b = 2
%>

测试一下 a + b : ${a + b}
```

> 第一行（紧跟“<%”之后）的 python 代码的缩进_**会被忽略**_，所以请避免使用 def、class、if、for 等依赖缩进的 python 代码。

> 函数中的 python 代码块在当前函数中可见，函数外的代码块在当前模板中全局可见。

  * 函数标签 <%def name="..." %>

> 函数是模板中最基本的调用单位，类似于 Python 中的函数：

```
<%def name="myfunc(x)">
    this is myfunc, x is ${x}
</%def>

调用: ${myfunc(7)}
```

  * <%call expr="..."%> 标签

> call 标签用于调用 <%def%> 标签，可传递额外的内嵌内容。稍后介绍。

## 使用模板 ##

为了演示 eurasia 标准模板的用法，这里将建立一个模板的应用范例。

```
#!/usr/bin/python2.5
#-*- coding: utf-8 -*-
from eurasia.template import Template

s = '''\                  # 字符串模板
<%def name="test1(a)">
    test1, a is ${a}
</%def>

<%def name="test2(b)">
    test2, b is ${b}
</%def>'''

tmpl = Template(s)        # 将字符串模板编译成可执行的 Python 模块

print tmpl.test1('hello') # 调用模板中的 "test1" 函数
print tmpl.test2('world') # 调用模板中的 "test2" 函数
```

在这个例子里，首先是一个字符串模板，带有 test1 和 test2 两个函数。

然后使用 Template() 编译成可执行的 python 模块，并对模板中的 test1、test2 函数进行调用。

结果是：

```
test1, a is hello
test2, b is world
```

<%def> 标签定义了模板的调用接口, 在这个例子中, 我们定义了 test1 和 test2 两个调用接口。

_我们已经确认 cpython 中存在的一个 bug 将导致 Template(s).test1('hello') **这种写法出错**。因此需要分开来写成：_

```
tmpl = Template(s)
tmpl.test1('hello')
```

_也就是上面这个例子中的写法。非 c 实现的 python 没有这个问题。_

## 一个更为复杂的例子 ##

本例中，将用到大量模板常用语法。包括有条件判断，循环等：

```
#!/usr/bin/python2.5
#-*- coding: utf-8 -*-
from eurasia.template import Template

s = '''\
<%def name="main(lst)">
--------------------------------------------
<% # python 代码块示范
class Foo:
    def test(self):
        return 'from foo.test()'

foo = Foo()

# 在 python 代码块中可以使用 write 函数进行内容输出
write('\\n' + foo.test())

%>

${Foo().test()}

--------------------------------------------
# 循环判断示范
%for i in lst:
    %if i == 1:
        ${1+2}
    %elif i == 2:
        ${'hello' + 'world'}
    %else:
        ${i}
    %endif
%endfor

--------------------------------------------
# 嵌入式函数示范, 只在 main 函数中可见
<%def name="bar()">
    this is bar
</%def>
${bar()}

--------------------------------------------
</%def>'''

tmpl = Template(s)
print tmpl.main([1, 2, 3, 4, 5])
```

## 使用 <%call> 标签定义宏 ##

<%call> 标签提供了一种较为高级的功能，也就是模板宏：

```
<%def name="macros(a, b)">
${caller.slot1(a)}
${caller.slot2(b)}
</%def>

----------------------------

<%def name="main()">
<%call expr="macros(1, 2)">
  <%def name="slot1(a)">
    slot1, ${a}
  </%def>

  <%def name="slot2(b)">
    slot1, ${b}
  </%def>
</%call>
</%def>
```

这里首先使用 <%def> 定义了一个名为 macros 的模板宏。

然后在 main 函数中使用 <%call> 调用 macros，其中定义了 slot1 和 slot2，在 macros 中可以通过 caller 取出。

该模板的入口函数是 main()。调用结果是：

```
slot1, 1
slot2, 2
```

  * 本例中在同一个模板中同时包涵了模板宏和宏调用，通常情况下，我们会将模板宏保存入外部文件。
  * call 只能在函数中 (<%def>) 调用。如果把 call 放置在模板顶层，模板在编译时会忽略 call 调用。

## 缓存编译结果 ##

eurasia 标准模板是一种编译型模板，字符串模板经过 Template 函数可以编译成可执行的 python 模块。

我们也可以仅仅把字符串模板转换成 python 源代码，在文件中保存起来，这样可以在下次使用时直接 import 进来，省去了编译过程，尽管这花不了多少时间。

这里我们将用到 template 的 compile 工具。

```
#!/usr/bin/python2.5
#-*- coding: utf-8 -*-
from eurasia.template import compile

s = '''\
<%def name="main()">
	hello world!
</%def>'''

# 得到 python 源码
code = compile(s)

# 保存为 python 模块文件
fd = open('cache.py', w)
fd.write(code)
fd.close()

# 以模块方式导入模板
import cache
print cache.main()
```

## <%namespace> 标签 ##

eurasia 标准模板中没有 Mako 中 <%namespace> 标签的等价物，这个功能相当于 Python 中的 import。

<%namespace> 需要一系列关于导入路径的配置，这比较复杂，因此 eurasia 换了一种方式。

Template 函数第二个参数可以设定模板中可见的环境。你可以在这里预先导入一些模板中用得到的东西，比如宏。

这个例子在讲解 <%call> 时出现过，这里把宏和调用部分拆离开来。

```
#!/usr/bin/python2.5
#-*- coding: utf-8 -*-
from eurasia.template import Template
s1 = '''\ # 定义模板宏
<%def name="macros(a, b)">
${caller.slot1(a)}
${caller.slot2(b)}
</%def>'''

s2 = '''\ # 调用模板宏
<%def name="main()">
<%call expr="macros(1, 2)">
  <%def name="slot1(a)">
    slot1, ${a}
  </%def>

  <%def name="slot2(b)">
    slot1, ${b}
  </%def>
</%call>
</%def>'''

tmpl1 = Template(s1)

# 使用 env 参数指定模板环境
tmpl2 = Template(s2, env={'macros': tmpl1.macros})

tmpl2.main()
```

| | | |
|:|:|:|
|  |  |
|  |


# 文件及 socket #

框架提供了专门的文件 IO 接口，用来提高系统性能。

而 socket 则是一种特殊的文件，也在这一节介绍。

## 文件 ##

使用 core.file(_**fileno**_) 接口对已打开的文件描述符（fileno）进行高效操作。

```
#!/usr/bin/python2.6
#-*- coding: utf-8 -*-

# epoll 不支持本地磁盘文件 patch 成 poll
from eurasia.pyev import *
mainloop = default_loop(EVBACKEND_POLL).loop

import os, sys
from eurasia import core
from traceback import print_exc
from eurasia.web import httpserver

# 打开调试输出
core.excepthook = lambda: print_exc(file=sys.stderr)

def handler(httpfile):
    httpfile.start_response('200 OK')

    # 文件读取
    fileno = os.open('test.txt', os.O_RDONLY|os.O_NONBLOCK)
    f = core.file(fileno)
    s = f.read()
    os.close(fileno)

    httpfile.sendall(s)
    httpfile.close()

httpd = httpserver(':8080', handler)
httpd.start()
mainloop()
```

  * eurasia 默认使用的 epoll 等后端无法处理磁盘文件，需要 patch 成 select 或 poll
  * 使用 os.open('test.txt', os.O\_NONBLOCK|...) 和 os.close(fileno) 来打开和关闭文件

在 unix 下，管道、socket、设备等等都是文件，都可以使用 core.file() 接口。

```
from os import popen
from eurasia.core import file

...

lsdir = popen('ls -alh')
fd = file(lsdir.fileno())
files = fd.readlines()

...

mainloop()
```

  * epoll 支持管道，无须 patch

## 文件对象接口列表 ##

| 接口类型： | 接口描述： | 解释： |
|:----------------|:----------------|:----------|
| 成员函数 | file.ready(_**timeout**=-1_) | 等待文件可操作，默认无超时 |
| 成员函数 | file.recv(_**size**=8192_, _**timeout**=-1_) | 当可读时返回尽可能多数据，需设置缓冲区 size |
| 成员函数 | file.send(_**data**_, _**timeout**=-1_) | 当可写时写入尽可能多数据，返回写入数据的字节数 |
| 成员函数 | file.read(_**size**=-1_, _**timeout**=-1_) | 读取指定 size，默认读取整个文件 |
| 成员函数 | file.readline(_**size**=-1_, _**timeout**=-1_) | 读取一行 |
| 成员函数 | file.sendall(_**data**_, _**timeout**=-1_) | 发送全部指定数据 |
| 成员函数 | file.close() | 关闭文件 |

  * file.recv() 不能和 file.read()、file.readline() 混用
  * file.send() 不能和 file.sendall() 混用

## socket ##

使用框架自带的 socket2 以替代 socket 标准库。

```
# from socket import socket, AF_INET, SOCK_STREAM
from eurasia.socket2 import socket, AF_INET, SOCK_STREAM

...

sock = socket(AF_INET, SOCK_STREAM)
sock.connect(('www.google.com', 80))
sock.sendall('GET / HTTP/1.0\r\n\r\n')
data = sock.read()

...
```

  * socket2.socket 不仅是 socket 也是 file 对象，可以使用 read()/readline()/sendall() 等接口
  * socket2.socket 对象用于创建客户端 socket
  * 出于性能考虑应总是使用 socket2

通过 socket2.install() 替换标准库。

```
from eurasia import socket2
socket2.install() # 这时标准库 socket 模块已经变成 socket2
import urllib     # urllib 将使用 socket2 模块

...

fd = urllib.urlopen('http://www.google.com/')
data = fd.read()

...
```

## tcp 服务器 ##

使用 server(_**addr**_, _**handler**_) 创建标准的 tcp 服务器。

这是一个 echo 服务。

```
# 文件名：test.py
from eurasia.server import server, mainloop
def handler(sock, addr, serv):
    data = sock.readline()
    while data.strip() != 'quit':
        sock.sendall(data)
        data = sock.readline()
    sock.close()

tcpd = server(':8080', handler)
tcpd.start()
mainloop()
```

  * handler 接受的三个参数，分别是：
    1. 客户端连接 _**sock**_，socket2.socket 对象
    1. 客户端地址 _**addr**_，tuple 类型，比如 ('192.168.0.101', 20000)
    1. 服务器对象 _**serv**_，也就是 server 本身

执行脚本，启动服务器。

```
$/usr/bin/python2 test.py
```

使用 telnet 连接到服务器，进行测试（输入 quit 退出测试）。

```
$telnet 127.0.0.1 8080
```

## 从超时恢复 ##

发生超时，会抛出 eurasia.core.timeout 异常。

当 file.recv()、file.send() 以及 socket.ready() 发生超时，没有产生任何 IO 操作，可以忽略超时。

```
from eurasia.core import timeout

# recv 超时
try:
    data = fd.recv(1024, 1.) # 读取 1024 字节，超时 1 秒
except timeout:
    data = ''

# send 超时
try:
    fd.send('hello world!', 1.) # 超时 1 秒
    sented = 'hello world!'
except timeout:
    sented = ''

# ready 超时
try:
    sock.ready(1.)
    ready = True
except timeout:
    ready = False
```

当 file.read()/readline() 发生超时，缓冲区可能已读取部分数据，但不影响下一次 file.read()/readline() 的读取。

```
try:
    data = file.read(1024, 1.)
except timeout:
    data = file.read(1024, 1.) # 重试
```

当 file.sendall() 发生超时，可以通过 timeout 对象的 num\_sent 属性，得到已经发送的字节数。

```
data = 'hello world!'
try:
    file.sendall(data, 1.)
except timeout, e:
    data = data[e.num_sent:] # 扣除已发送，得到剩余的数据
    file.sendall(data, 1.)
```


| | | |
|:|:|:|
|  |  |
|  |


# 使用 durus 数据库 #

通过 socket2.install() 可以使 eurasia 和 [durus](http://www.mems-exchange.org/software/durus/) 实现兼容。

首先需要一个 durus 数据库服务器。

```
from durus.file_storage import FileStorage
from durus.storage_server import StorageServer
storage = FileStorage('test.db')
server = StorageServer(storage=storage, address='test.sock')
server.serve()
```

在 eurasia 中通过 ClientStorage 连接数据库服务器。

```
from eurasia import socket2
socket2.install()
from time import time
from eurasia.web import httpserver, mainloop

form durus.persistent import Persistent
from durus.connection import Connection
from durus.client_storage import ClientStorage

class Guest(Persistent):
    def __init__(self, addr, port):
        self.addr = addr
        self.port = port

def handler(httpfile):
    conn = Connection(ClientStorage(address='test.sock'))
    root = conn.get_root()

    addr = httpfile.environ['REMOTE_ADDR']
    port = httpfile.environ['REMOTE_PORT']
    guest = Guest(addr, port)
    root[int(time())] = guest

    httpfile.start_response('200 OK', [('Content-Type', 'text/plain')])
    httpfile.sendall('added.')
    httpfile.close()
    conn.commit()
```
  * socket2.install() 必须在导入 durus 之前调用
  * 这个程序将每个访问者的地址和 port 存入数据库（使用 time() 做 key）


| | | |
|:|:|:|
|  |  |
|  |


# 框架功能介绍 #

这里介绍一些有用的框架工具。

## 使用 sleep 暂停线索 ##

core.sleep(_**sec**_) 是标准库 time.sleep() 在框架中的替代实现：

```
from eurasia.core import sleep

...

sleep(0.5) # 程序暂停 0.5 秒

...
```

## 创建协程 ##

在 eurasia 中可以直接使用 greenlet 库创建协程。

```
from time import time
from eurasia.greenlet import greenlet
from eurasia.core import sleep, mainloop
def loop():
    while 1:
        print time()
        sleep(1.)

greenlet(loop).switch()
mainloop()
```

## 调试 ##

通过设置 core.excepthook 安装调试钩子。

```
# -*- coding: utf-8 -*-
import sys, traceback
from eurasia import core
from eurasia.web import httpserver, mainloop

# 安装调试钩子
def excepthook():
    traceback.print_exc(file=sys.stderr)
core.excepthook = excepthook

def handler(httpfile):
    print not_exists # 这里会报错，变量 not_exists 不存在
    httpfile.close()

httpd = httpserver(('', 8080), handler)
httpd.start()
mainloop()
```

发布产品时，通过 core.without\_excepthook() 调用忽略调试钩子。

```
# -*- coding: utf-8 -*-
import sys, traceback
from eurasia import core
from eurasia.web import httpserver, mainloop

# 安装调试钩子
def excepthook():
    traceback.print_exc(file=sys.stderr)
core.excepthook = excepthook

...

core.without_excepthook() # 忽略前面设置的调试钩子
```

## 配置文件 ##

eurasia.pyetc 模块为配置文件的读取提供了支持。eurasia 认为使用 python 语法来编写配置文件是个好主意。

pyetc 中的 load(_**filename**_, _**`**`env**_) 函数能读取指定的 python 源码文件（文件后缀并不需要一定是“.py”）, 并设定配置文件中的可见环境 env。

load 函数能将指定源码文件转换成 python 模块并返回。

我们首先编写一个名为 httpd.conf 的配置文件：

```
Server(controller='Products.default.controller'
	port=8080)

user = 'nobody'
```

下面是对于这个配置文件的解析：

```
from eurasia import pyetc
config = {}
def Server(**args):
	config.update(args)

mod = pyetc.load('httpd.conf', env={Server:Server})
print 'Server:', config
print 'user', mod.user
```

  * pyetc 模块可以被用于导入执行任意指定路径和文件后缀的 python 模块

## 系统工具 ##

eurasia.utility 模块提供了一些常用的系统工具。

这里列出这些接口，以及说明：

| 接口：| 说明：|
|:---------|:---------|
| cpu\_count() | 返回所有 cpu 核心总数 |
| setuid(_**user**_) | 设定运行时身份，可以指定用户名或者用户 id |
| setprocname(_**procname**_) | 指定进程名 |
| dummy() | 禁止 stdout、stderr 输出 |

一个例子：

```
from eurasia import utility
utility.setprocname('hello')

...

sock.bind(('', 80)) # bind 到 80 需要 root 权限
utility.setuid('nobody') # 以 nobody 身份运行

...
```

  * 需要注意的是 utility.setprocname() 调用必须_**放在程序顶端**_否则就会出错


| | | |
|:|:|:|
|  |  |
|  |


# 部署 #

eurasia 3.1 不再原生提供对多核处理器和 daemon 的支持，以下是 eurasia 3.1 推荐的部署方案。

  * 支持多核

> 最简单的方法是在多个端口上启动多个独立的 eurasia 服务，数量与 cpu 核心相应，这样就可以充分使用 cpu 核心了。

> 然后通过 iptables（redirect）或者 lvs 做端口负载均衡即可。

  * 启动为 daemon

> eurasia 推荐使用 [daemon](http://www.libslack.org/daemon/) 命令启动服务器。

> 在多数 unix/linux 发行版上都可以非常方便地安装 daemon 命令。