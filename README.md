<div align="center">
    <a href="https://reactphp.org"><img src="https://rawgit.com/reactphp/branding/master/reactphp-logo.svg" alt="ReactPHP Logo" width="160"></a>
</div>
    
<br>
    
<div align="center">
    <strong>Event-driven, non-blocking I/O with PHP.</strong>
</div>

<br>

<div align="center">
    <a href="https://github.com/reactphp/reactphp/actions"><img src="https://github.com/reactphp/reactphp/workflows/CI/badge.svg" alt="Build Status"></a>
</div>

ReactPHP is a low-level library for event-driven programming in PHP. At its core is an event loop, on top of which it provides low-level utilities, such as: Streams abstraction, async DNS resolver, network client/server, HTTP client/server and interaction with processes. Third-party libraries can use these components to create async network clients/servers and more.

<br>


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

echo "Server running at http://127.0.0.1:8080\n";

$loop->run();
```

<div align="center">

This simple web server written in ReactPHP responds with "Hello World" for every request.

</div>

<br>


ReactPHP is a low-level library for event-driven programming in PHP. At its core
is an event loop, on top of which it provides low-level utilities, such as:
Streams abstraction, async DNS resolver, network client/server, HTTP
client/server and interaction with processes. Third-party libraries can use these
components to create async network clients/servers and more.

ReactPHP is production ready and battle-tested with millions of installations
from all kinds of projects around the world. Its event-driven architecture makes
it a perfect fit for efficient network servers and clients handling hundreds or
thousands of concurrent connections, long-running applications and many other
forms of cooperative multitasking with non-blocking I/O operations. What makes
ReactPHP special is its vivid ecosystem with hundreds of third-party libraries
allowing you to integrate with many existing systems, such as common network
services, database systems and other third-party APIs.

* **Production ready** and battle-tested.
* **Rock-solid** with stable long-term support (LTS) releases.
* **Requires no extensions** and runs on any platform - no excuses!
* Takes advantage of **optional extensions** to get better performance when available.
* **Highly recommends latest version of PHP 7+** for best performance and support.
* **Supports legacy PHP 5.3+ and HHVM** for maximum compatibility.
* **Well designed** and **reusable components**.
* **Decoupled parts** so they can be replaced by alternate implementations.
* Carefully **tested** (unit & functional).
* Promotes **standard PSRs** where possible for maximum interoperability.
* Aims to be **technology neutral**, so you can use your preferred application stack.
* Small **core team of professionals** supported by **large network** of outside contributors.

ReactPHP is non-blocking by default. Use workers for blocking I/O.
The event loop is based on the reactor pattern (hence the name) and strongly
inspired by libraries such as EventMachine (Ruby), Twisted (Python) and
Node.js (V8).

> This repository you're currently looking at is mostly used as a meta
  repository to discuss and plan all things @ReactPHP. See the individual
  components linked below for more details about each component, its
  documentation and source code.

## Core Components

* **EventLoop**
  ReactPHP's core reactor event-loop.
  [Read the documentation](1.Core-Components/EventLoop.md)

* **Stream**
  Event-driven readable and writable streams for non-blocking I/O in ReactPHP.
  [Read the documentation](1.Core-Components/Stream.md)

* **Promise**
  Promises/A implementation for PHP.
  [Read the documentation](1.Core-Components/Promise.md)


## Network Components

* **Socket**
  Async, streaming plaintext TCP/IP and secure TLS socket server and client connections for ReactPHP.
  [Read the documentation](2.Network-Components/Socket.md)

* **Datagram**
  Event-driven UDP client and server sockets for ReactPHP.
  [Read the documentation](2.Network-Components/Datagram.md)

## Protocol Components

* **HTTP**
  Event-driven, streaming plaintext HTTP and secure HTTPS server for ReactPHP.
  [Read the documentation](3.Protocol-Components/Http.md)

* **HTTPClient**
  Event-driven, streaming HTTP client for ReactPHP.
  [Read the documentation](3.Protocol-Components/HttpClient.md)

* **DNS**
  Async DNS resolver for ReactPHP.
  [Read the documentation](3.Protocol-Components/Dns.md)

## Utility Components

* **Cache**
  Async caching for ReactPHP.
  [Read the documentation](4.Utility-Components/Cache.md)

* **ChildProcess**
  Library for executing child processes.
  [Read the documentation](4.Utility-Components/ChildProcess.md)

* **PromiseTimer**
  Trivial timeout implementation for ReactPHP's Promise lib.
  [Read the documentation](4.Utility-Components/PromiseTimer.md)

* **PromiseStream**
  The missing link between Promise-land and Stream-land, built on top of ReactPHP.
  [Read the documentation](4.Utility-Components/PromiseStream.md)