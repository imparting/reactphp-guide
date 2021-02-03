Promise
=======

PHP的 [CommonJS Promises/A](http://wiki.commonjs.org/wiki/Promises/A) 的轻量级实现。

[![Build Status](https://travis-ci.org/reactphp/promise.svg?branch=master)](http://travis-ci.org/reactphp/promise)
[![Coverage Status](https://coveralls.io/repos/github/reactphp/promise/badge.svg?branch=master)](https://coveralls.io/github/reactphp/promise?branch=master)

> 主分支包含即将发布的3.0版本的代码。
> 对于当前稳定的2.x版本的代码，请检查 [2.x分支](https://github.com/reactphp/promise/tree/2.x)

> 即将发布的3.0版本将是此软件包的主发展方向。
> 但是，对于尚未安装PHP 7+的用户，我们仍将积极支持2.0和1.0。

目录
-----------------

1. [简介](#简介)
2. [概念](#概念)
   * [Deferred（延迟）](#deferred（延迟）)
   * [Promise（承诺）](#promise（承诺）)
3. [API](#api)
   * [Deferred](#deferred)
     * [Deferred::promise()](#deferredpromise)
     * [Deferred::resolve()](#deferredresolve)
     * [Deferred::reject()](#deferredreject)
   * [PromiseInterface](#promiseinterface)
     * [PromiseInterface::then()](#promiseinterfacethen)
     * [PromiseInterface::done()](#promiseinterfacedone)
     * [PromiseInterface::otherwise()](#promiseinterfaceotherwise)
     * [PromiseInterface::always()](#promiseinterfacealways)
     * [PromiseInterface::cancel()](#promiseinterfacecancel)
   * [Promise](#promise-2)
   * [Functions](#functions)
     * [resolve()](#resolve)
     * [reject()](#reject)
     * [all()](#all)
     * [race()](#race)
     * [any()](#any)
     * [some()](#some)
     * [map()](#map)
     * [reduce()](#reduce)
   * [PromisorInterface](#promisorinterface)
4. [示例](#示例)
   * [如何使用Deferred](#如何使用deferred)
   * [承诺转发的工作原理](#承诺转发的工作原理)
     * [转发履行](#转发履行)
     * [转发拒绝](#转发拒绝)
     * [转发履行和拒绝](#转发履行和拒绝)
   * [done() vs. then()](#done-vs-then)
5. [Credits](#credits)
6. [License](#license)

## 简介
------------

Promise是一个为PHP实现[CommonJS Promises/A](http://wiki.commonjs.org/wiki/Promises/A) 的库。

它还提供了其他一些与承诺相关的有用概念，例如加入多个承诺以及映射和减少承诺集合。

如果您以前从未听说过诺言，请先阅读[此内容](https://gist.github.com/3889970)

## 概念
--------

### Deferred（延迟）

**Deferred** 表示可能尚未完成的计算或工作单元。 通常（但并非总是）该计算将异步执行并在将来的某个时刻完成。

### Promise（承诺）

**Deferred** 表示计算本身，而 **Promise** 表示该计算的结果。 因此，每个**Deferred** 者都有一个承诺，可以充当其实际结果的占位符。

API
---

### Deferred

**Deferred** 表示其解析挂起的操作。它有单独的promise和resolver部分。

```php
$deferred = new React\Promise\Deferred();

$promise = $deferred->promise();

$deferred->resolve(mixed $value = null);
$deferred->reject(\Throwable $reason);
```

`promise` 方法返回延迟的承诺。

`resolve`和`reject` 方法控制延迟状态。

`Deferred`的构造函数接受一个可选的`$canceller`参数。有关更多信息，请参阅[Promise](#promise-2)

#### Deferred::promise()

```php
$promise = $deferred->promise();
```

返回延期的承诺，您可以将其分发给其他人，同时保留自行修改其状态的权限。

#### Deferred::resolve()

```php
$deferred->resolve(mixed $value = null);
```

解决`promise()`返回的承诺。

`$value`调用 `$onFulfilled`来通知所有消费者（它们通过`$promise->then()` 注册）。

如果`$value`本身是一个承诺，那么一旦这个承诺被解决，它将转换到这个承诺的状态。

#### Deferred::reject()

```php
$deferred->reject(\Throwable $reason);
```
拒绝`promise()`返回的承诺，表示延迟的计算失败。
所有消费者都会收到通知，方法是使用`$reason`调用`$onRejected`（它们通过`$promise->then()`注册）。

### PromiseInterface

`PromiseInterface` 为所有promise实现提供公共接口。
请参阅[Promise](#promise-2)以获取此包唯一公共实现。

一个承诺代表一个最终的结果，要么是实现（成功）和相关的结果，要么是拒绝（失败）和相关的原因。

一旦承诺处于履行或拒绝的状态，承诺就变得不可更改。
它的状态和结果（或错误）都不能修改。

#### PromiseInterface::then()

```php
$transformedPromise = $promise->then(callable $onFulfilled = null, callable $onRejected = null);
```
通过对承诺的履行或拒绝的结果，应用函数来转换承诺的状态值并返回转换结果的新承诺。

`then()` 方法用于一个promise注册新的已完成和拒绝处理程序（所有参数都是可选的）:

  * 一旦承诺实现，其结果将作为第一个参数传递，就会调用`$onFulfilled`
  * 一旦承诺被拒绝，其原因作为第一个参数传递，就会调用`$onRejected`

`$onFulfilled`或`$onRejected`的返回一个新的promise，无论哪个被调用，或者其中一个抛出异常则抛出异常。

promise 对在`then()`的同一调用中注册的处理回调做出以下保证:

  1.仅会调用`$onFulfilled` 或 `$onRejected`中的一个，两者不会都调用。
  
  2.`$onFulfilled` 和 `$onRejected` 不会被调用一次以上。

#### 另请参阅

* [resolve()](#resolve) - 创造承诺
* [reject()](#reject) - 创建拒绝的承诺
* [PromiseInterface::done()](#promiseinterfacedone)
* [done() vs. then()](#done-vs-then)

#### PromiseInterface::done()

```php
$promise->done(callable $onFulfilled = null, callable $onRejected = null);
```

如果承诺履行或拒绝，则耗费承诺的最终结果。

如果 `$onFulfilled` 或 `$onRejected`抛出或返回被拒绝的诺言，将导致致命错误(`E_USER_ERROR`)

由于`done()`的目的是消费而不是转换，所以`done()`总是返回`null`。

#### 另请参阅

* [PromiseInterface::then()](#promiseinterfacethen)
* [done() vs. then()](#done-vs-then)

#### PromiseInterface::otherwise()

```php
$promise->otherwise(callable $onRejected);
```

注册拒绝处理程序以进行承诺。 快捷操作方式如下:

```php
$promise->then(null, $onRejected);
```

另外，您可以传入提示`$onRejected`的`$reason`参数，仅捕获特定的错误。

```php
$promise
    ->otherwise(function (\RuntimeException $reason) {
        // Only catch \RuntimeException instances
        // All other types of errors will propagate automatically
    })
    ->otherwise(function (\Throwable $reason) {
        // Catch other errors
    )};
```

#### PromiseInterface::always()

```php
$newPromise = $promise->always(callable $onFulfilledOrRejected);
```

允许您在承诺链中执行“清理”类型的任务。

当`promise`被履行或被拒绝时调用`$onFulfilledOrRejected`(不带任何参数)回调

 * 如果`$promise`履行，并且`$onFulfilledOrRejected`成功返回，则`$newPromise`履行与`$promise`相同的值。
 * 如果`$promise`履行，并且`$onFulfilledOrRejected`抛出或返回被拒绝的承诺，`$newPromise`将抛出的异常或拒绝的承诺的原因。
 * 如果`$promise`拒绝，并且`$onFulfilledOrRejected`成功返回，则`$newPromise`与`$promise`以相同的原因拒绝。
 * 如果`$promise`拒绝，并且`$onFulfilledOrRejected`抛出或返回被拒绝的承诺，`$newPromise`将抛出的异常或拒绝的承诺的原因。

`always()`的行为于synchronous finally语句类似。当与`otherwise()`结合使用时，`always()`允许您编写于熟悉的同步catch/finally类似的代码。

考虑以下同步代码:

```php
try {
  return doSomething();
} catch (\Throwable $e) {
    return handleError($e);
} finally {
    cleanup();
}
```

可以编写类似的异步代码（`doSomething()`返回承诺）:

```php
return doSomething()
    ->otherwise('handleError')
    ->always('cleanup');
```

#### PromiseInterface::cancel()

```php
$promise->cancel();
```

`cancel()` 方法通知承诺创建者对操作的结果不再感兴趣。

一旦兑现了承诺（无论是履行还是拒绝），对承诺调用`cancel()`无效。

### Promise

创建一个`promise`，其状态由传`$resolver`函数控制。

```php
$resolver = function (callable $resolve, callable $reject) {
    // Do some work, possibly asynchronously, and then
    // resolve or reject.

    $resolve($awesomeResult);
    // or throw new Exception('Promise rejected');
    // or $resolve($anotherPromise);
    // or $reject($nastyError);
};

$canceller = function () {
    // Cancel/abort any running operations like network connections, streams etc.

    // Reject promise by throwing an exception
    throw new Exception('Promise cancelled');
};

$promise = new React\Promise\Promise($resolver, $canceller);
```

`promise`构造函数将接收一个`resolver`函数和一个可选的`canceller`函数，这两个函数都将使用3个参数进行调用:

  * `$resolve($value)` - 包装返回的`promise`的主要函数。 接受非承诺值或其他承诺。
当用非承诺值调用时，用该值实现承诺。当以另一个承诺被调用时，
例如 `$resolve($otherPromise)`，包装返回的`promise`等同于`$otherPromise`。
 * `$rejec($reason)` - 拒绝承诺的函数。建议只抛出异常，而不要使用`$reject()`。

如果`resolver`或`canceller`抛出异常，则将以该抛出的异常作为拒绝原因来拒绝承诺。

`resolver`函数将立即被调用，只有当所有使用者调用`promise`的`cancel()`方法时，`canceller`函数才会被调用。

### Functions

用于创建、连接、映射和减少承诺集合的有用函数。

所有处理承诺集合的功能（例如all()，race()，some()等）都支持取消。 
这意味着，如果您对返回的诺言调用`cancel()`，则集合中的所有诺言都会被取消。

#### resolve()

```php
$promise = React\Promise\resolve(mixed $promiseOrValue);
```
为`$promiseOrValue`创建一个承诺。

如果`$promiseOrValue`是一个值，它将是返回承诺

如果`$promiseOrValue`拥有`thenable`能力（提供`then()`方法的对象），则返回跟随`thenable`状态的可信承诺。

如果`$promiseOrValue`是一个承诺，它将按原样返回。

#### reject()

```php
$promise = React\Promise\reject(\Throwable $reason);
```

为`$reason`创建一个拒绝的`promise`。

注意[`\Throwable`](https://www.php.net/manual/en/class.throwable.php) 
PHP7中引入的接口包括两个用户区域
[`\Exception`](https://www.php.net/manual/en/class.exception.php) 和
[`\Error`](https://www.php.net/manual/en/class.error.php) 内部PHP错误。
通过强制使用`\Throwable`作为拒绝承诺的理由，任何语言错误或用户地异常都可以用来拒绝承诺。

#### all()

```php
$promise = React\Promise\all(array $promisesOrValues);
```

返回一个承诺，该承诺仅在`$promisesOrValues`中的所有项都已解析时才会解析。
返回的承诺的解析值将是一个数组，其中包含`$promisesOrValues`中每个项的解析值。

#### race()

```php
$promise = React\Promise\race(array $promisesOrValues);
```

发起一场允许一名获胜者参加的竞赛。返回一个承诺，该承诺的解决方式与第一个履行的承诺解决方式相同。

如果`$promisesOrValues`包含0项，则返回的承诺将**无限挂起**。

#### any()

```php
$promise = React\Promise\any(array $promisesOrValues);
```

返回将在`$promisesOrValues`中的任何一项兑现的承诺。返回的承诺的值将是触发项的值。

只有在`$promisesOrValues`中的 *所有* 项被拒绝时，返回的承诺才会被拒绝。
拒绝值将是`React\Promise\Exception\CompositeException`，其中包含所有拒绝原因。
拒绝原因可以通过`CompositeException::getThrowables()`获得。

如果`$promisesOrValues`包含0项，则返回的承诺也将拒绝，
并带有`React\Promise\Exception\LengthException`

#### some()

```php
$promise = React\Promise\some(array $promisesOrValues, integer $howMany);
```

返回一个承诺，当`$promisesOrValues`中至少有`$howMany`个提供的项兑现时，该承诺将被兑现。
返回的承诺的值将是一个长度为`$howMany`的数组，该数组包含首先解析的`$howMany`已兑现承诺的值。

如果`$howMany`项无法兑现（即`(count($promisesOrValues)-$howMany)+1`项拒绝），
则返回的承诺将拒绝。拒绝值将是一个`React\Promise\Exception\CompositeException`,
其中包含`(count($promisesOrValues)-$howMany)+1`拒绝原因。

拒绝原因可以通过`CompositeException::getExceptions()`获得。

如果`$promisesOrValues`包含的项目少于`$howMany`，则返回的承诺也将被拒绝，
并带有`React\Promise\Exception\LengthException`。

#### map()

```php
$promise = React\Promise\map(array $promisesOrValues, callable $mapFunc);
```

传统的map函数，类似于`array_map()`，但允许输入包含承诺 和/或 值，`$mapFunc`可以返回值或承诺。

map函数接收每个项作为参数，其中item是`$promisesOrValues`中的promise或value的完全解析值。

#### reduce()

```php
$promise = React\Promise\reduce(array $promisesOrValues, callable $reduceFunc, $initialValue = null);
```

传统的reduce函数，类似于`array_reduce()`，但输入可以包含承诺 和/或 值，
`$reduceFunc`可以返回值或承诺 *和* `$initialValue`可以是承诺或起始值。

### PromisorInterface

`React\Promise\PromisorInterface`提供实现承诺的公共接口。
`React\Promise\Deferred`实现了它，但是由于它是公共API的一部分，任何人都可以实现它。

示例
--------

### 如何使用Deferred

```php
function getAwesomeResultPromise()
{
    $deferred = new React\Promise\Deferred();

    // Execute a Node.js-style function using the callback pattern
    computeAwesomeResultAsynchronously(function (\Throwable $error, $result) use ($deferred) {
        if ($error) {
            $deferred->reject($error);
        } else {
            $deferred->resolve($result);
        }
    });

    // Return the promise
    return $deferred->promise();
}

getAwesomeResultPromise()
    ->then(
        function ($value) {
            // Deferred resolved, do something with $value
        },
        function (\Throwable $reason) {
            // Deferred rejected, do something with $reason
        }
    );
```

### 承诺转发的工作原理

几个简单的例子来展示`promise/A`转发机制是如何工作的。
当然，这些示例是精心设计的，在实际使用中，承诺链通常会分布在几个函数调用中，甚至是应用程序架构的几个级别。

#### 转发履行

已履行承诺将值转发给下一个承诺。
第一个承诺`$deferred->promise()`将用下面传递给`$deferred->resolve()`的值来进行解析。

每次调用`then()`都会返回一个新的承诺，该承诺将使用前一个处理程序的返回值进行解析。这里创建了一个承诺“管道”。

```php
$deferred = new React\Promise\Deferred();

$deferred->promise()
    ->then(function ($x) {
        // $x will be the value passed to $deferred->resolve() below
        // and returns a *new promise* for $x + 1
        return $x + 1;
    })
    ->then(function ($x) {
        // $x === 2
        // This handler receives the return value of the
        // previous handler.
        return $x + 1;
    })
    ->then(function ($x) {
        // $x === 3
        // This handler receives the return value of the
        // previous handler.
        return $x + 1;
    })
    ->then(function ($x) {
        // $x === 4
        // This handler receives the return value of the
        // previous handler.
        echo 'Resolve ' . $x;
    });

$deferred->resolve(1); // Prints "Resolve 4"
```

#### 转发拒绝

被拒绝的承诺的行为与`try/catch`类似，工作方式也与此类似:当你捕获一个异常时，你必须重新抛出进而继续向下传播。

同样，当您处理被拒绝的承诺时且传播拒绝，需通过返回被拒绝的承诺或实际抛出“重新抛出”它（因为promise将抛出的异常转换为拒绝）

```php
$deferred = new React\Promise\Deferred();

$deferred->promise()
    ->then(function ($x) {
        throw new \Exception($x + 1);
    })
    ->otherwise(function (\Exception $x) {
        // Propagate the rejection
        throw $x;
    })
    ->otherwise(function (\Exception $x) {
        // Can also propagate by returning another rejection
        return React\Promise\reject(
            new \Exception($x->getMessage() + 1)
        );
    })
    ->otherwise(function ($x) {
        echo 'Reject ' . $x->getMessage(); // 3
    });

$deferred->resolve(1);  // Prints "Reject 3"
```

#### 转发履行和拒绝

就像`try/catch`一样，您可以选择是否传播。转发履行和拒绝仍将以可预测的方式转发回调结果。

```php
$deferred = new React\Promise\Deferred();

$deferred->promise()
    ->then(function ($x) {
        return $x + 1;
    })
    ->then(function ($x) {
        throw new \Exception($x + 1);
    })
    ->otherwise(function (\Exception $x) {
        // Handle the rejection, and don't propagate.
        // This is like catch without a rethrow
        return $x->getMessage() + 1;
    })
    ->then(function ($x) {
        echo 'Mixed ' . $x; // 4
    });

$deferred->resolve(1);  // Prints "Mixed 4"
```

### done() vs. then()

黄金法则是:

    返回您的诺言，或调用done()方法.

乍一看，`then()` 和 `done()`看起来非常相似。 但是，有重要的区别。

`then()`的目的是转换`promise`的值，并将转换后的值传递或返回一个新的`promise`到代码的其他部分。

`done()`的目的是消费`promise`的值，并将转换后的值转移到代码中。

除了转换值之外，`then()`还允许您从中间错误中恢复或传播。 
未处理的任何错误都将由Promise机制捕获，并用于拒绝`then()`返回的承诺。

调用`done()`将错误的所有责任转移到代码中。
如果错误（抛出的异常或返回的拒绝）未能在您提供`done()`的`$onFulfilled` 或 `$onRejected`回调中捕获，它将导致致命错误。

```php
function getJsonResult()
{
    return queryApi()
        ->then(
            // Transform API results to an object
            function ($jsonResultString) {
                return json_decode($jsonResultString);
            },
            // Transform API errors to an exception
            function ($jsonErrorString) {
                $object = json_decode($jsonErrorString);
                throw new ApiErrorException($object->errorMessage);
            }
        );
}

// Here we provide no rejection handler. If the promise returned has been
// rejected, the ApiErrorException will be thrown
getJsonResult()
    ->done(
        // Consume transformed object
        function ($jsonResultObject) {
            // Do something with $jsonResultObject
        }
    );

// Here we provide a rejection handler which will either throw while debugging
// or log the exception
getJsonResult()
    ->done(
        function ($jsonResultObject) {
            // Do something with $jsonResultObject
        },
        function (ApiErrorException $exception) {
            if (isDebug()) {
                throw $exception;
            } else {
                logException($exception);
            }
        }
    );
```

Credits
-------

承诺是 [when.js](https://github.com/cujojs/when) 的一部分，作者[Brian Cavalier](https://github.com/briancavalier).

而且，大部分文档都是来源于`when.js`
[Wiki](https://github.com/cujojs/when/wiki) 和
[API docs](https://github.com/cujojs/when/blob/master/docs/api.md).

License
-------

Released under the [MIT](LICENSE) license.
