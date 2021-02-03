# HTTP

[![Build Status](https://travis-ci.org/reactphp/http.svg?branch=master)](https://travis-ci.org/reactphp/http)

[ReactPHP](https://reactphp.org/) 的事件驱动的流HTTP客户端和服务器实现。 

HTTP库是基于ReactPHP的[` Socket `](/2.Network-Components/Socket.md)组件和
 [` EventLoop `](/1.Core-Components/EventLoop.md) 组件为HTTP客户端和服务器提供了可重用的实现。

它的客户端组件允许您并发发送任意数量的异步HTTP/HTTPS请求。

它的服务器组件允许您构建明文HTTP和安全HTTPS服务器，接受来自HTTP客户端(如web浏览器)的传入HTTP请求。

这个库提供了异步、流的方式，因此您可以在不阻塞的情况下处理多个并发HTTP请求。

**目录**

* [快速开始](#快速开始)
* [客户端用法](#客户端用法)
    * [Request methods](#request-methods)
    * [Promises](#promises)
    * [Cancellation](#cancellation)
    * [Timeouts](#timeouts)
    * [Authentication](#authentication)
    * [Redirects](#redirects)
    * [Blocking](#blocking)
    * [Concurrency](#concurrency)
    * [Streaming response](#streaming-response)
    * [Streaming request](#streaming-request)
    * [HTTP proxy](#http-proxy)
    * [SOCKS proxy](#socks-proxy)
    * [SSH proxy](#ssh-proxy)
    * [Unix domain sockets](#unix-domain-sockets)
* [服务端用法](#服务端用法)
    * [Server](#server)
    * [listen()](#listen)
    * [Server Request](#server-request)
        * [Request parameters](#request-parameters)
        * [Query parameters](#query-parameters)
        * [Request body](#request-body)
        * [Streaming incoming request](#streaming-incoming-request)
        * [Request method](#request-method)
        * [Cookie parameters](#cookie-parameters)
        * [Invalid request](#invalid-request)
    * [Server Response](#server-response)
        * [Deferred response](#deferred-response)
        * [Streaming outgoing response](#streaming-outgoing-response)
        * [Response length](#response-length)
        * [Invalid response](#invalid-response)
        * [Default response headers](#default-response-headers)
    * [Middleware](#middleware)
        * [Custom middleware](#custom-middleware)
        * [Third-Party Middleware](#third-party-middleware)
* [API](#api)
    * [Browser](#browser)
        * [get()](#get)
        * [post()](#post)
        * [head()](#head)
        * [patch()](#patch)
        * [put()](#put)
        * [delete()](#delete)
        * [request()](#request)
        * [requestStreaming()](#requeststreaming)
        * [withTimeout()](#withtimeout)
        * [withFollowRedirects()](#withfollowredirects)
        * [withRejectErrorResponse()](#withrejecterrorresponse)
        * [withBase()](#withbase)
        * [withProtocolVersion()](#withprotocolversion)
        * [withResponseBuffer()](#withresponsebuffer)
    * [React\Http\Message](#reacthttpmessage)
        * [Response](#response)
        * [ServerRequest](#serverrequest)
        * [ResponseException](#responseexception)
    * [React\Http\Middleware](#reacthttpmiddleware)
        * [StreamingRequestMiddleware](#streamingrequestmiddleware)
        * [LimitConcurrentRequestsMiddleware](#limitconcurrentrequestsmiddleware)
        * [RequestBodyBufferMiddleware](#requestbodybuffermiddleware)
        * [RequestBodyParserMiddleware](#requestbodyparsermiddleware)
* [安装](#安装)
* [测试](#测试)
* [License](#license)

## 快速开始

[安装](#安装)后，您可以使用以下代码访问HTTP网络服务器并发送一些简单的HTTP GET请求:

```php
$loop = React\EventLoop\Factory::create();
$client = new React\Http\Browser($loop);

$client->get('http://www.google.com/')->then(function (Psr\Http\Message\ResponseInterface $response) {
    var_dump($response->getHeaders(), (string)$response->getBody());
});

$loop->run();
```

这是一个HTTP服务器，它对每个请求均以 `Hello World!` 响应。

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

$loop->run();
```

参阅[示例](https://github.com/reactphp/http/blob/v1.2.0/examples/).

## 客户端用法

### Request methods

最重要的是，该项目提供了一个[`Browser`](#browser)对象，它提供了几个类似HTTP协议方法的方法:

```php
$browser->get($url, array $headers = array());
$browser->head($url, array $headers = array());
$browser->post($url, array $headers = array(), string|ReadableStreamInterface $contents = '');
$browser->delete($url, array $headers = array(), string|ReadableStreamInterface $contents = '');
$browser->put($url, array $headers = array(), string|ReadableStreamInterface $contents = '');
$browser->patch($url, array $headers = array(), string|ReadableStreamInterface $contents = '');
```

每个方法都需要一个` $url `和一些可选参数来发送HTTP请求。
每个方法名称都匹配相应的HTTP请求方法，例如[` get() `](#get)方法发送一个HTTP ` get `请求。

您可以选择传递一个附加的 `$headers` 关联数组，该数组将与此HTTP请求一起发送。 
此外，如果给出了传出请求主体且其大小已知且为非空，则每个方法将自动添加匹配的`Content-Length`请求标头。 
对于空的请求正文，如果请求方法通常需要请求正文（仅适用于POST，PUT和PATCH HTTP请求方法），
则仅包含`Content-Length:0`请求标头。

如果您使用的是[streaming request body](#streaming-request)，默认使用`Transfer-Encoding:chunked`，
除非您明确传递匹配的`Content-Length`请求标头。

另请参阅[streaming request](#streaming-request)。

默认情况下上述所有方法使用HTTP / 1.1协议版本发送请求。 
如果要显式使用旧版HTTP / 1.0协议版本，则可以使用[`withProtocolVersion()`](#withprotocolversion)方法。
如果要使用任何其他甚至自定义的HTTP请求方法，则可以使用[`request()`](#request)方法。

以上每种方法都支持异步操作，并且使用[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface)
*实现*或*`Exception`拒绝*。
有关更多详细信息，请参见以下有关[Promises](/1.Core-Components/Promises.md)的章节。

### Promises

发送请求是异步的(非阻塞的)，所以实际上可以并发发送多个请求。

`Browser`将用[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface)
消息响应每个请求，但不能保证顺序。

发送请求使用[Promises](/1.Core-Components/Promises.md) 的接口，
该接口可以轻松响应HTTP请求完成时（即成功完成或因错误而被拒绝）:

```php
$browser->get($url)->then(
    function (Psr\Http\Message\ResponseInterface $response) {
        var_dump('Response received', $response);
    },
    function (Exception $error) {
        var_dump('There was an error', $error->getMessage());
    }
);
```

如果您觉得这很奇怪，也可以使用更传统的 [blocking API](#blocking).

请记住，使用完整的响应消息来解决`Promise`意味着整个响应主体都必须保留在内存中。

这很容易上手，并且对于较小的响应（例如常见的HTML页面或RESTful或JSON API请求）相当有效。

您可能还需要研究 [streaming API](#streaming-response):

* 如果你正在处理大量的并发请求(100+)或
* 如果你想在数据块发生时处理单个数据块(而不需要等待完整的响应体)或
* 如果你期望一个较大的响应体大小(例如下载二进制文件时，1 MiB或更多)或
* 如果您不确定响应正文的大小(在访问任意远程HTTP端点时，安全第一，而且响应正文的大小事先是未知的)。

### Cancellation

返回的承诺以这样一种方式实现:当它仍然挂起时，它可以被取消。
取消挂起的承诺将以异常拒绝其值，并清除所有底层资源。

```php
$promise = $browser->get($url);

$loop->addTimer(2.0, function () use ($promise) {
    $promise->cancel();
});
```

### Timeouts

该库使用非常有效的HTTP实现，因此大多数HTTP请求通常应在几毫秒内完成。 
但是，当通过不可靠的网络(Internet)发送HTTP请求时，可能会发生许多错误，并且一段时间后可能导致请求失败。 
这样，该库将PHP的`default_socket_timeout`设置（默认为60s）视为发送外发HTTP请求和等待成功响应的超时，
否则将取消待处理的请求并拒绝其值（带有`Exception`）。

请注意，此超时值包括创建基础传输连接，发送HTTP请求，
接收HTTP响应标头及其完整的响应主体以及任何最终的[redirects](#redirects) 。 
另请参阅下面的[redirects](#redirects) ，以配置要遵循的重定向数（或完全禁用后续重定向），
还可以在下面的[streaming](#streaming-response)中，不考虑将较大的响应主体纳入此超时范围。

您可以使用[`withTimeout()` method](#withtimeout)以秒为单位传递自定义超时值，如下所示:

```php
$browser = $browser->withTimeout(10.0);

$browser->get($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    // response received within 10 seconds maximum
    var_dump($response->getHeaders());
});
```

同样，您可以使用bool`false` 不超时，也可以使用bool`true`值来恢复默认处理。
有关更多详细信息，请参阅[`withTimeout()`](#withtimeout) 

如果您使用的是[streaming response body](#streaming-response) ，
则接收响应主体流所花费的时间将不包含在超时中。
这使您可以长时间保持此传入流的打开状态，例如在下载非常大的流或通过长期连接流式传输数据时。

如果您使用的是[streaming request body](#streaming-request) ，
则发送请求主体流所花费的时间将不包含在超时中。 
这使您可以长时间保持此传出流的打开状态，例如在上传非常大的流时。

请注意，此超时处理适用于更高级别的HTTP层。 较低的层（例如套接字和DNS）也可以应用（不同的）超时值。 
具体而言，基础套接字连接使用相同的`default_socket_timeout`设置来建立基础传输连接。
要控制此连接超时行为，您可以像这样[注入自定义`Connector`](#browser):

```php
$browser = new React\Http\Browser(
    $loop,
    new React\Socket\Connector(
        $loop,
        array(
            'timeout' => 5
        )
    )
);
```

### Authentication

该库使用 `Authorization: Basic …` 请求标头
支持[HTTP Basic Authentication](https://en.wikipedia.org/wiki/Basic_access_authentication) 
或允许您设置显式的`Authorization`请求标头。

默认情况下，该库不包含传出的`Authorization`请求标头。 
如果服务器需要身份验证，则可能会返回401（未经授权）状态代码，
默认情况下该状态代码将拒绝请求（另请参阅下面的[`withRejectErrorResponse()` method](#withrejecterrorresponse)）。

为了传递身份验证详细信息，您可以简单地将用户名和密码作为请求URL的一部分传递，如下所示:

```php
$promise = $browser->get('https://user:pass@example.com/api');
```

请注意，身份验证详细信息中的特殊字符必须进行编码，另请参阅 [`rawurlencode()`](https://www.php.net/manual/en/function.rawurlencode.php)
此示例将使用传出的`Authorization: Basic …`请求标头自动传递base64编码的身份验证详细信息。 
如果您要与之通信的HTTP端点需要任何其他身份验证方案，则也可以显式传递此标头。
使用(RESTful)HTTP API使用OAuth访问令牌或JSON Web令牌(JWT) :

```php
$token = 'abc123';

$promise = $browser->get(
    'https://example.com/api',
    array(
        'Authorization' => 'Bearer ' . $token
    )
);
```
在执行重定向时，默认情况下，`Authorization`请求头永远不会发送到任何远程主机。
当遵循`Location`响应头包含身份验证详细信息的重定向时，将为以下请求发送这些详细信息。
另请参阅下面的[redirects](#redirects) 

### Redirects

默认情况下，此库遵循任何重定向，并使用来自远程服务器的`Location`响应头遵守`3xx`（重定向）状态代码。
这个承诺将通过重定向链的最后一个响应来实现。

```php
$browser->get($url, $headers)->then(function (Psr\Http\Message\ResponseInterface $response) {
    // the final response will end up here
    var_dump($response->getHeaders());
});
```

任何重定向的请求都将遵循原始请求的语义，并将包含与原始请求相同的请求头，但下面列出的请求头除外。

如果原始请求包含请求正文，则此请求正文永远不会传递给重定向的请求。
因此，每个重定向的请求都将删除所有`Content-Length`和`Content-Type`请求头。

如果原始请求使用带有`Authorization`请求标头的HTTP身份验证，
则仅当重定向URL使用同一主机时，此请求标头才作为重定向请求的一部分传递。 
换句话说，由于可能的隐私/安全问题，`Authorization`请求标头将不会转发给其他外部主机。 
当遵循重定向时，其中`Location`响应头包含身份验证详细信息时，将为后续请求发送这些详细信息。

您可以使用[`withFollowRedirects()`](#withfollowredirects) 方法来控制要跟踪的最大重定向数，
或按原样返回任何重定向响应，并应用如下自定义重定向逻辑:

```php
$browser = $browser->withFollowRedirects(false);

$browser->get($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    // any redirects will now end up here
    var_dump($response->getHeaders());
});
```
有关详细信息，请参阅 [`withFollowRedirects()`](#withfollowredirects) 

### Blocking

如上所示，默认情况下，该库为您提供了功能强大的异步API。

但是，如果您想要将其集成到传统的阻塞环境中，
您应该考虑使用[clue/reactphp-block](https://github.com/clue/reactphp-block)

生成的阻塞代码可能如下所示:

```php
use Clue\React\Block;

$loop = React\EventLoop\Factory::create();
$browser = new React\Http\Browser($loop);

$promise = $browser->get('http://example.com/');

try {
    $response = Block\await($promise, $loop);
    // response successfully received
} catch (Exception $e) {
    // an error occured while performing the request
}
```

同样，您也可以同时处理多个请求并等待`Response`对象数组:

```php
$promises = array(
    $browser->get('http://example.com/'),
    $browser->get('http://www.example.org/'),
);

$responses = Block\awaitAll($promises, $loop);
```

有关更多详细信息，请参阅 [clue/reactphp-block](https://github.com/clue/reactphp-block#readme)

记住上面关于在内存中缓冲整个响应消息的注释。你可能还会看到以下章节中的[streaming API](#streaming-response)

### Concurrency

如上所示，这个库为您提供了一个强大的异步API。能够一次发送大量请求是这个项目的核心特性之一。
例如，您可以轻松地同时并发发送100个SQL请求查询处理。

记住，能力越大，责任越大。发送过多的请求可能会占用你过多的资源，
或者如果远程端看到来自您这边的请求数量不合理，甚至可能会禁止您。

```php
// 如果数组包含很多元素，请特别注意
foreach ($urls as $url) {
    $browser->get($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
        var_dump($response->getHeaders());
    });
}
```

因此，通常建议将发送方的并发限制为合理的值。 通常使用一个很小的限制，
因为一次执行十多个事情可能很容易使接收端不堪重负。
 您可以将[clue/reactphp-mq](https://github.com/clue/reactphp-mq) 用作轻量级的内存队列，
 以同时并行执行很多（但不是太多）的事情:
 
```php
// 将浏览器包装在一个队列对象中，该对象一次执行不超过10个操作
$q = new Clue\React\Mq\Queue(10, null, function ($url) use ($browser) {
    return $browser->get($url);
});

foreach ($urls as $url) {
    $q($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
        var_dump($response->getHeaders());
    });
}
```

超过并发性限制的其他请求将自动进入队列，直到其中一个挂起的请求完成。
这与现有的[Promise-based API](#promises) 很好地集成在一起。 
有关更多详细信息，请参考[clue/reactphp-mq](https://github.com/clue/reactphp-mq)

对于数以千计的未完成请求，此内存方法相当有效。 
如果要处理很大的输入列表（请考虑CSV或NDJSON文件中的数百万行），则可能需要考虑使用流方法。
了解更多细节，请参阅[clue/reactphp-flux](https://github.com/clue/reactphp-flux)

### Streaming response

以上所有示例均假设您要将整个响应主体存储在内存中。
这很容易上手，并且对于较小的响应也相当有效。

但是，在某些情况下，使用流方法通常是一个更好的方法，只需要占用一小块内存:

* 如果你正在处理大量的并发请求(100+)或
* 如果你想在数据块发生时处理单个数据块(而不需要等待完整的响应体)或
* 如果你期望一个较大的响应体大小(例如下载二进制文件时，1 MiB或更多)或
* 如果您不确定响应正文的大小(在访问任意远程HTTP端点时，安全第一，而且响应正文的大小事先是未知的)。

您可以使用[`requestStreaming()`](#requeststreaming) 方法发送任意HTTP请求并接收流式响应。 
它使用相同的HTTP消息API，但不将响应主体缓存在内存中。 它仅在接收到数据时以小块形式处理响应主体，
并通过[ReactPHP's Stream API](/1.Core-Components/Stream.md) 转发此数据。 
这适用于（任意数量）任意大小的响应。

这意味着它使用一个正常的
[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface) 进行解析，
可以像往常一样使用它来访问响应消息参数。
你可以像往常一样访问消息体，但是它现在也实现了
[ReactPHP's `ReadableStreamInterface`](https://github.com/reactphp/stream#readablestreaminterface)
以及部分[PSR-7 `StreamInterface`](https://www.php-fig.org/psr/psr-7/#34-psrhttpmessagestreaminterface)

```php
$browser->requestStreaming('GET', $url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    $body = $response->getBody();
    assert($body instanceof Psr\Http\Message\StreamInterface);
    assert($body instanceof React\Stream\ReadableStreamInterface);

    $body->on('data', function ($chunk) {
        echo $chunk;
    });

    $body->on('error', function (Exception $error) {
        echo 'Error: ' . $error->getMessage() . PHP_EOL;
    });

    $body->on('close', function () {
        echo '[DONE]' . PHP_EOL;
    });
});
```
另请参阅[stream download benchmark example](https://github.com/reactphp/http/blob/v1.2.0/examples/91-client-benchmark-download.php)
和[stream forwarding example](ehttps://github.com/reactphp/http/blob/v1.2.0/examples/21-client-request-streaming-to-stdout.php)

你可以在消息体上调用以下方法:

```php
$body->on($event, $callback);
$body->eof();
$body->isReadable();
$body->pipe(React\Stream\WritableStreamInterface $dest, array $options = array());
$body->close();
$body->pause();
$body->resume();
```

由于消息体处于流状态，调用以下方法没有多大意义:

```php
$body->__toString(); // ''
$body->detach(); // throws BadMethodCallException
$body->getSize(); // null
$body->tell(); // throws BadMethodCallException
$body->isSeekable(); // false
$body->seek(); // throws BadMethodCallException
$body->rewind(); // throws BadMethodCallException
$body->isWritable(); // false
$body->write(); // throws BadMethodCallException
$body->read(); // throws BadMethodCallException
$body->getContents(); // throws BadMethodCallException
```

请注意，使用流传输时，[timeouts](#timeouts)的应用方式略有不同。

在流传输模式下，超时值包括创建基础传输连接，发送HTTP请求，
接收HTTP响应标头以及任何可能的[redirects](#redirects) 。
特别是，超时值不考虑接收（可能很大）的响应主体。

如果要将流响应集成到更高级别的API中，则使用由Stream对象解析的Promise对象通常很不方便。
考虑考虑也使用[react/promise-stream](/4.Utility-Components/PromiseStream.md)

生成的流代码可能看起来像这样:

```php
use React\Promise\Stream;

function download(Browser $browser, string $url): React\Stream\ReadableStreamInterface {
    return Stream\unwrapReadable(
        $browser->requestStreaming('GET', $url)->then(function (Psr\Http\Message\ResponseInterface $response) {
            return $response->getBody();
        })
    );
}

$stream = download($browser, $url);
$stream->on('data', function ($data) {
    echo $data;
});
```

更多细节请参见[`requestStreaming()`](#requeststreaming) 方法。

### Streaming request

除了流化响应体之外，您还可以流化请求体。
如果你想发送较大的POST请求(上传文件等)或同时处理许多传出的流，这将非常有用。
不需要将body作为字符串传递，你可以简单地将一个实现
[ReactPHP's `ReadableStreamInterface`](/1.Core-Components/Stream.md#readablestreaminterface)
的实例传递给[request methods](#request-methods) ，如下所示:

```php
$browser->post($url, array(), $stream)->then(function (Psr\Http\Message\ResponseInterface $response) {
    echo 'Successfully sent.';
});
```

如果你正在使用一个流请求体(` React\Stream\ReadableStreamInterface `)，
它将默认使用` Transfer-Encoding: chunked `，
或者你必须显式地传入匹配的` Content-Length `请求头，如下所示:

```php
$body = new React\Stream\ThroughStream();
$loop->addTimer(1.0, function () use ($body) {
    $body->end("hello world");
});

$browser->post($url, array('Content-Length' => '11'), $body);
```

如果流式处理请求主体发出`error`事件或显式关闭而未首先发出成功的`end`事件，则请求将自动关闭并被拒绝。

### HTTP proxy

您还可以通过添加依赖项 [clue/reactphp-http-proxy](https://github.com/clue/reactphp-http-proxy) 
来建立HTTP连接代理服务器外发连接。

HTTP连接代理服务器（通常也称为`HTTPS代理`或`SSL代理`）通常用于通过中介（`proxy`）对HTTPS流量进行隧道传输，
以隐藏源地址（匿名）或绕过地址阻塞（地理阻塞）。
虽然许多（公共）HTTP连接代理服务器通常仅将此限制为HTTPS端口`443`，
但从技术上讲，它可以用于隧道任何基于TCP/IP的协议，例如纯HTTP和TLS加密的HTTPS。

```php
$proxy = new Clue\React\HttpProxy\ProxyConnector(
    'http://127.0.0.1:8080',
    new React\Socket\Connector($loop)
);

$connector = new React\Socket\Connector($loop, array(
    'tcp' => $proxy,
    'dns' => false
));

$browser = new React\Http\Browser($loop, $connector);
```

另请参阅[HTTP连接代理示例](https://github.com/reactphp/http/blob/v1.2.0/examples/11-client-http-connect-proxy.php)

### SOCKS proxy

您还可以通过添加依赖项 [clue/reactphp-socks](https://github.com/clue/reactphp-socks) 来建立SOCKS代理服务器外发连接。

SOCKS代理协议家族（SOCKS5、SOCKS4和SOCKS4a）通常用于通过中介（`proxy`）对HTTP（S）流量进行隧道传输，
以隐藏源地址（匿名性）或绕过地址阻塞（地理阻塞）。
虽然许多（公共）SOCKS代理服务器通常仅将此限制为HTTP（S）端口`80`和`443`，但从技术上讲，它可以用于任何基于TCP/IP的协议隧道传输。

```php
$proxy = new Clue\React\Socks\Client(
    'socks://127.0.0.1:1080',
    new React\Socket\Connector($loop)
);

$connector = new React\Socket\Connector($loop, array(
    'tcp' => $proxy,
    'dns' => false
));

$browser = new React\Http\Browser($loop, $connector);
```

另请参阅[SOCKS连接代理示例](https://github.com/reactphp/http/blob/v1.2.0/examples/12-client-socks-proxy.php).

### SSH proxy

通过添加依赖项[clue/reactphp-ssh-proxy](https://github.com/clue/reactphp-ssh-proxy) 
建立通过SSH服务器外发连接。

[Secure Shell (SSH)](https://en.wikipedia.org/wiki/Secure_Shell)
是一种安全的网络协议，最常用来访问远程服务器上的登录Shell。
它的架构允许它在单个连接上使用多个安全通道。此外，
这还可以用于创建“SSH隧道”，该隧道通常用于通过中介(“代理”)对HTTP(S)流量进行隧道，
以隐藏源地址(匿名)或绕过地址阻塞(地理阻塞)。
这可以用于隧道任何基于TCP/ ip的协议(HTTP, SMTP, IMAP等)，
允许您访问本地服务，否则无法从外部访问(防火墙后的数据库)，因此也可以用于纯HTTP和tls加密的HTTPS。

```php
$proxy = new Clue\React\SshProxy\SshSocksConnector('me@localhost:22', $loop);

$connector = new React\Socket\Connector($loop, array(
    'tcp' => $proxy,
    'dns' => false
));

$browser = new React\Http\Browser($loop, $connector);
```

请参阅 [SSH代理示例](https://github.com/reactphp/http/blob/v1.2.0/examples/13-client-ssh-proxy.php)

### Unix domain sockets

默认情况下，这个库分别支持`http://`和`https://`URL模式的明文TCP/IP传输和安全TLS连接。
当显式配置时，此库还支持Unix域套接字(UDS)。

为了使用UDS路径，您必须显式配置连接器以覆盖目标URL，这样请求URL中给出的主机名将不再用于建立连接:

```php
$connector = new React\Socket\FixedUriConnector(
    'unix:///var/run/docker.sock',
    new React\Socket\UnixConnector($loop)
);

$browser = new Browser($loop, $connector);

$client->get('http://localhost/info')->then(function (Psr\Http\Message\ResponseInterface $response) {
    var_dump($response->getHeaders(), (string)$response->getBody());
});
```

请参阅[Unix域套接字(UDS)示例](https://github.com/reactphp/http/blob/v1.2.0/examples/14-client-unix-domain-sockets.php)


## 服务端用法

### Server

`React\Http\Server`类负责处理传入的连接，然后处理每个传入的Http请求。

当接收到一个完整的HTTP请求时，它将调用给定的请求处理程序函数。
这个请求处理函数需要传递给构造函数，并将被相应的 [request](#server-request) 对象调用，
并期望返回一个 [response](#server-response) 对象:

```php
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    return new React\Http\Message\Response(
        200,
        array(
            'Content-Type' => 'text/plain'
        ),
        "Hello World!\n"
    );
});
```

每个传入的HTTP请求消息始终由
[PSR-7`ServerRequestInterface`](https://www.php-fig.org/psr/psr-7/#321-psrhttpmessageserverrequestinterface) 表示，
有关详细信息，请参见下面的 [request](#server-request) 章节。

每个传出的HTTP响应消息始终由
[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface) 表示,
有关更多详细信息，请参见下面的 [response](#server-response) 章节。

需要通过[' listen() '](#listen)方法将` Server `附加到
[' React\Socket\ServerInterface '](/2.Network-Components/Socket.md#serverinterface)
的实例上开始监听任何传入的连接，这将在下一章中描述。
在最简单的形式中，你可以将它附加到
[' React\Socket\Server '](/2.Network-Components/Socket.md#server) ，
以便启动一个明文HTTP服务器，如下所示:

```php
$server = new React\Http\Server($loop, $handler);

$socket = new React\Socket\Server('0.0.0.0:8080', $loop);
$server->listen($socket);
```

更多细节请参阅[' listen() '](#listen)方法和
[hello world server 示例](https://github.com/reactphp/http/blob/v1.2.0/examples/51-server-hello-world.php)

默认情况下，` Server `会在内存中缓冲并解析完整的传入HTTP请求。
当接收到完整的请求头和请求体时，它将调用给定的请求处理程序函数。
这意味着传递给请求处理函数的[request](#server-request)对象将与PSR-7 (http-message)完全兼容。
这为80%的用例提供了正常的默认值，并且推荐使用这个库，除非您确定自己知道自己在做什么。

另一方面，将完整的HTTP请求缓冲到内存中，直到您的请求处理程序函数能够处理它们，
这意味着该类必须采用许多限制，以避免消耗过多的内存。为了获得更高级的配置，
使用[` php.ini `](https://www.php.net/manual/en/ini.core.php) 中的设置默认值。
这是一个PHP设置列表，这个类使用它们各自的默认值:

```
memory_limit 128M
post_max_size 8M // capped at 64K

enable_post_data_reading 1
max_input_nesting_level 64
max_input_vars 1000

file_uploads 1
upload_max_filesize 2M
max_file_uploads 20
```

特别是，`post_max_size`设置限制了单个HTTP请求在缓冲其请求体时允许消耗多少内存。
这需要加以限制，因为服务器可以并发处理大量请求，否则服务器可能会潜在地消耗大量内存。
为了在默认情况下支持更高的并发性，这个值被限制为`64K`。
如果你指定一个更高的值，默认情况下它只允许`64K`。

如果一个请求超过了这个限制，它的请求主体将被忽略，它将像一个没有请求主体的请求一样被处理。
有关覆盖此设置的显式配置，请参见下面的内容。

默认情况下，该类会尽量避免为缓冲多个并发的HTTP请求而消耗超过`memory_limit`的一半。
因此，使用上面的默认设置最大`128M`，它将尝试不超过`64M`来缓冲多个并发的HTTP请求。
因此，在上述默认设置下，它将把HTTP请求的并发性限制为`1024`。

必须为您的PHP ini设置分配合理的值。
通常建议不支持使用大型HTTP请求正文（例如，大文件上传）来缓冲传入的HTTP请求。 
如果要增加此缓冲区的大小，则还必须增加总内存限制，以允许更多并发请求（设置`memory_limit 512M`或更多）或显式限制并发性。

为了覆盖上面的默认缓冲，你可以显式地配置`Server`。
你可以使用[`LimitConcurrentRequestsMiddleware`](#limitconcurrentrequestsmiddleware)
和[`RequestBodyBufferMiddleware`](#requestbodybuffermiddleware)(见下面)来显式地配置一次可以处理的请求的总数:

```php
$server = new React\Http\Server(
    $loop,
    new React\Http\Middleware\StreamingRequestMiddleware(),
    new React\Http\Middleware\LimitConcurrentRequestsMiddleware(100), // 100个并发缓冲处理程序
    new React\Http\Middleware\RequestBodyBufferMiddleware(2 * 1024 * 1024), // 每请求2 MiB
    new React\Http\Middleware\RequestBodyParserMiddleware(),
    $handler
);
```

在这个例子中，我们允许一次最多处理100个并发请求，每个请求最多可以缓冲`2M`。
这意味着您可能必须为传入的请求体缓冲区保留最大的`200M`内存。
相应地，您需要调整`memory_limit` ini设置，
以允许这些缓冲区加上您实际的应用程序逻辑内存需求(考虑`512M`或更多)。

>当内部没有给出[`StreamingRequestMiddleware`](#streamingrequestmiddleware) 时，
 此类会自动分配这些中间件处理程序。因此，可以使用此示例覆盖所有默认设置以实现自定义限制。

作为在内存中缓冲整个请求主体的一种替代方法，您还可以使用流传输方法，其中仅一小部分数据必须保留在内存中:

```php
$server = new React\Http\Server(
    $loop,
    new React\Http\Middleware\StreamingRequestMiddleware(),
    $handler
);
```

在这个例子中，接收到HTTP请求头之后，接收到可能更大的HTTP请求体之前，将调用request handler函数。
这意味着传递给请求处理函数的[request](#server-request)可能与PSR-7不完全兼容。
这是专门为帮助处理更高级的用例而设计的，在这些用例中，您希望完全控制使用传入的HTTP请求主体和并发设置。
有关更多详细信息，请参阅下面的[流式传入请求](#streaming-incoming-request)

### listen()

使用 `listen(React\Socket\ServerInterface $socket): void` 方法侦听给定Socket服务器实例上的HTTP请求。

给定的[`React\Socket\ServerInterface`](/2.Network-Components/Socket.md#serverinterface) 负责发出基础的流连接。 
需要将此HTTP服务器连接到该服务器，以便处理任何连接并将传入的流数据作为传入的HTTP请求消息进行处理。 
您可以将其以最常见的形式附加到[`React\Socket\Server`](/2.Network-Components/Socket.md#server)
以便启动纯文本HTTP服务器:

```php
$server = new React\Http\Server($loop, $handler);

$socket = new React\Socket\Server('0.0.0.0:8080', $loop);
$server->listen($socket);
```

另请参阅[hello world server示例](https://github.com/reactphp/http/blob/v1.2.0/examples/51-server-hello-world.php)

此示例将开始在所有接口（公开）上的备用HTTP端口`8080`上侦听HTTP请求。
作为替代方案，使用反向代理并让此HTTP服务器仅通过使用侦听地址`127.0.0.1:8080`来在localhost（环回）接口上侦听是常见做法。
这样，您可以将应用程序托管在默认的HTTP端口`80`上，并且仅将特定的请求路由到此HTTP服务器。

同样，通常建议使用反向代理设置来接受默认HTTPS端口`443`（TLS终止）上的安全HTTPS请求，并且仅将纯文本请求路由到此HTTP服务器。 
另外，您也可以使用安全的TLS侦听地址将此HTTP服务器附加到[`React\Socket\Server`](/2.Network-Components/Socket.md#server) 上，
从而对此HTTP服务器接受安全的HTTPS请求，证书文件和可选的`passphrase`，如下所示:

```php
$server = new React\Http\Server($loop, $handler);

$socket = new React\Socket\Server('tls://0.0.0.0:8443', $loop, array(
    'local_cert' => __DIR__ . '/localhost.pem'
));
$server->listen($socket);
```

另请参阅[hello world HTTPS示例](https://github.com/reactphp/http/blob/v1.2.0/examples/61-server-hello-world-https.php)

### Server Request

如上所示，[`Server`](#server) 类负责处理传入的连接，然后处理每个传入的HTTP请求。

一旦请求被服务端接收，请求对象将被处理。
这个请求对象实现了
[PSR-7 `ServerRequestInterface`](https://www.php-fig.org/psr/psr-7/#321-psrhttpmessageserverrequestinterface) ，
这反过来扩展了
[PSR-7 `RequestInterface`](https://www.php-fig.org/psr/psr-7/#32-psrhttpmessagerequestinterface) ，
并将其传递给回调函数。

 ```php 
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    $body = "The method of the request is: " . $request->getMethod();
    $body .= "The requested path is: " . $request->getUri()->getPath();

    return new React\Http\Message\Response(
        200,
        array(
            'Content-Type' => 'text/plain'
        ),
        $body
    );
});
```

有关请求对象的更多细节，请参阅
[PSR-7 ' ServerRequestInterface '](https://www.php-fig.org/psr/psr-7/#321-psrhttpmessageserverrequestinterface)
和[PSR-7 ' RequestInterface '](https://www.php-fig.org/psr/psr-7/#32-psrhttpmessagerequestinterface)
的文档。

#### Request parameters

`getServerParams(): mixed[]`方法可以用于获取类似于`$_SERVER`变量的服务器端参数。
目前有以下参数:

* `REMOTE_ADDR`
  请求发送者的IP地址
* `REMOTE_PORT`
  请求发送者的端口
* `SERVER_ADDR`
  服务器IP地址
* `SERVER_PORT`
  服务器的端口
* `REQUEST_TIME`
  当收到完整的请求头时的Unix时间戳(整数)，类似于` time() `
* `REQUEST_TIME_FLOAT`
  当收到完整的请求头时的Unix时间戳， 类似于` microtime(true) `
* `HTTPS`
  如果请求使用HTTPS，则设置为`on`，否则不会设置

```php 
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    $body = "Your IP is: " . $request->getServerParams()['REMOTE_ADDR'];

    return new React\Http\Message\Response(
        200,
        array(
            'Content-Type' => 'text/plain'
        ),
        $body
    );
});
```

请参阅 [whatsmyip server示例](https://github.com/reactphp/http/blob/v1.2.0/examples/53-server-whatsmyip.php).

>高级:注意，如果您正在侦听Unix域套接字(UDS)路径，则不会设置地址参数，因为该协议缺乏主机/端口的概念。

#### Query parameters

`getQueryParams(): array` 方法可以用于获取`$_GET`变量的查询参数。

```php
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    $queryParams = $request->getQueryParams();

    $body = 'The query parameter "foo" is not set. Click the following link ';
    $body .= '<a href="/?foo=bar">to use query parameter in your request</a>';

    if (isset($queryParams['foo'])) {
        $body = 'The value of "foo" is: ' . htmlspecialchars($queryParams['foo']);
    }

    return new React\Http\Message\Response(
        200,
        array(
            'Content-Type' => 'text/html'
        ),
        $body
    );
});
```

上面示例中的响应将返回一个带有链接的响应体。URL包含查询参数` foo `，值为` bar `。
使用 [`htmlentities`](https://www.php.net/manual/en/function.htmlentities.php) ，
如本例中防止[跨站点脚本(缩写为XSS)](https://en.wikipedia.org/wiki/Cross-site_scripting)

请参阅 [server query parameters示例](https://github.com/reactphp/http/blob/v1.2.0/examples/54-server-query-parameter.php).

#### Request body

默认情况下，[`Server`](#server)将在内存中缓冲并解析完整的请求体。这意味着给定的请求对象包括已解析的请求主体和上传的文件。

>作为默认缓冲逻辑的替代，你也可以使用
 [`StreamingRequestMiddleware`](#streamingrequestmiddleware) 。
 跳转到下一章，学习更多关于如何处理
 [流式传入请求](#streaming-incoming-request)。

如上所示，每个传入的HTTP请求总是由
[PSR-7 `ServerRequestInterface`](https://www.php-fig.org/psr/psr-7/#321-psrhttpmessageserverrequestinterface) 表示。
这个接口提供了几个在处理传入请求体时很有用的方法，如下所示。

`getParsedBody(): null|array|object` 方法可以用来获取解析的请求体，
类似于[PHP的`$_POST`变量](https://www.php.net/manual/en/reserved.variables.post.php).
如果不能解析请求体，这个方法可能返回一个(可能是嵌套的)包含所有体参数的数组结构，或者返回一个` null `值。
默认情况下，该方法只会返回使用
`Content-Type: application/x-www-form-urlencoded`或`Content-Type: multipart/form-data`
请求头(通常用于HTML表单提交数据的`POST`请求)的解析数据。

```php
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    $name = $request->getParsedBody()['name'] ?? 'anonymous';

    return new React\Http\Message\Response(
        200,
        array(),
        "Hello $name!\n"
    );
});
```

更多细节请参阅 [form upload示例](https://github.com/reactphp/http/blob/v1.2.0/examples/62-server-form-upload.php)

`getBody(): StreamInterface` 方法可以用来从请求体中获取原始数据，类似于
[PHP's `php://input` stream](https://www.php.net/manual/en/wrappers.php.php#wrappers.php.input).
该方法返回由
[PSR-7 `StreamInterface`](https://www.php-fig.org/psr/psr-7/#34-psrhttpmessagestreaminterface)
表示的请求体的实例。
这是特别有用,当使用自定义请求主体,否则不会被解析在默认情况下,
比如一个JSON(`Content-Type: application/json`)或XML(`Content-Type: application/xml`)
请求主体(通常用于`POST`, `PUT` 或 `PATCH`请求基于JSON或RESTful/RESTish APIs)。

```php
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    $data = json_decode((string)$request->getBody());
    $name = $data->name ?? 'anonymous';

    return new React\Http\Message\Response(
        200,
        array('Content-Type' => 'application/json'),
        json_encode(['message' => "Hello $name!"])
    );
});
```

更多细节请参阅 [JSON API 服务示例](https://github.com/reactphp/http/blob/v1.2.0/examples/59-server-json-api.php)

`getUploadedFiles(): array`方法可以用于在这个请求中获取上传的文件，类似于
[PHP的` $_FILES `变量](https://www.php.net/manual/en/reserved.variables.files.php)
这个方法返回一个(可能是嵌套的)包含所有文件上传的数组结构，每个文件都由
[PSR-7 ` UploadedFileInterface `](https://www.php-fig.org/psr/psr-7/#36-psrhttpmessageuploadedfileinterface) 表示。
这个数组只有在使用` Content-Type: multipart/form-data `请求头(通常用于HTML文件上传的` POST `请求)时才会被填充。

```php
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    $files = $request->getUploadedFiles();
    $name = isset($files['avatar']) ? $files['avatar']->getClientFilename() : 'nothing';

    return new React\Http\Message\Response(
        200,
        array(),
        "Uploaded $name\n"
    );
});
```

更多细节请参阅  [form upload服务示例](https://github.com/reactphp/http/blob/v1.2.0/examples/62-server-form-upload.php) 

`getSize(): ?int`方法可以用来获取请求体的大小，类似于PHP的`$_SERVER['CONTENT_LENGTH']`变量。
此方法返回以字节数衡量的请求体的完整大小，该大小由消息边界定义。
如果请求消息不包含请求正文(如简单的`GET`请求)，则该值可以为`0`。
这个方法操作缓冲的请求体，即请求体的大小总是已知的，
即使请求没有指定` Content-Length `请求头或当使用` Transfer-Encoding: chunked ` HTTP/1.1请求。

>注意:`Server`会自动处理带有附加的`Expect: 100-continue`请求头的请求。
 当HTTP/1.1客户端想要发送更大的请求体时，他们可能只发送带有附加的`Expect: 100-continue`请求头的请求头，
 并在发送实际的(更大的)消息体之前等待。在这种情况下，服务器将自动向客户端发送一个中间的`HTTP/1.1 100 Continue`响应。
 这样可以确保您将收到请求正文，而不会出现预期的延迟。
#### Streaming incoming request

如果你使用高级的[`StreamingRequestMiddleware`](#streamingrequestmiddleware) ，请求对象将在收到请求头后被处理。
这意味着这种情况的发生与接收(可能更大的)请求体无关(例如*before*)。

>注意，这是被认为是高级使用的非标准行为。跳到上一章了解更多关于如何处理缓冲的 [request body](#request-body)

虽然这在PHP生态系统中可能并不常见，但实际上这是一种非常强大的方法，它为您提供了一些其他方法无法实现的优势:

* 在接收到一个较大的请求体之前对请求进行响应，例如拒绝一个未经认证的请求或一个超过允许的消息长度的请求(文件上传)。
* 在请求正文的其余部分到达之前，或者发送者正在缓慢地流数据时，开始处理请求正文的部分。
* 处理一个大的请求体而不需要在内存中缓冲任何东西，例如接受一个巨大的文件上传或可能无限的请求体流。

`getBody(): StreamInterface`方法可以用于访问请求体流。
在流模式下，这个方法返回一个流实例，它实现了
[PSR-7 `StreamInterface`](https://www.php-fig.org/psr/psr-7/#34-psrhttpmessagestreaminterface)
和[ReactPHP `ReadableStreamInterface`](https://github.com/reactphp/stream#readablestreaminterface) 。
然而，大多数[PSR-7 `StreamInterface`](https://www.php-fig.org/psr/psr-7/#34-psrhttpmessagestreaminterface)
方法都是在控制同步请求体的假设下设计的。
鉴于这并不适用于此服务器，以下
[PSR-7 `StreamInterface`](https://www.php-fig.org/psr/psr-7/#34-psrhttpmessagestreaminterface)
方法不会被使用，也不应该被调用:`tell()`, `eof()`, `seek()`, `rewind()`, `write()` 和 `read()`。
如果这是一个问题，你的用例和/或你想访问上传的文件，它是强烈建议使用buffered[request body](#request-body)
或使用[`RequestBodyBufferMiddleware`](#requestbodybuffermiddleware) 代替。
[ReactPHP `ReadableStreamInterface`](https://github.com/reactphp/stream#readablestreaminterface)
让你访问传入的请求体作为单独的块:

```php
$server = new React\Http\Server(
    $loop,
    new React\Http\Middleware\StreamingRequestMiddleware(),
    function (Psr\Http\Message\ServerRequestInterface $request) {
        $body = $request->getBody();
        assert($body instanceof Psr\Http\Message\StreamInterface);
        assert($body instanceof React\Stream\ReadableStreamInterface);

        return new React\Promise\Promise(function ($resolve, $reject) use ($body) {
            $bytes = 0;
            $body->on('data', function ($data) use (&$bytes) {
                $bytes += strlen($data);
            });

            $body->on('end', function () use ($resolve, &$bytes){
                $resolve(new React\Http\Message\Response(
                    200,
                    array(
                        'Content-Type' => 'text/plain'
                    ),
                    "Received $bytes bytes\n"
                ));
            });

            // an error occures e.g. on invalid chunked encoded data or an unexpected 'end' event
            $body->on('error', function (\Exception $exception) use ($resolve, &$bytes) {
                $resolve(new React\Http\Message\Response(
                    400,
                    array(
                        'Content-Type' => 'text/plain'
                    ),
                    "Encountered error after $bytes bytes: {$exception->getMessage()}\n"
                ));
            });
        });
    }
);
```
上面的示例只是计算在请求体中接收到的字节数。这可以用作缓冲或处理请求主体的框架。

更多细节请参阅 [streaming request服务示例](https://github.com/reactphp/http/blob/v1.2.0/examples/63-server-streaming-request.php)

只要请求体流上有新数据可用，就会触发` data `事件。

服务器还会自动使用`Transfer-Encoding: chunked`处理任何传入请求的解码，并且只会将实际负载作为数据发出。

当请求体流成功终止时，将触发` end `事件，也就是说，它被读取到预期的结束。

如果请求流包含` Transfer-Encoding: chunked `的无效数据，或者在接收到完整的请求流之前连接关闭，则会触发` error `事件。

服务器将自动停止读取连接，并丢弃所有传入的数据，而不是关闭连接。

响应消息仍然可以发送(除非连接已经关闭)。

` close `事件将在` error `或` end `事件之后触发。

有关请求体流的更多细节，请参阅[ReactPHP ` ReadableStreamInterface `](/1.Core-Components/Stream.md#readablestreaminterface)

`getSize(): ?int`方法可以用来获取请求体的大小，类似于PHP的`$_SERVER['CONTENT_LENGTH']`变量。

此方法返回以字节数衡量的请求体的完整大小，该大小由消息边界定义。

如果请求消息不包含请求正文(如简单的`GET`请求)，则该值可以为`0`。

该方法操作流请求体，即当使用` Transfer-Encoding: chunked ` HTTP/1.1请求时，请求体大小可能是未知的(` null `)。

```php 
$server = new React\Http\Server(
    $loop,
    new React\Http\Middleware\StreamingRequestMiddleware(),
    function (Psr\Http\Message\ServerRequestInterface $request) {
        $size = $request->getBody()->getSize();
        if ($size === null) {
            $body = 'The request does not contain an explicit length.';
            $body .= 'This example does not accept chunked transfer encoding.';

            return new React\Http\Message\Response(
                411,
                array(
                    'Content-Type' => 'text/plain'
                ),
                $body
            );
        }

        return new React\Http\Message\Response(
            200,
            array(
                'Content-Type' => 'text/plain'
            ),
            "Request body size: " . $size . " bytes\n"
        );
    }
);
```

>注意: `Server` 会自动处理带有附加的`Expect: 100-continue`请求头的请求。
 当HTTP/1.1客户端想要发送一个更大的请求体时，他们可能只发送带有附加的`Expect: 100-continue`请求头的请求头，
 并在发送实际的(更大的)消息体之前等待。在这种情况下，服务器将自动向客户端发送一个中间的`HTTP/1.1 100 Continue`响应。
 这样可以确保您将收到请求正文，而不会出现预期的延迟。

#### Request method

请注意，服务器支持*any*请求方法(包括自定义和非标准方法)，
以及HTTP规范中为每种方法定义的所有请求目标格式，
包括*normal*`origin-form`请求以及`absolute-form`和`authority-form`的代理请求。

`getUri(): UriInterface`方法可以用来获得有效的请求URI，它提供了对各个URI组件的访问。

注意(取决于给定的`request-target(请求目标)`)某些URI组件可能存在也可能不存在，
例如，`getPath(): string`方法将为`asterisk-form`或`authority-form`的请求返回一个空字符串。

`getHost(): string`方法将返回由有效请求URI确定的主机，如果HTTP/1.0客户端没有指定，则默认为本地套接字地址(即没有` host `报头)。

`getScheme(): string`方法将返回`http`或`https`，这取决于请求是否通过到目标主机的安全TLS连接进行。

仅当此URI方案非标准时，才会清理`Host`头标头值以匹配此主机组件和端口组件。

您可以使用`getMethod(): string`和`getRequestTarget(): string`来检查此请求是否被接受，
并且可能希望拒绝带有适当错误代码的其他请求，例如`400`（错误请求）或`405`（不允许使用方法）。

>`CONNECT`方法在隧道设置(HTTPS代理)中是有用的，而不是大多数HTTP服务器想要关心的东西。
 请注意，如果您想处理这个方法，客户端可能会发送一个不同于` Host `报头值的请求目标(比如删除默认端口)，
 并且请求目标在转发时必须优先。

#### Cookie parameters

`getCookieParams(): string[]` 方法可以用来获取当前请求发送的所有cookie。

```php 
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    $key = 'react\php';

    if (isset($request->getCookieParams()[$key])) {
        $body = "Your cookie value is: " . $request->getCookieParams()[$key];

        return new React\Http\Message\Response(
            200,
            array(
                'Content-Type' => 'text/plain'
            ),
            $body
        );
    }

    return new React\Http\Message\Response(
        200,
        array(
            'Content-Type' => 'text/plain',
            'Set-Cookie' => urlencode($key) . '=' . urlencode('test;more')
        ),
        "Your cookie has been set."
    );
});
```

上面的例子将尝试在第一次访问时设置cookie，并尝试在所有后续尝试时打印cookie值。
请注意这个例子是如何使用` urlencode() `函数来编码非字母数字字符的。
内部解码cookie的名称和值时也使用这种编码(这与其他实现一致，比如PHP的cookie函数)。

更多细节请参阅  [cookie server示例](https://github.com/reactphp/http/blob/v1.2.0/examples/55-server-cookie-handling.php)

#### Invalid request

`Server`类同时支持HTTP/1.1和HTTP/1.0请求消息。如果客户端发送了一个无效的请求消息，
使用了一个无效的HTTP协议版本或发送了一个无效的` Transfer-Encoding `请求头值，
服务器将自动发送一个` 400 `(Bad Request)HTTP错误响应到客户端并关闭连接。

除此之外，它将发出一个` error `事件，可以用于日志记录:

```php
$server->on('error', function (Exception $e) {
    echo 'Error: ' . $e->getMessage() . PHP_EOL;
});
```

请注意，如果您没有从请求处理程序函数返回一个有效的响应对象，服务器也将发出一个` error `事件。

另请参阅[invalid response](#invalid-response)

### Server Response

传递给[`Server`](#server)构造函数的回调函数负责处理请求并返回响应，该响应将被传递给客户端。

这个函数必须要返回一个实例实现
[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface)
对象或(ReactPHP承诺)(/1.Core-Components/Promise.md),解决了用
[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface) 对象。

这个项目提供了一个[`Response` class](#response)，它实现了
[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface)

最简单的使用方式:

```php 
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    return new React\Http\Message\Response(
        200,
        array(
            'Content-Type' => 'text/plain'
        ),
        "Hello World!\n"
    );
});
```

我们在整个项目示例中都使用这个[`Response` class](#response)，但也可以随意使用
[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface)

另请参阅 [`Response` class](#response)

#### Deferred response

上面的示例直接返回响应，因为它不需要处理时间。

使用数据库、文件系统或长时间计算(实际上每个操作都需要>=1ms)来创建响应，将降低服务器的速度。
为了防止这种情况发生，你应该使用[ReactPHP Promise](https://github.com/reactphp/promise#reactpromise) 。

这个例子展示了一个长期的运行的服务:

```php
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) use ($loop) {
    return new Promise(function ($resolve, $reject) use ($loop) {
        $loop->addTimer(1.5, function() use ($resolve) {
            $response = new React\Http\Message\Response(
                200,
                array(
                    'Content-Type' => 'text/plain'
                ),
                "Hello world"
            );
            $resolve($response);
        });
    });
});
```

上面的示例将在1.5秒后创建响应。

这个例子表明，如果您的响应需要时间来创建，那么您需要Promise。

当请求体结束时，` ReactPHP Promise `将在` Response `对象中解析。

如果客户端在承诺仍然挂起时关闭连接，承诺将自动被取消。

promise cancel处理程序可用于清除在这种情况下分配的任何挂起的资源(如果适用的话)。

如果一个承诺在客户端关闭后被解析，它将被忽略。

#### Streaming outgoing response

这个项目中的` Response `类支持为响应体添加一个实现
[ReactPHP `ReadableStreamInterface`](https://github.com/reactphp/stream#readablestreaminterface)
的实例。

因此，您可以将数据流直接输入响应体。

请注意[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface)
的其他实现可能只支持字符串。

```php
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) use ($loop) {
    $stream = new ThroughStream();

    $timer = $loop->addPeriodicTimer(0.5, function () use ($stream) {
        $stream->write(microtime(true) . PHP_EOL);
    });

    $loop->addTimer(5, function() use ($loop, $timer, $stream) {
        $loop->cancelTimer($timer);
        $stream->end();
    });

    return new React\Http\Message\Response(
        200,
        array(
            'Content-Type' => 'text/plain'
        ),
        $stream
    );
});
```

上面的示例将每0.5秒向客户端发送当前的Unix时间戳(以微秒为浮点数)，并在5秒后结束。

这只是一个您可以使用流的示例，您也可以通过小块发送大量数据，或将其用于需要计算的主体数据。

如果请求处理程序使用一个已经关闭的响应流进行解析，它将简单地发送一个空响应体。

如果客户端在流仍然打开的情况下关闭连接，则响应流将自动关闭。

如果在客户端关闭后使用流体解析Promise，则响应流将自动关闭。

` close `事件可以用来清理在这种情况下分配的任何挂起的资源(如果适用的话)。

>请注意，如果你使用一个实现ReactPHP的
 [' DuplexStreamInterface '](https://github.com/reactphp/stream#duplexstreaminterface)
 的体流实例(例如上面例子中的` ThroughStream `)，必须特别小心。
>在*大多数*情况下，这只会消耗流的可读端，并转发(发送)流发出的任何数据，从而完全忽略流的可写端。
 然而，如果这是对` CONNECT `方法的` 101 `(交换协议)响应或` 2xx `(成功)响应，它也将*写入*数据到流的可写端。
 这可以通过拒绝所有使用` CONNECT `方法的请求(这是大多数*正常*源HTTP服务器可能会做的)
 或确保只使用[ReactPHP's `ReadableStreamInterface`](https://github.com/reactphp/stream#readablestreaminterface) 
 来避免。

>101(交换协议)响应代码对于更高级的`升级`请求很有用，比如升级到WebSocket协议，或者实现超出HTTP规范和HTTP库范围的自定义协议逻辑。
 如果你想处理` Upgrade: WebSocket `头文件，你可能需要考虑使用[Ratchet](http://socketo.me/) 来代替。
 如果你想处理一个自定义协议，你可能需要查看[HTTP specs](https://tools.ietf.org/html/rfc7230#section-6.7) ，
 也可以查看[示例 #81和#82](https://github.com/reactphp/http/blob/v1.2.0/examples/) 了解更多细节。
 特别是，`101`(交换协议)响应代码不能被使用，除非你发送一个`Upgrade`响应头值，这个值也出现在HTTP/1.1的`Upgrade`请求头值中。
 在这种情况下，服务器会自动发送` Connection: upgrade `报头值，所以你不必这么做。

>`CONNECT`方法在隧道设置(HTTPS代理)中是有用的，而不是大多数原始HTTP服务器想要关心的东西。
 HTTP规范为这个方法定义了一个不透明的`隧道模式`，并且没有使用消息体。
 出于一致性原因，该库在响应体中使用了` DuplexStreamInterface `，用于隧道化的应用程序数据。
 这意味着对`CONNECT`请求的`2xx`(成功)响应实际上可以为隧道化的应用程序数据使用流响应体，
 这样客户端通过连接发送的任何原始数据都将通过管道通过可写流进行消费。
 请注意，虽然HTTP规范没有使用`CONNECT`请求的请求体，但可能仍然存在一个请求体。
 正常的请求体处理在这里适用，只有在请求体被处理之后(在大多数情况下应该是空的)连接才会转向`隧道模式`。
 更多细节请参阅[HTTP CONNECT服务示例](https://github.com/reactphp/http/blob/v1.2.0/examples/72-server-http-connect-proxy.php)

#### Response length

如果知道响应正文的大小，则会自动添加一个`Content-Length`响应头。这是最常见的用例，例如当像这样使用` string `响应体时:

```php 
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) {
    return new React\Http\Message\Response(
        200,
        array(
            'Content-Type' => 'text/plain'
        ),
        "Hello World!\n"
    );
});
```

如果响应正文大小未知，则无法自动添加`Content-Length`响应标头。当使用没有显式`Content Length`响应头的
[streaming -outgoing response](#streaming-outgoing-response)
时，HTTP/1.1 响应消息将自动使用`Transfer Encoding:chunked`，
而旧版HTTP/1.0 响应消息将包含普通响应正文。
如果您知道流式处理响应主体的长度，则可能需要像下面这样显式指定它:

```php
$server = new React\Http\Server($loop, function (Psr\Http\Message\ServerRequestInterface $request) use ($loop) {
    $stream = new ThroughStream();

    $loop->addTimer(2.0, function () use ($stream) {
        $stream->end("Hello World!\n");
    });

    return new React\Http\Message\Response(
        200,
        array(
            'Content-Length' => '13',
            'Content-Type' => 'text/plain',
        ),
        $stream
    );
});
```

根据HTTP规范，对` HEAD `请求的任何响应以及任何带有` 1xx `(信息)、
` 204 `(无内容)或` 304 `(未修改)状态码的响应都将*不*包含消息体。
这意味着你的回调不需要特别注意这个，任何响应体都会被忽略。

同样，对` CONNECT `请求的任何` 2xx `(成功)响应，任何带有` 1xx `(信息)
或` 204 `(无内容)状态码的响应将*不*包含` Content- length `或` Transfer-Encoding `头，
因为这些不适用于这些消息。

请注意，对`HEAD`请求的响应和带有`304`（未修改）状态码的任何响应可能包括这些头，即使消息不包含响应正文，
因为如果相同的请求使用（无条件的）`GET`，那么这些头将应用于消息。

#### Invalid response

如上所示，每个传出的HTTP响应始终由
[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface) 表示。
如果您的请求处理函数返回无效值或引发未处理的 `Exception` 或 `Throwable` ，则服务器将自动向客户端发送`500`（内部服务器错误）HTTP错误响应。
最重要的是，它将发出一个`error`事件，可用于记录，如下所示:

```php
$server->on('error', function (Exception $e) {
    echo 'Error: ' . $e->getMessage() . PHP_EOL;
    if ($e->getPrevious() !== null) {
        echo 'Previous: ' . $e->getPrevious()->getMessage() . PHP_EOL;
    }
});
```

请注意，如果客户端发送的无效HTTP请求从未到达您的请求处理程序函数，服务器也将发出“error”事件。
有关详细信息，请参见[invalid request](#invalid-request) 。
此外，[streaming incoming request](#streaming-incoming-request)
主体还可以在请求主体上发出`error` 事件。

如果发生未处理的错误，服务器只会发送一个非常通用的`500` (Interval Server Error) HTTP错误响应，
而不会向客户端发送任何其他详细信息。虽然我们知道这可能会使初始调试更加困难，
但这也意味着服务器在默认情况下不会向外部泄漏任何应用程序细节或堆栈跟踪信息。
通常建议在请求处理函数中捕获任何` Exception `或` Throwable `，或者使用[` middleware `](#middleware)
中创建自己的HTTP响应信息来避免这种通用错误处理。

#### Default response headers

当请求处理程序函数返回响应时，响应将由[`Server`](#server)处理，然后发送回客户端。
`Server: ReactPHP/1` 响应头将被自动添加。 您可以像这样添加自定义的`Server`响应标头:

```php
$server = new React\Http\Server($loop, function (ServerRequestInterface $request) {
    return new React\Http\Message\Response(
        200,
        array(
            'Server' => 'PHP/3'
        )
    );
});
```

如果您根本不想发送此`Sever`响应标头（例如，当您不想公开底层服务器软件时），则可以使用如下所示的空字符串值:

```php
$server = new React\Http\Server($loop, function (ServerRequestInterface $request) {
    return new React\Http\Message\Response(
        200,
        array(
            'Server' => ''
        )
    );
});
```

如果未指定日期和时间，则会自动添加一个`Date`响应标题和当前系统日期和时间。 
您也可以添加自定义的`Date`响应标头，如下所示:

```php
$server = new React\Http\Server($loop, function (ServerRequestInterface $request) {
    return new React\Http\Message\Response(
        200,
        array(
            'Date' => gmdate('D, d M Y H:i:s \G\M\T')
        )
    );
});
```

如果您不想发送此`Date`响应标头（例如，当您没有适当的时钟依赖时），
则可以使用如下所示的空字符串值:

```php
$server = new React\Http\Server($loop, function (ServerRequestInterface $request) {
    return new React\Http\Message\Response(
        200,
        array(
            'Date' => ''
        )
    );
});
```

`Server`类会自动添加请求的协议版本，所以你不必自己添加。
例如，如果客户端使用HTTP/1.1协议版本发送请求，无论请求处理程序函数返回的是什么版本，
响应消息也将使用相同的协议版本，

注意，目前不支持持久连接(` Connection: keep-alive `)。
因此不管您设置的是什么头值，HTTP/1.1响应消息将自动包含` Connection: close `头。

### Middleware

如上所示，[`Server`](#server)接受一个请求处理程序参数，
该参数负责处理传入的HTTP请求，然后创建并返回传出的HTTP响应。
许多常见的用例都涉及到在将传入的HTTP请求传递给最终的业务逻辑请求处理程序之前对其进行验证、处理和操作。
因此，该项目支持中间件请求处理程序的概念。

#### Custom middleware

中间件请求处理程序应该遵循以下规则:

* 它是一个有效的`callable`。
* 接受一个实现
  [PSR-7 `ServerRequestInterface`](https://www.php-fig.org/psr/psr-7/#321-psrhttpmessageserverrequestinterface)
  的实例作为第一个参数和一个可选的`callable`作为第二个参数。

* 它返回以下任意一个:
 * 一个实例实现[`Promise\resolve()`](/1.Core-Components/Promise.md#resolve) 直接消费。
 * 任何可以被[`Promise\resolve()`](/1.Core-Components/Promise.md#resolve) 消费的承诺，
   解析为[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface) 延迟消费。
 * 它可能会抛出一个` Exception `(或者返回一个被拒绝的promise)来表示一个错误条件并中止。

* 它调用` $next($request) `继续处理下一个中间件请求处理程序，或显式返回而不调用` $next `中止。
 * ` $next `请求处理程序(递归地)调用链上的下一个请求处理程序，并返回(或抛出)如上所示。
 * 在调用`$next($request)`来更改下一个中间件操作的传入请求之前，可以修改` $request `。
 * ` $next `返回值可能被用来修改输出响应。
 * 如果你想实现自定义`retry`逻辑等，`$next`请求处理程序可能会被多次调用。

注意，这个非常简单的定义允许您使用匿名函数或使用魔术类`__invoke()`的任何类。
这使您可以轻松动态地创建自定义的中间件请求处理程序，或使用基于类的方法轻松使用现有的中间件实现。

尽管此项目确实提供了*使用*中间件实现的方法，但它的目标并不是*定义*中间件实现应该是什么样子。
我们意识到存在一个生动的中间件实现生态系统，并且正在努力通过
[PSR-15](https://www.php-fig.org/psr/psr-15/) (HTTP Server Request Handlers)
标准化它们之间的接口并支持它。
因此，这个项目只捆绑了一些中间件实现，这些实现是匹配PHP的请求行为所必需的（见下文），
并积极鼓励[Third-Party Middleware](#third-party-middleware) 实现。

为了使用中间件请求处理程序，只需将上面定义的所有可调用对象的数组传递给[`Server`](#server) 。
下面的示例添加了一个中间件请求处理程序，它将当前时间作为头添加到请求中（`request time`），
并添加了一个最终的请求处理程序，它总是返回一个不带正文的200代码:

```php
$server = new React\Http\Server(
    $loop,
    function (Psr\Http\Message\ServerRequestInterface $request, callable $next) {
        $request = $request->withHeader('Request-Time', time());
        return $next($request);
    },
    function (Psr\Http\Message\ServerRequestInterface $request) {
        return new React\Http\Message\Response(200);
    }
);
```

>请注意，中间件请求处理程序和最终请求处理程序有一个非常简单(且类似)的接口。
 唯一的区别是最终的请求处理程序没有接收到` $next `处理程序。

同样，您可以使用` $next `中间件请求处理程序函数的结果来修改传出响应。
请注意，根据上面的文档，` $next `中间件请求处理程序可以直接返回
[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface)
或者返回一个包装在延迟解决承诺中的。
为了简化这两个路径的处理，你可以简单地把它包装在[`Promise\resolve()`](/1.Core-Components/Promise.md/#resolve)
调用中，像这样:

```php
$server = new React\Http\Server(
    $loop,
    function (Psr\Http\Message\ServerRequestInterface $request, callable $next) {
        $promise = React\Promise\resolve($next($request));
        return $promise->then(function (ResponseInterface $response) {
            return $response->withHeader('Content-Type', 'text/html');
        });
    },
    function (Psr\Http\Message\ServerRequestInterface $request) {
        return new React\Http\Message\Response(200);
    }
);
```

请注意，` $next `中间件请求处理程序也可能抛出一个` Exception `(或返回一个被拒绝的承诺)，如上所示。
前面的例子没有捕获任何异常，因此会向`Server`发出一个错误条件信号。
另外，你也可以捕获任何`Exception`来实现自定义错误处理逻辑(或日志记录等)，方法是将其包装在
[` Promise `](/1.Core-Components/Promise.md/#promise)中，如下所示:

```php
$server = new React\Http\Server(
    $loop,
    function (Psr\Http\Message\ServerRequestInterface $request, callable $next) {
        $promise = new React\Promise\Promise(function ($resolve) use ($next, $request) {
            $resolve($next($request));
        });
        return $promise->then(null, function (Exception $e) {
            return new React\Http\Message\Response(
                500,
                array(),
                'Internal error: ' . $e->getMessage()
            );
        });
    },
    function (Psr\Http\Message\ServerRequestInterface $request) {
        if (mt_rand(0, 1) === 1) {
            throw new RuntimeException('Database error');
        }
        return new React\Http\Message\Response(200);
    }
);
```

#### Third-Party Middleware

尽管此项目确实提供了*使用*中间件实现的方法，但它的目标并不是*定义*中间件实现应该是什么样子。
我们意识到存在一个生动的中间件实现生态系统，并且正在努力通过
[PSR-15](https://www.php-fig.org/psr/psr-15/) (HTTP Server Request Handlers)
标准化它们之间的接口并支持它。
因此，这个项目只捆绑了一些中间件实现，这些实现是匹配PHP的请求行为所必需的（见下文），
并积极鼓励[Third-Party Middleware](#third-party-middleware) 实现。

虽然我们希望在` react/http `中直接支持PSR-15，但我们知道这个接口并不是专门针对async api的，
因此也没有利用[deferred responses](#deferred-response) 的承诺。要点在于，当PSR-15强制使用
[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface)
返回值时，我们也接受`PromiseInterface<ResponseInterface>`。因此，我们建议使用外部
[PSR-15 middleware adapter](https://github.com/friends-of-reactphp/http-middleware-psr15-adapter) ，
它使用这些返回值的fly monkey补丁，这使得在这个包中使用大多数PSR-15中间件成为可能，而不需要任何更改。

除此之外，您还可以使用上面的[middleware definition](#middleware)
来创建定制的中间件。第三方中间件的非详尽列表可以在
[middleware wiki](https://github.com/reactphp/reactphp/wiki/Users#http-middleware)
中找到。如果你创建了一个自定义中间件，请确保让所有人都知道并将其添加到这个列表中。

## API

### Browser

`React\Http\Browser` 负责发送Http请求到您的Http服务器，并保持跟踪等待返回的Http响应。
它还将所有内容注册到主[`EventLoop`](/1.Core-Components/EventLoop.md#用法)

```php
$loop = React\EventLoop\Factory::create();

$browser = new React\Http\Browser($loop);
```

如果您需要自定义连接器设置(DNS解析、TLS参数、超时、代理服务器等)，您可以显式地传递
[` ConnectorInterface `](https://github.com/reactphp/socket#connectorinterface)
的自定义实例:

```php
$connector = new React\Socket\Connector($loop, array(
    'dns' => '127.0.0.1',
    'tcp' => array(
        'bindto' => '192.168.10.1:0'
    ),
    'tls' => array(
        'verify_peer' => false,
        'verify_peer_name' => false
    )
));

$browser = new React\Http\Browser($loop, $connector);
```

> 请注意，`browser`类是final类，不应该被继承，它很可能在将来的版本中被标记为final。
#### get()

`get(string $url, array $headers = array()): PromiseInterface<ResponseInterface>`方法可用于发送HTTP GET请求

```php
$browser->get($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    var_dump((string)$response->getBody());
});
```

请参阅 [GET request client示例](https://github.com/reactphp/http/blob/v1.2.0/examples/01-client-get-request.php).

#### post()

`post(string $url, array $headers = array(), string|ReadableStreamInterface $contents = ''): PromiseInterface<ResponseInterface>`
方法可用于发送HTTP POST请求

```php
$browser->post(
    $url,
    [
        'Content-Type' => 'application/json'
    ],
    json_encode($data)
)->then(function (Psr\Http\Message\ResponseInterface $response) {
    var_dump(json_decode((string)$response->getBody()));
});
```

请参阅 [POST JSON client示例](https://github.com/reactphp/http/blob/v1.2.0/examples/04-client-post-json.php).

这种方法也常用来提交HTML表单数据:

```php
$data = [
    'user' => 'Alice',
    'password' => 'secret'
];

$browser->post(
    $url,
    [
        'Content-Type' => 'application/x-www-form-urlencoded'
    ],
    http_build_query($data)
);
```

如果发出的请求主体是一个` string `，该方法将自动添加一个匹配的` Content-Length `请求头。
如果你正在使用一个流请求体(` ReadableStreamInterface `)，它将默认使用` Transfer-Encoding: chunked `，
或者你必须显式地传递一个匹配的` Content-Length `请求头，如下所示:

```php
$body = new React\Stream\ThroughStream();
$loop->addTimer(1.0, function () use ($body) {
    $body->end("hello world");
});

$browser->post($url, array('Content-Length' => '11'), $body);
```

#### head()

`head(string $url, array $headers = array()): PromiseInterface<ResponseInterface>` 方法可用于发送HTTP HEAD请求

```php
$browser->head($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    var_dump($response->getHeaders());
});
```

#### patch()

`patch(string $url, array $headers = array(), string|ReadableStreamInterface $contents = ''): PromiseInterface<ResponseInterface>`
方法可用于发送HTTP PATCH请求

```php
$browser->patch(
    $url,
    [
        'Content-Type' => 'application/json'
    ],
    json_encode($data)
)->then(function (Psr\Http\Message\ResponseInterface $response) {
    var_dump(json_decode((string)$response->getBody()));
});
```

如果发出的请求主体是一个` string `，该方法将自动添加一个匹配的` Content-Length `请求头。
如果你正在使用一个流请求体(` ReadableStreamInterface `)，它将默认使用` Transfer-Encoding: chunked `，
或者你必须显式地传递一个匹配的` Content-Length `请求头，如下所示:

```php
$body = new React\Stream\ThroughStream();
$loop->addTimer(1.0, function () use ($body) {
    $body->end("hello world");
});

$browser->patch($url, array('Content-Length' => '11'), $body);
```

#### put()

`put(string $url, array $headers = array()): PromiseInterface<ResponseInterface>`
方法可用于发送HTTP PUT请求

```php
$browser->put(
    $url,
    [
        'Content-Type' => 'text/xml'
    ],
    $xml->asXML()
)->then(function (Psr\Http\Message\ResponseInterface $response) {
    var_dump((string)$response->getBody());
});
```

请参阅[PUT XML client示例](https://github.com/reactphp/http/blob/v1.2.0/examples/05-client-put-xml.php).

如果发出的请求主体是一个` string `，该方法将自动添加一个匹配的` Content-Length `请求头。
如果你正在使用一个流请求体(` ReadableStreamInterface `)，它将默认使用` Transfer-Encoding: chunked `，
或者你必须显式地传递一个匹配的` Content-Length `请求头，如下所示:

```php
$body = new React\Stream\ThroughStream();
$loop->addTimer(1.0, function () use ($body) {
    $body->end("hello world");
});

$browser->put($url, array('Content-Length' => '11'), $body);
```

#### delete()

`delete(string $url, array $headers = array()): PromiseInterface<ResponseInterface>`
方法可用于发送HTTP DELETE请求

```php
$browser->delete($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    var_dump((string)$response->getBody());
});
```

#### request()

`request(string $method, string $url, array $headers = array(), string|ReadableStreamInterface $body = ''): PromiseInterface<ResponseInterface>`
方法发送一个任意的HTTP请求。

发送HTTP请求的首选方法是使用上面的[request methods](#request-methods) ，
例如[`get()`](#get)方法发送HTTP ` GET `请求。

另外，如果你想使用一个自定义的HTTP请求方法，你可以使用这个方法:

```php
$browser->request('OPTIONS', $url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    var_dump((string)$response->getBody());
});
```

如果请求正文的大小已知且非空，此方法将自动添加一个匹配的` Content-Length `请求头。
对于一个空的请求正文，如果请求方法通常需要一个请求正文(仅适用于` POST `， ` PUT `和` PATCH `)，
则如果将只包含一个` Content-Length: 0 `的请求头。

如果你正在使用一个流请求体(` ReadableStreamInterface `)，
它将默认使用` Transfer-Encoding: chunked `，
或者你必须显式地传递一个匹配的` Content-Length `请求头，
如下所示:

```php
$body = new React\Stream\ThroughStream();
$loop->addTimer(1.0, function () use ($body) {
    $body->end("hello world");
});

$browser->request('POST', $url, array('Content-Length' => '11'), $body);
```

#### requestStreaming()

`requestStreaming(string $method, string $url, array $headers = array(), string|ReadableStreamInterface $body = ''): PromiseInterface<ResponseInterface>`
方法可以用来发送任意的HTTP请求和接收流响应，而不需要缓冲响应体。

发送HTTP请求的首选方法是使用上述[request methods](#request-methods) ，
例如[`get()`](#get)方法发送HTTP`GET`请求。 
默认情况下，这些方法中的每一个都会在内存中缓冲整个响应主体。 
这很容易上手，并且对于较小的响应也相当有效。

在某些情况下，最好使用流传输方法，其中只需要在内存中保留一小块。
您可以使用此方法发送任意HTTP请求并接收流式响应。
它使用相同的HTTP消息API，但不将响应主体缓存在内存中。 
它仅在接收到数据时以小块形式处理响应主体，并通过
[ReactPHP的Stream API](/1.Core-Components/Stream.md)
转发此数据。这适用于（任意数量）任意大小的响应。

```php
$browser->requestStreaming('GET', $url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    $body = $response->getBody();
    assert($body instanceof Psr\Http\Message\StreamInterface);
    assert($body instanceof React\Stream\ReadableStreamInterface);

    $body->on('data', function ($chunk) {
        echo $chunk;
    });

    $body->on('error', function (Exception $error) {
        echo 'Error: ' . $error->getMessage() . PHP_EOL;
    });

    $body->on('close', function () {
        echo '[DONE]' . PHP_EOL;
    });
});
```
另请参见[ReactPHP's `ReadableStreamInterface`](/1.Core-Components/Stream.md#readablestreaminterface)
和[streaming response](#streaming-response) ，以获取更多详细信息，示例和可能的用例。

如果传出请求主体的大小已知且非空，则此方法将自动添加匹配的`Content-Length`请求标头。
对于空的请求主体，如果请求方法通常需要一个请求主体（仅适用于`POST`, `PUT` 和 `PATCH`），
则仅包含`Content-Length: 0`请求标头。

如果您使用的是流式请求主体（`ReadableStreamInterface`），
则默认使用`Transfer-Encoding: chunked`，
否则您必须显式地传递匹配的`Content-Length`请求标头，如下所示:

```php
$body = new React\Stream\ThroughStream();
$loop->addTimer(1.0, function () use ($body) {
    $body->end("hello world");
});

$browser->requestStreaming('POST', $url, array('Content-Length' => '11'), $body);
```

#### withTimeout()

`withTimeout(bool|number $timeout): Browser`方法可用于更改等待挂起请求的最大超时。

您可以传入秒数作为新的超时值:

```php
$browser = $browser->withTimeout(10.0);
```
您可以传入布尔值`false`来禁用任何超时。 在这种情况下，请求可以永远保持待处理状态:

```php
$browser = $browser->withTimeout(false);
```

您可以传入布尔值true来重新启用默认超时处理。 这将遵守PHP的`default_socket_timeout`设置（默认为60s）:

```php
$browser = $browser->withTimeout(true);
```

另请参阅 [timeouts](#timeouts) 了解有关超时处理的更多详细信息。

请注意，[`Browser`](#browser)是一个不可变的对象，
即此方法实际上返回一个* 新的 * [`Browser`](#browser) 实例，并应用了给定的超时值。

#### withFollowRedirects()

`withFollowRedirects(bool|int $followRedirects): Browser` 方法可用于更改遵循HTTP重定向。 

您可以传入重定向的最大数量来追踪:

```php
$browser = $browser->withFollowRedirects(5);
```

当重定向次数超过时，请求将自动被拒绝。你可以传入一个` 0 `来拒绝任何遇到的重定向请求:

```php
$browser = $browser->withFollowRedirects(0);

$browser->get($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    // only non-redirected responses will now end up here
    var_dump($response->getHeaders());
});
```

您可以传入布尔值`false`来禁用任何重定向。在这种情况下，
请求将使用重定向响应进行解析，而不是遵循`Location`响应头:

```php
$browser = $browser->withFollowRedirects(false);

$browser->get($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    // any redirects will now end up here
    var_dump($response->getHeaderLine('Location'));
});
```

您可以传入布尔值`true`来重新启用默认重定向处理。
默认情况下，最多遵循10个重定向:

```php
$browser = $browser->withFollowRedirects(true);
```

另请参阅[redirects](#redirects) ，以获取有关重定向处理的更多详细信息。

请注意，[`Browser`](#browser)是一个不可变的对象，
即该方法实际上返回一个*新的* [`Browser`](#browser)实例，并应用了给定的重定向设置。

#### withRejectErrorResponse()

`withRejectErrorResponse(bool $obeySuccessCode): Browser`方法可用于更改是否拒绝不成功的HTTP响应状态代码（4xx和5xx）。 

您可以传入布尔值`false`以禁用拒绝使用4xx或5xx响应状态代码的传入响应。在这种情况下，请求将通过指示错误条件的响应消息进行解析:

```php
$browser = $browser->withRejectErrorResponse(false);

$browser->get($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    // any HTTP response will now end up here
    var_dump($response->getStatusCode(), $response->getReasonPhrase());
});
```

您可以传入布尔值`true`来重新启用默认状态代码处理。
默认情况下，使用[`ResponseException`](#responseexception)
拒绝4xx或5xx范围内的任何响应状态代码:

```php
$browser = $browser->withRejectErrorResponse(true);

$browser->get($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    // any successful HTTP response will now end up here
    var_dump($response->getStatusCode(), $response->getReasonPhrase());
}, function (Exception $e) {
    if ($e instanceof React\Http\Message\ResponseException) {
        // any HTTP response error message will now end up here
        $response = $e->getResponse();
        var_dump($response->getStatusCode(), $response->getReasonPhrase());
    } else {
        var_dump($e->getMessage());
    }
});
```

请注意，[`Browser`](#browser)是一个不可变的对象，
即该方法实际上返回一个*新的* [`Browser`](#browser)实例，并应用了给定的设置。

#### withBase()

`withBase(string|null $baseUrl): Browser` 方法可用于更改解析相对URL的基本URL。 

如果您配置了一个基本URL，那么对相对URL的任何请求都将首先通过相对于给定的绝对基本URL解析这个请求来处理。
这支持解析相对路径引用（如`../`等）。
这对于所有端点（URL）都位于公共基URL下的（RESTful）API调用特别有用。

```php
$browser = $browser->withBase('http://api.example.com/v3/');

// will request http://api.example.com/v3/users
$browser->get('users')->then(…);
```
您可以传入`null`基础URL以返回不使用基础URL的新实例:

```php
$browser = $browser->withBase(null);
```

因此，任何使用相对URL的请求都无法完成，并且在不发送请求的情况下将被拒绝。

如果给定的` $baseUrl `参数不是一个有效的URL，这个方法将抛出一个` InvalidArgumentException `。

注意，[`Browser`](#browser)是一个不可变对象，也就是说，
` withBase() `方法实际上会返回一个*新的* [` Browser `](#browser)实例，其中应用了给定的base URL。

#### withProtocolVersion()

`withProtocolVersion(string $protocolVersion): Browser` 方法用来更改HTTP协议的版本，该版本将用于所有后续请求。

以上所有的 [request methods](#request-methods) 默认以HTTP/1.1的方式发送请求。
这是首选的HTTP协议版本，它还提供了与传统HTTP/1.0服务器的良好向后兼容性。
所以一般很少更改这个协议版本。

如果你想使用旧的HTTP/1.0协议版本，你可以使用这个方法:

```php
$browser = $browser->withProtocolVersion('1.0');

$browser->get($url)->then(…);
```

注意，[`Browser`](#browser)是一个不可变对象，也就是说，
` withBase() `方法实际上会返回一个*新的* [` Browser `](#browser)实例，其中应用了新协议版本。

#### withResponseBuffer()

`withResponseBuffer(int $maximumSize): Browser`方法用来改变缓冲响应体的最大大小。

发送HTTP请求的首选方法是使用上述 [request methods](#request-methods) ，
例如 [`get()`](#get) 方法发送HTTP`GET`请求。 默认情况下，这些方法中的每一个都会在内存中缓冲整个响应主体。 
这很容易上手，并且对于较小的响应也相当有效。

默认情况下，响应主体缓冲区将限制为16 MiB。 如果响应正文超过此值，则该请求将被拒绝。

您可以传入最大字节数来缓冲:

```php
$browser = $browser->withResponseBuffer(1024 * 1024);

$browser->get($url)->then(function (Psr\Http\Message\ResponseInterface $response) {
    // response body will not exceed 1 MiB
    var_dump($response->getHeaders(), (string) $response->getBody());
});
```

请注意，对于每个待处理的请求，响应主体缓冲区必须保留在内存中，直到其传输完成为止，
并且仅在满足待处理的请求之后才将其释放。 所以通常不建议增加此最大缓冲区大小以允许更大的响应主体。
相反，您可以使用[`requestStreaming()` method](#requeststreaming)
接收任意大小的响应而无需缓冲。 因此，此最大缓冲区大小设置对流响应没有影响。

注意，[`Browser`](#browser)是一个不可变对象，也就是说，
` withBase() `方法实际上会返回一个*新的* [` Browser `](#browser)实例，其中应用了给定的设置。

### React\Http\Message

#### Response

`React\Http\Message\Response`类可用于表示外发服务器响应消息。 

```php
$response = new React\Http\Message\Response(
    200,
    array(
        'Content-Type' => 'text/html'
    ),
    "<html>Hello world!</html>\n"
);
```

这个类实现了
[PSR-7 `ResponseInterface`](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface)
这又扩展了
[PSR-7 `MessageInterface`](https://www.php-fig.org/psr/psr-7/#31-psrhttpmessagemessageinterface).

>此实现构建在现有传入响应消息的基础上，并且只添加所需的流支持。这个基类被认为是将来可能改变的实现细节。

#### ServerRequest

`React\Http\Message\ServerRequest`类可以用来代表传入的服务器请求消息。 

此类实现了
[PSR-7 `ServerRequestInterface`](https://www.php-fig.org/psr/psr-7/#321-psrhttpmessageserverrequestinterface)
又扩展了
[PSR-7 `RequestInterface`](https://www.php-fig.org/psr/psr-7/#32-psrhttpmessagerequestinterface)
继而扩展了
[PSR-7 `MessageInterface`](https://www.php-fig.org/psr/psr-7/#31-psrhttpmessagemessageinterface).

这主要用于内部表示每个传入的请求消息。
同样，您也可以在测试用例中使用这个类来测试您的web应用程序如何响应某些HTTP请求。

>此实现建立在现有传出请求消息的顶部，并且仅添加必需的服务器方法。 该基类被认为是将来可能会更改的实现细节。

#### ResponseException

` React\Http\Message\ResponseException `是一个` Exception `子类，
如果远程服务器返回一个不成功的状态码(除了2xx或3xx)，它将被用来拒绝请求承诺。
你可以通过[`withRejectErrorResponse()` method](#withrejecterrorresponse)
来控制这个行为。

`getCode(): int` 方法可以用来返回HTTP响应状态码。

`getResponse(): ResponseInterface`方法可用于访问其底层的响应对象。

### React\Http\Middleware

#### StreamingRequestMiddleware

`React\Http\Middleware\StreamingRequestMiddleware`可以用来处理流请求体(没有缓冲)传入的请求。

这允许您处理任意大小的请求，而无需将请求体缓冲在内存中。
相反，它将把请求体表示为
[`ReadableStreamInterface`](/1.Core-Components/Stream.md#readablestreaminterface) ，
它会在接收到数据时发出数据块:

```php
$server = new React\Http\Server(array(
    new React\Http\Middleware\StreamingRequestMiddleware(),
    function (Psr\Http\Message\ServerRequestInterface $request) {
        $body = $request->getBody();
        assert($body instanceof Psr\Http\Message\StreamInterface);
        assert($body instanceof React\Stream\ReadableStreamInterface);

        return new React\Promise\Promise(function ($resolve) use ($body) {
            $bytes = 0;
            $body->on('data', function ($chunk) use (&$bytes) {
                $bytes += \count($chunk);
            });
            $body->on('close', function () use (&$bytes, $resolve) {
                $resolve(new React\Http\Message\Response(
                    200,
                    [],
                    "Received $bytes bytes\n"
                ));
            });
        });
    }
));
```

更多请参阅 [streaming incoming request](#streaming-incoming-request)

另外，这个中间件可以结合使用[`LimitConcurrentRequestsMiddleware`](#limitconcurrentrequestsmiddleware)
和[`RequestBodyBufferMiddleware`](#requestbodybuffermiddleware)
(见下文)来显式配置一次可以处理的请求总数:

```php
$server = new React\Http\Server(array(
    new React\Http\Middleware\StreamingRequestMiddleware(),
    new React\Http\Middleware\LimitConcurrentRequestsMiddleware(100), //100个并发缓冲处理程序
    new React\Http\Middleware\RequestBodyBufferMiddleware(2 * 1024 * 1024), // 每请求 2 MiB
    new React\Http\Middleware\RequestBodyParserMiddleware(),
    $handler
));
```

>这个类内部被用作`标记`，以不触发`Server`中的默认请求缓冲行为。它本身不实现任何逻辑。

#### LimitConcurrentRequestsMiddleware

`React\Http\Middleware\LimitConcurrentRequestsMiddleware`可用于限制并发执行的处理程序的数量。

如果调用这个中间件，它将检查挂起处理程序的数量是否低于允许的限制，然后简单地调用下一个处理程序，它将返回下一个处理程序返回(或抛出)的任何内容。

如果挂起的处理程序的数量超过了允许的限制，请求将被排队（其流主体将被暂停），并且它将返回一个挂起的承诺。
一旦挂起的处理程序返回（或抛出），它将选择最早的请求并调用下一个处理程序（其流主体将被恢复）。

以下示例显示了如何使用此中间件来确保一次调用不超过10个处理程序: 
```php
$server = new React\Http\Server(
    $loop,
    new React\Http\Middleware\LimitConcurrentRequestsMiddleware(10),
    $handler
);
```
此中间件通常与[`RequestBodyBufferMiddleware`](#requestbodybuffermiddleware) 
（见下文）结合使用，限制一次可以缓冲的请求总数:

```php
$server = new React\Http\Server(
    $loop,
    new React\Http\Middleware\StreamingRequestMiddleware(),
    new React\Http\Middleware\LimitConcurrentRequestsMiddleware(100), //100个并发缓冲处理程序
    new React\Http\Middleware\RequestBodyBufferMiddleware(2 * 1024 * 1024), // 每请求 2 MiB
    new React\Http\Middleware\RequestBodyParserMiddleware(),
    $handler
);
```

更复杂的示例包括限制可以一次缓冲的请求总数，然后确保实际的请求处理程序仅处理一个请求，而没有任何并发处理:

```php
$server = new React\Http\Server(
    $loop,
    new React\Http\Middleware\StreamingRequestMiddleware(),
    new React\Http\Middleware\LimitConcurrentRequestsMiddleware(100), //100个并发缓冲处理程序
    new React\Http\Middleware\RequestBodyBufferMiddleware(2 * 1024 * 1024), // 每请求 2 MiB
    new React\Http\Middleware\RequestBodyParserMiddleware(),
    new React\Http\Middleware\LimitConcurrentRequestsMiddleware(1), // 仅执行1个处理程序（无并发）
    $handler
);
```

#### RequestBodyBufferMiddleware

`React\Http\Middleware\RequestBodyBufferMiddleware`是一个内置的中间件，它可以用来在内存中缓冲整个传入的请求体。
如果请求处理程序需要完全的PSR-7兼容性，并且不需要默认的流请求体处理，则这很有用。
构造函数接受一个可选参数，即最大请求体大小。
如果未提供，它将使用PHP配置中的` post_max_size `(默认8 MiB)。

(注意，将使用匹配的SAPI的值，这在大多数情况下是CLI配置。)

任何请求主体超过此限制的传入请求都将被接受，但其请求主体将被丢弃(空请求主体)。
这样做是为了避免在内存中保留一个过大的传入请求(例如，考虑一个2GB的文件上传)。
这允许下一个中间件处理程序仍然处理这个请求，但它将看到一个空的请求体。
这类似于PHP的默认行为，即如果超过这个限制则不会解析正文。然而，与PHP的默认行为不同，原始请求体不能通过' PHP://input '获得。

`RequestBodyBufferMiddleware`将缓冲主体已知大小的请求(即指定了` Content-Length `头)
以及主体未知大小的请求(即` Transfer-Encoding: chunked `头)。

所有请求将被缓冲在内存中，直到到达请求主体末尾，然后使用完整的缓冲请求调用下一个中间件处理程序。
类似地，这将立即为具有空请求主体的请求（例如简单的`GET`请求）和已经缓冲的请求（例如由于其他中间件）调用下一个中间件处理程序。

请注意，给定的缓冲区大小限制将分别应用于每个请求。
这意味着，如果您允许2 MiB限制，然后接收1000个并发请求，则仅为这些缓冲区最多可以分配2000 MiB。
因此，强烈建议将其与[`LimitConcurrentRequestsMiddleware`](#limitconcurrentrequestsmiddleware)
（请参阅上文）一起使用，以限制并发请求的总数。

Usage:

```php
$server = new React\Http\Server(
    $loop,
    new React\Http\Middleware\StreamingRequestMiddleware(),
    new React\Http\Middleware\LimitConcurrentRequestsMiddleware(100),  //100个并发缓冲处理程序
    new React\Http\Middleware\RequestBodyBufferMiddleware(16 * 1024 * 1024), // 16 MiB
    function (Psr\Http\Message\ServerRequestInterface $request) {
        // $request->getBody() 的请求体可用，无需流式传输 
        return new React\Http\Message\Response(200);
    },
);
```

#### RequestBodyParserMiddleware

`React\Http\Middleware\RequestBodyParserMiddleware` 接受一个完全缓冲的请求体
(通常来自[`RequestBodyBufferMiddleware`](#requestbodybuffermiddleware))，
并从传入的Http请求体解析表单值和上传文件。

这个中间件处理程序负责从使用` Content-Type: application/x-www-form-urlencoded `或
` Content-Type: multipart/form-data `的HTTP请求中应用值，
以类似PHP默认的超全局变量` $_POST `和` $_FILES `。

与其依赖这些超全局变量，你可以使用PSR-7定义的` $request->getParsedBody() `和` $request->getUploadedFiles() `方法。

相应地，每个文件上传将被表示为实现
[PSR-7 `UploadedFileInterface`](https://www.php-fig.org/psr/psr-7/#36-psrhttpmessageuploadedfileinterface)
的实例。

由于其阻塞性质，` moveTo() `方法不可用，而是抛出`RuntimeException`。

你可以使用` $contents = (string)$file->getStream(); `来访问文件内容并将其持久化到您喜欢的数据存储中。

```php
$handler = function (Psr\Http\Message\ServerRequestInterface $request) {
    // 如果有的话，现在可以从 $request->getParsedBody() 中获得解析的表单字段
    $body = $request->getParsedBody();
    $name = isset($body['name']) ? $body['name'] : 'unnamed';

    $files = $request->getUploadedFiles();
    $avatar = isset($files['avatar']) ? $files['avatar'] : null;
    if ($avatar instanceof Psr\Http\Message\UploadedFileInterface) {
        if ($avatar->getError() === UPLOAD_ERR_OK) {
            $uploaded = $avatar->getSize() . ' bytes';
        } elseif ($avatar->getError() === UPLOAD_ERR_INI_SIZE) {
            $uploaded = 'file too large';
        } else {
            $uploaded = 'with error';
        }
    } else {
        $uploaded = 'nothing';
    }

    return new React\Http\Message\Response(
        200,
        array(
            'Content-Type' => 'text/plain'
        ),
        $name . ' uploaded ' . $uploaded
    );
};

$server = new React\Http\Server(
    $loop,
    new React\Http\Middleware\StreamingRequestMiddleware(),
    new React\Http\Middleware\LimitConcurrentRequestsMiddleware(100),  //100个并发缓冲处理程序
    new React\Http\Middleware\RequestBodyBufferMiddleware(16 * 1024 * 1024), // 16 MiB
    new React\Http\Middleware\RequestBodyParserMiddleware(),
    $handler
);
```

详情请参阅 [form upload server示例](https://github.com/reactphp/http/blob/v1.2.0/examples/62-server-form-upload.php)

默认情况下，中间件遵循[`upload_max_filesize`](https://www.php.net/manual/en/ini.core.php#ini.upload-max-filesize)
(默认`2M`) ini设置。
超过此限制的文件将被拒绝，错误为` UPLOAD_ERR_INI_SIZE `。
你可以通过显式地将最大文件大小(以字节为单位)作为第一个参数传递给构造函数来控制每个文件上传的最大文件大小，如下所示:

```php
new React\Http\Middleware\RequestBodyParserMiddleware(8 * 1024 * 1024); // 每个请求文件8 MiB
```

默认情况下，这个中间件尊重[`file_uploads`](https://www.php.net/manual/en/ini.core.php#ini.file-uploads) (默认的` 1 `)和
[` max_file_uploads `](https://www.php.net/manual/en/ini.core.php#ini.max-file-uploads) (默认的` 20 `)ini设置。

这些设置可以控制是否有文件，以及在单个请求中可以上传多少文件。
如果在单个请求中上传更多文件，额外的文件将被忽略，` getUploadedFiles() `方法将返回一个截断的数组。

请注意，提交时保留为空的上载字段不计入此限制。

你可以通过像这样显式地将第二个参数传递给构造函数来控制每个请求上传文件的最大数量:

```php
new React\Http\Middleware\RequestBodyParserMiddleware(10 * 1024, 100); //100个文件，每个10 KiB
```

>请注意，此中间件处理程序仅解析请求主体中已缓冲的所有内容。
 如上例所示，请求主体必须由先前的中间件处理程序缓冲。
 该先前的中间件处理程序还负责拒绝超出允许的消息大小（例如大文件上传）的传入请求。
 上面使用的[`RequestBodyBufferMiddleware`](requestbodybuffermiddleware)
 仅丢弃过多的请求主体，从而导致主体为空。
 如果您使用此中间件而没有先进行缓冲，则它将尝试解析一个空的（流式）主体，因此可能会假定一个空的数据结构。
 更多细节请参阅[`RequestBodyBufferMiddleware`](#equestbodybuffermiddleware)。
  
>该中间件遵循PHP的`MAX_FILE_SIZE`隐藏字段。
 超过此限制的文件将被拒绝，并显示`UPLOAD_ERR_FORM_SIZE`错误。

>该中间件遵循[`max_input_vars`](https://www.php.net/manual/en/info.configuration.php#ini.max-input-vars) （默认为`1000`）和
 [`max_input_nesting_level`](https://www.php.net/manual/en/info.configuration.php#ini.max-input-nesting-level) （默认为`64`）ini设置。
 
>请注意，此中间件会忽略[`enable_post_data_reading`](https://www.php.net/manual/en/ini.core.php#ini.enable-post-data-reading) （默认为1）ini 设置，
 因为在这里没有什么意义，只能由更高级别的实现来完成。
 
>如果您想遵循这个设置，就必须检查它的值，并有效地避免完全使用这个中间件。

## 安装

推荐的安装这个库的方法是[通过Composer](https://getcomposer.org)。
[Composer 新手?](https://getcomposer.org/doc/00-intro.md)

该项目遵循[SemVer](https://semver.org/) ，
默认安装最新支持的版本:

```bash
$ composer require react/http:^1.2
```

有关版本升级的详细信息，请参见[CHANGELOG](https://reactphp.org/http/changelog.html)

该项目旨在在任何平台上运行，因此不需要任何PHP扩展，并支持通过 `PHP 7+`和`HHVM在旧版PHP 5.3`上运行。

强烈推荐在这个项目中使用*PHP 7+*。

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
如果您不想运行这些，则可以像这样跳过它们:

```bash
$ php vendor/bin/phpunit --exclude-group internet
```

## License

MIT, see [LICENSE file](https://reactphp.org/http/license.html).
