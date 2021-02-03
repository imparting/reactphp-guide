# Datagram

[![Build Status](https://travis-ci.org/reactphp/datagram.svg?branch=master)](https://travis-ci.org/reactphp/datagram)

[ReactPHP](https://reactphp.org)的事件驱动UDP数据报套接字客户端和服务器。 

## 快速开始

[安装](#安装)后，可以使用以下代码连接到正在侦听`localhost:1234`的UDP服务器，发送和接收UDP数据报:

```php
$loop = React\EventLoop\Factory::create();
$factory = new React\Datagram\Factory($loop);

$factory->createClient('localhost:1234')->then(function (React\Datagram\Socket $client) {
    $client->send('first');

    $client->on('message', function($message, $serverAddress, $client) {
        echo 'received "' . $message . '" from ' . $serverAddress. PHP_EOL;
    });
});

$loop->run();
```

参阅 [示例](https://github.com/reactphp/datagram/blob/v1.5.0/examples).

## 用法

这个库的API是按照node建模的。[UDP / Datagram Sockets (dgram.Socket)](https://nodejs.org/api/dgram.html)

## 安装

推荐的安装这个库的方法是[通过Composer](https://getcomposer.org)。
[Composer 新手?](https://getcomposer.org/doc/00-intro.md)

该项目遵循[SemVer](https://semver.org/) ，
默认安装最新支持的版本:

```bash
$ composer require react/datagram:^1.5
```

有关版本升级的详细信息，请参阅[CHANGELOG](https://reactphp.org/datagram/changelog.html)

该项目旨在在任何平台上运行，因此不需要任何PHP扩展，并支持通过当前*PHP 7+*和*HHVM在旧版PHP 5.3*上运行。
强烈建议对此项目使用*PHP 7+*。

## 测试

要运行测试套件，首先需要克隆这个存储库，然后安装所有依赖项[通过Composer](https://getcomposer.org):

```bash
$ composer install
```

要运行测试套件，请转到项目根目录并运行:

```bash
$ php vendor/bin/phpunit
```

## License

MIT, see [LICENSE file](https://reactphp.org/datagram/license.html).
