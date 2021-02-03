# EventLoop

[![Build Status](https://travis-ci.org/reactphp/event-loop.svg?branch=master)](https://travis-ci.org/reactphp/event-loop)

[ReactPHP](https://reactphp.org/)的核心事件循环，用于事件I/O

为了使基于异步的库可互操作，它们需要使用相同的事件循环。 该组件提供了一个任何库都可以定位的通用公共`LoopInterface`，
这使它们可以在同一循环中使用，并由用户控制一个 [`run()`](#run)调用。

**目录**

* [快速开始](#快速开始)
* [用法](#用法)
  * [Factory](#factory)
    * [create()](#create)
  * [Loop implementations](#loop-implementations)
    * [StreamSelectLoop](#streamselectloop)
    * [ExtEventLoop](#exteventloop)
    * [ExtLibeventLoop](#extlibeventloop)
    * [ExtLibevLoop](#extlibevloop)
    * [ExtEvLoop](#extevloop)
    * [ExtUvLoop](#extuvloop)
  * [LoopInterface](#loopinterface)
    * [run()](#run)
    * [stop()](#stop)
    * [addTimer()](#addtimer)
    * [addPeriodicTimer()](#addperiodictimer)
    * [cancelTimer()](#canceltimer)
    * [futureTick()](#futuretick)
    * [addSignal()](#addsignal)
    * [removeSignal()](#removesignal)
    * [addReadStream()](#addreadstream)
    * [addWriteStream()](#addwritestream)
    * [removeReadStream()](#removereadstream)
    * [removeWriteStream()](#removewritestream)
* [安装](#安装)
* [测试](#测试)
* [License](#license)

## 快速开始

这里是一个仅使用事件循环构建的异步HTTP服务器。

```php
$loop = React\EventLoop\Factory::create();

$server = stream_socket_server('tcp://127.0.0.1:8080');
stream_set_blocking($server, false);

$loop->addReadStream($server, function ($server) use ($loop) {
    $conn = stream_socket_accept($server);
    $data = "HTTP/1.1 200 OK\r\nContent-Length: 3\r\n\r\nHi\n";
    $loop->addWriteStream($conn, function ($conn) use (&$data, $loop) {
        $written = fwrite($conn, $data);
        if ($written === strlen($data)) {
            fclose($conn);
            $loop->removeWriteStream($conn);
        } else {
            $data = substr($data, $written);
        }
    });
});

$loop->addPeriodicTimer(5, function () {
    $memory = memory_get_usage() / 1024;
    $formatted = number_format($memory, 3).'K';
    echo "Current memory usage: {$formatted}\n";
});

$loop->run();
```

查看[示例](https://github.com/reactphp/event-loop/blob/v1.1.1/examples).

## 用法

典型的应用程序:开始时创建的单个事件循环，程序结束时运行。

```php
// [1]
$loop = React\EventLoop\Factory::create();

// [2]
$loop->addPeriodicTimer(1, function () {
    echo "Tick\n";
});

$stream = new React\Stream\ReadableResourceStream(
    fopen('file.txt', 'r'),
    $loop
);

// [3]
$loop->run();
```

1. 在程序开始时创建循环实例[`React\EventLoop\Factory::create()`](#create) 
并选择最佳的可用[循环实现](#loop-implementations).
2.循环实例可直接使用或传递给库和应用程序代码。
在此示例中，向事件循环注册了一个定期定时器，该循环每秒输出一次 `Tick` ，
并使用ReactPHP[stream组件](Stream.md)
创建[可读流](Stream.md#readablestreaminterface)进行演示。

3.循环在程序最后通过单个[`$loop->run()`](#run)运行。

### Factory

`Factory` 类是一个提供便捷创建循环实例的最佳类
[事件循环实现](#loop-implementations).

#### create()

`create():LoopInterface` 方法可用于创建新的事件循环
例子:

```php
$loop = React\EventLoop\Factory::create();
```

该方法返回实现[`LoopInterface`](#loopinterface)的实例，[事件循环实现](#loop-implementations)是一个具体实现。

该方法通常只能在程序开始时调用一次。

### Loop implementations

除了[`LoopInterface`](#loopinterface)之外，还有一些提供了事件循环实现。

所有事件循环都支持以下功能:

* 文件描述符轮询
* 一次性定时器
* 周期定时器
* 在未来循环中延迟执行

对于此软件包的大多数使用者而言，底层事件循环实现是具体实现。
您应该使用[`Factory`](#factory)自动创建一个新实例。

高级！ 如果您明确需要某个事件循环实现，则可以手动实例化以下类之一。

请注意，您可能必须为其安装相应的PHP扩展，否则它们将在创建时抛出 `BadMethodCallException` 异常。 

#### StreamSelectLoop

基于`stream_select()`的事件循环。
使用[`stream_select()`](https://www.php.net/manual/en/function.stream-select.php) 函数，它是唯一一个使用PHP开箱即用的实现。

在php5.3到php7+和HHVM上，这个事件循环是开箱即用的。这意味着不需要额外安装其他扩展，而且这个库可以在所有支持的PHP的平台上工作。
因此，如果您没有安装下面列出的事件循环扩展， [`Factory`](#factory)将默认使用此事件循环。

在后台，它执行一个简单的 `select` 系统调用。
该系统调用限于 `FD_SETSIZE` 的最大文件描述符数量（取决于平台，通常为1024），并以 `O(m)` （ `m` 是传递的最大文件描述符数量）进行缩放。
这意味着在同时处理数千个流时，您可能会遇到问题，并且在这种情况下，您可能想研究使用下面列出的事件循环实现之一替代。
如果您的用例属于仅涉及一次处理数十个或几百个流，则此事件循环实现的执行效果非常好。 

如果要使用信号处理（另请参阅下面的[`addSignal()`](#addsignal) ），此事件循环实现需要安装`ext-pcntl`扩展。 

此扩展仅适用于类Unix平台，不支持Windows, 该扩展通常作为许多PHP发行版的一部分安装。

如果缺少此扩展名（或您正在Windows上运行），则不支持信号处理，将抛出`BadMethodCallException`异常。 

PHP 7.3之前的版本，该事件循环都依赖于时钟时间来安排将来的定时器，因为单调时间源仅从PHP 7.3+ 可用（`hrtime()`）。

尽管这并不影响大部分应用程序，但这对于依赖于高时间精度的程序或受不连续时间调整（时间跳跃）影响的系统而言，是一个重要的区别。
这意味着，如果您安排一个定时器在PHP <7.3上的30秒内触发，然后将系统时间向前调整20秒，则该定时器可能会在10秒内触发。
另请参阅[`addTimer()`](#addtimer)。

#### ExtEventLoop

基于`ext-event`的事件循环。 

使用[`event` PECL extension](https://pecl.php.net/package/event) 扩展，与libevent相同。 

此循环可用于PHP 5.4到PHP 7+。

#### ExtEvLoop

基于`ext-ev`的事件循环。 

使用 [`ev` PECL extension](https://pecl.php.net/package/ev) 扩展，提供 `libev` 库的接口。

此循环可用于PHP 5.4到PHP 7+。

#### ExtUvLoop

基于`ext-uv`的事件循环。 

使用 [`uv` PECL extension](https://pecl.php.net/package/uv) 扩展， 提供 `libuv` 库的接口。

此循环可用于PHP 7+。

#### ExtLibeventLoop

基于`ext-libevent`的事件循环。 

使用 [`libevent` PECL extension](https://pecl.php.net/package/libevent) 扩展。
`libevent` 本身支持许多特定于系统的后端（epoll，kqueue）。 

此事件循环仅适用于PHP 5。 

存在用于PHP 7 的版本 [非正式更新](https://github.com/php/pecl-event-libevent/pull/2) ，但已知由于SEGFAULT导致定期崩溃。

重申一下:不建议在PHP 7上使用此事件循环。
因此[`Factory`](#factory)将不会尝试在PHP 7上使用此事件循环。 

已知只有在流变得可读（边缘触发）时才触发事件侦听器，并且如果从一开始就已经可读该流，则可能不会触发。 

这也意味着，当数据仍留在PHP的内部流缓冲区中时，该流可能不被视为可读。 

因此，在这种情况下，建议使用`stream_set_read_buffer($stream, 0);`禁用PHP的内部读取缓冲区。
另请参阅[`addReadStream()`](#addreadstream)。 

#### ExtLibevLoop

基于`ext-libev`的事件循环。 

使用[非正式的 `libev` 扩展名](https://github.com/m4rw3r/php-libev) 扩展，与libevent相同。 

此循环仅适用于PHP 5。
PHP 7的更新[不太可能](https://github.com/m4rw3r/php-libev/issues/8) 

### LoopInterface

#### run()

使用 `run(): void` 方法执行事件循环，直到没有任务执行为止。

对于大部分应用程序，该方法是事件循环上唯一直接可见的调用。

根据经验，通常建议将所有内容附加到同一循环实例，然后在应用程序的最底端运行一次循环。

```php
$loop->run();
```

此方法将使循环保持运行状态，直到没有其他任务可以执行为止。 换句话说:此方法将阻塞直到最后一个定时器，流 和/或 信号被删除为止。

同样，必须确保应用程序实际调用此方法一次。如果将侦听器添加到循环中而没有执行该方法，任何附加的监听器将不再等待，应用程序会直接退出。

循环已在运行时，不能调用此方法。此方法在显式调用[`stop()`ped](#stop)后或由于以前不再有任何操作而自动停止后，可能会被多次调用。

#### stop()

运行`stop(): void`方法将停止正在运行的事件循环。

此方法为高级用法，应小心使用。通常建议仅当循环不再有任何事情要做时才自动停止。

此方法用于显式指示事件循环停止:

```php
$loop->addTimer(3.0, function () use ($loop) {
    $loop->stop();
});
```
对当前未运行的循环实例或已停止的循环实例调用此方法无效。

#### addTimer()

`addTimer(float $interval, callable $callback): TimerInterface` 方法可用于将要在设定间隔（秒）后调用回调。

定时器回调函数必须能够接受单个参数，定时器实例也是由这个方法返回的，或者您可以使用一个完全没有参数的函数。

定时器回调函数不能抛出`Exception`。

定时器回调函数的返回值将被忽略，并且没有任何影响，因此出于性能原因，建议您不要返回任何过多的数据结构。

与[`addPeriodicTimer()`](#addperiodictimer)不同，此方法将确保在设定间隔（秒）后只调用一次回调。

您可以调用[`cancelTimer`](#canceltimer)来取消挂起的定时器。

```php
$loop->addTimer(0.8, function () {
    echo 'world!' . PHP_EOL;
});

$loop->addTimer(0.3, function () {
    echo 'hello ';
});
```

[示例#1](https://github.com/reactphp/event-loop/blob/v1.1.1/examples).

如果要在回调函数中访问变量，可以将任意数据变量通过 `use` 绑定到回调闭包中，如下所示:

```php
function hello($name, LoopInterface $loop)
{
    $loop->addTimer(1.0, function () use ($name) {
        echo "hello $name\n";
    });
}

hello('Tester', $loop);
```

此接口不强制任意间隔的定时器，因此如果您依赖毫秒或以下的非常高的精度，可能需要特别小心。除非另有说明，否则事件循环实现应尽力提供高精度间隔，并应至少提供毫秒精度。
已知许多现有的事件循环实现提供微秒精度，但通常不建议依赖这种高精度。

类似地，不能保证在同一时间（在其可能的精度范围内）被调度执行的定时器的执行顺序。

此接口建议事件循环实现应使用单调时间源（如果可用）。假设单调时间源在默认情况下仅在php7.3中可用，那么事件循环实现可能会退回到使用挂钟时间。

虽然这不会影响许多常见的用例，但对于依赖高时间精度或受不连续时间调整（时间跳跃）影响的系统的程序来说，这是很重要的一点。

这意味着如果你安排一个定时器在30秒后触发，然后调整你的系统时间向前20秒，定时器仍应在30秒后触发。

有关详细信息，请参阅[事件循环实现](#loop-implementations) 

#### addPeriodicTimer()

`addPeriodicTimer(float $interval, callable $callback): TimerInterface` 方法可用于将要在设定间隔（秒）后重复调用的回调。

定时器回调函数必须能够接受单个参数，定时器实例也是由这个方法返回的，或者您可以使用一个完全没有参数的函数。

定时器回调函数不能抛出`Exception`。

定时器回调函数的返回值将被忽略，并且没有任何影响，因此出于性能原因，建议您不要返回任何过多的数据结构。

与[`addTimer()`](#addtimer)不同，此方法将确保在设定间隔（秒）后重复调用回调。

您可以调用[`cancelTimer`](#canceltimer)来取消挂起的定时器。

```php
$timer = $loop->addPeriodicTimer(0.1, function () {
    echo 'tick!' . PHP_EOL;
});

$loop->addTimer(1.0, function () use ($loop, $timer) {
    $loop->cancelTimer($timer);
    echo 'Done' . PHP_EOL;
});
```

[示例#2](https://github.com/reactphp/event-loop/blob/v1.1.1/examples).

如果要限制执行次数，可以将控制变量通过`use`绑定的回调闭包中，如下所示:

```php
function hello($name, LoopInterface $loop)
{
    $n = 3;
    $loop->addPeriodicTimer(1.0, function ($timer) use ($name, $loop, &$n) {
        if ($n > 0) {
            --$n;
            echo "hello $name\n";
        } else {
            $loop->cancelTimer($timer);
        }
    });
}

hello('Tester', $loop);
```


此接口不强制任意间隔的定时器，因此如果您依赖毫秒或以下的非常高的精度，可能需要特别小心。除非另有说明，否则事件循环实现应尽力提供高精度间隔，并应至少提供毫秒精度。
已知许多现有的事件循环实现提供微秒精度，但通常不建议依赖这种高精度。

类似地，不能保证在同一时间（在其可能的精度范围内）被调度执行的定时器的执行顺序。

此接口建议事件循环实现应使用单调时间源（如果可用）。假设单调时间源在默认情况下仅在php7.3中可用，那么事件循环实现可能会退回到使用挂钟时间。

虽然这不会影响许多常见的用例，但对于依赖高时间精度或受不连续时间调整（时间跳跃）影响的系统的程序来说，这是很重要的一点。

这意味着如果你安排一个定时器在30秒后触发，然后调整你的系统时间向前20秒，定时器仍应在30秒后触发。

有关详细信息，请参阅[事件循环实现](#loop-implementations) 

此外，由于每次调用后都要进行重新调度，周期性定时器可能会发生定时器漂移。 因此，通常不建议在毫秒级或以下的高精度间隔中使用此方法。

#### cancelTimer()

`cancelTimer(TimerInterface $timer): void` 方法可用于取消待处理的定时器。 

[`addPeriodicTimer()`](#addperiodictimer) [示例#2](https://github.com/reactphp/event-loop/blob/v1.1.1/examples).

对没有添加到循环实例的定时器或已取消的定时器调用此方法无效。

#### futureTick()

`futureTick(callable $listener): void` 方法可用于安排在事件循环的未来时刻调用的回调。

这与间隔为 `0` 秒的定时器非常相似，但是它不会被插入定时器队列中，进而减少队列开销。 

tick回调函数必须能够接受零参数。tick回调函数不抛出`Exception`。
tick回调函数的返回值将被忽略并且不起作用，因此出于性能原因，建议您不要返回任何过多的数据结构。

如果要在回调函数中访问变量，可以将任意数据变量通过 `use` 绑定到回调闭包中，如下所示:

```php
function hello($name, LoopInterface $loop)
{
    $loop->futureTick(function () use ($name) {
        echo "hello $name\n";
    });
}

hello('Tester', $loop);
```
与定时器不同，tick回调保证按其入队的顺序执行。同样，一旦将回调放入队列，就无法取消此操作。

这通常用于将较大的任务分解为较小的步骤（一种协作式多任务处理形式）。

```php
$loop->futureTick(function () {
    echo 'b';
});
$loop->futureTick(function () {
    echo 'c';
});
echo 'a';
```

[示例#3](https://github.com/reactphp/event-loop/blob/v1.1.1/examples).

#### addSignal()

`addSignal(int $signal, callable $listener): void` 方法可用于注册一个侦听器，以便在此过程捕获到信号时得到通知。

这对于从`supervisor` 或 `systemd`之类的工具捕获用户中断信号或关闭信号很有用。

通过此方法添加的信号，侦听器回调函数必须能够接受单个参数或者您可以使用完全不带参数的函数。

侦听器回调函数不得抛出 `Exception`

监听器回调函数的返回值将被忽略并且不起作用，因此出于性能原因，建议您不要返回任何过多的数据结构。

```php
$loop->addSignal(SIGINT, function (int $signal) {
    echo 'Caught user interrupt signal' . PHP_EOL;
});
```

[示例#4](https://github.com/reactphp/event-loop/blob/v1.1.1/examples).

信号仅在类似Unix的平台上可用，由于操作系统限制，不支持Windows。
如果此平台不支持信号，例如缺少所需的扩展名，则此方法可能抛出`BadMethodCallException`。

**注意 :一个监听器只能添加到同一信号一次，多次添加将被忽略。**

#### removeSignal()

`removeSignal(int $signal, callable $listener): void` 方法可用于删除先前添加的信号监听器。 

```php
$loop->removeSignal(SIGINT, $listener);
```

删除未注册的监听器将被忽略。

#### addReadStream()

> 高级！ 请注意，此底层API被视为高级用法。 大多数用例可能应该使用更高级别的API来替代 [可读流API](https://github.com/reactphp/stream#readablestreaminterface) 

`addReadStream(resource $stream, callable $callback): void` 方法可用于注册侦听器，以便在流准备好读取时收到通知。

第一个参数必须是一个有效的流资源，它支持检查循环实现是否可以读取它。

单个流资源不能添加多次。

但可以先调用[`removeReadStream()`](#removereadstream)，或者使用单个侦听器对此事件做出反应，然后从此侦听器中进行调度。如果循环实现不支持给定的资源类型，则此方法可能引发`Exception`

添加的流资源侦听器回调函数必须能够接受单个参数或者您可以使用完全没有参数的函数。

侦听器回调函数不能抛出`Exception`

侦听器回调函数的返回值将被忽略，并且没有任何影响，因此出于性能原因，建议您不要返回任何过多的数据结构。

如果要在回调函数中访问变量，可以将任意数据变量通过 `use` 绑定到回调闭包中，如下所示:

```php
$loop->addReadStream($stream, function ($stream) use ($name) {
    echo $name . ' said: ' . fread($stream);
});
```

[示例#11](https://github.com/reactphp/event-loop/blob/v1.1.1/examples).

您可以调用 [`removeReadStream()`](#removereadstream) 删除此流的读取事件侦听器。

无法保证多个流同时准备就绪时侦听器的执行顺序。

某些事件循环实现仅在流*变得*可读（边缘触发）时才触发侦听器，并且如果从一开始就已经可读该流，则可能不会触发。

这也意味着，当数据仍留在PHP的内部流缓冲区中时，该流可能不被视为可读。

在这种情况下，建议使用`stream_set_read_buffer($stream, 0);`禁用PHP的内部读取缓冲区。

#### addWriteStream()
> 高级！ 请注意，此底层API被视为高级用法。大多数用例可能应该使用更高级别的API替代[可写流API](https://github.com/reactphp/stream#writablestreaminterface)

`addWriteStream(resource $stream, callable $callback): void` 方法可用于注册一个侦听器，以便在流准备好写入时得到通知。

第一个参数务必是有效的流资源，它支持检查是否已准备好通过此循环实现进行写入。

单个流资源不得多次添加。
相反，请先调用[`removeWriteStream()`](#removewritestream)或使用单个侦听器对此事件做出反应，然后从该事件进行分派监听。
如果此循环实现不支持给定的资源类型，则此方法可以抛出 `Exception`

添加的流资源侦听器回调函数必须能够接受单个参数或者您可以使用完全没有参数的函数。

侦听器回调函数不得抛出`Exception`

监听器回调函数的返回值将被忽略并且不起作用，因此出于性能原因，建议您不要返回任何过多的数据结构。

如果要在回调函数中访问变量，可以将任意数据变量通过 `use` 绑定到回调闭包中，如下所示:

```php
$loop->addWriteStream($stream, function ($stream) use ($name) {
    fwrite($stream, 'Hello ' . $name);
});
```

[示例#12](https://github.com/reactphp/event-loop/blob/v1.1.1/examples).

您可以调用 [`removeWriteStream()`](#removewritestream) 删除此流的写入事件侦听器。

无法保证多个流同时准备就绪时侦听器的执行顺序。

#### removeReadStream()

`removeReadStream(resource $stream): void` 方法可用于删除给定流的可读事件监听器。 

从循环中删除已删除的流，或尝试删除从未添加过或无效的流时此方法无效。

#### removeWriteStream()

`removeWriteStream(resource $stream): void` 方法可用于删除给定流的可写事件监听器。 

从循环中删除已删除的流，或尝试删除从未添加过或无效的流时此方法无效。

## 安装

推荐安装 [通过Composer](https://getcomposer.org).
[Composer新手?](https://getcomposer.org/doc/00-intro.md)

该项目遵循 [SemVer](https://semver.org/).
这将安装最新的受支持版本:

```bash
$ composer require react/event-loop:^1.1.1
```

另请参阅 [CHANGELOG](https://reactphp.org/event-loop/changelog.html) ，以获取有关版本升级的详细信息。

该项目旨在在任何平台上运行，因此不需要任何PHP扩展，并支持通过 `PHP 7+`和`HHVM在旧版PHP 5.3`上运行。

强烈建议在这个项目中使用PHP7+。

建议安装任何一个事件循环扩展，但完全是可选的。
有关详细信息，请参阅[事件循环实现](#loop-implementations)

## 测试

要运行测试套件，首先需要克隆此存储库，然后安装所有依赖项 [通过Composer](https://getcomposer.org):

```bash
$ composer install
```

要运行测试套件，请转到项目根目录并运行:

```bash
$ php vendor/bin/phpunit
```

## License

MIT, see [LICENSE file](https://reactphp.org/event-loop/license.html).
