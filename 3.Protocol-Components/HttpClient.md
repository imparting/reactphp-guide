# 弃用的通知

该软件包现已迁移到[react/http](3.Protocol-Components/Http.md) ，并且仅出于BC原因而存在。

```bash
$ composer require react/http
```

如果您以前使用过此软件包，则升级可能需要一两分钟。
新的API已更新为使用Promises和PSR-7消息抽象。
这意味着它现在比以往更强大，更易于使用:

```php
// 旧版
$client = new React\HttpClient\Client($loop);
$request = $client->request('GET', 'https://example.com/');
$request->on('response', function ($response) {
    $response->on('data', function ($chunk) {
        echo $chunk;
    });
});
$request->end();

// 新版
$browser = new React\Http\Browser($loop);
$browser->get('https://example.com/')->then(function (Psr\Http\Message\ResponseInterface $response) {
    echo $response->getBody();
});
```

参阅 [react/http](3.Protocol-Components/Http.md#客户端用法)
