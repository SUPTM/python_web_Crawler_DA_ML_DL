# 10 socketserver服务器

在第9章我们采用最基础的socket套接字实现了静态web服务器。实际上python提供了一个比较高级的模块socketserver，仅需创建一个继承此模块的子类并重载handle方法即可替代实现之前的创建、绑定、监听等socket语句操作。本章就来介绍socketserver。

socketserver模块的常见类见下表：

| 类 | 描述 |
| :--- | :--- |
| BaseServer | 包含核心服务器功能和mix-in类的钩子；仅用于推导，这样不会创建这个类的实例，是TCPServer或UDPServer类的父类 |
| TCPServer/UDPServer | 基础的网络同步TCP/UDP服务器 |
| UnixStreamServer/UnixDatagramServer | 基于文件的基础同步TCP/UDP服务器 |
| ForkingMixin/ThreadingMixIn | 核心派出或线程功能，只用作mix-in类与一个服务器类配合实现一些异步性；不能直接实例化这个类 |
| ForkingTCPServer/ForkingUDPServer | ForkingMixIn和TCPServer/UDPServer的组合 |
| ThreadingTCPServer/ThreadingUDPServer | ThreadingMixIn和TCPServer/UDPServer的组合 |
| BaseRequestHandler | 包含处理服务请求的核心功能；无法创建这样的类实例；StreamRequestHandler或DatagramRequestHandler的父类 |
| StreamRequestHandler/DatagramRequestHandler | 实现TCP/UDP服务器的服务处理器 |

可以很清晰的看到可以将这些类分为两大模块：创建服务器的类和处理请求的类。

## 10.1 socketserver实现TCP服务器

就像上表中看到的那样，用socketserver来实现服务器，只需要两种类：创建服务器类和处理请求类。本小节使用TCPServer类作为TCP服务器，使用BaseRequestHandler的自定义子类来实现处理请求。

```python
'''net10_tcp_server_sample.py'''
from socketserver import TCPServer as TCP
from socketserver import BaseRequestHandler as BRH
from time import ctime

HOST = 'localhost'
PORT = 8888
ADDR = (HOST, PORT)


class MyRequestHandler(BRH):
    """自定义请求处理类，继承BaseRequestHandler"""
    def handle(self):
        """重载handle方法"""
        print('...connected from:', self.client_address)
        self.data = self.request.recv(1024).strip()  # 读取数据
        cli_data = self.data.decode()
        print('%s send data: %s' % (self.client_address, cli_data))
        self.request.send(('[%s] %s' % (ctime(), cli_data)).encode('utf-8'))  # 发送数据


if __name__ == '__main__':
    tcpServ = TCP(ADDR, MyRequestHandler) # 创建TCP服务器实例对象，并调用自定义请求处理子类
    print('waiting for connection...')
    tcpServ.serve_forever() # 调用TCPServer的serve_forever来永久运行TCP服务器
```

 首先我们直接创建了TCPServer实例对象tcpServ，它调用了自定义的BaseRequestHandler子类MyRequestHandler来完成请求处理。然后通过TCPServer类的serve\_forever\(\)方法使得服务器永久运行。在MyRequestHandler类中，我们重载了它的父类BRH中的handle\(\)方法来实现自定义的请求处理。

可以很清晰的看到，之前我们在socket中所编写的绑定、监听、连接等代码均不需要我们写了，底层已经封装好了。我们只需专注于请求数据处理那里就可以了，这极大的减轻了开发工作量。这就是socketserver的本意。如下为这段代码的执行结果，很明显，完全和socket编写的一样，但更省事。

```
waiting for connection...
...connected from: ('127.0.0.1', 64394)
('127.0.0.1', 64394) send data: hello socketserver tcp server
```

![](/assets/socketserver1.png)

为了更清晰的明白它的原理，我们在这里看一下TCPServer类的初始化方法相关的几个方法，其他方法感兴趣读者可自行查看源代码进行分析。

```python
   address_family = socket.AF_INET

    socket_type = socket.SOCK_STREAM

    request_queue_size = 5

    allow_reuse_address = False

    def __init__(self, server_address, RequestHandlerClass, bind_and_activate=True):
        """Constructor.  May be extended, do not override."""
        BaseServer.__init__(self, server_address, RequestHandlerClass)
        self.socket = socket.socket(self.address_family,
                                    self.socket_type)
        if bind_and_activate:
            try:
                self.server_bind()
                self.server_activate()
            except:
                self.server_close()
                raise
    def server_bind(self):
        """Called by constructor to bind the socket.

        May be overridden.

        """
        if self.allow_reuse_address:
            self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.socket.bind(self.server_address)
        self.server_address = self.socket.getsockname()

    def server_activate(self):
        """Called by constructor to activate the server.

        May be overridden.

        """
        self.socket.listen(self.request_queue_size)

    def server_close(self):
        """Called to clean-up the server.

        May be overridden.

        """
        self.socket.close()
```

可以看到在初始化方法中，首先调用继承了它的父类BaseServer的初始化方法，然后设置了服务套接字；最后设置了地址绑定、监听等，和我们上一章的HTTPServer类的初始化方法实现的功能基本一致。也就是说，我们上一章写的服务器类HTTPServer已经被socketserver封装好啦，我们只需使用就可以啦。

然后我们来看一下BaseRequestHandler类的源码：

```python
class BaseRequestHandler:
    def __init__(self, request, client_address, server):
        self.request = request
        self.client_address = client_address
        self.server = server
        self.setup()
        try:
            self.handle()
        finally:
            self.finish()

    def setup(self):
        pass

    def handle(self):
        pass

    def finish(self):
        pass
```

可以看到，BaseRequestHandler类的初始化方法就是获取连接套接字，相当于socket中的accept。然后调用handle方法。而handle方法中什么都没有，就等用户自定义重载啦。这就像我们第9章第一节那样的代码框架一样，架子搭好，就等用户登台唱戏了。

总而言之，它就是把开发人员从底层socket代码中解放出来然后专注于数据传输处理就可以了。这就是框架的思想，读者若有心，会在工作或学习中发现，众多框架都是基于这种把开发人员从底层繁琐重复代码编写工作中解脱出来的思想。

## 10.2 基于socketserver实现静态web服务器

就像上一节了解的那样，本节我们通过socketserver中的ThreadingTCPServer和BaseRequestHandler来实现第九章多线程实现的静态web服务器。我们仅仅需要把之前的HTTPServer自定义处理方法handlerequest的代码复制过来，并稍加修改即可。足见工作量极大降低了。完整源码如下：

```python
'''net10_tcp_threading.py'''
from socketserver import ThreadingTCPServer as TCP
from socketserver import BaseRequestHandler as BRH
import re

HOST = 'localhost'
PORT = 8888
ADDR = (HOST, PORT)
VERSION = 8.0  # web服务器版本号
STATIC_PATH = './static/'


class MyRequestHandler(BRH):
    """自定义请求处理类，继承BaseRequestHandler"""

    def handle(self):
        """重载handle方法"""
        print(self.client_address, '连接了服务器')
        request_data = self.request.recv(2048).decode('utf-8')
        request_header_lines = request_data.splitlines()
        # print(request_header_lines[0])  # 第一行为请求头信息
        # 解析请求头，获取具体请求信息
        pattern = r'[^/]+(/[^ ]*)'
        request_html_name = re.match(pattern, request_header_lines[0]).group(1)
        # 根据解析到的内容补全将要读取文件的路径
        if request_html_name == '/':
            request_html_name = STATIC_PATH + 'baidu.html'
        else:
            request_html_name = STATIC_PATH + request_html_name

        # 根据文件情况来返回相应的信息
        try:
            html_file = open(request_html_name, 'rb')
        except FileNotFoundError:
            # 文件不存在，则返回文件不存在，并返回状态码404
            resp_headers = 'HTTP/1.1 404 not found\r\n'
            resp_headers += "Server: PWB" + str(VERSION) + '\r\n'
            resp_headers += '\r\n'
            resp_body = '==== 404 file not found===='.encode('utf-8')
        else:
            # 文件存在，则读取文件内容，并返回状态码200
            resp_headers = "HTTP/1.1 200 OK\r\n"  # 200代表响应成功并找到资源
            resp_headers += "Server: PWB" + str(VERSION) + '\r\n'  # 告诉浏览器服务器
            resp_headers += '\r\n'  # 空行隔开body
            resp_body = html_file.read()  # 显示内容为读取的文件内容
            html_file.close()
        finally:
            resp_data = resp_headers.encode('utf-8') + resp_body  # 结合响应头和响应体
            # 发送相应数据至浏览器
            self.request.send(resp_data)
            self.request.close()  # HTTP短连接，请求完即关闭TCP连接


if __name__ == '__main__':
    tcpServ = TCP(ADDR, MyRequestHandler)  # 创建TCP服务器实例对象，并调用自定义请求处理子类
    print('web server:PWB %s on port %d...\n' % (VERSION, PORT))
    tcpServ.serve_forever()  # 调用TCPServer的serve_forever来永久运行TCP服务器
```

## 10.3 应用流程示例

笔者认为在不考虑效率的前提下，只要根据以下socketserver流程便可以构建相应的服务器。读者可自由组合发挥。

![](/assets/socketserver5.png)



