# Socket

[![Build Status](https://travis-ci.org/reactphp/socket.svg?branch=master)](https://travis-ci.org/reactphp/socket)

[ReactPHP](https://reactphp.org/) 的异步，流式纯文本TCP / IP以及安全的TLS套接字服务器和客户端连接。

套接字库基于[`EventLoop`](1.Core-Components/EventLoop.md) 
和 [` Stream `](1.Core-Components/Stream.md) 组件为套接字层服务器和客户端提供了可重用的接口。

服务器组件允许您构建接受来自网络客户端连接的网络服务器(如HTTP服务器)。

客户端组件允许您构建建立到网络服务器外发连接的网络客户端(例如HTTP或数据库客户端)。

这个库为这些提供了异步、流的方式，因此您可以在不阻塞的情况下处理多个并发连接。

**目录**

* [快速开始](#快速开始)
* [连接用法](#连接用法)
  * [ConnectionInterface](#connectioninterface)
    * [getRemoteAddress()](#getremoteaddress)
    * [getLocalAddress()](#getlocaladdress)
* [服务端用法](#服务端用法)
  * [ServerInterface](#serverinterface)
    * [connection event](#connection-event)
    * [error event](#error-event)
    * [getAddress()](#getaddress)
    * [pause()](#pause)
    * [resume()](#resume)
    * [close()](#close)
  * [Server](#server)
  * [高级服务端使用](#高级服务端使用)
    * [TcpServer](#tcpserver)
    * [SecureServer](#secureserver)
    * [UnixServer](#unixserver)
    * [LimitingServer](#limitingserver)
      * [getConnections()](#getconnections)
* [客服端用法](#客服端用法)
  * [ConnectorInterface](#connectorinterface)
    * [connect()](#connect)
  * [Connector](#connector)
  * [高级客户端使用](#高级客户端使用)
    * [TcpConnector](#tcpconnector)
    * [HappyEyeBallsConnector](#happyeyeballsconnector)
    * [DnsConnector](#dnsconnector)
    * [SecureConnector](#secureconnector)
    * [TimeoutConnector](#timeoutconnector)
    * [UnixConnector](#unixconnector)
    * [FixUriConnector](#fixeduriconnector)
* [安装](#安装)
* [测试](#测试)
* [License](#license)

## 快速开始

如果您发送任何连接，这是一个关闭连接的服务器：

```php
$loop = React\EventLoop\Factory::create();
$socket = new React\Socket\Server('127.0.0.1:8080', $loop);

$socket->on('connection', function (React\Socket\ConnectionInterface $connection) {
    $connection->write("Hello " . $connection->getRemoteAddress() . "!\n");
    $connection->write("Welcome to this amazing server!\n");
    $connection->write("Here's a tip: don't say anything.\n");

    $connection->on('data', function ($data) use ($connection) {
        $connection->close();
    });
});

$loop->run();
```

另请参阅[示例](https://github.com/reactphp/socket/blob/v1.6.0/examples)

这是一个客户端，该客户端输出所述服务器的输出，然后尝试向其发送字符串：

```php
$loop = React\EventLoop\Factory::create();
$connector = new React\Socket\Connector($loop);

$connector->connect('127.0.0.1:8080')->then(function (React\Socket\ConnectionInterface $connection) use ($loop) {
    $connection->pipe(new React\Stream\WritableResourceStream(STDOUT, $loop));
    $connection->write("Hello World!\n");
});

$loop->run();
```

## 连接用法

### ConnectionInterface

`ConnectionInterface` 用于表示任何传入和传出的连接，例如普通的TCP / IP连接。

传入或传出连接是实现React [`DuplexStreamInterface`](1.Core-Components/Stream.md#duplexstreaminterface) 的双工流（可读和可写）。

它包含已建立到/来自此连接的本地和远程地址（客户端IP）的附加属性。

最常见的情况是，所有实现[`ServerInterface`](#serverinterface)的类都会触发实现这个`ConnectionInterface`的实例，
并由所有实现[`ConnectorInterface`](#connectorinterface)的类使用。

因为` ConnectionInterface `实现了底层的[`DuplexStreamInterface`](1.Core-Components/Stream.md#duplexstreaminterface)
所以你可以像往常一样使用它的所有事件和方法:

```php
$connection->on('data', function ($chunk) {
    echo $chunk;
});

$connection->on('end', function () {
    echo 'ended';
});

$connection->on('error', function (Exception $e) {
    echo 'error: ' . $e->getMessage();
});

$connection->on('close', function () {
    echo 'closed';
});

$connection->write($data);
$connection->end($data = null);
$connection->close();
// …
```

更多细节，请参阅[`DuplexStreamInterface`](1.Core-Components/Stream.md#duplexstreaminterface).

#### getRemoteAddress()

`getRemoteAddress(): ?string` 方法返回已经建立连接的完整远程地址(URI)。

```php
$address = $connection->getRemoteAddress();
echo 'Connection with ' . $address . PHP_EOL;
```

如果此时无法确定或未知远程地址(比如在连接关闭后)，它可能会返回一个` NULL `值。

否则，它将返回完整地址(URI)作为字符串值，例如` tcp://127.0.0.1:8080 `，
`tcp://[::1]:80`， `tls://127.0.0.1:443`， `unix://example.sock`
或 `unix:///path/to/example.sock`

请注意，单个URI组件是特定于应用程序的，并依赖于底层传输协议。

如果这是一个基于TCP/IP的连接，而你只想要远程IP，你可以这样使用:

```php
$address = $connection->getRemoteAddress();
$ip = trim(parse_url($address, PHP_URL_HOST), '[]');
echo 'Connection with ' . $ip . PHP_EOL;
```

#### getLocalAddress()

`getLocalAddress(): ?string` 方法返回已经建立连接的完整本地地址(URI)。

```php
$address = $connection->getLocalAddress();
echo 'Connection with ' . $address . PHP_EOL;
```

如果此时无法确定或未知本地地址(比如在连接关闭后)，它可能会返回一个` NULL `值。

否则，它将返回完整地址(URI)作为字符串值，例如` tcp://127.0.0.1:8080 `，
`tcp://[::1]:80`， `tls://127.0.0.1:443`， `unix://example.sock`
或 `unix:///path/to/example.sock`

请注意，单个URI组件是特定于应用程序的，并依赖于底层传输协议。

这个方法是[`getRemoteAddress()`](#getremoteaddress) 方法的补充，所以不应该混淆它们。

如果你的` TcpServer `实例正在监听多个接口(例如使用地址` 0.0.0.0 `)，
你可以使用这个方法找出哪个接口实际上接受了这个连接(例如一个公共的或本地的接口)。

如果您的系统具有多个接口（例如WAN和LAN接口），则可以使用此方法找出实际用于该连接的接口。

## 服务端用法

### ServerInterface

`ServerInterface` 提供一个接收流连接的接口，比如一个正常的TCP/IP连接。

大多数更高级别的组件（例如HTTP服务器）都接受实现此接口的实例，以接受传入的流连接。

通常这是通过依赖项注入完成的，便于你将该实现替换为该接口的其他实现。

这意味着您应该对此接口进行类型提示，而不是对此接口的具体实现。

除了定义一些方法之外，这个接口还实现了[`EventEmitterInterface`](https://github.com/igorw/evenement) ，它允许您对某些事件作出反应。

#### connection event

当建立了一个新的连接时，就会触发` connection `事件，例如，一个新的客户端连接到这个服务器套接字:

```php
$server->on('connection', function (React\Socket\ConnectionInterface $connection) {
    echo 'new connection' . PHP_EOL;
});
```

请参阅[`ConnectionInterface`](#connectioninterface)以获得有关处理传入连接的更多细节。

#### error event

当从客户端接收新连接时发生错误时，将触发`error`事件。

```php
$server->on('error', function (Exception $e) {
    echo 'error: ' . $e->getMessage() . PHP_EOL;
});
```

注意，这不是一个致命错误事件，也就是说，即使在这个事件之后，服务器仍然在监听新的连接。

#### getAddress()

`getAddress(): ?string` 方法可以用来返回服务器当前正在监听的完整地址(URI)。

```php
$address = $server->getAddress();
echo 'Server listening on ' . $address . PHP_EOL;
```
如果此时地址无法确定或未知(例如在套接字关闭后)，它可能会返回一个` NULL `值。

否则，它将返回完整地址(URI)作为字符串值，例如`tcp://127.0.0.1:8080`, `tcp://[::1]:80`, 
`tls://127.0.0.1:443`，`unix://example.sock` 或 `unix:///path/to/example.sock`

请注意，单个URI组件是特定于应用程序的，并依赖于底层传输协议。

如果这是一个基于TCP/IP的服务器，你只需要本地端口，你可以这样使用:

```php
$address = $server->getAddress();
$port = parse_url($address, PHP_URL_PORT);
echo 'Server listening on port ' . $port . PHP_EOL;
```

#### pause()

`pause(): void`方法可以用于暂停接受新的传入连接。

从`EventLoop`中删除套接字资源，从而停止接受新连接。注意，监听套接字保持活动并不是关闭了。

这意味着新的传入连接将在操作系统`backlog`中保持挂起状态，直到其可配置的`backlog`被填满为止。

一旦积压被填满，操作系统可能会拒绝新传入的连接，直到再次通过恢复接受新的连接来耗尽积压。

一旦服务器暂停，就不应该触发更多的`connection`事件。

```php
$server->pause();

$server->on('connection', assertShouldNeverCalled());
```

此方法仅供参考，但通常不推荐使用，服务器可以继续触发`connection`事件。

除非另有说明，成功打开的服务器不应在暂停状态下启动。

您可以通过再次调用` resume() `继续处理事件。

请注意，这两个方法都可以被调用多次，特别是多次调用` pause() `无效。

同样，在` close() `之后调用这个函数也是无效操作。

#### resume()

`resume(): void`方法可以用来恢复接收新的连接。

在前一个` pause() `之后重新将套接字资源附加到`EventLoop`。

```php
$server->pause();

$loop->addTimer(1.0, function () use ($server) {
    $server->resume();
});
```

请注意，这两个方法都可以被多次调用，之前没有调用` pause() `函数，调用` resume() `无效。

同样，在` close() `之后调用这个函数也是一个无效操作。

#### close()

`close(): void`方法可以用来关闭这个监听套接字。

这将停止侦听这个套接字上的新传入连接。

```php
echo 'Shutting down server socket' . PHP_EOL;
$server->close();
```

在同一个实例上多次调用这个方法是无效的。

### Server

` Server `类是这个包中的主要类，它实现了[`ServerInterface`](#serverinterface)，
并允许您接受传入的流连接，例如明文TCP/IP或安全TLS连接流。

Unix域套接字也可以接受连接。

```php
$server = new React\Socket\Server(8080, $loop);
```

如上所述，` $uri `参数可以只包含一个端口，在这种情况下，服务器将默认侦听本地主机地址` 127.0.0.1 `，这意味着从系统外部无法访问该地址。

为了使用一个随机的端口分配，你可以使用端口` 0 `:

```php
$server = new React\Socket\Server(0, $loop);
$address = $server->getAddress();
```

为了改变套接字正在监听的主机，你可以给构造函数的第一个参数提供一个IP地址，可以在IP前面加上` tcp:// `方案:

```php
$server = new React\Socket\Server('192.168.0.1:8080', $loop);
```

如果你想监听一个IPv6地址，你必须用方括号括起主机:

```php
$server = new React\Socket\Server('[::1]:8080', $loop);
```

要在Unix域套接字(UDS)路径上监听，必须在URI前面加上` Unix:// `方案:

```php
$server = new React\Socket\Server('unix:///tmp/server.sock', $loop);
```

如果给定的URI无效，不包含端口，任何其他方案，或者包含主机名，它将抛出一个` InvalidArgumentException `:

```php
// throws InvalidArgumentException due to missing port
$server = new React\Socket\Server('127.0.0.1', $loop);
```

如果给定的URI看起来是有效的，但是监听失败(比如端口已经被使用或者端口低于1024可能需要root访问等等)，
它将抛出一个` RuntimeException `:

```php
$first = new React\Socket\Server(8080, $loop);

// throws RuntimeException because port is already in use
$second = new React\Socket\Server(8080, $loop);
```
>请注意，这些错误条件可能因您的系统和/或配置而异。
 有关实际错误条件的详细信息，请参阅异常消息和代码。

您还可以为底层流套接字资源指定[TCP socket context options](https://www.php.net/manual/en/context.socket.php)
如下所示:

```php
$server = new React\Socket\Server('[::1]:8080', $loop, array(
    'tcp' => array(
        'backlog' => 200,
        'so_reuseport' => true,
        'ipv6_v6only' => true
    )
));
```
>请注意，可用的[socket context options](https://www.php.net/manual/en/context.socket.php),
 它们的默认值和更改这些选项的效果可能会根据您的系统和/或PHP版本而有所不同，传递未知的上下文选项没有效果。
 除非明确给出，否则` backlog `上下文选项默认为` 511 `。
 出于BC原因，您还可以将TCP套接字上下文选项作为一个简单的数组传递，而不必将其包装在` TCP `键下的另一个数组中。

您可以启动一个安全TLS(以前称为SSL)服务器，只需在` TLS:// ` URI前添加一个前缀。

它内部将等待明文TCP/IP连接，然后对每个连接执行TLS握手。

因此，它需要有效的[TLS上下文选项](https://www.php.net/manual/en/context.ssl.php) ,
如果您使用PEM编码的证书文件,它在其最基本的形式可能看起来像这样:
```php
$server = new React\Socket\Server('tls://127.0.0.1:8080', $loop, array(
    'tls' => array(
        'local_cert' => 'server.pem'
    )
));
```

> 注意，证书文件不会在实例化时加载，而是在传入连接初始化其TLS上下文时加载。
  这意味着任何无效的证书文件路径或内容只会在以后的时间导致`error`事件。 

如果您的私钥已使用密码加密，则必须像这样指定它：

```php
$server = new React\Socket\Server('tls://127.0.0.1:8000', $loop, array(
    'tls' => array(
        'local_cert' => 'server.pem',
        'passphrase' => 'secret'
    )
));
```

默认情况下，此服务器支持TLSv1.0 +，并且不支持旧版SSLv2 / SSLv3。
从PHP 5.6+开始，您还可以显式选择要与远程端协商的TLS版本：

```php
$server = new React\Socket\Server('tls://127.0.0.1:8000', $loop, array(
    'tls' => array(
        'local_cert' => 'server.pem',
        'crypto_method' => STREAM_CRYPTO_METHOD_TLSv1_2_SERVER
    )
));
```
>请注意，使用[TLS context options](https://www.php.net/manual/en/context.ssl.php),
 它们的默认值和更改这些值的效果可能会因您的系统和/或PHP版本而异。
 外部上下文数组允许您同时使用`tcp`（可能还有更多）上下文选项。
 传递未知的上下文选项无效。
 如果您不使用`tls：//`方案，那么传递`tls`上下文选项将无效。

每当客户端连接时，它将通过实现[`ConnectionInterface`](#connectioninterface)的连接实例触发`connection`事件：

```php
$server->on('connection', function (React\Socket\ConnectionInterface $connection) {
    echo 'Plaintext connection from ' . $connection->getRemoteAddress() . PHP_EOL;
    
    $connection->write('hello there!' . PHP_EOL);
    …
});
```

另请参阅 [`ServerInterface`](#serverinterface)

>注意，Server类是TCP / IP套接字的具体实现。
 如果要在更高级别的协议实现中类型提示，则应改用通用的 [`ServerInterface`](#serverinterface)

### 高级服务端使用

#### TcpServer

` TcpServer `类实现 [`ServerInterface`](#serverinterface)，并负责接受明文TCP/IP连接。

```php
$server = new React\Socket\TcpServer(8080, $loop);
```

如上所述，` $uri `参数可以只包含一个端口，在这种情况下，服务器将默认侦听本地主机地址` 127.0.0.1 `，这意味着从系统外部无法访问该地址。

为了使用一个随机的端口分配，你可以使用端口` 0 `:

```php
$server = new React\Socket\TcpServer(0, $loop);
$address = $server->getAddress();
```

为了改变套接字正在监听的主机，你可以通过提供给构造函数的第一个参数提供一个IP地址，在此参数之前有一个` tcp:// `方案:

```php
$server = new React\Socket\TcpServer('192.168.0.1:8080', $loop);
```

如果你想监听一个IPv6地址，你必须用方括号括起主机:

```php
$server = new React\Socket\TcpServer('[::1]:8080', $loop);
```

如果给定的URI无效，不包含端口，任何其他方案，或者包含主机名，则将抛出`InvalidArgumentException`：

```php
// throws InvalidArgumentException due to missing port
$server = new React\Socket\TcpServer('127.0.0.1', $loop);
```

如果给定的URI似乎有效，但是对其进行侦听失败（例如，如果端口已在使用中，或者端口低于1024，则可能需要root用户访问权限等），
它将抛出`RuntimeException`：

```php
$first = new React\Socket\TcpServer(8080, $loop);

// throws RuntimeException because port is already in use
$second = new React\Socket\TcpServer(8080, $loop);
```

> 请注意，这些错误情况可能会因您的系统和/或配置而异。
  有关实际错误情况的更多详细信息，请参阅异常消息和代码。

您还可以为底层流套接字资源指定[socket context options](https://www.php.net/manual/en/context.socket.php) ，如下所示:

```php
$server = new React\Socket\TcpServer('[::1]:8080', $loop, array(
    'backlog' => 200,
    'so_reuseport' => true,
    'ipv6_v6only' => true
));
```

>请注意，可用的[socket context options](https://www.php.net/manual/en/context.socket.php),
 它们的默认值和更改这些选项的效果可能会根据您的系统和/或PHP版本而有所不同，传递未知的上下文选项没有效果。
 除非明确给出，否则` backlog `上下文选项默认为` 511 `。

当客户端连接时，它将发出一个` connection `事件，该事件的连接实例实现了 [`ConnectionInterface`](#connectioninterface):

```php
$server->on('connection', function (React\Socket\ConnectionInterface $connection) {
    echo 'Plaintext connection from ' . $connection->getRemoteAddress() . PHP_EOL;
    
    $connection->write('hello there!' . PHP_EOL);
    …
});
```

更多细节请参阅[`ServerInterface`](#serverinterface)

#### SecureServer

` SecureServer `类实现了[`ServerInterface`](#serverinterface)，负责提供安全的TLS(以前称为SSL)服务器。

它通过包装一个[`TcpServer`](#tcpserver)实例实现，该实例等待明文TCP/IP连接，然后对每个连接执行TLS握手。

因此，它需要有效的[TLS context options](https://www.php.net/manual/en/context.ssl.php) ，
如果您使用PEM编码的证书文件,其最基本的形式可能看起来像这样，:

```php
$server = new React\Socket\TcpServer(8000, $loop);
$server = new React\Socket\SecureServer($server, $loop, array(
    'local_cert' => 'server.pem'
));
```
>注意，证书文件不会在实例化时加载，而是在传入连接初始化其TLS上下文时加载。
 这意味着任何无效的证书文件路径或内容只会在以后的时间导致`error`事件。 

如果你的私钥是用密码加密的，你必须这样指定:

```php
$server = new React\Socket\TcpServer(8000, $loop);
$server = new React\Socket\SecureServer($server, $loop, array(
    'local_cert' => 'server.pem',
    'passphrase' => 'secret'
));
```

默认情况下，此服务器支持TLSv1.0 +，并且不支持旧版SSLv2 / SSLv3。 
从PHP 5.6+开始，您还可以显式选择要与远程端协商的TLS版本：

```php
$server = new React\Socket\TcpServer(8000, $loop);
$server = new React\Socket\SecureServer($server, $loop, array(
    'local_cert' => 'server.pem',
    'crypto_method' => STREAM_CRYPTO_METHOD_TLSv1_2_SERVER
));
```

>请注意，使用[TLS context options](https://www.php.net/manual/en/context.ssl.php) ，
 它们的默认值和更改它们的效果可能会因系统和/或PHP版本而异。
 传递未知的上下文选项无效。

每当客户端完成TLS握手时，它将发出带有实现[`ConnectionInterface`](#connectioninterface)的连接实例的`connection`事件：

```php
$server->on('connection', function (React\Socket\ConnectionInterface $connection) {
    echo 'Secure connection from' . $connection->getRemoteAddress() . PHP_EOL;
    
    $connection->write('hello there!' . PHP_EOL);
    …
});
```

每当客户端未能成功执行TLS握手时，客户端都会触发`error`事件，然后关闭基础TCP / IP连接：

```php
$server->on('error', function (Exception $e) {
    echo 'Error' . $e->getMessage() . PHP_EOL;
});
```
另请参阅[`ServerInterface`](#serverinterface)

请注意，` SecureServer `类是TLS套接字的具体实现。
如果你想在高级协议实现中类型提示，你应该使用通用的[`ServerInterface`](#serverinterface)来代替。

>高级用法：尽管允许将任何`ServerInterface`作为第一个参数，但应该将`TcpServer`实例作为第一个参数传递，除非您知道自己在做什么。
 `SecureServer`内部必须在底层流资源上设置所需的TLS上下文选项。
 这些资源不会通过此包中定义的任何接口，而只能通过内部`Connection`类公开。
 `TcpServer`类保证发出实现`ConnectionInterface`的连接，并使用内部`Connection`类来公开这些底层资源。
 如果使用自定义`ServerInterface`且其`connection`事件不满足此要求，`SecureServer`将触发`error`事件，然后关闭连接。

#### UnixServer

` UnixServer `类实现了[`ServerInterface`](#serverinterface)，并负责接受Unix域套接字(UDS)上的连接。

```php
$server = new React\Socket\UnixServer('/tmp/server.sock', $loop);
```
如上所述，` $uri `参数只能由一个套接字路径或以` unix:// `方案为前缀的套接字。

如果给定的URI看起来是有效的，但是监听失败(比如socket已经在使用或者文件不能访问等等)，
它将抛出一个`RuntimeException`:

```php
$first = new React\Socket\UnixServer('/tmp/same.sock', $loop);

// throws RuntimeException because socket is already in use
$second = new React\Socket\UnixServer('/tmp/same.sock', $loop);
```

> 请注意，这些错误条件可能因您的系统和/或配置而异。
  特别是，当UDS路径已经存在且不能被绑定时，Zend PHP只会报告"Unknown error"。
  在这种情况下，您可能需要检查指定UDS路径上的` is_file() `，以报告更友好的错误消息。
  有关实际错误条件的详细信息，请参阅异常消息和代码。

当客户端连接时，它将发出一个` connection `事件，该事件的连接实例实现了 [`ConnectionInterface`](#connectioninterface):

```php
$server->on('connection', function (React\Socket\ConnectionInterface $connection) {
    echo 'New connection' . PHP_EOL;

    $connection->write('hello there!' . PHP_EOL);
    …
});
```
更多细节请参阅 [`ServerInterface`](#serverinterface)

#### LimitingServer

` LimitingServer `装饰器包装了一个给定的` ServerInterface `，并负责限制和跟踪到这个服务器实例的打开连接。

每当底层服务器发出` connection `事件时，将检查其限制，做出以下两种情况之一
 - 通过将该连接添加到打开的连接列表中来跟踪该连接，然后触发` connection `事件
 - 或者当连接超出限制时拒绝(关闭)连接，并将触发` error `事件。

当一个连接关闭时，它将从打开的连接列表中删除该连接。

```php
$server = new React\Socket\LimitingServer($server, 100);
$server->on('connection', function (React\Socket\ConnectionInterface $connection) {
    $connection->write('hello there!' . PHP_EOL);
    …
});
```

更多细节请参阅[第二个示例](https://github.com/reactphp/socket/blob/v1.6.0/examples) 

您必须传递打开连接的最大数量，以确保一旦超过这个限制，服务器将自动拒绝(关闭)连接。
在本例中，它将发出一个` error `事件来通知此情况，而不会触发` connection `事件。

```php
$server = new React\Socket\LimitingServer($server, 100);
$server->on('connection', function (React\Socket\ConnectionInterface $connection) {
    $connection->write('hello there!' . PHP_EOL);
    …
});
```

您可以传递一个` null `限制，以便不限制打开连接的数量，并一直接受新连接，直到耗尽操作系统资源(例如打开的文件句柄)。
如果您不想注意应用限制，但仍然想使用` getConnections() `方法，这很有用。

您可以配置服务器连接限制上限设置，到达上限就暂停接受新连接。在这种情况下，它将暂停底层服务器，不再处理任何新连接，因此也不再关闭任何过多的连接。

底层操作系统负责保持等待连接的积压，直到达到极限为止，此时它将开始拒绝新的连接。

当服务器低于连接限制，它将继续使用`backlog`中的连接，并在每个连接上处理未完成的数据。

这种模式对于一些设计为等待响应消息的协议(比如HTTP)可能很有用，但是对于要求立即响应的其他协议(比如交互式聊天中的“welcome”消息)就不适用了。

```php
$server = new React\Socket\LimitingServer($server, 100, true);
$server->on('connection', function (React\Socket\ConnectionInterface $connection) {
    $connection->write('hello there!' . PHP_EOL);
    …
});
```

##### getConnections()

`getConnections(): ConnectionInterface[]`方法可以用来返回一个包含所有当前活动连接的数组。

```php
foreach ($server->getConnection() as $connection) {
    $connection->write('Hi!');
}
```

## 客服端用法

### ConnectorInterface

`ConnectorInterface` 提供一个用于建立流连接的接口，例如普通的TCP / IP连接。

这是此包中定义的主要接口，并且在整个React的广阔生态系统中使用。

大多数更高级别的组件（例如HTTP，数据库或其他网络服务客户端）都接受实现此接口的实例，创建其与基础网络服务的TCP / IP连接。

通常这是通过依赖项注入完成的，便于你将该实现替换为该接口的其他实现。

该接口仅提供一种方法：

#### connect()

`connect(string $uri): PromiseInterface<ConnectionInterface,Exception>`方法可用于创建到给定远程地址的流式连接。

返回一个[Promise](1.Core-Components/Promise.md) ，
它在成功时以实现[`ConnectionInterface`](#connectioninterface)的流来实现，
或者在连接不成功时以`Exception`拒绝。 ：

```php
$connector->connect('google.com:443')->then(
    function (React\Socket\ConnectionInterface $connection) {
        // connection successfully established
    },
    function (Exception $error) {
        // failed to connect due to $error
    }
);
```

另请参阅[`ConnectionInterface`](#connectioninterface) 

返回的Promise必须以这样的方式实现：在尚待处理时可以将其取消。 
取消未决的承诺必须以`Exception`拒绝其值。 它应清理所有适用的基础资源和参考：

```php
$promise = $connector->connect($uri);

$promise->cancel();
```

### Connector

` Connector `类是这个包中的主要类，它实现了[`ConnectorInterface`](#connectorinterface)接口，并允许您创建流连接。

您可以使用此连接器创建任何类型的流连接，例如明文TCP/IP、安全TLS或本地Unix连接流。

它绑定到主事件循环，可以像这样使用:

```php
$loop = React\EventLoop\Factory::create();
$connector = new React\Socket\Connector($loop);

$connector->connect($uri)->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});

$loop->run();
```

为了创建一个明文TCP/IP连接，你可以简单地传递一个主机和端口组合:

```php
$connector->connect('www.google.com:80')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});
```

> 如果你没有在目标URI中指定一个URI方案，它将假定` tcp:// `为默认值，并建立一个明文TCP/IP连接。
  注意，TCP/IP连接需要目的地主机和端口
  像上面一样，所有其他URI组件都是可选的。

In order to create a secure TLS connection, you can use the `tls://` URI scheme
like this:
创建一个安全的TLS连接，你可以使用` tls:// ` URI方案:

```php
$connector->connect('tls://www.google.com:443')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});
```

创建一个本地的Unix域套接字连接，你可以使用` unix:// ` URI方案:

```php
$connector->connect('unix:///tmp/demo.sock')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});
```
>[`getRemoteAddress()`](#getremoteaddress)方法将返回给定给` connect() `方法的目标Unix域套接字(UDS)路径，
 包括` unix:// `方案，例如` unix:///tmp/demo.sock `。
 [`getLocalAddress()`](#getlocaladdress)方法很可能返回一个` null `值，因为这个值不适用于这里的UDS连接。

在底层，`Connector`被实现为此软件包中实现的较低层连接器的*高层门面*。 这意味着它还共享所有功能和实现细节。

如果要在更高级别的协议实现中类型提示，则应改用通用的[`ConnectorInterface`](#connectorinterface)

从`v1.4.0`开始，`Connector`类默认使用[happy eyeballs algorithm](https://en.wikipedia.org/wiki/Happy_Eyeballs)
在指定主机名时自动通过IPv4或IPv6连接。
它会自动尝试同时使用IPv4和IPv6连接(更偏向IPv6)，从而避免用户使用不完善的IPv6连接或设置所面临的常见问题。
如果你想恢复到只做一个IPv4查找并且只尝试一个IPv4连接的旧行为，你可以这样设置`Connector`:

```php
$connector = new React\Socket\Connector($loop, array(
    'happy_eyeballs' => false
));
```
同样，您还可以如下影响默认的DNS行为。

`Connector`类将尝试检测您的系统DNS设置（如果无法确定您的系统设置，并使用Google的公共DNS服务器`8.8.8.8`作为备用），
默认情况下会将所有公共主机名解析为基础IP地址。
如果您确定要使用自定义DNS服务器（例如本地DNS中继或公司范围的DNS服务器），则可以按以下方式设置`Connector`：

```php
$connector = new React\Socket\Connector($loop, array(
    'dns' => '127.0.1.1'
));

$connector->connect('localhost:80')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});
```

如果您想直接连接IP地址，不使用DNS解析器，可以这样设置`Connector`：

```php
$connector = new React\Socket\Connector($loop, array(
    'dns' => false
));

$connector->connect('127.0.0.1:80')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});
```

高级:如果你需要一个自定义的DNS `React\Dns\Resolver\ResolverInterface`实例，你也可以这样设置你的连接器:

```php
$dnsResolverFactory = new React\Dns\Resolver\Factory();
$resolver = $dnsResolverFactory->createCached('127.0.1.1', $loop);

$connector = new React\Socket\Connector($loop, array(
    'dns' => $resolver
));

$connector->connect('localhost:80')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});
```

默认情况下，`tcp://` 和 `tls://` URI方案将使用超时值，遵循` default_socket_timeout ` ini 设置(默认为60s)。
如果你想要一个自定义的超时值，你可以这样设置:

```php
$connector = new React\Socket\Connector($loop, array(
    'timeout' => 10.0
));
```

同样，如果你不想使用超时，并让操作系统处理它，你可以传递一个`bool`标志，像这样:

```php
$connector = new React\Socket\Connector($loop, array(
    'timeout' => false
));
```

默认情况下，`Connector`支持`tcp://`, `tls://` 和 `unix://` URI模式。
如果你想禁止用，你可以这样设置:

```php
// 只允许安全的TLS连接
$connector = new React\Socket\Connector($loop, array(
    'tcp' => false,
    'tls' => true,
    'unix' => false,
));

$connector->connect('tls://google.com:443')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});
```

`tcp://` 和 `tls://`也接受传递给底层连接器附加的上下文选项。

如果你想显式地传递额外的上下文选项，你可以想这样传递上下文选项数组:

```php
// 允许不安全的TLS连接
$connector = new React\Socket\Connector($loop, array(
    'tcp' => array(
        'bindto' => '192.168.0.1:0'
    ),
    'tls' => array(
        'verify_peer' => false,
        'verify_peer_name' => false
    ),
));

$connector->connect('tls://localhost:443')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});
```
默认情况下，此连接器支持TLSv1.0 +，并且不支持旧版SSLv2 / SSLv3。 
从PHP 5.6+开始，您还可以显式选择要与远程端协商的TLS版本：

```php
$connector = new React\Socket\Connector($loop, array(
    'tls' => array(
        'crypto_method' => STREAM_CRYPTO_METHOD_TLSv1_2_CLIENT
    )
));
```

>有关上下文选项的更多详细信息，请参阅有关[socket context options](https://www.php.net/manual/en/context.socket.php)
 和[SSL context options](https://www.php.net/manual/en/context.ssl.php)

高级：默认情况下，`Connector`支持`tcp://`, `tls://` 和 `unix://` URI方案。
为此，它会自动设置所需的连接器类。如果您想显式传递自定义连接器，
则可以传递一个实现`ConnectorInterface`的实例，如下所示：

```php
$dnsResolverFactory = new React\Dns\Resolver\Factory();
$resolver = $dnsResolverFactory->createCached('127.0.1.1', $loop);
$tcp = new React\Socket\HappyEyeBallsConnector($loop, new React\Socket\TcpConnector($loop), $resolver);

$tls = new React\Socket\SecureConnector($tcp, $loop);

$unix = new React\Socket\UnixConnector($loop);

$connector = new React\Socket\Connector($loop, array(
    'tcp' => $tcp,
    'tls' => $tls,
    'unix' => $unix,

    'dns' => false,
    'timeout' => false,
));

$connector->connect('google.com:80')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});
```

>`tcp：//`连接器将始终由DNS解析器包装，禁用DNS除外。 
 在这种情况下，` tcp：// `连接器将接收实际的主机名执行查找，而不是仅接收解析的IP地址。
 在内部，自动创建的` tls：// `连接器始终包装基础的`tcp：//`连接器，
 以在启用安全TLS模式之前建立基础的纯文本TCP / IP连接。 
 如果您只想将自定义基础` tcp：//`连接器仅用于安全的TLS连接，则可以像上面那样显式地传递` tls：// `连接器。
 `tcp：//`和`tls：//`连接器将始终由`TimeoutConnector`包装，禁用超时除外。

### 高级客户端使用

#### TcpConnector

`TcpConnector`类实现[`ConnectorInterface`](#connectorinterface)，并允许您创建到任何IP端口组合的纯文本TCP / IP连接：

```php
$tcpConnector = new React\Socket\TcpConnector($loop);

$tcpConnector->connect('127.0.0.1:80')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});

$loop->run();
```

参阅 [示例](https://github.com/reactphp/socket/blob/v1.6.0/examples).

挂起的连接可以通过取消其挂起的承诺来取消，如下所示：
```php
$promise = $tcpConnector->connect('127.0.0.1:80');

$promise->cancel();
```

对挂起的承诺调用`cancel()`将关闭底层套接字资源，从而取消挂起的TCP/IP连接，并拒绝生成的承诺。

您可以选择将其他[socket context options](https://www.php.net/manual/en/context.socket.php) 传递给构造函数，如下所示：

```php
$tcpConnector = new React\Socket\TcpConnector($loop, array(
    'bindto' => '192.168.0.1:0'
));
```

请注意，此类仅允许您连接到IP端口组合。
如果给定的URI无效，不包含有效的IP地址和端口或包含任何其他方案，则它将以`InvalidArgumentException`拒绝：

如果给定的URI似乎有效，但是连接失败（例如，远程主机拒绝连接等），它将以`RuntimeException`拒绝。

如果要连接到主机名-端口组合，请参见以下章节。

>高级用法:`TcpConnector`内部为每个流资源分配一个空的*context*资源。
 如果目标URI包含一个`hostname`查询参数，则它的值将用于设置TLS对等名称。
 `SecureConnector`和`DnsConnector`会使用它来验证对端名称，如果您想要自定义TLS对端名称，也可以使用它。

#### HappyEyeBallsConnector

` HappyEyeBallsConnector `类实现了[`ConnectorInterface`](#connectorinterface)，
并允许您创建到任何主机名-端口组合的纯文本TCP/IP连接。
内部实现了happy eyeballs算法[`RFC6555`](https://tools.ietf.org/html/rfc6555)
和[`RFC8305`](https://tools.ietf.org/html/rfc8305) 来支持IPv6和IPv4主机名。

它通过装饰给定的`TcpConnector`实例来实现，
首先通过DNS(如果适用的话)查找给定的域名，然后建立到已解析的目标IP地址的底层TCP/IP连接。

设置你的DNS解析器和底层TCP连接器:

```php
$dnsResolverFactory = new React\Dns\Resolver\Factory();
$dns = $dnsResolverFactory->createCached('8.8.8.8', $loop);

$dnsConnector = new React\Socket\HappyEyeBallsConnector($loop, $tcpConnector, $dns);

$dnsConnector->connect('www.google.com:80')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});

$loop->run();
```

参阅 [示例](https://github.com/reactphp/socket/blob/v1.6.0/examples).

挂起的连接可以通过取消其挂起的承诺来取消，如下所示：

```php
$promise = $dnsConnector->connect('www.google.com:80');

$promise->cancel();
```

对挂起的承诺调用` cancel() `将取消基础DNS查找和/或基础TCP/IP连接，并拒绝产生的承诺。

>高级用法:`HappyEyeBallsConnector`内部依赖于一个`Resolver(解析器)`来查找给定主机名的IP地址。
 然后，它将用这个IP的主机名替换目标URI中的主机名，并附加一个`hostname`查询参数，并将这个更新后的URI传递给底层连接器。
 Happy Eye Balls算法描述为给定的主机名查找IPv6和IPv4地址，因此该连接器发送两个DNS查找A和AAAA记录。
 然后，它使用所有IP地址(包括v6和v4)，并尝试以50ms的间隔连接到所有IP地址。在IPv6和IPv4地址之间切换。
 当连接建立时，所有其他DNS查找和连接尝试都被取消。

#### DnsConnector

`DnsConnector`类实现了[`ConnectorInterface`](#connectorinterface)，并允许您创建到任何主机名-端口组合的纯文本TCP/IP连接。

它通过装饰给定的`TcpConnector`实例来实现，首先通过DNS(如果适用的话)查找给定的域名，然后建立到已解析的目标IP地址的底层TCP/IP连接。

这样设置你的DNS解析器和底层TCP连接器:

```php
$dnsResolverFactory = new React\Dns\Resolver\Factory();
$dns = $dnsResolverFactory->createCached('8.8.8.8', $loop);

$dnsConnector = new React\Socket\DnsConnector($tcpConnector, $dns);

$dnsConnector->connect('www.google.com:80')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write('...');
    $connection->end();
});

$loop->run();
```

参阅 [示例](https://github.com/reactphp/socket/blob/v1.6.0/examples).

挂起的连接可以通过取消其挂起的承诺来取消，如下所示：

```php
$promise = $dnsConnector->connect('www.google.com:80');

$promise->cancel();
```

对挂起的承诺调用` cancel() `将取消基础DNS查找和/或基础TCP/IP连接，并拒绝产生的承诺。

>高级用法:` DnsConnector `内部依赖于` React\Dns\Resolver\ResolverInterface `来查找给定主机名的IP地址。
 然后，它将用这个IP替换目标URI中的主机名，并附加一个`hostname`查询参数，并将这个更新后的URI传递给底层连接器。
 因此，底层连接器负责创建到目标IP地址的连接，而此查询参数可用于检查原始主机名，并由`TcpConnector`用于设置TLS对等名称。
 如果显式地给出了`hostname`，则不会修改此查询参数，如果您想要自定义TLS对等端名称会，这会很有用。

#### SecureConnector

`SecureConnector`类实现了[`ConnectorInterface`](#connectorinterface)，并允许您创建到任何主机名-端口组合的安全TLS(以前称为SSL)连接。
通过装饰给定的`DnsConnector`实例来实现，首先创建一个明文TCP/IP连接，然后在此流上启用TLS加密。

```php
$secureConnector = new React\Socket\SecureConnector($dnsConnector, $loop);

$secureConnector->connect('www.google.com:443')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write("GET / HTTP/1.0\r\nHost: www.google.com\r\n\r\n");
    ...
});

$loop->run();
```

参阅 [示例](https://github.com/reactphp/socket/blob/v1.6.0/examples).

挂起的连接可以通过取消其挂起的承诺来取消，如下所示：

```php
$promise = $secureConnector->connect('www.google.com:443');

$promise->cancel();
```

对挂起的承诺调用` cancel() `将取消底层TCP/IP连接和/或SSL/TLS协商，并拒绝产生的承诺。

您可以选择传递额外的[SSL context options](https://www.php.net/manual/en/context.ssl.php) 到构造函数，像这样：

```php
$secureConnector = new React\Socket\SecureConnector($dnsConnector, $loop, array(
    'verify_peer' => false,
    'verify_peer_name' => false
));
```

默认情况下，此连接器支持TLSv1.0 +，并且不支持旧版SSLv2 / SSLv3。 从PHP 5.6+开始，您还可以显式选择要与远程端协商的TLS版本：

```php
$secureConnector = new React\Socket\SecureConnector($dnsConnector, $loop, array(
    'crypto_method' => STREAM_CRYPTO_METHOD_TLSv1_2_CLIENT
));
```

> 高级用法：` SecureConnector ` 内部依赖于在基础流资源上设置所需的*context options*。
  因此，它应该与连接器堆栈中某处的`TcpConnector`一起使用，以便它可以为每个流资源分配一个空的*context*资源并验证对等名称
  否则所有流资源都将使用单个共享的*default context*资源，可能会导致TLS对等名称不匹配错误或某些难以跟踪的竞争条件。

#### TimeoutConnector

`TimeoutConnector`类实现了[`ConnectorInterface`](#connectorinterface)，并允许您将超时处理添加到现有的连接器实例中。

通过装饰给定的 [`ConnectorInterface`](#connectorinterface)实例并启动一个计时器来完成，如果时间太长，该计时器将自动拒绝并中止连接尝试。

```php
$timeoutConnector = new React\Socket\TimeoutConnector($connector, 3.0, $loop);

$timeoutConnector->connect('google.com:80')->then(function (React\Socket\ConnectionInterface $connection) {
    // connection succeeded within 3.0 seconds
});
```

另请参阅 [示例](https://github.com/reactphp/socket/blob/v1.6.0/examples).

挂起的连接可以通过取消其挂起的承诺来取消，如下所示：

```php
$promise = $timeoutConnector->connect('google.com:80');

$promise->cancel();
```
对挂起的承诺调用` cancel() `将取消基础连接尝试，中止计时器并拒绝产生的承诺。

#### UnixConnector

` UnixConnector `类实现了[`ConnectorInterface`](#connectorinterface) ，并允许你连接到Unix域套接字(UDS)路径，如下所示:

```php
$connector = new React\Socket\UnixConnector($loop);

$connector->connect('/tmp/demo.sock')->then(function (React\Socket\ConnectionInterface $connection) {
    $connection->write("HELLO\n");
});

$loop->run();
```

连接到Unix域套接字是一个原子操作，即它的承诺将立即兑现(履行或拒绝)。
因此，对产生的promise调用` cancel() `不起作用。

>[`getRemoteAddress()`](#getremoteaddress)方法将返回与` connect() `方法相同的目标Unix域套接字(UDS)路径，
 前面加上` unix:// `方案，例如` unix:///tmp/demo.sock `。
 [`getLocalAddress()`]方法很可能返回一个` null `值，因为这个值不适用于这里的UDS连接。

#### FixedUriConnector

` FixedUriConnector `类实现了[`ConnectorInterface`](#connectorinterface)，
并装饰现有的连接器，以始终使用固定的、预先配置的URI。

这对于不支持特定uri的用户很有用，比如当你想显式连接到Unix域套接字(UDS)路径，而不是连接到高级API假设的默认地址:

```php
$connector = new React\Socket\FixedUriConnector(
    'unix:///var/run/docker.sock',
    new React\Socket\UnixConnector($loop)
);

// destination will be ignored, actually connects to Unix domain socket
$promise = $connector->connect('localhost:80');
```

## 安装

推荐的安装这个库的方法是[通过Composer](https://getcomposer.org)。
[Composer 新手?](https://getcomposer.org/doc/00-intro.md)

该项目遵循[SemVer](https://semver.org/) ，
默认安装最新支持的版本:

```bash
$ composer require react/socket:^1.6
```

有关版本升级的详细信息，请参阅[CHANGELOG](https://reactphp.org/socket/changelog.html)

该项目旨在在任何平台上运行，因此不需要任何PHP扩展，并支持通过当前*PHP 7+*和*HHVM在旧版PHP 5.3*上运行。
强烈建议对此项目使用*PHP 7+*，一是因为它的性能得到了很大的提高，二是因为旧版PHP版本需要采取以下几种解决方法。

安全TLS连接从PHP 5.6开始进行了一些重大升级，默认设置更加安全，而旧版本则需要显式设置上下文选项。
该库对这些上下文选项不承担任何责任，因此，由该库的使用者负责设置适当的上下文选项。

PHP <7.3.3（和PHP <7.2.15）存在一个bug，其中feof()可能导致TLS记录上100%的CPU使用率（阻塞）。
我们尝试通过始终消耗完整个接收缓冲区来避免TLS缓冲区中的过时数据来解决此问题。 众所周知，这可以解决CPU使用率过高的问题，
但这可能会在高吞吐量场景中导致非常大的数据块。由于网络I/O缓冲区或受影响版本上的恶意对等体，仍然可能触发错误行为，强烈建议升级PHP。

PHP < 7.1.4(和PHP < 7.0.18)在通过TLS流写入大量数据时存在一个bug。
我们试图通过将写块大小限制为8192字节来解决这个问题，这仅适用于较老的PHP版本。
这只是一种变通方法，对受影响的版本有显著的性能损失。

这个项目也支持在HHVM上运行。
注意，HHVM < 3.8不支持安全TLS连接，因为它缺乏所需的` stream_socket_enable_crypto() `函数。
因此，尝试在受影响的版本上创建安全TLS连接将返回一个被拒绝的承诺。

我们的测试套件也涉及此问题，它将跳过受影响版本的相关测试。

## 测试

要运行测试套件，首先需要克隆这个存储库，然后安装所有依赖项[通过Composer](https://getcomposer.org):

```bash
$ composer install
```

要运行测试套件，请转到项目根目录并运行:

```bash
$ php vendor/bin/phpunit
```

该测试套件还包含许多依赖稳定internet连接的功能集成测试。
如果您不想运行这些，则可以像这样跳过它们：

```bash
$ php vendor/bin/phpunit --exclude-group internet
```

## License

MIT, see [LICENSE file](https://reactphp.org/socket/license.html).
