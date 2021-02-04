# PromiseStream

[![Build Status](https://travis-ci.org/reactphp/promise-stream.svg?branch=master)](https://travis-ci.org/reactphp/promise-stream)

[ReactPHP](https://reactphp.org/) 在Promise-land和Stream-land之间缺少的链接。

**目录**

* [用法](#用法)
  * [buffer()](#buffer)
  * [first()](#first)
  * [all()](#all)
  * [unwrapReadable()](#unwrapreadable)
  * [unwrapWritable()](#unwrapwritable)
* [安装](#安装)
* [测试](#测试)
* [License](#license)

## 用法

这个轻量级的库仅包含一些简单的功能。
所有功能都位于`React\Promise\Stream`命名空间下。

以下示例假定您使用与此类似的导入语句：

```php
use React\Promise\Stream;

Stream\buffer(…);
```

或者，也可以用它们完整的命名空间来引用它们:

```php
\React\Promise\Stream\buffer(…);
```

### buffer()

`buffer(ReadableStreamInterface<string> $stream, ?int $maxLength = null): PromiseInterface<string,Exception>`
函数可用于创建一个`Promise`，它可以用流数据缓冲区解析。

```php
$stream = accessSomeJsonStream();

Stream\buffer($stream)->then(function ($contents) {
    var_dump(json_decode($contents));
});
```

流关闭后，`Promise`将与所有连接的数据块一起解析。

如果流已经关闭，则`Promise`将以空字符串解析。

如果流发出错误，则`Promise`将拒绝。

如果`Promise`被取消，则`Promise`将被拒绝。

`$maxLength`参数(可选)默认为无限制。 如果设定了最大长度，并且流在结束之前发出更多数据，
则将通过`OverflowException`拒绝诺言。

```php
$stream = accessSomeToLargeStream();

Stream\buffer($stream, 1024)->then(function ($contents) {
    var_dump(json_decode($contents));
}, function ($error) {
    //当流缓冲区超过最大大小时或当流发生错误时触发此处逻辑
    //在此示例中为1024字节，
});
```

### first()

`first(ReadableStreamInterface|WritableStreamInterface $stream, string $event = 'data'): PromiseInterface<mixed,Exception>`
函数可用于创建`Promise`，该`Promise`将在设定事件首次触发时解析。 

```php
$stream = accessSomeJsonStream();

Stream\first($stream)->then(function ($chunk) {
    echo 'The first chunk arrived: ' . $chunk;
});
```

无论第一个事件发出什么，promise都将解析，如果该事件不传递任何数据，则使用`null`。
如果不传递自定义事件名称，则它将等待第一个`data`事件，并使用包含第一个数据块的字符串进行解析。

如果流发出错误，则`promise`将拒绝 - 除`error`事件。

一旦流关闭，`promise`就会被拒绝 - 除`close`事件。

如果流已关闭，则`promise`将被拒绝。

如果`promise`被取消，它将被拒绝。

### all()

`all(ReadableStreamInterface|WritableStreamInterface $stream, string $event = 'data'): PromiseInterface<array,Exception>`
函数可用于创建一个`Promise`，该函数用一个包含所有事件数据的数组进行解析。

```php
$stream = accessSomeJsonStream();

Stream\all($stream)->then(function ($chunks) {
    echo 'The stream consists of ' . count($chunks) . ' chunk(s)';
});
```

`promise`将使用一个数组来解析所有发出的事件，如果事件不传递任何数据，则使用`null`。
如果不传递自定义事件名称，则它将等待所有`data`事件，并使用包含所有数据块的数组进行解析。

一旦流关闭，`promise`将用数组解析。

如果流已经关闭，则`promise`将使用空数组解析。

如果流发出错误，`promise`将被拒绝。

如果`promise`被取消，它将被拒绝。

### unwrapReadable()

`unwrapReadable(PromiseInterface<ReadableStreamInterface,Exception> $promise): ReadableStreamInterface`
函数可用于解包解析为`ReadableStreamInterface`的`promise`。

该函数立即返回一个可读的流实例（实现`ReadableStreamInterface`），用作将来的承诺解析的代理。
一旦Promise通过`ReadableStreamInterface`解析，其数据将通过管道传输到输出流。

```php
//$promise = someFunctionWhichResolvesWithAStream();
$promise = startDownloadStream($uri);

$stream = Stream\unwrapReadable($promise);

$stream->on('data', function ($data) {
    echo $data;
});

$stream->on('end', function () {
    echo 'DONE';
});
```

如果`promise`被拒绝或通过` ReadableStreamInterface `以外的任何实例实现，
那么输出流将发出一个` error `事件并关闭:

```php
$promise = startDownloadStream($invalidUri);

$stream = Stream\unwrapReadable($promise);

$stream->on('error', function (Exception $error) {
    echo 'Error: ' . $error->getMessage();
});
```

这里的` $promise `应该是挂起的，也就是说，在调用这个函数时，它不应该被履行或拒绝。
如果给定的承诺已经履行，并且没有使用`ReadableStreamInterface`实例解决，那么您将无法接收到`error`事件。
您可以随时` close() `结果流，这将尝试` cancel() `未完成的承诺或尝试` close() `基础流。

```php
$promise = startDownloadStream($uri);

$stream = Stream\unwrapReadable($promise);

$loop->addTimer(2.0, function () use ($stream) {
    $stream->close();
});
```

### unwrapWritable()

`unwrapWritable(PromiseInterface<WritableStreamInterface,Exception> $promise): WritableStreamInterface`
 函数可用于解包解析为`WritableStreamInterface`的`promise`。

该函数立即返回一个可写的流实例（实现`WritableStreamInterface`），用作将来的承诺解析的代理。
对`Promise`的解析将在此实例中进行的所有写操作都缓存在内存中。
一旦Promise通过`WritableStreamInterface`解析，您写入代理的所有数据都将透明地转发到内部流。

```php
//$promise = someFunctionWhichResolvesWithAStream();
$promise = startUploadStream($uri);

$stream = Stream\unwrapWritable($promise);

$stream->write('hello');
$stream->end('world');

$stream->on('close', function () {
    echo 'DONE';
});
```

如果给定的承诺已经履行，并且没有使用`WritableStreamInterface`实例解决，则输出流将发出一个`error`事件并关闭： 

```php
$promise = startUploadStream($invalidUri);

$stream = Stream\unwrapWritable($promise);

$stream->on('error', function (Exception $error) {
    echo 'Error: ' . $error->getMessage();
});
```

给定的` $promise `应该是挂起的，也就是说，在调用这个函数时，它不应该被履行或拒绝。
如果给定的承诺已经履行，并且没有使用`WritableStreamInterface`实例解决，那么您将无法接收` error `事件。

您可以随时` close() `结果流，这将尝试` cancel() `未完成的承诺或尝试` close() `基础流。

```php
$promise = startUploadStream($uri);

$stream = Stream\unwrapWritable($promise);

$loop->addTimer(2.0, function () use ($stream) {
    $stream->close();
});
```

## 安装

推荐的安装这个库的方法是[通过Composer](https://getcomposer.org)。
[Composer 新手?](https://getcomposer.org/doc/00-intro.md)

该项目遵循[SemVer](https://semver.org/) ，
默认安装最新支持的版本:

```bash
$ composer require react/promise-stream:^1.2
```

有关版本升级的详细信息，请参见[CHANGELOG](https://reactphp.org/promise-stream/changelog.html) 。

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

MIT, see [LICENSE file](https://reactphp.org/promise-stream/license.html).
