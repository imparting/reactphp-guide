# Stream

[![Build Status](https://travis-ci.org/reactphp/stream.svg?branch=master)](https://travis-ci.org/reactphp/stream)

事件驱动的可读流和可写流，用于 [ReactPHP](https://reactphp.org/) 中的非阻塞I/O

为了使 [EventLoop](https://github.com/reactphp/event-loop) 更容易使用，该组件引入了强大的“流”概念。
流允许您以小块的方式高效地处理大量数据(比如一个多GB的文件下载)，而不必一次将所有数据存储在内存中。
与PHP中的流非常相似，但有一个更适合异步、非阻塞I/O的接口。

**目录**

* [Stream用法](#stream用法)
  * [ReadableStreamInterface](#readablestreaminterface)
    * [data事件](#data事件)
    * [end事件](#end事件)
    * [error事件](#error事件)
    * [close事件](#close事件)
    * [isReadable()](#isreadable)
    * [pause()](#pause)
    * [resume()](#resume)
    * [pipe()](#pipe)
    * [close()](#close)
  * [WritableStreamInterface](#writablestreaminterface)
    * [drain事件](#drain事件)
    * [pipe事件](#pipe事件)
    * [error事件](#error-事件)
    * [close事件](#close-事件)
    * [isWritable()](#iswritable)
    * [write()](#write)
    * [end()](#end)
    * [close()](#close-1)
  * [DuplexStreamInterface](#duplexstreaminterface)
* [Creating streams](#creating-streams)
  * [ReadableResourceStream](#readableresourcestream)
  * [WritableResourceStream](#writableresourcestream)
  * [DuplexResourceStream](#duplexresourcestream)
  * [ThroughStream](#throughstream)
  * [CompositeStream](#compositestream)
* [用法](#用法)
* [安装](#安装)
* [测试](#测试)
* [License](#license)
* [More](#more)

## stream用法

ReactPHP在其整个生态系统中使用“流”的概念，为处理任意数据内容和大小的流提供一致的高级抽象。
虽然流本身是一个非常底层的概念，但它可以作为一个强大的抽象来构建更高级的组件和协议。
如果你不熟悉这个概念，可以把它们想象成水管(pipe):
你可以从一个水源中取水，也可以产生水并将其输送到任何目的地水槽(sink)。

同样，流可以是

* 可读(如`STDIN`终端输入)或
* 可写(如`STDOUT`终端输出)或
* 双工(既可读又可写，例如TCP/IP连接)

因此，这个包定义了以下三个接口

* [`ReadableStreamInterface`](#readablestreaminterface)
* [`WritableStreamInterface`](#writablestreaminterface)
* [`DuplexStreamInterface`](#duplexstreaminterface)

### ReadableStreamInterface

`ReadableStreamInterface` 负责为只读流和双工流的可读端。

除了定义一些方法之外，这个接口还实现了`EventEmitterInterface`允许你对特定的事件做出响应。

事件回调函数必须是一个有效的 `callable` ，遵守严格的参数定义，并且必须完全按照文档中描述的那样接受事件参数。

事件回调函数绝不能抛出`Exception`。

事件回调函数的返回值将被忽略，并且没有任何效果，因此出于性能原因，建议您不要返回任何过多的数据结构。

这个接口的每个实现都必须遵循这些事件语义，才能被认为是合法的流。

> 请注意，此接口的高级实现可能会选择使用专用语义来定义附加事件，
  这些专用语义未定义为此低级流规范的一部分。
  与这些事件语义的一致性超出了此接口的范围，
  因此您可能还必须参考此类更高级别实现的文档。

#### data事件

当从该源流读取/接收数据时，将触发`data`事件。事件接收传入数据的单个混合参数。

```php
$stream->on('data', function ($data) {
    echo $data;
});
```

该事件可能被触发任意次，如果该流根本不发送任何数据，则可能为零次。
在`end`或`close`事件之后不应该触发它。

给定的`$data`参数可能是混合类型，但通常建议它应该是`string`值，
或者可以使用允许表示为`string`的类型，以实现最大的兼容性。

许多常见流(如TCP/IP连接或基于文件的流)将发出原始(二进制)有效负载数据，
这些数据通过网络接收为`string`值块。

由于这种基于流的特性，发送方可以发送任意数量不同大小的块。
不能保证接收到的数据块与发送方打算发送的帧完全相同。
换句话说，许多底层协议(如TCP/IP)以块的形式传输数据，
这些块可能介于单字节到几十千字节之间。
为了实现正确的消息帧，您可能需要对这些数据块应用更高级别的协议。
  
#### end事件

源流成功到达流尾(EOF)后，将触发`end`事件。

```php
$stream->on('end', function () {
    echo 'END';
});
```

该事件最多触发一次，或者根本不触发，这取决于是否检测到成功结束。

它不应该在前一个`end`或`close`事件之后触发。
如果流未成功结束而关闭（例如在前一个`close`事件之后），则不能触发该事件。

流结束后，必须切换到不可读模式，另请参见[`isReadable()`](#isreadable)

只有成功到达*end*时才会触发此事件，如果流被不可恢复的错误中断或显式关闭则不会触发此事件。
并不是所有的流都知道“成功的结束”这个概念。

许多用例涉及检测流何时关闭(终止)，在这种情况下，您应该使用`close`事件。

流发出`end`事件后，通常应该跟在`close`事件后面。

如果远程端关闭连接或成功读取文件句柄直到其结束(EOF)，许多公共流（如TCP/IP连接或基于文件的流）都将发出此事件。

请注意，不应将此事件与`end()`方法混淆。
此事件定义从源流*读取*的成功结束，而`end()`方法定义向目标流*写入*的成功结束。

#### error事件

通常是在尝试从该流读取时发生致命错误，则会触发`error`事件。

事件为错误实例接收一个`Exception`参数。

```php
$server->on('error', function (Exception $e) {
    echo 'Error: ' . $e->getMessage() . PHP_EOL;
});
```

一旦流检测到致命错误（如致命的传输错误）或意外的`data`或过早的`end`事件之后，就会触发此事件。

它不应在前一个`error`, `end` 或 `close`事件之后触发。
如果这不是致命的错误情况，例如没有导致任何数据丢失的临时网络问题，则不会触此事件。

出现流错误后，它必须关闭流，因此后面应该有一个`close`事件，
然后切换到不可读模式，另请参见`close()`和`isReadable()`。

许多常见的流（例如TCP/IP连接或基于文件的流）只处理数据传输，
并不对数据边界进行假设（例如意外的`data`或过早的`end`事件）。

换言之，许多较底层的协议（例如TCP/IP）可能会选择只在出现致命传输错误时触发事件，并在响应时关闭(终止)流。

如果这个流是`DuplexStreamInterface`，你也应该注意到流的可写端也实现了`error`事件。

换句话说，在读取或写入流时可能发生错误，这应该导致相同的错误处理。

#### close事件

一旦流关闭（终止），将触发`close`事件。

```php
$stream->on('close', function () {
    echo 'CLOSED';
});
```

根据流是否终止，这个事件应该被触发一次，或者根本不触发。
它不应该在前一个`close`事件之后触发。

流关闭后，必须切换到不可读模式，
另请参见[`isReadable()`](#isreadable)。

与`end`事件不同，每当流关闭时都应触发此事件，而不管此事件是由于不可恢复的错误隐式发生的，还是在任何一方关闭流时显式发生的。

如果只想检测*成功*结束，则应改用`end`事件。

许多常见的流（例如TCP / IP连接或基于文件的流）很可能在读取*成功*结束事件或致命的传输错误事件之后选择触发此事件。

如果此流是`DuplexStreamInterface`，则您还应该注意该流的可写端`close`事件的实现。

换句话说，在接收到该事件之后，流必须切换到不可写和不可读取模式，另请参见 [`isWritable()`](#iswritable)。

注意，该事件不应与`end`事件混淆。

#### isReadable()

`isReadable(): bool`方法可用于检查此流是否处于可读状态（尚未关闭）。

此方法可用于检查流是否仍然接受传入的数据事件，或者它是否已结束或关闭。

一旦流不可读，就不再发出`data`或`end`事件。

```php
assert($stream->isReadable() === false);

$stream->on('data', assertNeverCalled());
$stream->on('end', assertNeverCalled());
```

成功打开的流始终必须以可读模式启动。

一旦流结束或关闭，它必须切换到非可读模式。

这可以随时发生，通过`close()`显式地发生，或者由于远程关闭或不可恢复的传输错误而隐式地发生。

流一旦切换到非可读模式，就绝不能回到可读模式。

如果此流是`DuplexStreamInterface`，则您还应该注意该流的可写端`isWritable()`方法的实现。

除非这是一个半开双工流，否则它们通常应该具有相同的返回值。

#### pause()

`pause(): void`方法可用于暂停读取传入的数据事件。 

从事件循环中删除数据源文件描述符。 这使您可以限制传入的数据。

除非另有说明，否则成功打开的流不应暂停。

流暂停后，就不应再触发`data`或`end`事件

```php
$stream->pause();

$stream->on('data', assertShouldNeverCalled());
$stream->on('end', assertShouldNeverCalled());
```

该方法仅是建议性的，通常不建议调用，但该流可以继续`emit`(发出)`data`事件。

您可以通过再次调用`resume()`来继续处理事件。

注意，这两种方法都可以被调用多次，多次调用`pause()`无效。

另见[`resume()`](#resume)

#### resume()

`resume(): void` 方法可用于恢复`data`事件。 

在`pause()`之后重新连接数据源。

```php
$stream->pause();

$loop->addTimer(1.0, function () use ($stream) {
    $stream->resume();
});
```

请注意，这两个方法都可以被调用任意次数，在没有`pause()`的情况下调用`resume()`无效。
 
请参见[`pause()`](#pause)

#### pipe()

`pipe(WritableStreamInterface $dest, array $options = [])` 方法可将此可读源中的所有数据通过管道传输到给定的可写目标源。

自动将所有传入数据发送到目标源。根据目标源可以处理的内容自动限制源。

```php
$source->pipe($dest);
```

同样，您也可以通过管道将实现`DuplexStreamInterface`的实例导入自身，以便回写接收到的所有数据。

对于TCP/IP `echo`服务，这是一个有用的特性:

```php
$connection->pipe($connection);
```
这个方法按原样返回目标流，可以用来建立管道流链:

```php
$source->pipe($decodeGzip)->pipe($filterBadWords)->pipe($dest);
```
默认情况下，一旦源流发出`end()`事件，就会对目标流调用`end()`。可以这样禁用：

```php
$source->pipe($dest, array('end' => false));
```
请注意，这只适用于`end`事件。
如果源流上发生 `error` 或显式`close`事件，则您必须手动关闭目标流：

```php
$source->pipe($dest);
$source->on('close', function () use ($dest) {
    $dest->end('BYE!');
});
```

如果源流不可读（关闭状态），则这是不可操作的。

```php
$source->close();
$source->pipe($dest); // 禁止操作
```

如果目标流不可写（关闭状态），则这将简单地限制（暂停）源流：

```php
$dest->close();
$source->pipe($dest); // calls $source->pause()
```

同样，如果目标流在管道仍处于活动状态时关闭，它还将限制（暂停）源流：

```php
$source->pipe($dest);
$dest->close(); // calls $source->pause()
```

一旦管道成功设置，目标流必须发出一个`pipe`事件，源流必须有一个 event 参数。

#### close()

`close(): void` 方法可以用来关闭流(强制)。

```php
$stream->close();
```

一旦流被关闭，它应该触发一个`close`事件。
请注意，此事件不应触发多次。

调用此方法后，流必须切换到不可读模式，另请参见[`isReadable()`](#isreadable)。
这意味着不应再触发`data`或`end`事件。

```php
$stream->close();
assert($stream->isReadable() === false);

$stream->on('data', assertNeverCalled());
$stream->on('end', assertNeverCalled());
```
如果此流是`DuplexStreamInterface`，则还应该注意流的可写端`close()`方法的实现。

换句话说，调用此方法后，流必须切换到不可写和不可读模式，另请参见[`iswriteable()`](#iswritable)。

请注意，此方法不应与`end()`方法混淆。

### WritableStreamInterface

`WritableStreamInterface` 为只写流和双工流的可写端接口。

除了定义一些方法外，这个接口还实现了`EventEmitterInterface`，它允许您对某些事件做出反应。

事件回调函数必须是一个有效的`callable`，它遵循严格的参数定义，并且必须完全按照文档所述接受事件参数。

事件回调函数不能抛出`Exception`。

事件回调函数的返回值将被忽略并且没有任何影响，因此出于性能原因，建议您不要返回任何过多的数据结构。

此接口的每个实现都必须遵循这些事件语义，才能被视为合法的流。

> 请注意，此接口的更高级别的实现可以选择使用未定义为该底层级别流规范一部分的专用语义来定义其他事件。
  与这些事件语义的一致性超出了此接口的范围，因此您可能还必须参考此类更高级别的实现的文档。

#### drain事件

每写入缓冲区满时且有更多数据到达时，就会发出`drain`事件。

```php
$stream->on('drain', function () use ($stream) {
    echo 'Stream is now ready to accept more data';
});
```
每写入缓冲区满时且有更多数据到达时，就会发出`drain`事件。

换句话说，这个事件可以被触发多次，如果缓冲区不满，则该事件可能是零次。

如果缓冲区不满，则不应触发此事件。

该事件主要在内部使用，有关更多详细信息，请参见[`write()`](#write)

#### pipe事件

当一个可读流`pipe()`进入数据时，`pipe`事件将被触发。
事件接收源流的一个`ReadableStreamInterface`参数。

```php
$stream->on('pipe', function (ReadableStreamInterface $source) use ($stream) {
    echo 'Now receiving piped data';

    // explicitly close target if source emits an error
    $source->on('error', function () use ($stream) {
        $stream->close();
    });
});

$source->pipe($stream);
```

对于每个成功导入目标流的可读流，此事件必须触发一次。

换句话说，这个事件可以被触发多次，如果没有数据流通过管道进入这个流，则可能是零次。

如果源不可读(已经关闭)或目标不可写(已经关闭)，则绝不能触发此事件。

此事件主要在内部使用，请参阅[`pipe()`](#pipe)了解更多细节。

#### error 事件

一旦发生致命错误，则会触发`error`事件，通常是在试图写入该流时。
事件为错误实例接收一个`Exception`对象参数。

```php
$stream->on('error', function (Exception $e) {
    echo 'Error: ' . $e->getMessage() . PHP_EOL;
});
```
一旦流检测到致命错误(例如致命传输错误)，就会触发此事件。

它不应该在前一个`error` 或 `close`事件之后触发。

如果不出现一个致命的错误情况，例如没有导致任何数据丢失的临时网络问题，则不会触发。

在流出错后，它必须关闭流，因此应该紧跟着一个`close`事件，然后切换到非可写模式，参见[`close()`](#close)和[`isWritable()`](#iswritable)。

许多常见流（例如TCP / IP连接或基于文件的流）仅处理数据传输，并且可能会选择仅针对致命的传输错误触发一次，然后将其关闭（终止）作为响应。

如果这个流是一个`DuplexStreamInterface`，您还应该注意流的可读端`error`事件的实现。
换句话说，在读取或写入流时可能发生错误，这将导致相同的错误处理。

#### close 事件

一旦流关闭（终止），将发出`close`事件。

```php
$stream->on('close', function () {
    echo 'CLOSED';
});
```

根据流是否终止，此事件应触发一次或从不触发。
它不会在前一个`close`事件之后触发。

流关闭后，必须切换到不可写模式，
另请参见[`isWritable()`](#iswritable)

无论是由于不可恢复的错误而隐式触发还是在任何一方关闭流时显式触发，只要流关闭，都应触发此事件。

许多常见的流(例如TCP/IP连接或基于文件的流)可能会选择在`end()`方法刷新缓冲区后、在接收到*成功* `end` 事件或致命的传输`error`事件后触发此事件。

如果此流是`DuplexStreamInterface`，则还应注意该流的可读端`close`事件的实现。

换句话说，接收到该事件后，流必须切换到不可写和不可读取模式，另请参见[`isReadable()`](#isreadable)。

注意，该事件不应与`end`事件混淆。

#### isWritable()

`isWritable(): bool`方法可用于检查此流是否处于可写状态（尚未关闭）。

此方法可用于检查流是否仍接受写入数据，或者是否已结束或关闭。
将数据写入不可写流是不可操作的：

```php
assert($stream->isWritable() === false);

$stream->write('end'); // NO-OP
$stream->end('end'); // NO-OP
```

成功打开的流必须始终以可写模式。

一旦流结束或关闭，它必须切换到不可写模式。

这种情况随时可能发生，可以通过`end()`或`close()`显式发生，也可以由于远程关闭或不可恢复的传输错误而隐式发生。

一旦流切换到不可写模式，它就不能转换回可写模式。

如果此流是`DuplexStreamInterface`实现，则还应该注意流的可读端`isReadable()`方法的实现。除非这是半开放双工流，否则它们通常应该具有相同的返回值。

#### write()

使用`write(mixed $data): bool`方法将数据写入流。

必须使用布尔值`true`来确认成功写入，这意味着要么立即写入（刷新）数据，要么对数据进行缓冲和调度以备将来写入。

请注意，这个接口无法控制显式刷新缓冲数据，因为寻找合适的刷新时间超出了这个接口的范围，要由这个接口的实现来决定。

许多常见的流（例如TCP / IP连接或基于文件的流）可以选择缓冲所有给定的数据，并通过使用底层的`EventLoop`来检查资源何时实际可写来计划将来的刷新。

如果流不能处理写入（或刷新）数据的操作，它应该发出一个`error`事件，如果流不能从这个错误中恢复，则可能`close()`该流。

如果在添加`$data`后内部缓冲区已满，那么`write()`应该返回`false`，表明调用者应该停止发送数据，直到缓冲区耗尽。

一旦缓冲区准备好接受更多数据，流应该发送`drain`事件。

同样，如果流是不可写的(已经处于关闭状态)，它一定不能处理给定的`$data`，并且应该返回`false`，表明调用者应该停止发送数据。

给定的`$data`参数可能是混合类型，但通常建议它应该是一个`string`值，或者使用允许转化为`string`的类型，以最大限度地兼容。

许多常见的流（例如TCP / IP连接或基于文件的流）仅接受原始（二进制）有效载荷数据，这些数据通过网络作为`string`值的块传输。

由于这种基于流的特性，发送方可以发送任意数量大小不同的块。无法保证这些数据块将以发送方打算发送的完全相同的顺序接收。

换言之，许多较底层的协议（如TCP/IP）以块的形式传输数据，这些块的大小可能介于单个字节到几十千字节之间。
您需要对这些数据块应用更高级的协议，以便实现正确的消息帧。

#### end()

`end(mixed $data = null): void`方法可用于成功结束流（可选地发送一些最终数据）。

这个方法可以用来成功地结束流，例如，在发送出所有当前缓存的数据后关闭流。

```php
$stream->write('hello');
$stream->write('world');
$stream->end();
```
如果当前没有缓冲的数据，也没有需要刷新的数据，那么这个方法可以立即`close()`流。

如果缓冲区中仍有需要首先刷新的数据，则该方法应该尝试写出这些数据，然后才使用`close()`关闭流。

一旦流关闭，它会触发`close`事件。

请注意，这个接口无法控制显式刷新缓冲数据，因为寻找合适的刷新时间超出了这个接口的范围，要由这个接口的实现来决定。

许多常见的流（例如TCP / IP连接或基于文件的流）可以选择缓冲所有给定的数据，并通过使用底层的`EventLoop`来检查资源何时实际可写来计划将来的刷新。

您可以选择在结束流之前将一些最终数据传递给流。 如果将非`null`值指定为`$data`，则此方法的行为就像在没有结束之前调用`write($data)`一样。

```php
// shorter version
$stream->end('bye');

// same as longer version
$stream->write('bye');
$stream->end();
```

调用此方法后，流必须切换到不可写模式，另请参见[`isWritable()`](#iswritable)。

这意味着不可能再进行写操作，因此任何其他的`write()`或`end()`调用均无效。

```php
$stream->end();
assert($stream->isWritable() === false);

$stream->write('nope'); // NO-OP
$stream->end(); // NO-OP
```
如果此流是`DuplexStreamInterface`实现，则调用此方法也应结束其可读端，除非该流支持半开模式。

换句话说，调用此方法后，这些流应该切换到不可写和不可读取的模式，另请参见[`isReadable()`](#isreadable)

这意味着在这种情况下，流不再应该发出任何`data`或`end`事件。 
流可能会选择使用`pause()`方法逻辑，但必须特别注意确保对`resume()`方法的后续调用不应继续发出可读事件。

注意，该方法不应与`close()`方法混淆。

#### close()

`close(): void` 方法可用于（强制）关闭流。

此方法可用于强制关闭流，即在不等待刷新任何缓冲数据的情况下关闭流。
如果缓冲区中仍有数据，则会丢弃此数据。

```php
$stream->close();
```

一旦流关闭，它应该发出一个`close`事件。
请注意，不应多次触发此事件。

调用此方法后，流必须切换到不可写模式，另请参见[`isWritable()`](#iswritable)。
这意味着不可能再进行写操作，因此任何其他的`write()`或`end()`调用均无效。

```php
$stream->close();
assert($stream->isWritable() === false);

$stream->write('nope'); // NO-OP
$stream->end(); // NO-OP
```

注意，该方法不应与`end()`方法混淆。

与`end()`方法不同，此方法不处理任何现有缓冲区，而只是丢弃缓冲区内容。

同样，也可以在对流调用`end()`之后调用此方法，以停止等待流刷新其最终数据。

同样，为了停止等待流刷新其最终数据，也可以在流上调用`end()`之后调用此方法。

```php
$stream->end();
$loop->addTimer(1.0, function () use ($stream) {
    $stream->close();
});
```

如果此流是`DuplexStreamInterface`，则还应该注意流的可读端如何实现`close()` 方法。

换句话说，调用此方法后，流必须切换到不可写和不可读模式，另请参见[`isReadable()`](#isreadable)。

### DuplexStreamInterface

`DuplexStreamInterface`为双工流（可读写）提供接口。

它建立在用于可读和可写流的现有接口之上，并遵循完全相同的方法和事件语义。

如果您是这个概念的新手，则应该先阅读`ReadableStreamInterface`和`WritableStreamInterface`。

除了定义一些方法外，该接口还实现了`EventEmitterInterface`，
它使您能够对`ReadbleStreamInterface`和`WritableStreamInterface`上定义的相同事件做出反应。

事件回调函数必须是一个有效的`callable`，遵守严格的参数定义，并且必须完全按照文档中描述接受事件参数。

事件回调函数绝不能抛出`Exception`。

事件回调函数的返回值将被忽略，并且没有任何效果，因此出于性能原因，建议您不要返回任何过多的数据结构。

这个接口的每个实现都必须遵循这些事件语义，才能被认为是合法流。

> 请注意，此接口的高级实现可能会选择使用专用语义来定义附加事件，
  这些专用语义未定义为此低级流规范的一部分。
  与这些事件语义的一致性超出了此接口的范围，
  因此您可能还必须参考此类更高级别实现的文档。

另请参阅 [`ReadableStreamInterface`](#readablestreaminterface)和[`WritableStreamInterface`](#writablestreaminterface)。

## Creating streams

 * ReactPHP在其整个生态系统中都使用`streams`的概念，所以这个包的许多高级用户只处理[流使用](#stream-usage)。
   流实例通常是在一些更高级别的组件中创建的，许多用户实际上从来不需要处理创建流实例的问题。

 * 如果你想接受传入或建立传出的明文TCP/IP或安全TLS socket连接流，使用[react/socket](2.Network-Components/Socket.md) 

 * 如果你想接收一个http请求体流，请使用[react/http](3.Protocol-Components/Http.md)

 * 如果你想通过诸如STDIN, STDOUT, STDERR等进程管道与子进程通信，请使用[react/child-process](4.Utility-Components/ChildProcess.md)

 * 如果你想对文件系统进行读写操作，请使用 [react/filesystem](https://github.com/reactphp/filesystem)

 * 参见最后一章[更多真实应用](#more)。

但是，如果您正在编写一个底层组件，或者想要从一个流资源创建一个流实例，那么下面的章节就是为您准备的。

>请注意，以下示例使用`fopen()`和`stream_socket_client()`只是为了说明。
 这些函数不应该在真正的异步程序中使用，因为每个调用可能需要几秒钟才能完成，否则将阻塞`EventLoop`。
 此外，`fopen()` 调用将在某些平台上返回一个文件句柄，这可能是所有`EventLoop`实现所支持的，也可能不是。
 作为一种替代方案，您可能希望使用上面列出的高级库。 

### ReadableResourceStream

`ReadableResourceStream`是PHP流资源[`ReadableStreamInterface`](#readablestreaminterface)的具体实现。

这可以用来表示只读资源，比如以可读模式打开的文件流，或者像`STDIN`这样的流:

```php
$stream = new ReadableResourceStream(STDIN, $loop);
$stream->on('data', function ($chunk) {
    echo $chunk;
});
$stream->on('end', function () {
    echo 'END';
});
```

请参阅[`ReadableStreamInterface`](#readablestreaminterface).

构造函数的第一个参数必须是一个以读取模式打开的有效的流资源(例如:`fopen()`的模式`r`)。

否则，它将抛出一个`InvalidArgumentException`:

```php
// throws InvalidArgumentException
$stream = new ReadableResourceStream(false, $loop);
```
另请参阅[`DuplexResourceStream`](#readableresourcestream)了解读写流资源。

该类内部试图在流资源上启用非阻塞模式，这可能不支持所有的流资源。

最值得注意的是，Windows上的管道(STDIN等)不支持这一点。

如果失败，它将抛出`RuntimeException`:

```php
// throws RuntimeException on Windows
$stream = new ReadableResourceStream(STDIN, $loop);
```
一旦使用有效的流资源调用构造函数，该类将负责底层的流资源。

您应该只使用它的公共API，而不应该手动干扰底层的流资源。

该类接受一个可选参数`int|null $readChunkSize`，该参数控制一次从流中读取的最大缓冲区大小(以字节为单位)。

您可以在这里使用`null`值来应用其默认值。

除非您知道自己在做什么，否则不应该更改此值。

这可以是一个正数，这意味着一次最多可以从底层流资源读取X个字节。注意，如果流资源当前可用的字节数小于X字节，则实际读取的字节数可能更低。

这可以是`-1`，表示从底层流资源中`读取所有可用的内容`。

这应该读取直到流资源不再可读(即底层缓冲区耗尽)，注意这并不一定意味着它到达了`EOF`。

```php
$stream = new ReadableResourceStream(STDIN, $loop, 8192);
```
>PHP bug警告:如果PHP进程在没有`STDIN`流的情况下显式启动，
 那么尝试从`STDIN`读取数据可能会从其他流资源返回数据。
 如果以空流(如`php test.php < /dev/null`而不是`php test.php <&-`)开始，则不会发生这种情况。
 请参阅[#81](https://github.com/reactphp/stream/issues/81) 了解更多细节。

### WritableResourceStream

`WritableResourceStream`是PHP流资源的[`WritableStreamInterface`](#writablestreaminterface)的具体实现。

这可以用来表示只写的资源，比如以可写模式打开的文件流，或者像`STDOUT`或`STDERR`这样的流:

```php
$stream = new WritableResourceStream(STDOUT, $loop);
$stream->write('hello!');
$stream->end();
```

请参阅[`WritableStreamInterface`](#writablestreaminterface)

构造函数的第一个参数必须是打开用于写入的有效流资源。
否则，它将抛出一个`InvalidArgumentException`:

```php
// throws InvalidArgumentException
$stream = new WritableResourceStream(false, $loop);
```

另请参阅[`DuplexResourceStream`](#readableresourcestream)了解读写流资源。

该类内部试图在流资源上启用非阻塞模式，这可能不支持所有的流资源。

最值得注意的是，Windows上的管道(STDOUT、STDERR等)不支持这一点。

如果失败，它将抛出`RuntimeException`:

```php
// throws RuntimeException on Windows
$stream = new WritableResourceStream(STDOUT, $loop);
```

一旦使用有效的流资源调用构造函数，该类将负责底层的流资源。

您应该只使用它的公共API，而不应该手动干扰底层的流资源。

对这个类的任何`write()`调用都不会立即执行，而是在`EventLoop`报告流资源准备好接受数据后异步执行。

为此，它使用一个内存缓冲区字符串来收集所有未完成的写操作。

这个缓冲区应用了一个软限制，它定义了在调用者停止发送进一步数据之前，它愿意接受多少数据。

该类接受一个可选参数`int|null $writeBufferSoftLimit`，以字节为单位控制最大缓冲区大小。

您可以在这里使用`null`值来应用其默认值。

否则不应该更改此值，除非您知道自己在做什么。

```php
$stream = new WritableResourceStream(STDOUT, $loop, 8192);
```

该类接受一个可选参数`int|null $writeChunkSize`，该参数以字节为单位控制一次写入流的最大缓冲区大小。

您可以在这里使用`null`值来应用其默认值。

除非您知道自己在做什么，否则不应该更改此值。

这可以是一个正数，这意味着一次最多将写入X个字节到底层流资源。注意，如果流资源当前可用的字节数小于X字节，则实际写入的字节数可能更低。

这可以是`-1`，意思是`将所有可用的内容写入底层流资源`。

```php
$stream = new WritableResourceStream(STDOUT, $loop, null, 8192);
```
请参阅[`write()`](#write)了解更多细节。

### DuplexResourceStream

` DuplexResourceStream `是PHP流资源[`DuplexStreamInterface`](#duplexstreaminterface)的具体实现。

用来表示读写资源，比如以读写模式打开的文件流，或者像TCP/IP连接这样的流:

```php
$conn = stream_socket_client('tcp://google.com:80');
$stream = new DuplexResourceStream($conn, $loop);
$stream->write('hello!');
$stream->end();
```

请参阅[`DuplexStreamInterface`](#duplexstreaminterface) 了解更多细节。

构造函数的第一个参数必须是一个有效的流资源，该流资源被打开用于读取*和*写入。

否则，它将抛出一个`InvalidArgumentException`:

```php
// throws InvalidArgumentException
$stream = new DuplexResourceStream(false, $loop);
```

另请参阅只读的[`ReadableResourceStream`](#readableresourcestream)和只写流资源的[`WritableResourceStream`](#writableresourcestream)。

该类内部试图在流资源上启用非阻塞模式，这可能不支持所有的流资源。

最值得注意的是，Windows上的管道(STDOUT、STDERR等)不支持这一点。

如果失败，它将抛出`RuntimeException`:

```php
// throws RuntimeException on Windows
$stream = new DuplexResourceStream(STDOUT, $loop);
```
一旦使用有效的流资源调用构造函数，该类将负责底层的流资源。

您应该只使用它的公共API，而不应该手动干扰底层的流资源。

该类接受一个可选参数`int|null $readChunkSize`，该参数控制一次从流中读取的最大缓冲区大小(以字节为单位)。

您可以在这里使用`null`值来应用其默认值。

除非您知道自己在做什么，否则不应该更改此值。

这可以是一个正数，这意味着一次最多可以从底层流资源读取X个字节。注意，如果流资源当前可用的字节数小于X字节，则实际读取的字节数可能更低。

这可以是`-1`，表示从底层流资源中`读取所有可用的内容`。

这应该读取直到流资源不再可读(即底层缓冲区耗尽)，注意这并不一定意味着它到达了`EOF`。

```php
$conn = stream_socket_client('tcp://google.com:80');
$stream = new DuplexResourceStream($conn, $loop, 8192);
```

对这个类的任何`write()`调用都不会立即执行，而是在`EventLoop`报告流资源准备好接受数据后异步执行。

为此，它使用一个内存缓冲区字符串来收集所有未完成的写操作。

这个缓冲区应用了一个软限制，它定义了在调用者停止发送进一步数据之前，它愿意接受多少数据。

这个类接受另一个可选参数`WritableStreamInterface|null $buffer`，控制这个流的写行为。

您可以在这里使用`null`值来应用其默认值。

除非您知道自己在做什么，否则不应该更改此值。

如果你想改变写缓冲区软限制，你可以传递一个[`WritableResourceStream`](#writableresourcestream) 的实例，像这样:

```php
$conn = stream_socket_client('tcp://google.com:80');
$buffer = new WritableResourceStream($conn, $loop, 8192);
$stream = new DuplexResourceStream($conn, $loop, null, $buffer);
```
参见 [`WritableResourceStream`](#writableresourcestream) 了解更多细节。

### ThroughStream

` ThroughStream `实现了[`DuplexStreamInterface`](#duplexstreaminterface) ，并将任何你写入它的数据传递到它的可读端。

```php
$through = new ThroughStream();
$through->on('data', $this->expectCallableOnceWith('hello'));

$through->write('hello');
```

同样，[`end()` 方法](#end)将结束流并触发[`end`](#end-event)，然后[`close()`](#close-1)流。

[`close()` 方法](#close-1) 将关闭流并发出[`close`](#close-event).

相应地，这也可以像这样在[`pipe()`](#pipe)上下文中使用:

```php
$through = new ThroughStream();
$source->pipe($through)->pipe($dest);
```

可选，它的构造函数接受任何可调用的函数，然后这些函数将被用来 *filter（过滤）* 任何写入它的数据。
此函数在传递到可写端时接收单个数据参数，并且在传递到可读端时必须返回数据：

```php
$through = new ThroughStream('strtoupper');
$source->pipe($through)->pipe($dest);
```
请注意，这个类不假设任何数据类型。这可用于转换数据，例如将任何结构化数据转换为换行符分隔的JSON（NDJSON）流，如下所示：

```php
$through = new ThroughStream(function ($data) {
    return json_encode($data) . PHP_EOL;
});
$through->on('data', $this->expectCallableOnceWith("[2, true]\n"));

$through->write(array(2, true));
```

允许回调函数抛出`Exception`。在这种情况下，流将发出一个`error`事件，然后[`close()`](#close-1)流。

```php
$through = new ThroughStream(function ($data) {
    if (!is_string($data)) {
        throw new \UnexpectedValueException('Only strings allowed');
    }
    return $data;
});
$through->on('error', $this->expectCallableOnce()));
$through->on('close', $this->expectCallableOnce()));
$through->on('data', $this->expectCallableNever()));

$through->write(2);
```

### CompositeStream

` CompositeStream `实现了[`DuplexStreamInterface`](#duplexstreaminterface)，
并可用于从两个分别实现[`ReadableStreamInterface`](#readablestreaminterface)和
[`WritableStreamInterface`](#writablestreaminterface)的单独流中创建一个双工流。

这对于一些可能需要单个[`DuplexStreamInterface`](#duplexstreaminterface) 的api很有用，
或者只是因为像这样使用单个流实例通常更方便:

```php
$stdin = new ReadableResourceStream(STDIN, $loop);
$stdout = new WritableResourceStream(STDOUT, $loop);

$stdio = new CompositeStream($stdin, $stdout);

$stdio->on('data', function ($chunk) use ($stdio) {
    $stdio->write('You said: ' . $chunk);
});
```
这是一个合法流，它从底层流转发所有的流事件，并将所有的流调用转发给底层流。

如果你 `write()` 写入双工流，它将简单地将 `write()` 写入可写端并返回其状态。

如果` end() `双工流，则可写流将` end() `，可读流将` pause() `。

如果` close() `双工流，两个输入流都将被关闭。

如果两个输入流中的任何一个发出` close `事件，双工流也将关闭。

如果两个输入流中的任何一个在构造双工流时已经关闭，它将` close() `另一端并返回一个关闭的流。

## 用法

下面的例子可以用来将源文件的内容管道到目标文件中，而不必将整个文件读入内存:

```php
$loop = new React\EventLoop\StreamSelectLoop;

$source = new React\Stream\ReadableResourceStream(fopen('source.txt', 'r'), $loop);
$dest = new React\Stream\WritableResourceStream(fopen('destination.txt', 'w'), $loop);

$source->pipe($dest);

$loop->run();
```
>注意，这个例子使用` fopen() `只是为了说明。
 在真正的异步程序中不应该使用这种方法，因为文件系统本身就是阻塞的，而且每次调用都可能需要几秒钟的时间。
 参见[创建流](#creating-streams)获取更复杂的示例。

## 安装

推荐的安装这个库的方法是[通过Composer](https://getcomposer.org)。
[Composer 新手?](https://getcomposer.org/doc/00-intro.md)

该项目遵循[SemVer](https://semver.org/) ，
默认安装最新支持的版本:

```bash
$ composer require react/stream:^1.1.1
```

有关版本升级的详细信息，请参见[CHANGELOG](https://reactphp.org/stream/changelog.html) 。

该项目旨在在任何平台上运行，因此不需要任何PHP扩展，并支持通过 `PHP 7+`和`HHVM在旧版PHP 5.3`上运行。

强烈推荐在这个项目中使用*PHP 7+*，因为它有巨大的性能改进。

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

MIT, see [LICENSE file](https://reactphp.org/stream/license.html).

## More

* 有关在实际应用程序中如何创建流的更多信息，请参见[创建流](#creating-streams)。
* 请参阅我们的[用户Wiki](https://github.com/reactphp/react/wiki/Users) 
和[Packagist依赖项](https://packagist.org/packages/react/stream/dependents) 
在实际应用程序中使用流的软件包列表。
