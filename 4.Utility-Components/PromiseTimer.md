# PromiseTimer

[![Build Status](https://travis-ci.org/reactphp/promise-timer.svg?branch=master)](https://travis-ci.org/reactphp/promise-timer)

在[ReactPHP](https://reactphp.org/)
的基础上为`Promise`构建的一个简单的超时实现。

**目录**

* [用法](#用法)
  * [timeout()](#timeout)
    * [Timeout cancellation](#timeout-cancellation)
    * [Cancellation handler](#cancellation-handler)
    * [Input cancellation](#input-cancellation)
    * [Output cancellation](#output-cancellation)
    * [Collections](#collections)
  * [resolve()](#resolve)
    * [Resolve cancellation](#resolve-cancellation)
  * [reject()](#reject)
    * [Reject cancellation](#reject-cancellation)
  * [TimeoutException](#timeoutexception)
* [安装](#安装)
* [测试](#测试)
* [License](#license)

## 用法

这个轻量级的库仅包含一些简单的功能。
所有功能都位于`React\Promise\Timer`命名空间下。

以下示例假定您使用与此类似的导入语句：

```php
use React\Promise\Timer;

Timer\timeout(…);
```

或者，也可以用它们完整的命名空间来引用它们:

```php
\React\Promise\Timer\timeout(…);
```

### timeout()

`timeout(PromiseInterface $promise, $time, LoopInterface $loop)` 函数可以用来`取消`*耗时太长*的操作。
您需要传入一个`$promise`，表示一个挂起的操作和超时参数。
它返回一个新的`Promise`，具有以下兑现行为： 

* 如果传入的`$promise`在`$time`秒之前被履行，则用其履行值兑现最终的`promise`。
* 如果传入的`$promise`在`$time`秒之前被拒绝，则以其拒绝值产生拒绝的`promise`。
* 如果传入的`$promise`在`$time`秒之前未兑现，则取消操作并产生 [`TimeoutException`](#timeoutexception) 拒绝的`promise`。

设定的`$time`值将用于启动计时器，该计时器一旦触发将*取消*挂起的操作。
这意味着，如果传递的值很小（或为负），它仍将启动计时器，从而在将来的最早时间触发。

如果传入的`$promise`已经被兑现，那么产生的`promise`将立即履行或拒绝，而无需启动计时器。

仅处理解析值的常见用例如下所示：

```php
$promise = accessSomeRemoteResource();
Timer\timeout($promise, 10.0, $loop)->then(function ($value) {
    // the operation finished within 10.0 seconds
});
```

一个完整的示例如下所示：

```php
$promise = accessSomeRemoteResource();
Timer\timeout($promise, 10.0, $loop)->then(
    function ($value) {
        // the operation finished within 10.0 seconds
    },
    function ($error) {
        if ($error instanceof Timer\TimeoutException) {
            // the operation has failed due to a timeout
        } else {
            // the input operation has failed due to some other error
        }
    }
);
```

或者，如果您使用的是[react/promise v2.2.0](/1.Core-Components/Promise.md)
或更高版本，请执行以下操作：

```php
Timer\timeout($promise, 10.0, $loop)
    ->then(function ($value) {
        // the operation finished within 10.0 seconds
    })
    ->otherwise(function (Timer\TimeoutException $error) {
        // the operation has failed due to a timeout
    })
    ->otherwise(function ($error) {
        // the input operation has failed due to some other error
    })
;
```

#### Timeout cancellation

如上所述，[`timeout()`](#timeout) 函数花费的时间太长，则会*取消*底层操作。最终的承诺会被[`TimeoutException`](#timeoutexception)拒绝。

然而,底层传入`$promise`发生了什么比较棘手的事情:
定时器一旦触发后,我们将尝试调用[`$promise->cancel()`](/1.Core-Components/Promise.md#promiseinterfacecancel)
传入`$promise`依次调用它的[取消处理程序](#cancellation-handler) 。

这意味着实际上是在传入` $promise `来处理[取消支持](/1.Core-Components/Promise.md#promiseinterface)。

* 常见的用例包括清理资源，如打开的网络套接字或文件句柄，或终止外部进程或计时器。
* 如果传入的`$promise`不支持取消，这是不允许的。
  这意味着，虽然最终的promise仍将被拒绝，但底层传入的`$promise`可能仍处于挂起状态，因此可能继续占用资源。
  
有关取消处理程序的详细信息，请参见下一章。

#### Cancellation handler

例如，上述操作的实现如下所示:

```php
function accessSomeRemoteResource()
{
    return new Promise(
        function ($resolve, $reject) use (&$socket) {
            // 创建承诺，就会调用此方法
            // 常见的用例涉及打开资源并最终进行释放资源
            $socket = createSocket();
            $socket->on('data', function ($data) use ($resolve) {
                $resolve($data);
            });
        },
        function ($resolve, $reject) use (&$socket) {
            // 在此promise上调用'cancel()'后，该函数会被调用
            // 常见的用例包括清除资源然后拒绝
            $socket->close();
            $reject(new \RuntimeException('Operation cancelled'));
        }
    );
}
```

在此示例中，`$promise->cancel()`将调用已注册的取消处理程序，该处理程序随后关闭网络套接字并拒绝`Promise`实例。

如果没有向` Promise `构造函数传递取消处理程序，那么调用它的` cancel() `方法无效。可能是挂起状态，可以继续消耗资源。

有关承诺取消的更多细节，请参阅
[Promise文档](/1.Core-Components/Promise.md#promiseinterface)

#### Input cancellation

不管超时处理如何，您也可以在任何时候显式地`cancel()` `$promise`。
因此`timeout()`处理不会影响`$promise`的取消，如下例所示：

```php
$promise = accessSomeRemoteResource();
$timeout = Timer\timeout($promise, 10.0, $loop);

$promise->cancel();
```

注册的[cancellation handler](#cancellation-handler) 负责处理` cancel() `调用:

* 如上所述，常见的应用涉及资源清理，然后*拒绝* `Promise`。
  如果传入的` $promise `被拒绝，那么超时将被中止，产生的promise也将被拒绝。

* 如果传入的` $promise `仍然挂起，那么超时将继续运行，直到计时器到期。
  如果传入` $promise `没有注册[cancellation handler](#cancellation-handler) ，那么超时也继续运行，直到计时器到期。

#### Output cancellation

同样，你也可以像这样显式地` cancel() `生成如下承诺:

```php
$promise = accessSomeRemoteResource();
$timeout = Timer\timeout($promise, 10.0, $loop);

$timeout->cancel();
```

请注意，这与上面的 [input cancellation](#input-cancellation) 示例非常相似。因此，它的行为也非常相似。
对结果`promise`调用`cancel()`只会尝试`cancel()`传入`$promise`。
这意味着我们不承担结果的责任，完全由传入`$promise`来支持取消处理。
注册的 [cancellation handler](#cancellation-handler) 负责处理`cancel()`调用：

* 如上所述，常见的用法涉及到资源清理，然后会*拒绝*这个`$promise`。
  如果传入`$promise`被拒绝，那么超时将被中止，并且产生的承诺也将被拒绝。
* 如果传入`$promise`仍然挂起，则超时将继续运行，直到计时器过期。
  如果传入`$promise`没有注册 [cancellation handler](#cancellation-handler) ，则超时也将继续运行，直到计时器过期。

要再次重申，请注意，对结果`Promise`调用`cancel()`仅会尝试取消传入`$promise`。
然后由传入`$promise`的取消处理程序来解决`$promise`。
如果在发生超时时传入`$promise`仍然兑现，则将触发常规的[timeout cancellation](#timeout-cancellation) 处理，
并通过 [`TimeoutException`](#timeoutexception) 有效地拒绝输出`$promise`。

这样做是为了与 [timeout cancellation](#timeout-cancellation) 处理保持一致，并且还因为通常是这样使用的：

```php
$timeout = Timer\timeout(accessSomeRemoteResource(), 10.0, $loop);

$timeout->cancel();
```

如上所述，该示例按预期工作，并清理了分配给传入`$promise`的所有资源。

请注意，如果传入`$promise`不支持取消，则此操作无效。
这意味着，虽然超时后仍然会拒绝产生的`promise`，但底层传入的`$promise`可能仍然是挂起的，因此可以继续消耗资源。

#### Collections

如果要等待多个promise兑现，则可以使用如下所示的常规promise函数：

```php
$promises = array(
    accessSomeRemoteResource(),
    accessSomeRemoteResource(),
    accessSomeRemoteResource()
);

$promise = \React\Promise\all($promises);

Timer\timeout($promise, 10, $loop)->then(function ($values) {
    // *所有*的诺言得以实现
});
```

适用于所有promise集合的函数，即all()，race()，any()，some()等。

有关Promise函数的更多详细信息，请参考[Promise文档](/1.Core-Components/Promise.md#functions)

### resolve()

可以使用`resolve($time, LoopInterface $loop)`函数来创建一个新的Promise，它以`$time`为履行值，在`$time`秒内解析。

```php
Timer\resolve(1.5, $loop)->then(function ($time) {
    echo 'Thanks for waiting ' . $time . ' seconds' . PHP_EOL;
});
```

在内部，设定的` $time `值将用于启动一个计时器，一旦触发，就会兑现承诺。
这意味着，如果您传递一个非常小(或负值)的值，它仍然会启动一个计时器，因此将在未来尽可能早的时间触发。

#### Resolve cancellation

您可以在任何时候显式`cancel()`生成的计时器承诺：

```php
$timer = Timer\resolve(2.0, $loop);

$timer->cancel();
```

这将中止计时器并通过`RuntimeException`来`拒绝`。

### reject()

函数`reject($time, LoopInterface $loop)`可以用来创建一个新的承诺，该承诺在` $time `秒内通过` TimeoutException `拒绝。

```php
Timer\reject(2.0, $loop)->then(null, function (TimeoutException $e) {
    echo 'Rejected after ' . $e->getTimeout() . ' seconds ' . PHP_EOL;
});
```

设定的`$time`值将用于启动计时器，该计时器一旦触发将拒绝承诺。
这意味着，如果传递的值很小（或为负），它仍将启动计时器，从而在将来的最早时间触发。

此函数是对 [`resolve()`](#resolve) 函数的补充，可以用作更高级别的`Promise`开发者的基本构建块。

#### Reject cancellation

你可以在任何时候显式地` cancel() `产生的计时器承诺:

```php
$timer = Timer\reject(2.0, $loop);

$timer->cancel();
```

这将中止计时器并通过`RuntimeException`来“拒绝”。

### TimeoutException

` TimeoutException `扩展了PHP内置的` RuntimeException `。

`getTimeout()`方法可用于获取以秒为单位的超时值。

## 安装

推荐的安装这个库的方法是[通过Composer](https://getcomposer.org)。
[Composer 新手?](https://getcomposer.org/doc/00-intro.md)

该项目遵循[SemVer](https://semver.org/) ，
默认安装最新支持的版本:

```bash
$ composer require react/promise-timer:^1.6
```

有关版本升级的详细信息，请参见[CHANGELOG](https://reactphp.org/promise-timer/changelog.html) 。

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

## License

MIT, see [LICENSE file](https://reactphp.org/promise-timer/license.html).
