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
它不应该在前一个' close '事件之后触发。

After the stream is closed, it MUST switch to non-readable mode,
see also `isReadable()`.

Unlike the `end` event, this event SHOULD be emitted whenever the stream
closes, irrespective of whether this happens implicitly due to an
unrecoverable error or explicitly when either side closes the stream.
If you only want to detect a *successful* end, you should use the `end`
event instead.

Many common streams (such as a TCP/IP connection or a file-based stream)
will likely choose to emit this event after reading a *successful* `end`
event or after a fatal transmission `error` event.

If this stream is a `DuplexStreamInterface`, you should also notice
how the writable side of the stream also implements a `close` event.
In other words, after receiving this event, the stream MUST switch into
non-writable AND non-readable mode, see also `isWritable()`.
Note that this event should not be confused with the `end` event.

#### isReadable()

The `isReadable(): bool` method can be used to
check whether this stream is in a readable state (not closed already).

This method can be used to check if the stream still accepts incoming
data events or if it is ended or closed already.
Once the stream is non-readable, no further `data` or `end` events SHOULD
be emitted.

```php
assert($stream->isReadable() === false);

$stream->on('data', assertNeverCalled());
$stream->on('end', assertNeverCalled());
```

A successfully opened stream always MUST start in readable mode.

Once the stream ends or closes, it MUST switch to non-readable mode.
This can happen any time, explicitly through `close()` or
implicitly due to a remote close or an unrecoverable transmission error.
Once a stream has switched to non-readable mode, it MUST NOT transition
back to readable mode.

If this stream is a `DuplexStreamInterface`, you should also notice
how the writable side of the stream also implements an `isWritable()`
method. Unless this is a half-open duplex stream, they SHOULD usually
have the same return value.

#### pause()

The `pause(): void` method can be used to
pause reading incoming data events.

Removes the data source file descriptor from the event loop. This
allows you to throttle incoming data.

Unless otherwise noted, a successfully opened stream SHOULD NOT start
in paused state.

Once the stream is paused, no futher `data` or `end` events SHOULD
be emitted.

```php
$stream->pause();

$stream->on('data', assertShouldNeverCalled());
$stream->on('end', assertShouldNeverCalled());
```

This method is advisory-only, though generally not recommended, the
stream MAY continue emitting `data` events.

You can continue processing events by calling `resume()` again.

Note that both methods can be called any number of times, in particular
calling `pause()` more than once SHOULD NOT have any effect.

See also `resume()`.

#### resume()

The `resume(): void` method can be used to
resume reading incoming data events.

Re-attach the data source after a previous `pause()`.

```php
$stream->pause();

$loop->addTimer(1.0, function () use ($stream) {
    $stream->resume();
});
```

Note that both methods can be called any number of times, in particular
calling `resume()` without a prior `pause()` SHOULD NOT have any effect.
 
See also `pause()`.

#### pipe()

The `pipe(WritableStreamInterface $dest, array $options = [])` method can be used to
pipe all the data from this readable source into the given writable destination.

Automatically sends all incoming data to the destination.
Automatically throttles the source based on what the destination can handle.

```php
$source->pipe($dest);
```

Similarly, you can also pipe an instance implementing `DuplexStreamInterface`
into itself in order to write back all the data that is received.
This may be a useful feature for a TCP/IP echo service:

```php
$connection->pipe($connection);
```

This method returns the destination stream as-is, which can be used to
set up chains of piped streams:

```php
$source->pipe($decodeGzip)->pipe($filterBadWords)->pipe($dest);
```

By default, this will call `end()` on the destination stream once the
source stream emits an `end` event. This can be disabled like this:

```php
$source->pipe($dest, array('end' => false));
```

Note that this only applies to the `end` event.
If an `error` or explicit `close` event happens on the source stream,
you'll have to manually close the destination stream:

```php
$source->pipe($dest);
$source->on('close', function () use ($dest) {
    $dest->end('BYE!');
});
```

If the source stream is not readable (closed state), then this is a NO-OP.

```php
$source->close();
$source->pipe($dest); // NO-OP
```

If the destinantion stream is not writable (closed state), then this will simply
throttle (pause) the source stream:

```php
$dest->close();
$source->pipe($dest); // calls $source->pause()
```

Similarly, if the destination stream is closed while the pipe is still
active, it will also throttle (pause) the source stream:

```php
$source->pipe($dest);
$dest->close(); // calls $source->pause()
```

Once the pipe is set up successfully, the destination stream MUST emit
a `pipe` event with this source stream an event argument.

#### close()

The `close(): void` method can be used to
close the stream (forcefully).

This method can be used to (forcefully) close the stream.

```php
$stream->close();
```

Once the stream is closed, it SHOULD emit a `close` event.
Note that this event SHOULD NOT be emitted more than once, in particular
if this method is called multiple times.

After calling this method, the stream MUST switch into a non-readable
mode, see also `isReadable()`.
This means that no further `data` or `end` events SHOULD be emitted.

```php
$stream->close();
assert($stream->isReadable() === false);

$stream->on('data', assertNeverCalled());
$stream->on('end', assertNeverCalled());
```

If this stream is a `DuplexStreamInterface`, you should also notice
how the writable side of the stream also implements a `close()` method.
In other words, after calling this method, the stream MUST switch into
non-writable AND non-readable mode, see also `isWritable()`.
Note that this method should not be confused with the `end()` method.

### WritableStreamInterface

The `WritableStreamInterface` is responsible for providing an interface for
write-only streams and the writable side of duplex streams.

Besides defining a few methods, this interface also implements the
`EventEmitterInterface` which allows you to react to certain events.

The event callback functions MUST be a valid `callable` that obeys strict
parameter definitions and MUST accept event parameters exactly as documented.
The event callback functions MUST NOT throw an `Exception`.
The return value of the event callback functions will be ignored and has no
effect, so for performance reasons you're recommended to not return any
excessive data structures.

Every implementation of this interface MUST follow these event semantics in
order to be considered a well-behaving stream.

> Note that higher-level implementations of this interface may choose to
  define additional events with dedicated semantics not defined as part of
  this low-level stream specification. Conformance with these event semantics
  is out of scope for this interface, so you may also have to refer to the
  documentation of such a higher-level implementation.

#### drain事件

The `drain` event will be emitted whenever the write buffer became full
previously and is now ready to accept more data.

```php
$stream->on('drain', function () use ($stream) {
    echo 'Stream is now ready to accept more data';
});
```

This event SHOULD be emitted once every time the buffer became full
previously and is now ready to accept more data.
In other words, this event MAY be emitted any number of times, which may
be zero times if the buffer never became full in the first place.
This event SHOULD NOT be emitted if the buffer has not become full
previously.

This event is mostly used internally, see also `write()` for more details.

#### pipe事件

The `pipe` event will be emitted whenever a readable stream is `pipe()`d
into this stream.
The event receives a single `ReadableStreamInterface` argument for the
source stream.

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

This event MUST be emitted once for each readable stream that is
successfully piped into this destination stream.
In other words, this event MAY be emitted any number of times, which may
be zero times if no stream is ever piped into this stream.
This event MUST NOT be emitted if either the source is not readable
(closed already) or this destination is not writable (closed already).

This event is mostly used internally, see also `pipe()` for more details.

#### error 事件

The `error` event will be emitted once a fatal error occurs, usually while
trying to write to this stream.
The event receives a single `Exception` argument for the error instance.

```php
$stream->on('error', function (Exception $e) {
    echo 'Error: ' . $e->getMessage() . PHP_EOL;
});
```

This event SHOULD be emitted once the stream detects a fatal error, such
as a fatal transmission error.
It SHOULD NOT be emitted after a previous `error` or `close` event.
It MUST NOT be emitted if this is not a fatal error condition, such as
a temporary network issue that did not cause any data to be lost.

After the stream errors, it MUST close the stream and SHOULD thus be
followed by a `close` event and then switch to non-writable mode, see
also `close()` and `isWritable()`.

Many common streams (such as a TCP/IP connection or a file-based stream)
only deal with data transmission and may choose
to only emit this for a fatal transmission error once and will then
close (terminate) the stream in response.

If this stream is a `DuplexStreamInterface`, you should also notice
how the readable side of the stream also implements an `error` event.
In other words, an error may occur while either reading or writing the
stream which should result in the same error processing.

#### close 事件

The `close` event will be emitted once the stream closes (terminates).

```php
$stream->on('close', function () {
    echo 'CLOSED';
});
```

This event SHOULD be emitted once or never at all, depending on whether
the stream ever terminates.
It SHOULD NOT be emitted after a previous `close` event.

After the stream is closed, it MUST switch to non-writable mode,
see also `isWritable()`.

This event SHOULD be emitted whenever the stream closes, irrespective of
whether this happens implicitly due to an unrecoverable error or
explicitly when either side closes the stream.

Many common streams (such as a TCP/IP connection or a file-based stream)
will likely choose to emit this event after flushing the buffer from
the `end()` method, after receiving a *successful* `end` event or after
a fatal transmission `error` event.

If this stream is a `DuplexStreamInterface`, you should also notice
how the readable side of the stream also implements a `close` event.
In other words, after receiving this event, the stream MUST switch into
non-writable AND non-readable mode, see also `isReadable()`.
Note that this event should not be confused with the `end` event.

#### isWritable()

The `isWritable(): bool` method can be used to
check whether this stream is in a writable state (not closed already).

This method can be used to check if the stream still accepts writing
any data or if it is ended or closed already.
Writing any data to a non-writable stream is a NO-OP:

```php
assert($stream->isWritable() === false);

$stream->write('end'); // NO-OP
$stream->end('end'); // NO-OP
```

A successfully opened stream always MUST start in writable mode.

Once the stream ends or closes, it MUST switch to non-writable mode.
This can happen any time, explicitly through `end()` or `close()` or
implicitly due to a remote close or an unrecoverable transmission error.
Once a stream has switched to non-writable mode, it MUST NOT transition
back to writable mode.

If this stream is a `DuplexStreamInterface`, you should also notice
how the readable side of the stream also implements an `isReadable()`
method. Unless this is a half-open duplex stream, they SHOULD usually
have the same return value.

#### write()

The `write(mixed $data): bool` method can be used to
write some data into the stream.

A successful write MUST be confirmed with a boolean `true`, which means
that either the data was written (flushed) immediately or is buffered and
scheduled for a future write. Note that this interface gives you no
control over explicitly flushing the buffered data, as finding the
appropriate time for this is beyond the scope of this interface and left
up to the implementation of this interface.

Many common streams (such as a TCP/IP connection or file-based stream)
may choose to buffer all given data and schedule a future flush by using
an underlying EventLoop to check when the resource is actually writable.

If a stream cannot handle writing (or flushing) the data, it SHOULD emit
an `error` event and MAY `close()` the stream if it can not recover from
this error.

If the internal buffer is full after adding `$data`, then `write()`
SHOULD return `false`, indicating that the caller should stop sending
data until the buffer drains.
The stream SHOULD send a `drain` event once the buffer is ready to accept
more data.

Similarly, if the the stream is not writable (already in a closed state)
it MUST NOT process the given `$data` and SHOULD return `false`,
indicating that the caller should stop sending data.

The given `$data` argument MAY be of mixed type, but it's usually
recommended it SHOULD be a `string` value or MAY use a type that allows
representation as a `string` for maximum compatibility.

Many common streams (such as a TCP/IP connection or a file-based stream)
will only accept the raw (binary) payload data that is transferred over
the wire as chunks of `string` values.

Due to the stream-based nature of this, the sender may send any number
of chunks with varying sizes. There are no guarantees that these chunks
will be received with the exact same framing the sender intended to send.
In other words, many lower-level protocols (such as TCP/IP) transfer the
data in chunks that may be anywhere between single-byte values to several
dozens of kilobytes. You may want to apply a higher-level protocol to
these low-level data chunks in order to achieve proper message framing.

#### end()

The `end(mixed $data = null): void` method can be used to
successfully end the stream (after optionally sending some final data).

This method can be used to successfully end the stream, i.e. close
the stream after sending out all data that is currently buffered.

```php
$stream->write('hello');
$stream->write('world');
$stream->end();
```

If there's no data currently buffered and nothing to be flushed, then
this method MAY `close()` the stream immediately.

If there's still data in the buffer that needs to be flushed first, then
this method SHOULD try to write out this data and only then `close()`
the stream.
Once the stream is closed, it SHOULD emit a `close` event.

Note that this interface gives you no control over explicitly flushing
the buffered data, as finding the appropriate time for this is beyond the
scope of this interface and left up to the implementation of this
interface.

Many common streams (such as a TCP/IP connection or file-based stream)
may choose to buffer all given data and schedule a future flush by using
an underlying EventLoop to check when the resource is actually writable.

You can optionally pass some final data that is written to the stream
before ending the stream. If a non-`null` value is given as `$data`, then
this method will behave just like calling `write($data)` before ending
with no data.

```php
// shorter version
$stream->end('bye');

// same as longer version
$stream->write('bye');
$stream->end();
```

After calling this method, the stream MUST switch into a non-writable
mode, see also `isWritable()`.
This means that no further writes are possible, so any additional
`write()` or `end()` calls have no effect.

```php
$stream->end();
assert($stream->isWritable() === false);

$stream->write('nope'); // NO-OP
$stream->end(); // NO-OP
```

If this stream is a `DuplexStreamInterface`, calling this method SHOULD
also end its readable side, unless the stream supports half-open mode.
In other words, after calling this method, these streams SHOULD switch
into non-writable AND non-readable mode, see also `isReadable()`.
This implies that in this case, the stream SHOULD NOT emit any `data`
or `end` events anymore.
Streams MAY choose to use the `pause()` method logic for this, but
special care may have to be taken to ensure a following call to the
`resume()` method SHOULD NOT continue emitting readable events.

Note that this method should not be confused with the `close()` method.

#### close()

The `close(): void` method can be used to
close the stream (forcefully).

This method can be used to forcefully close the stream, i.e. close
the stream without waiting for any buffered data to be flushed.
If there's still data in the buffer, this data SHOULD be discarded.

```php
$stream->close();
```

Once the stream is closed, it SHOULD emit a `close` event.
Note that this event SHOULD NOT be emitted more than once, in particular
if this method is called multiple times.

After calling this method, the stream MUST switch into a non-writable
mode, see also `isWritable()`.
This means that no further writes are possible, so any additional
`write()` or `end()` calls have no effect.

```php
$stream->close();
assert($stream->isWritable() === false);

$stream->write('nope'); // NO-OP
$stream->end(); // NO-OP
```

Note that this method should not be confused with the `end()` method.
Unlike the `end()` method, this method does not take care of any existing
buffers and simply discards any buffer contents.
Likewise, this method may also be called after calling `end()` on a
stream in order to stop waiting for the stream to flush its final data.

```php
$stream->end();
$loop->addTimer(1.0, function () use ($stream) {
    $stream->close();
});
```

If this stream is a `DuplexStreamInterface`, you should also notice
how the readable side of the stream also implements a `close()` method.
In other words, after calling this method, the stream MUST switch into
non-writable AND non-readable mode, see also `isReadable()`.

### DuplexStreamInterface

The `DuplexStreamInterface` is responsible for providing an interface for
duplex streams (both readable and writable).

It builds on top of the existing interfaces for readable and writable streams
and follows the exact same method and event semantics.
If you're new to this concept, you should look into the
`ReadableStreamInterface` and `WritableStreamInterface` first.

Besides defining a few methods, this interface also implements the
`EventEmitterInterface` which allows you to react to the same events defined
on the `ReadbleStreamInterface` and `WritableStreamInterface`.

The event callback functions MUST be a valid `callable` that obeys strict
parameter definitions and MUST accept event parameters exactly as documented.
The event callback functions MUST NOT throw an `Exception`.
The return value of the event callback functions will be ignored and has no
effect, so for performance reasons you're recommended to not return any
excessive data structures.

Every implementation of this interface MUST follow these event semantics in
order to be considered a well-behaving stream.

> Note that higher-level implementations of this interface may choose to
  define additional events with dedicated semantics not defined as part of
  this low-level stream specification. Conformance with these event semantics
  is out of scope for this interface, so you may also have to refer to the
  documentation of such a higher-level implementation.

See also [`ReadableStreamInterface`](#readablestreaminterface) and
[`WritableStreamInterface`](#writablestreaminterface) for more details.

## Creating streams

ReactPHP uses the concept of "streams" throughout its ecosystem, so that
many higher-level consumers of this package only deal with
[stream usage](#stream-usage).
This implies that stream instances are most often created within some
higher-level components and many consumers never actually have to deal with
creating a stream instance.

* Use [react/socket](https://github.com/reactphp/socket)
  if you want to accept incoming or establish outgoing plaintext TCP/IP or
  secure TLS socket connection streams.
* Use [react/http](https://github.com/reactphp/http)
  if you want to receive an incoming HTTP request body streams.
* Use [react/child-process](https://github.com/reactphp/child-process)
  if you want to communicate with child processes via process pipes such as
  STDIN, STDOUT, STDERR etc.
* Use experimental [react/filesystem](https://github.com/reactphp/filesystem)
  if you want to read from / write to the filesystem.
* See also the last chapter for [more real-world applications](#more).

However, if you are writing a lower-level component or want to create a stream
instance from a stream resource, then the following chapter is for you.

> Note that the following examples use `fopen()` and `stream_socket_client()`
  for illustration purposes only.
  These functions SHOULD NOT be used in a truly async program because each call
  may take several seconds to complete and would block the EventLoop otherwise.
  Additionally, the `fopen()` call will return a file handle on some platforms
  which may or may not be supported by all EventLoop implementations.
  As an alternative, you may want to use higher-level libraries listed above.

### ReadableResourceStream

The `ReadableResourceStream` is a concrete implementation of the
[`ReadableStreamInterface`](#readablestreaminterface) for PHP's stream resources.

This can be used to represent a read-only resource like a file stream opened in
readable mode or a stream such as `STDIN`:

```php
$stream = new ReadableResourceStream(STDIN, $loop);
$stream->on('data', function ($chunk) {
    echo $chunk;
});
$stream->on('end', function () {
    echo 'END';
});
```

See also [`ReadableStreamInterface`](#readablestreaminterface) for more details.

The first parameter given to the constructor MUST be a valid stream resource
that is opened in reading mode (e.g. `fopen()` mode `r`).
Otherwise, it will throw an `InvalidArgumentException`:

```php
// throws InvalidArgumentException
$stream = new ReadableResourceStream(false, $loop);
```

See also the [`DuplexResourceStream`](#readableresourcestream) for read-and-write
stream resources otherwise.

Internally, this class tries to enable non-blocking mode on the stream resource
which may not be supported for all stream resources.
Most notably, this is not supported by pipes on Windows (STDIN etc.).
If this fails, it will throw a `RuntimeException`:

```php
// throws RuntimeException on Windows
$stream = new ReadableResourceStream(STDIN, $loop);
```

Once the constructor is called with a valid stream resource, this class will
take care of the underlying stream resource.
You SHOULD only use its public API and SHOULD NOT interfere with the underlying
stream resource manually.

This class takes an optional `int|null $readChunkSize` parameter that controls
the maximum buffer size in bytes to read at once from the stream.
You can use a `null` value here in order to apply its default value.
This value SHOULD NOT be changed unless you know what you're doing.
This can be a positive number which means that up to X bytes will be read
at once from the underlying stream resource. Note that the actual number
of bytes read may be lower if the stream resource has less than X bytes
currently available.
This can be `-1` which means "read everything available" from the
underlying stream resource.
This should read until the stream resource is not readable anymore
(i.e. underlying buffer drained), note that this does not neccessarily
mean it reached EOF.

```php
$stream = new ReadableResourceStream(STDIN, $loop, 8192);
```

> PHP bug warning: If the PHP process has explicitly been started without a
  `STDIN` stream, then trying to read from `STDIN` may return data from
  another stream resource. This does not happen if you start this with an empty
  stream like `php test.php < /dev/null` instead of `php test.php <&-`.
  See [#81](https://github.com/reactphp/stream/issues/81) for more details.

### WritableResourceStream

The `WritableResourceStream` is a concrete implementation of the
[`WritableStreamInterface`](#writablestreaminterface) for PHP's stream resources.

This can be used to represent a write-only resource like a file stream opened in
writable mode or a stream such as `STDOUT` or `STDERR`:

```php
$stream = new WritableResourceStream(STDOUT, $loop);
$stream->write('hello!');
$stream->end();
```

See also [`WritableStreamInterface`](#writablestreaminterface) for more details.

The first parameter given to the constructor MUST be a valid stream resource
that is opened for writing.
Otherwise, it will throw an `InvalidArgumentException`:

```php
// throws InvalidArgumentException
$stream = new WritableResourceStream(false, $loop);
```

See also the [`DuplexResourceStream`](#readableresourcestream) for read-and-write
stream resources otherwise.

Internally, this class tries to enable non-blocking mode on the stream resource
which may not be supported for all stream resources.
Most notably, this is not supported by pipes on Windows (STDOUT, STDERR etc.).
If this fails, it will throw a `RuntimeException`:

```php
// throws RuntimeException on Windows
$stream = new WritableResourceStream(STDOUT, $loop);
```

Once the constructor is called with a valid stream resource, this class will
take care of the underlying stream resource.
You SHOULD only use its public API and SHOULD NOT interfere with the underlying
stream resource manually.

Any `write()` calls to this class will not be performed instantly, but will
be performed asynchronously, once the EventLoop reports the stream resource is
ready to accept data.
For this, it uses an in-memory buffer string to collect all outstanding writes.
This buffer has a soft-limit applied which defines how much data it is willing
to accept before the caller SHOULD stop sending further data.

This class takes an optional `int|null $writeBufferSoftLimit` parameter that controls
this maximum buffer size in bytes.
You can use a `null` value here in order to apply its default value.
This value SHOULD NOT be changed unless you know what you're doing.

```php
$stream = new WritableResourceStream(STDOUT, $loop, 8192);
```

This class takes an optional `int|null $writeChunkSize` parameter that controls
this maximum buffer size in bytes to write at once to the stream.
You can use a `null` value here in order to apply its default value.
This value SHOULD NOT be changed unless you know what you're doing.
This can be a positive number which means that up to X bytes will be written
at once to the underlying stream resource. Note that the actual number
of bytes written may be lower if the stream resource has less than X bytes
currently available.
This can be `-1` which means "write everything available" to the
underlying stream resource.

```php
$stream = new WritableResourceStream(STDOUT, $loop, null, 8192);
```

See also [`write()`](#write) for more details.

### DuplexResourceStream

The `DuplexResourceStream` is a concrete implementation of the
[`DuplexStreamInterface`](#duplexstreaminterface) for PHP's stream resources.

This can be used to represent a read-and-write resource like a file stream opened
in read and write mode mode or a stream such as a TCP/IP connection:

```php
$conn = stream_socket_client('tcp://google.com:80');
$stream = new DuplexResourceStream($conn, $loop);
$stream->write('hello!');
$stream->end();
```

See also [`DuplexStreamInterface`](#duplexstreaminterface) for more details.

The first parameter given to the constructor MUST be a valid stream resource
that is opened for reading *and* writing.
Otherwise, it will throw an `InvalidArgumentException`:

```php
// throws InvalidArgumentException
$stream = new DuplexResourceStream(false, $loop);
```

See also the [`ReadableResourceStream`](#readableresourcestream) for read-only
and the [`WritableResourceStream`](#writableresourcestream) for write-only
stream resources otherwise.

Internally, this class tries to enable non-blocking mode on the stream resource
which may not be supported for all stream resources.
Most notably, this is not supported by pipes on Windows (STDOUT, STDERR etc.).
If this fails, it will throw a `RuntimeException`:

```php
// throws RuntimeException on Windows
$stream = new DuplexResourceStream(STDOUT, $loop);
```

Once the constructor is called with a valid stream resource, this class will
take care of the underlying stream resource.
You SHOULD only use its public API and SHOULD NOT interfere with the underlying
stream resource manually.

This class takes an optional `int|null $readChunkSize` parameter that controls
the maximum buffer size in bytes to read at once from the stream.
You can use a `null` value here in order to apply its default value.
This value SHOULD NOT be changed unless you know what you're doing.
This can be a positive number which means that up to X bytes will be read
at once from the underlying stream resource. Note that the actual number
of bytes read may be lower if the stream resource has less than X bytes
currently available.
This can be `-1` which means "read everything available" from the
underlying stream resource.
This should read until the stream resource is not readable anymore
(i.e. underlying buffer drained), note that this does not neccessarily
mean it reached EOF.

```php
$conn = stream_socket_client('tcp://google.com:80');
$stream = new DuplexResourceStream($conn, $loop, 8192);
```

Any `write()` calls to this class will not be performed instantly, but will
be performed asynchronously, once the EventLoop reports the stream resource is
ready to accept data.
For this, it uses an in-memory buffer string to collect all outstanding writes.
This buffer has a soft-limit applied which defines how much data it is willing
to accept before the caller SHOULD stop sending further data.

This class takes another optional `WritableStreamInterface|null $buffer` parameter
that controls this write behavior of this stream.
You can use a `null` value here in order to apply its default value.
This value SHOULD NOT be changed unless you know what you're doing.

If you want to change the write buffer soft limit, you can pass an instance of
[`WritableResourceStream`](#writableresourcestream) like this:

```php
$conn = stream_socket_client('tcp://google.com:80');
$buffer = new WritableResourceStream($conn, $loop, 8192);
$stream = new DuplexResourceStream($conn, $loop, null, $buffer);
```

See also [`WritableResourceStream`](#writableresourcestream) for more details.

### ThroughStream

The `ThroughStream` implements the
[`DuplexStreamInterface`](#duplexstreaminterface) and will simply pass any data
you write to it through to its readable end.

```php
$through = new ThroughStream();
$through->on('data', $this->expectCallableOnceWith('hello'));

$through->write('hello');
```

Similarly, the [`end()` method](#end) will end the stream and emit an
[`end` event](#end-event) and then [`close()`](#close-1) the stream.
The [`close()` method](#close-1) will close the stream and emit a
[`close` event](#close-event).
Accordingly, this is can also be used in a [`pipe()`](#pipe) context like this:

```php
$through = new ThroughStream();
$source->pipe($through)->pipe($dest);
```

Optionally, its constructor accepts any callable function which will then be
used to *filter* any data written to it. This function receives a single data
argument as passed to the writable side and must return the data as it will be
passed to its readable end:

```php
$through = new ThroughStream('strtoupper');
$source->pipe($through)->pipe($dest);
```

Note that this class makes no assumptions about any data types. This can be
used to convert data, for example for transforming any structured data into
a newline-delimited JSON (NDJSON) stream like this:

```php
$through = new ThroughStream(function ($data) {
    return json_encode($data) . PHP_EOL;
});
$through->on('data', $this->expectCallableOnceWith("[2, true]\n"));

$through->write(array(2, true));
```

The callback function is allowed to throw an `Exception`. In this case,
the stream will emit an `error` event and then [`close()`](#close-1) the stream.

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

The `CompositeStream` implements the
[`DuplexStreamInterface`](#duplexstreaminterface) and can be used to create a
single duplex stream from two individual streams implementing
[`ReadableStreamInterface`](#readablestreaminterface) and
[`WritableStreamInterface`](#writablestreaminterface) respectively.

This is useful for some APIs which may require a single
[`DuplexStreamInterface`](#duplexstreaminterface) or simply because it's often
more convenient to work with a single stream instance like this:

```php
$stdin = new ReadableResourceStream(STDIN, $loop);
$stdout = new WritableResourceStream(STDOUT, $loop);

$stdio = new CompositeStream($stdin, $stdout);

$stdio->on('data', function ($chunk) use ($stdio) {
    $stdio->write('You said: ' . $chunk);
});
```

This is a well-behaving stream which forwards all stream events from the
underlying streams and forwards all streams calls to the underlying streams.

If you `write()` to the duplex stream, it will simply `write()` to the
writable side and return its status.

If you `end()` the duplex stream, it will `end()` the writable side and will
`pause()` the readable side.

If you `close()` the duplex stream, both input streams will be closed.
If either of the two input streams emits a `close` event, the duplex stream
will also close.
If either of the two input streams is already closed while constructing the
duplex stream, it will `close()` the other side and return a closed stream.

## 用法

The following example can be used to pipe the contents of a source file into
a destination file without having to ever read the whole file into memory:

```php
$loop = new React\EventLoop\StreamSelectLoop;

$source = new React\Stream\ReadableResourceStream(fopen('source.txt', 'r'), $loop);
$dest = new React\Stream\WritableResourceStream(fopen('destination.txt', 'w'), $loop);

$source->pipe($dest);

$loop->run();
```

> Note that this example uses `fopen()` for illustration purposes only.
  This should not be used in a truly async program because the filesystem is
  inherently blocking and each call could potentially take several seconds.
  See also [creating streams](#creating-streams) for more sophisticated
  examples.

## 安装

The recommended way to install this library is [through Composer](https://getcomposer.org).
[New to Composer?](https://getcomposer.org/doc/00-intro.md)

This project follows [SemVer](https://semver.org/).
This will install the latest supported version:

```bash
$ composer require react/stream:^1.1.1
```

See also the [CHANGELOG](CHANGELOG.md) for details about version upgrades.

This project aims to run on any platform and thus does not require any PHP
extensions and supports running on legacy PHP 5.3 through current PHP 7+ and HHVM.
It's *highly recommended to use PHP 7+* for this project due to its vast
performance improvements.

## 测试

To run the test suite, you first need to clone this repo and then install all
dependencies [through Composer](https://getcomposer.org):

```bash
$ composer install
```

To run the test suite, go to the project root and run:

```bash
$ php vendor/bin/phpunit
```

The test suite also contains a number of functional integration tests that rely
on a stable internet connection.
If you do not want to run these, they can simply be skipped like this:

```bash
$ php vendor/bin/phpunit --exclude-group internet
```

## License

MIT, see [LICENSE file](LICENSE).

## More

* See [creating streams](#creating-streams) for more information on how streams
  are created in real-world applications.
* See our [users wiki](https://github.com/reactphp/react/wiki/Users) and the
  [dependents on Packagist](https://packagist.org/packages/react/stream/dependents)
  for a list of packages that use streams in real-world applications.
