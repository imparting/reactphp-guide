# Dns

[![Build Status](https://travis-ci.org/reactphp/dns.svg?branch=master)](https://travis-ci.org/reactphp/dns)

用于[ReactPHP](https://reactphp.org/) 的异步DNS解析器。 

DNS组件的重点是提供异步DNS解析。
但是，它实际上是用于处理DNS消息的工具包，可以轻松地用于创建DNS服务器。

**目录**

* [基本用法](#基本用法)
* [Caching](#caching)
  * [Custom cache adapter](#custom-cache-adapter)
* [ResolverInterface](#resolverinterface)
  * [resolve()](#resolve)
  * [resolveAll()](#resolveall)
* [高级用法](#高级用法)
  * [UdpTransportExecutor](#udptransportexecutor)
  * [TcpTransportExecutor](#tcptransportexecutor)
  * [SelectiveTransportExecutor](#selectivetransportexecutor)
  * [HostsFileExecutor](#hostsfileexecutor)
* [安装](#安装)
* [测试](#测试)
* [License](#license)
* [参考文献](#参考文献)

## 基本用法

最基本的用法是通过解析器工厂创建解析器。你只需要给它一个`nameserver`，然后你就可以开始解析了。

```php
$loop = React\EventLoop\Factory::create();

$config = React\Dns\Config\Config::loadSystemConfigBlocking();
$server = $config->nameservers ? reset($config->nameservers) : '8.8.8.8';

$factory = new React\Dns\Resolver\Factory();
$dns = $factory->create($server, $loop);

$dns->resolve('igor.io')->then(function ($ip) {
    echo "Host: $ip\n";
});

$loop->run();
```

请参阅[示例](https://github.com/reactphp/dns/blob/v1.4.0/examples)

`Config`类可用于加载系统默认配置。这是一个可以访问文件系统和块的操作。
理想情况下，这个方法应该在循环开始前只执行一次，而不是在循环运行时重复执行。
请注意，如果无法加载系统配置，则此类可能会返回一个*empty*配置。
因此，如果找不到默认`nameserver`，您可能希望像上面那样应用默认`nameserver`。

> 请注意，工厂在创建解析器实例时会从文件系统加载hosts文件一次。
  理想情况下，这个方法应该在循环开始前只执行一次，而不是在循环运行时重复执行。

## Caching

通过将解析程序配置为使用`CachedExecutor`来缓存结果:

```php
$loop = React\EventLoop\Factory::create();

$config = React\Dns\Config\Config::loadSystemConfigBlocking();
$server = $config->nameservers ? reset($config->nameservers) : '8.8.8.8';

$factory = new React\Dns\Resolver\Factory();
$dns = $factory->createCached($server, $loop);

$dns->resolve('igor.io')->then(function ($ip) {
    echo "Host: $ip\n";
});

...

$dns->resolve('igor.io')->then(function ($ip) {
    echo "Host: $ip\n";
});

$loop->run();
```

如果第一个调用在第二个调用之前返回，则将仅执行一个查询。
第二个结果将从内存中的缓存中提供。
这对于需要多次查找相同主机名的长时间运行的脚本特别有用。

请参阅[示例3](https://github.com/reactphp/dns/blob/v1.4.0/examples).

### Custom cache adapter

默认情况下，上述操作将使用内存缓存。

你也可以指定一个自定义缓存实现[`CacheInterface`](4.Utility-Components/Cache.md)来缓存:

```php
$cache = new React\Cache\ArrayCache();
$loop = React\EventLoop\Factory::create();
$factory = new React\Dns\Resolver\Factory();
$dns = $factory->createCached('8.8.8.8', $loop, $cache);
```

另请参阅wiki了解更多其他缓存实现 [cache实现](https://github.com/reactphp/react/wiki/Users#cache-implementations).

## ResolverInterface

### resolve()

`resolve(string $domain): PromiseInterface<string,Exception>` 方法可以用于将给定的`$domain`解析为单个IPv4地址(键入`A`查询)。

```php
$resolver->resolve('reactphp.org')->then(function ($ip) {
    echo 'IP for reactphp.org is ' . $ip . PHP_EOL;
});
```

向DNS服务器发送给定`$domain`的DNS查询，成功后返回单个IP地址。

如果DNS服务器为该查询发送一个包含多个IP地址的DNS响应消息，它将从响应中随机选择一个IP地址。
如果你想要完整的IP地址列表或者想要发送不同类型的查询，你可使用[`resolveAll()`](#resolveall) 方法。

如果DNS服务器发送指示错误代码的DNS响应消息，则此方法将拒绝并返回`RecordNotFoundException`。 它的消息和代码可用于检查响应代码。

如果DNS通信失败，并且服务器没有以有效的响应消息进行响应，则该消息将以`Exception`拒绝。

可以通过取消其待处理的承诺来取消待处理的DNS查询，如下所示:

```php
$promise = $resolver->resolve('reactphp.org');

$promise->cancel();
```

### resolveAll()

`resolveAll(string $host, int $type): PromiseInterface<array,Exception>`方法可用于解析给定`$domain`的所有记录值并查询$type。

```php
$resolver->resolveAll('reactphp.org', Message::TYPE_A)->then(function ($ips) {
    echo 'IPv4 addresses for reactphp.org ' . implode(', ', $ips) . PHP_EOL;
});

$resolver->resolveAll('reactphp.org', Message::TYPE_AAAA)->then(function ($ips) {
    echo 'IPv6 addresses for reactphp.org ' . implode(', ', $ips) . PHP_EOL;
});
```

这是此程序包中的主要方法之一。它会将给定`$domain`的DNS查询发送到您的DNS服务器，
并在成功后返回一个包含所有记录值的列表。

如果DNS服务器发送的DNS响应消息包含此查询的一个或多个记录，它将返回一个列表，其中包含响应中的所有记录值。
您可以使用`Message :: TYPE_ *`常量来控制将发送哪种查询类型。
请注意，此方法始终返回记录值列表，但是每种记录值类型取决于查询类型。
例如，它返回用于A型查询的IPv4地址，用于AAAA型查询的IPv6地址，用于NS型，CNAME和PTR型查询的主机名以及用于其他查询的结构化数据。
有关更多详细信息，请参见`Record`文档。

如果DNS服务器发送指示错误代码的DNS响应消息，则此方法将拒绝并返回`RecordNotFoundException`。它的消息和代码可用于检查响应代码。

如果DNS通信失败，并且服务器没有以有效的响应消息进行响应，则该消息将以`Exception`拒绝。

可以通过取消其待处理的承诺来取消待处理的DNS查询，如下所示:

```php
$promise = $resolver->resolveAll('reactphp.org', Message::TYPE_AAAA);

$promise->cancel();
```

## 高级用法

### UdpTransportExecutor

` UdpTransportExecutor `可用于通过UDP传输发送DNS查询。

这是发送一个DNS查询到您的DNS服务器的主类，由`Resolver`内部用于实际的消息传输。

对于更高级的用法，可以直接使用这个类。

下面的示例查找`igor.io`的`IPv6`地址。
```php
$loop = Factory::create();
$executor = new UdpTransportExecutor('8.8.8.8:53', $loop);

$executor->query(
    new Query($name, Message::TYPE_AAAA, Message::CLASS_IN)
)->then(function (Message $message) {
    foreach ($message->answers as $answer) {
        echo 'IPv6: ' . $answer->data . PHP_EOL;
    }
}, 'printf');

$loop->run();
```

请参阅 [示例4](https://github.com/reactphp/dns/blob/v1.4.0/examples).

请注意，此执行器未实现超时机制，您可能希望与`TimeoutExecutor`结合使用，如下所示:

```php
$executor = new TimeoutExecutor(
    new UdpTransportExecutor($nameserver, $loop),
    3.0,
    $loop
);
```

另请注意，此执行器使用不可靠的UDP传输，并且它不实现任何重试逻辑，因此您可能希望与`RetryExecutor`结合使用，如下所示:

```php
$executor = new RetryExecutor(
    new TimeoutExecutor(
        new UdpTransportExecutor($nameserver, $loop),
        3.0,
        $loop
    )
);
```

注意，这个执行器是完全异步的，因此允许并发执行任意数量的查询。
您应该限制应用程序中并发查询的数量，否则很可能会在解析器端面临速率限制和禁令。
对于许多常见的应用程序，当第一个查询仍然挂起时，你可能想要避免多次发送相同的查询，
所以你可能想要结合使用`CoopExecutor`:

```php
$executor = new CoopExecutor(
    new RetryExecutor(
        new TimeoutExecutor(
            new UdpTransportExecutor($nameserver, $loop),
            3.0,
            $loop
        )
    )
);
```

>此类仅出于组织原因而使用PHP的UDP套接字，并且没有利用 [react/datagram](2.Network-Components/Datagram.md)
 的优势，以避免两个包之间的循环依赖。 高级组件应该利用数据报组件，而不是从头重新实现这个套接字逻辑。

### TcpTransportExecutor

`TcpTransportExecutor`类可用于通过TCP / IP流传输发送DNS查询。

这是将DNS查询发送到DNS服务器的主要类之一。

对于更高级的用法，可以直接使用此类。
以下示例查询`reactphp.org`的`IPv6`地址。

```php
$loop = Factory::create();
$executor = new TcpTransportExecutor('8.8.8.8:53', $loop);

$executor->query(
    new Query($name, Message::TYPE_AAAA, Message::CLASS_IN)
)->then(function (Message $message) {
    foreach ($message->answers as $answer) {
        echo 'IPv6: ' . $answer->data . PHP_EOL;
    }
}, 'printf');

$loop->run();
```

请参阅[示例#92](https://github.com/reactphp/dns/blob/v1.4.0/examples)

请注意，此执行器未实现超时机制，您可能希望与`TimeoutExecutor`结合使用，如下所示:

```php
$executor = new TimeoutExecutor(
    new TcpTransportExecutor($nameserver, $loop),
    3.0,
    $loop
);
```

与`UdpTransportExecutor`不同，该类使用可靠的TCP/IP传输，因此不必实现任何重试逻辑。
注意，这个执行器是完全异步的，因此允许并发执行查询。第一个查询将建立到DNS服务器的TCP/IP套接字连接，该连接将在短时间内保持打开状态。

其他查询将自动重用到DNS服务器的这个现有套接字连接，将在这个连接上管道多个请求，并将在短时间内保持空闲连接打开。
如果您只是偶尔发送查询，初始TCP/IP连接开销可能会导致轻微的延迟 - 当在现有连接上发送大量并发查询时，
它会变得越来越高效，并避免创建许多并发套接字(如基于udp的executor)。
您可能仍然希望限制应用程序中的(并发)查询数量，或者您可能面临解析器端的速率限制和禁令。
对于许多常见的应用程序，当第一个查询仍然挂起时，你可能想要避免多次发送相同的查询，所以你可能想要结合使用` CoopExecutor `:

```php
$executor = new CoopExecutor(
    new TimeoutExecutor(
        new TcpTransportExecutor($nameserver, $loop),
        3.0,
        $loop
    )
);
```

>这个类使用PHP的TCP/IP套接字，并且没有利用 [react/socket](2.Network-Components/Socket.md)
的优势，以避免两个包之间的循环依赖。高级组件应该利用套接字组件，而不是从头重新实现这个套接字逻辑。

### SelectiveTransportExecutor

` SelectiveTransportExecutor `类可用于通过UDP或TCP/IP流传输发送DNS查询。
此类将自动选择正确的传输协议向DNS服务器发送DNS查询。它总是首先尝试通过更有效的UDP传输发送它。
如果此查询产生与大小相关的问题(截断消息)，则它将通过TCP/IP传输重试。

对于更高级的用法，可以直接使用此类。
以下示例查询`reactphp.org`的`IPv6`地址。

```php
$executor = new SelectiveTransportExecutor($udpExecutor, $tcpExecutor);

$executor->query(
    new Query($name, Message::TYPE_AAAA, Message::CLASS_IN)
)->then(function (Message $message) {
    foreach ($message->answers as $answer) {
        echo 'IPv6: ' . $answer->data . PHP_EOL;
    }
}, 'printf');
```

注意，这个执行器仅实现为给定DNS查询选择正确传输的逻辑。
实现正确的传输逻辑、实现超时和任何重试逻辑都由给定的executor决定，
更多细节请参见 [`UdpTransportExecutor`](#udptransportexecutor) 和
[`TcpTransportExecutor`](#tcptransportexecutor)

注意，这个执行器是完全异步的，因此允许并发执行任意数量的查询。
您应该限制应用程序中并发查询的数量，否则很可能会在解析器端面临速率限制和禁令。
对于许多常见的应用程序，当第一个查询仍然挂起时，你可能想要避免多次发送相同的查询，
所以你可能想要像这样结合使用` CoopExecutor `:

```php
$executor = new CoopExecutor(
    new SelectiveTransportExecutor(
        $datagramExecutor,
        $streamExecutor
    )
);
```

### HostsFileExecutor

注意，上面的`UdpTransportExecutor`类总是执行一个实际的DNS查询。
如果你还想从你的hosts文件中获取条目，你可以使用以下代码:

```php
$hosts = \React\Dns\Config\HostsFile::loadFromPathBlocking();

$executor = new UdpTransportExecutor('8.8.8.8:53', $loop);
$executor = new HostsFileExecutor($hosts, $executor);

$executor->query(
    new Query('localhost', Message::TYPE_A, Message::CLASS_IN)
);
```

## 安装

推荐的安装这个库的方法是[通过Composer](https://getcomposer.org)。
[Composer 新手?](https://getcomposer.org/doc/00-intro.md)

该项目遵循[SemVer](https://semver.org/) ，
默认安装最新支持的版本:

```bash
$ composer require react/dns:^1.4
```

有关版本升级的详细信息，请参见[CHANGELOG](https://reactphp.org/dns/changelog.html) 。

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

MIT, see [LICENSE file](https://reactphp.org/dns/license.html).

## 参考文献

* [RFC 1034](https://tools.ietf.org/html/rfc1034) 域名-概念和设施
* [RFC 1035](https://tools.ietf.org/html/rfc1035) 域名-实现和规范
