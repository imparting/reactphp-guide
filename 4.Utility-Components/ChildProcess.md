# Child Process

[![Build Status](https://travis-ci.org/reactphp/child-process.svg?branch=master)](https://travis-ci.org/reactphp/child-process)

使用[ReactPHP](https://reactphp.org/)
执行子进程的事件驱动库。

该库将 [Program Execution](http://php.net/manual/en/book.exec.php)
与[EventLoop](https://github.com/reactphp/event-loop)
集成在一起。

已启动的子进程可能会发出信号，并在终止时发出`exit`事件。

此外，进程I / O流（即`STDIN`，`STDOUT`，`STDERR`）公开为[Streams](https://github.com/reactphp/stream)

**目录**

* [快速开始](#快速开始)
* [Process](#process)
  * [Stream特性](#stream特性)
  * [Command](#command)
  * [Termination](#termination)
  * [自定义pipes](#自定义pipes)
  * [Sigchild兼容性](#sigchild兼容性)
  * [Windows兼容性](#windows兼容性)
* [安装](#安装)
* [测试](#测试)
* [License](#license)

## 快速开始

```php
$loop = React\EventLoop\Factory::create();

$process = new React\ChildProcess\Process('echo foo');
$process->start($loop);

$process->stdout->on('data', function ($chunk) {
    echo $chunk;
});

$process->on('exit', function($exitCode, $termSignal) {
    echo 'Process exited with code ' . $exitCode . PHP_EOL;
});

$loop->run();
```

请参阅[示例](https://github.com/reactphp/child-process/blob/v0.6.1/examples)

## Process

### Stream特性

进程一旦启动，它的I/O流将被构造为`React\Stream\ReadableStreamInterface`和`React\Stream\WritableStreamInterface`的实例。
在调用`start（）`之前，未设置这些属性。 进程终止后，流将关闭但不会取消设置。

遵循通用的Unix约定，该库将使用匹配标准I / O流的三个管道（默认情况下如下）启动每个子进程。
您可以对常用用例使用命名引用，也可以将它们作为具有所有三个管道的数组访问。

* `$stdin`  或 `$pipes[0]` 是 `WritableStreamInterface`
* `$stdout` 或 `$pipes[1]` 是 `ReadableStreamInterface`
* `$stderr` 或 `$pipes[2]` 是 `ReadableStreamInterface`

请注意，可以通过显式传递[自定义管道](#custom-pipes) 来覆盖此默认配置，
所以您可以不设置它们或为它们分配不同的值。
特别要注意的是，[Windows支持](#windows-compatibility) 受限制，因为它不支持非阻塞`STDIO`管道。
`$pipes`数组将始终包含对所有已配置管道的引用，并且始终将标准I / O引用设置为引用符合上述约定的管道。 
有关更多详细信息，请参见[自定义管道](#custom-pipes) 。

因为它们都实现了底层的[`ReadableStreamInterface`](https://github.com/reactphp/stream#readablestreaminterface)
或[`WritableStreamInterface`](https://github.com/reactphp/stream#writablestreaminterface) ，
你可以像往常一样使用它们的事件和方法:

```php
$process = new Process($command);
$process->start($loop);

$process->stdout->on('data', function ($chunk) {
    echo $chunk;
});

$process->stdout->on('end', function () {
    echo 'ended';
});

$process->stdout->on('error', function (Exception $e) {
    echo 'error: ' . $e->getMessage();
});

$process->stdout->on('close', function () {
    echo 'closed';
});

$process->stdin->write($data);
$process->stdin->end($data = null);
// …
```

要了解更多信息，请参阅
[`ReadableStreamInterface`](https://github.com/reactphp/stream#readablestreaminterface)
和 
[`WritableStreamInterface`](https://github.com/reactphp/stream#writablestreaminterface)

### Command

` Process `类允许你传递任何类型的命令行字符串:

```php
$process = new Process('echo test');
$process->start($loop);
```

命令行字符串通常由一个以空格分隔的列表组成，其中包含主可执行文件bin和任意数量的参数。
需要注意转义或引用参数，特别是在传递用户输入时。
同样，请记住，尤其是在Windows上，路径名包含空格和其他特殊字符是很常见的。
如果要运行这样的二进制文件，则必须确保使用`escapeshellarg()`将其作为单个参数引用，如下所示：

```php
$bin = 'C:\\Program files (x86)\\PHP\\php.exe';
$file = 'C:\\Users\\me\\Desktop\\Application\\main.php';

$process = new Process(escapeshellarg($bin) . ' ' . escapeshellarg($file));
$process->start($loop);
```

默认情况下，在Unix上，PHP将通过将命令行字符串包装在`sh`命令中来启动进程，
因此第一个示例将在Unix上的引擎盖下实际执行`sh-c echo test`。
在Windows上，它不会通过将进程包装在shell中来启动进程。

这是一个非常有用的特性，因为它不仅允许您传递单个命令，而且实际上还允许您传递任何类型的shell命令行，
并且可以使用命令链（使用`&&`、`||`、`;`和其他命令）启动多个子命令，
并允许您重定向`STDIO`流（使用`2>&1`和家族命令）。
这可用于传递完整的命令行，并从包装shell命令接收生成的`STDIO`流，如下所示：

```php
$process = new Process('echo run && demo || echo failed');
$process->start($loop);
```

>请注意，[Windows支持](#windows-compatibility)受到限制，因为它根本不支持`STDIO`流，
 而且默认情况下进程不会在包装shell中运行。如果要运行shell内置函数，如`echo hello`或`sleep 10`，
 可能需要在命令行前面加上一个显式shell，如`cmd/c echo hello`。

换句话说，底层shell负责管理此命令行并启动各个子命令，并适当地连接其`STDIO`流。
这意味着`Process`类将仅从包装shell接收生成的`STDIO`流，因此它将包含完整的输入/输出，而无法辨别单个子命令的输入/输出。
如果要识别单个子命令的输出，则可能需要实现一些更高级别的协议逻辑，例如在每个子命令之间打印显式边界，如下所示：

```php
$process = new Process('cat first && echo --- && cat second');
$process->start($loop);
```

另一种办法是，考虑一次启动一个进程，监听其`exit`事件，有条件地启动链中的下一个进程。
这将给你一个机会来配置后续的进程I/O流:

```php
$first = new Process('cat first');
$first->start($loop);

$first->on('exit', function () use ($loop) {
    $second = new Process('cat second');
    $second->start($loop);
});
```

请记住，PHP对Unix上的所有命令行都使用shell包装器。
虽然这对于更复杂的命令行似乎是合理的，但实际上这也适用于运行最简单的单个命令：

```php
$process = new Process('yes');
$process->start($loop);
```

实际上，这将产生一个类似于Unix上的命令层次结构：

```
5480 … \_ php example.php
5481 …    \_ sh -c yes
5482 …        \_ yes
```

这意味着获取底层进程PID或发送信号，实际上目标却是shell包装器，这种情况下可能不是您期望的结果。

如果你不想显示这个shell包装器进程，在Unix平台上你可以在命令串前加上` exec `，这将导致shell包装器进程被我们的进程所取代:

```php
$process = new Process('exec yes');
$process->start($loop);
```

这将显示与以下类似的结果命令层次结构：

```
5480 … \_ php example.php
5481 …    \_ yes
```

这样获取底层进程PID和发送信号现在将以预期的实际命令为目标。

注意，在这种情况下，命令行不会在shell包装器中运行。
这意味着在使用` exec `时，无法传递命令行，比如那些包含命令链或重定向`STDIO`流的命令行。

根据经验，大多数命令都可以在shell包装器中正常运行。
如果您传递了一个完整的命令行(或者不确定)，您很可能会保留shell包装器。
如果您运行在Unix上，并且希望只传递一个单独的命令，您可能需要考虑在命令字符串前面加上` exec `，以避免shell包装器。

### Termination

每当进程不再运行时，都会发出`exit`事件。
事件侦听器将接收退出代码和终止信号作为两个参数：

```php
$process = new Process('sleep 10');
$process->start($loop);

$process->on('exit', function ($code, $term) {
    if ($term === null) {
        echo 'exit with code ' . $code . PHP_EOL;
    } else {
        echo 'terminated with signal ' . $term . PHP_EOL;
    }
});
```
注意，如果进程已经终止，则` $code `为` null `，但无法确定退出码(例如[sigchild compatibility](#sigchild-compatibility)被禁用)。
同样，除非进程响应发送给它的未捕获信号而终止，否则`$term`为`null`。
这并不是这个项目的限制，而是在POSIX系统上如何公开退出码和信号的实际限制，更多细节请参阅
[这里](https://unix.stackexchange.com/questions/99112/default-exit-code-when-process-is-terminated)

还值得注意的是，进程终止取决于事先关闭的所有文件描述符。
这意味着所有 [process pipes](#stream-properties) 将在` exit `事件之前触发` close `事件，
而在` exit `事件之后不会再有` data `事件到达。
因此如果这些管道中的任何一个处于暂停状态(` pause() `方法或内部由于` pipe() `调用)，
此检测可能不会触发。

可以使用`terminate(?int $signal = null): bool`方法向进程发送信号（默认为SIGTERM）。
根据您向进程发送的信号以及它是否已注册信号处理程序，这可以用于仅向进程发送信号，甚至可以强制终止它。 

```php
$process->terminate(SIGUSR1);
```

如果要强制终止进程，请记住上面的部分。
如果您的进程生成子进程或隐式使用[shell包装器](#command) ，
则其文件描述符可能会继承给子进程，终止主进程不一定会终止整个进程树。
强烈建议您在终止进程时相应地将所有进程管道显式`close()`：

```php
$process = new Process('sleep 10');
$process->start($loop);

$loop->addTimer(2.0, function () use ($process) {
    foreach ($process->pipes as $pipe) {
        $pipe->close();
    }
    $process->terminate();
});
```

对于许多简单的程序，这些极其复杂的步骤也可以通过在命令行前面加上`exec`来避免，
以避免包装shell及其继承的进程管道[如上所述](#command)

```php
$process = new Process('exec sleep 10');
$process->start($loop);

$loop->addTimer(2.0, function () use ($process) {
    $process->terminate();
});
```

许多命令行程序需要等待` STDIN `上的数据，并在此管道关闭时干净地终止。
例如，下面的方法可以用来`软关闭`一个`cat`进程:

```php
$process = new Process('cat');
$process->start($loop);

$loop->addTimer(2.0, function () use ($process) {
    $process->stdin->end();
});
```

虽然进程管道和终止可能会让新手感到困惑，但上面的属性实际上允许对进程终止进行一些细粒度的控制，
比如首先尝试软关闭，然后在超时后应用强制关闭。

### 自定义pipes

按照常见的Unix约定，默认情况下，这个库将使用匹配标准I/O流的三个管道启动每个子进程。
对于更高级的用例，传入自定义管道可能很有用，例如显式地传入额外的文件描述符(FDs)或覆盖默认进程管道。

注意，传递自定义管道被认为是高级用法，需要更深入地理解Unix文件描述符，
以及它们如何继承到子进程并在多处理应用程序中共享。

如果你不想使用默认的标准I/O管道，你可以像这样显式地将一个包含文件描述符规范的数组传递给构造函数:

```php
$fds = array(
    // 标准输入输出管道用于stdin/stdout/stderr
    0 => array('pipe', 'r'),
    1 => array('pipe', 'w'),
    2 => array('pipe', 'w'),

    // 使用实例文件或开放资源的FDs
    4 => array('file', '/dev/null', 'r'),
    6 => fopen('log.txt','a'),
    8 => STDERR,

    // socket的FDs示例
    10 => fsockopen('localhost', 8080),
    12 => stream_socket_server('tcp://0.0.0.0:4711')
);

$process = new Process($cmd, null, null, $fds);
$process->start($loop);
```

除非您的用例有其他特殊需求，否则强烈建议您（至少）传入上面给出的标准I/O管道。
文件描述符规范接受与基础[`proc_open()`](http://php.net/proc_open)格式完全相同的参数。

一旦进程启动，`$pipes`数组将始终包含对配置的所有管道的引用，
并且标准I/O引用将始终设置为引用与常见Unix约定匹配的管道。
此库支持任意数量的管道和附加的文件描述符，
但是许多作为子进程运行的常见应用程序都希望父进程正确地分配这些文件描述符。

### Sigchild兼容性

该项目使用了一种变通方法来提高PHP在使用`--enable-sigchild`选项编译时的兼容性。
这应该不会影响大多数安装，因为默认情况下这个配置选项并没有被使用，
而且许多发行版(如Debian和Ubuntu)默认情况下都不使用这个选项。
一些使用[Oracle OCI8](http://php.net/manual/en/book.oci8.php)
的安装可能会使用这个配置选项来绕过` defunct `进程。

当PHP使用`--enable-sigchild`选项编译时，子进程的退出码不能通过` proc_close() `或` proc_get_status() `可靠地确定。
为了解决这个问题，我们使用附加管道执行子进程，并使用该管道检索其退出代码。

这个方法会带来一些开销，所以我们只在必要的时候触发它，并且当我们检测到PHP已经使用`--enable-sigchild`选项编译时触发它。
因为PHP没有提供一种可靠的方法来检测这个选项，所以我们尝试检查` phpinfo() `函数的PHP配置选项的输出。

`setSigchildEnabled(bool $sigchild): void`静态方法可以像这样显式地启用或禁用这种行为:

```php
// 高级:默认不推荐
Process::setSigchildEnabled(true);
```

请注意，在此方法调用之后实例化的所有进程都将受到影响。
如果在受影响的PHP安装上禁用了此解决方案，` exit `事件可能接收到` null `，而不是如上所述的实际退出代码。

一些发行版会忽略` phpinfo() `中的配置选项，所以自动检测在某些情况下可能无法启用这个解决方案。
然后，您可以像上面所给出的那样显式地启用它。

**注意:** 最初的功能来自Symfony的 [Process](https://github.com/symfony/process) 组件。

### Windows兼容性

由于平台限制，此库仅为Windows上的派生子进程提供有限的支持。
特别是，PHP不允许在没有阻塞的情况下访问标准I/O管道。因此，
此项目不允许使用默认进程管道构造子进程，在Windows上默认情况下抛出`LogicException`：

```php
// 在Windows上抛出`LogicException`
$process = new Process('ping example.com');
$process->start($loop);
```

如果要在Windows上运行子进程，有许多替代方法和解决方法，如下所述，每种方法都有自己的优缺点：

* 该软件包可以在[`Linux的Windows子系统(WSL)`](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux) 上正常运行。
  当您控制应用程序的部署方式时，如果要在Windows上运行此程序包，
  建议使用[安装WSL](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide)

* 如果只关心子进程的退出代码以检查其执行是否成功，则可以使用[自定义pipes](#自定义pipes)省略任何标准的I / O管道，如下所示：
    ```php
    $process = new Process('ping example.com', null, null, array());
    $process->start($loop);

    $process->on('exit', function ($exitcode) {
        echo 'exit with ' . $exitcode . PHP_EOL;
    });
    ```
    同样，如果您的子进程使用[Socket组件](/2.Network-Components/Socket.md)
    通过套接字与远程服务器甚至是父进程进行通信，这也很有用。 
    如果您可以控制子进程与父进程之间的通信方式这是最佳选择。

*   如果你只关心子进程执行后的命令输出，你可以使用[自定义pipes](#自定义pipes)
    来配置传递给子进程的文件句柄，而不是像这样的管道:

    ```php
    $process = new Process('ping example.com', null, null, array(
        array('file', 'nul', 'r'),
        $stdout = tmpfile(),
        array('file', 'nul', 'w')
    ));
    $process->start($loop);

    $process->on('exit', function ($exitcode) use ($stdout) {
        echo 'exit with ' . $exitcode . PHP_EOL;

        // rewind to start and then read full file (demo only, this is blocking).
        // reading from shared file is only safe if you have some synchronization in place
        // or after the child process has terminated.
        rewind($stdout);
        echo stream_get_contents($stdout);
        fclose($stdout);
    });
    ```

    注意，这个例子使用` tmpfile() ` / ` fopen() `只是为了说明。
    在真正的异步程序中不应该使用这种方法，因为文件系统本身就是阻塞的，而且每次调用都可能需要几秒钟的时间。
    另一个替代方案是[Filesystem component](https://github.com/reactphp/filesystem)

*   如果你想以流方式访问命令输出，你可以使用重定向生成一个额外的进程，
    将你的标准I/O流转发到套接字，并使用[自定义pipes](#custom pipes)
    来省略任何实际的标准I/O管道，就像这样:

    ```php
    $server = new React\Socket\Server('127.0.0.1:0', $loop);
    $server->on('connection', function (React\Socket\ConnectionInterface $connection) {
        $connection->on('data', function ($chunk) {
            echo $chunk;
        });
    });

    $command = 'ping example.com | foobar ' . escapeshellarg($server->getAddress());
    $process = new Process($command, null, null, array());
    $process->start($loop);

    $process->on('exit', function ($exitcode) use ($server) {
        $server->close();
        echo 'exit with ' . $exitcode . PHP_EOL;
    });
    ```

    请注意，这将产生另一个虚构的`foobar`助手程序，以使用实际子进程的标准输出。
    这实际上类似于上面建议的在子进程中使用套接字连接，但是在这种情况下不需要修改实际的子进程。

    在本例中，虚构的`foobar`助手程序可以通过使用标准输入并将所有数据转发到如下套接字连接中：

    ```php
    $socket = stream_socket_client($argv[1]);
    do {
        fwrite($socket, $data = fread(STDIN, 8192));
    } while (isset($data[0]));
    ```

    因此，此示例也可以使用普通PHP运行，而不必依赖任何外部帮助程序，如：

    ```php
    $code = '$s=stream_socket_client($argv[1]);do{fwrite($s,$d=fread(STDIN, 8192));}while(isset($d[0]));';
    $command = 'ping example.com | php -r ' . escapeshellarg($code) . ' ' . escapeshellarg($server->getAddress());
    $process = new Process($command, null, null, array());
    $process->start($loop);
    ```

    请参阅[example #23](https://github.com/reactphp/child-process/blob/v0.6.1/examples/23-forward-socket.php).

    请注意，这只是为了说明，如果您不想让其他进程冒着连接到服务器套接字的风险，那么您可能希望在实际生产使用中实现一些适当的错误检查和/或套接字验证。
    在这种情况下，我们建议查看优秀的[创建进程-windows](https://github.com/cubiclesoft/createprocess-windows)

此外，请注意，`Process`的[command](#command)将传递给底层的Windows API
([`CreateProcess`](https://docs.microsoft.com/en-us/windows/desktop/api/processthreadsapi/nf-processthreadsapi-createprocessa))
默认情况下，进程不会在shell包装器中启动。这意味着诸如`echo hello`或`sleep 10`之类的shell内置函数可能必须以如下显式shell命令作为前缀：

```php
$process = new Process('cmd /c echo hello', null, null, $pipes);
$process->start($loop);
```

## 安装

推荐的安装这个库的方法是[通过Composer](https://getcomposer.org)。
[Composer 新手?](https://getcomposer.org/doc/00-intro.md)

默认安装最新支持的版本:

```bash
$ composer require react/child-process:^0.6.1
```

有关版本升级的详细信息，请参见[CHANGELOG](https://reactphp.org/child-process/changelog.html)

该项目旨在在任何平台上运行，因此不需要任何PHP扩展，并支持通过 `PHP 7+`和`HHVM在旧版PHP 5.3`上运行。

强烈推荐在这个项目中使用*PHP 7+*。

见上文关于限制的说明[Windows Compatibility](#windows兼容性).

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

MIT, see [LICENSE file](https://reactphp.org/child-process/license.html).
