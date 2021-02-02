<div align="center">
    <a href="https://reactphp.org"><img src="/reactphp-logo.png" alt="ReactPHP 中文文档" title="ReactPHP 中文文档" width="120px"></a>
</div>
    
<br>
    
<div align="center">
    <strong>事件驱动,非阻塞I/O的PHP</strong>
</div>

<br>

<div align="center">
    <a href="https://github.com/reactphp/reactphp/actions"><img src="https://github.com/reactphp/reactphp/workflows/CI/badge.svg" alt="Build Status"></a>
</div>

>ReactPHP是PHP中用于事件驱动编程的底层库。它的核心是一个事件循环，在此基础上它提供了底层实用程序，例如：流抽象、异步DNS解析器、网络客户端/服务器、HTTP客户端/服务器以及进程间通信。第三方库可以使用这些组件创建异步网络客户端/服务器等。

<br>

```php
$loop = React\EventLoop\Factory::create();

$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    return new React\Http\Message\Response(
        200,
        array(
            'Content-Type' => 'text/plain'
        ),
        "Hello World!\n"
    );
});

$socket = new React\Socket\Server(8080, $loop);
$server->listen($socket);

echo "Server running at http://127.0.0.1:8080\n";

$loop->run();
```

<div align="center">

这个用ReactPHP编写的简单的Web服务器对每个请求都响应 "Hello World" 

</div>

<br>

ReactPHP是PHP中用于事件驱动编程的底层库。

核心是一个事件循环，它在其上提供底层实用程序，
例如：流抽象，异步DNS解析器，网络客户端/服务器，HTTP客户机/服务器和与进程的交互。

第三方库可以使用这些用于创建异步网络客户端/服务器等的组件。

ReactPHP已经可用于生产环境，并且经过来自世界各地的各种项目数百万次的安装测试。

由于事件驱动架构，使它非常适合高效的网络服务器和处理数百或数千个并发连接，长期运行的应用程序和许多其他无阻塞I / O操作的协作多任务形式。

ReactPHP之所以与众不同，是因为其活跃的生态系统具有数百个第三方库，可让您与许多现有系统集成 ，例如公共网络服务、数据库系统和其他第三方API。

* **生产就绪**，并经过了实战测试。
* **稳固的**，具有稳定的长期支持（LTS）版本。
* **不需要扩展**，并且可以在任何平台上运行-没有任何借口！
* 利用**可选扩展**来获得更好的性能（如果可用）。
* **强烈建议使用最新版本的PHP 7 +**，以获得最佳性能和支持。
* **支持旧版PHP 5.3+和HHVM**，以实现最大兼容性。
* **精心设计的**和**可重复使用的组件**。
* **分离的零件**，因此可以用其他实现方式替换它们。
* 经过**测试**（单位和功能）。
* 尽可能采用**标准PSR**，以实现最大的互操作性。
* 旨在“技术中立”，因此您可以使用首选的应用程序堆栈。
* 小型**核心专业团队**由外部贡献者的**大型网络**支持。 

默认情况下，ReactPHP是非阻塞的，workers阻止I / O。
事件循环基于反应堆模式（因此命名），并且强烈受诸如EventMachine（Ruby），Twisted（Python）和
Node.js（V8）。 

## 核心组件

* **EventLoop**
  ReactPHP的核心反应器event-loop
  [文档](1.Core-Components/EventLoop.md)

* **Stream**
  事件驱动的可读写流，用于ReactPHP中的非阻塞I / O
  [文档](1.Core-Components/Stream.md)

* **Promise**
  Promises/A 的PHP实现
  [文档](1.Core-Components/Promise.md)


## 网络组件

* **Socket**
  异步，流式传输纯文本TCP / IP以及安全TLS套接字服务器和客户端连接
  [文档](2.Network-Components/Socket.md)

* **Datagram**
  事件驱动的UDP客户端和服务器套接字
  [文档](2.Network-Components/Datagram.md)

## 协议组件

* **HTTP**
  事件驱动的流式纯文本HTTP和安全HTTPS服务器
  [文档](3.Protocol-Components/Http.md)

* **HTTPClient**
  事件驱动的HTTP流客户端
  [文档](3.Protocol-Components/HttpClient.md)

* **DNS**
  异步DNS解析器
  [文档](3.Protocol-Components/Dns.md)

## 实用组件

* **Cache**
  异步缓存
  [文档](4.Utility-Components/Cache.md)

* **ChildProcess**
  执行子进程的库。
  [文档](4.Utility-Components/ChildProcess.md)

* **PromiseTimer**
  ReactPHP的Promise库的简单超时实现。
  [文档](4.Utility-Components/PromiseTimer.md)

* **PromiseStream**
  在ReactPHP之上构建的Promise和Stream之间的衔接环节。 
  [文档](4.Utility-Components/PromiseStream.md)