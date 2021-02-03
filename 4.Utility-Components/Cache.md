# Cache

[![Build Status](https://travis-ci.org/reactphp/cache.svg?branch=master)](https://travis-ci.org/reactphp/cache)

[ReactPHP](https://reactphp.org/)
的基于[Promise](https://github.com/reactphp/promise)
的异步缓存接口。 

缓存组件提供了一个基于[Promise](https://github.com/reactphp/promise)
的[`CacheInterface`](#cacheinterface) 和其在内存中的[`ArrayCache`](#arraycache) 实现。

这允许使用者在界面上键入提示，并允许第三方提供替代的实现。

该项目的主要灵感来自[PSR-16:缓存库的通用接口](https://www.php-fig.org/psr/psr-16/) ，
但使用更适合异步、非阻塞应用程序的接口。

**目录**

* [用法](#用法)
  * [CacheInterface](#cacheinterface)
    * [get()](#get)
    * [set()](#set)
    * [delete()](#delete)
    * [getMultiple()](#getmultiple)
    * [setMultiple()](#setmultiple)
    * [deleteMultiple()](#deletemultiple)
    * [clear()](#clear)
    * [has()](#has)
  * [ArrayCache](#arraycache)
* [常见用法](#常见用法)
  * [Fallback get](#fallback-get)
  * [Fallback-get-and-set](#fallback-get-and-set)
* [安装](#安装)
* [测试](#测试)
* [License](#license)

## 用法

### CacheInterface

`CacheInterface`描述此组件的主接口。

允许使用者针对接口键入提示，并允许第三方提供替代实现。

#### get()

`get(string $key, mixed $default = null): PromiseInterface<mixed>`方法可以用来从缓存中获取值。

成功时，该方法将使用缓存的值进行解析;当找不到任何项目或发生错误时，该方法将返回` $default `值进行解析。
同样，过期的缓存项(一旦生命周期过期)被认为是缓存事项。
```php
$cache
    ->get('foo')
    ->then('var_dump');
```

本例获取键` foo `的值并将其传递给` var_dump `函数。您可以使用 [promises](1.Core-Components/Promise.md) 提供的任何组合。

#### set()

`set(string $key, mixed $value, ?float $ttl = null): PromiseInterface<bool>`方法可以用来在缓存中存储一个值。

此方法将在成功时使用` true `解决问题，发生错误时使用` false `解决问题。如果缓存实现必须通过网络来存储它，则可能需要一段时间。

` $ttl `参数(可选)设置该缓存项的最大生命周期(以秒为单位)。如果忽略此参数(或` null `)，只要底层实现支持，该项将一直留在缓存中。
尝试访问过期的缓存项会导致缓存丢失，参阅['get()'](#get)。

```php
$cache->set('foo', 'bar', 60);
```

这个例子最终将键` foo `的值设置为` bar `。如果它已经存在，它将被覆盖。

此接口不强制任何特定的TTL分辨率，因此如果您依赖毫秒或以下的非常高精度，则可能需要特别小心。
已知许多现有的缓存实现提供微秒或毫秒的精度，但通常不建议依赖这种高精度。

这个接口建议缓存实现应该使用单调的时间源(如果可用的话)。默认情况下，单调时间源只有在PHP 7.3才可用，因此缓存实现可能会退回到使用时钟时间。
虽然这不会影响许多常见的用例，但对于依赖于高时间精度的程序和依赖于不连续时间调整(时间跳跃)的系统来说，这是一个重要的区别。
这意味着，如果您存储一个TTL为30秒的缓存项，然后将系统时间向前调整20秒，该缓存项仍将在30秒内过期。

#### delete()

`delete(string $key): PromiseInterface<bool>`方法可以用来从缓存中删除值。

成功时此方法将解析为`true`，当发生错误时将使用`false`解析。 
当在缓存中没有找到用于`$key`的项时，它也解析为`true`。 
如果缓存实现必须通过网络删除它，则可能需要一段时间。

```php
$cache->delete('foo');
```

这个例子最终从缓存中删除了键` foo `。与` set() `一样，这可能不会立即发生，并且返回Promise来保证删除项是否已经从缓存中移除。

#### getMultiple()

`getMultiple(string[] $keys, mixed $default = null): PromiseInterface<array>`方法可以通过它们唯一的键来检索多个缓存项。

此方法将在成功时使用缓存值数组进行解析，或在找不到项或发生错误时使用给定的`$default`值进行解析。
同样，过期的缓存项（一旦过了有效期）被视为缓存未命中。

```php
$cache->getMultiple(array('name', 'age'))->then(function (array $values) {
    $name = $values['name'] ?? 'User';
    $age = $values['age'] ?? 'n/a';

    echo $name . ' is ' . $age . PHP_EOL;
});
```

此示例获取`name`和`age`键的缓存项，并打印一些示例输出。 您可以使用
[promises](1.Core-Components/Promise.md) 提供的任何组合。 

#### setMultiple()

可以使用`setMultiple(array $values, ?float $ttl = null): PromiseInterface<bool>`方法在缓存中持久化一组键值对，其中`ttl`是可选的。

此方法将在成功时使用`true`进行解析，或在发生错误时使用`false`进行解析。如果缓存实现必须通过网络来存储它，则可能需要一段时间。

`$ttl`参数(可选)设置这些缓存项的最大生命周期(以秒为单位)。如果省略这个参数(或` null `)，只要底层实现支持，这些项就会一直留在缓存中。
尝试访问过期的缓存项会导致缓存丢失，请参阅[`getMultiple()`](#getmultiple)

```php
$cache->setMultiple(array('foo' => 1, 'bar' => 2), 60);
```

本例最终将值列表设置为`foo`键为`1`值，`bar`键为`2`。如果某些键已经存在，它们将被重写。

#### deleteMultiple()


`deleteMultiple(string[] $keys): PromiseInterface<bool>` 方法可以用于在一次操作中删除多个缓存项。

此方法将在成功时使用`true`进行解析，或在发生错误时使用`false`进行解析。
当在缓存中找不到`$keys`的项时，它也会解析为`true`。如果缓存实现必须通过网络删除它，则可能需要一段时间。

```php
$cache->deleteMultiple(array('foo', 'bar, 'baz'));
```

本例最终从缓存中删除键`foo`, `bar`和`baz`。
与`setMultiple()`一样，这可能不会立即发生，并且会返回一个承诺，以保证该项是否已从缓存中删除。

#### clear()

`clear(): PromiseInterface<bool>`方法可用于清除整个缓存。

此方法将在成功时使用`true`进行解析，或在发生错误时使用`false`进行解析。如果缓存实现必须通过网络删除它，则可能需要一段时间。

```php
$cache->clear();
```

这个例子最终会从缓存中删除所有的键。与`deleteMultiple()`一样，这可能不会立即发生，并且会返回一个承诺，以保证是否已从缓存中删除所有项。

#### has()

`has(string $key): PromiseInterface<bool>`方法可用于确定缓存中是否存在项。

此方法将在成功时使用`true`进行解析，在找不到项或出现错误时使用`false`进行解析。
同样，过期的缓存项（一旦生命周期过期）被视为缓存未命中。

```php
$cache
    ->has('foo')
    ->then('var_dump');
```

下面的例子检查键` foo `的值是否设置在缓存中，并将结果传递给` var_dump `函数。您可以使用
[promises](1.Core-Components/Promise.md) 提供的任何组合。
注意:建议`has()`仅用于缓存预热类型的目的，而不要在`get/set`的实时应用程序操作中使用，
因为此方法受竞态条件的限制，在竞态条件下，`has()`将立即返回`true`，
然后另一个脚本可以将其删除，从而使应用程序的状态过期。

### ArrayCache

`ArrayCache`提供了 [`CacheInterface`](#cacheinterface) 在内存中的实现。

```php
$cache = new ArrayCache();

$cache->set('foo', 'bar');
```

它的构造函数接受一个`?int $limit`参数(可选)来限制在LRU缓存中存储的最大量。
如果您向该实例添加更多项，它将自动删除最近使用最少使用的项(LRU)。

例如，该代码片段将覆盖第一个值，并只存储最后两项:
```php
$cache = new ArrayCache(2);

$cache->set('foo', '1');
$cache->set('bar', '2');
$cache->set('baz', '3');
```

已知在使用PHP 7.3之前的任何版本时，此缓存实现都依赖于时钟时间来计划将来的缓存过期时间，
因为单调时间源仅在PHP 7.3起可用(`hrtime()`)。 
尽管这并不影响许多常见用例，但这对于依赖于高时间精度的程序或受不连续时间调整（时间跳跃）影响的系统而言，是一个重要的区别。

这意味着，如果在PHP<7.3上存储TTL为30s的缓存项，然后将系统时间向前调整20s，则该缓存项可能在10s内到期。 另请参见[`set()`](#set)

## 常见用法

### Fallback get

缓存的一个常见用例是尝试获取缓存的值，如果没有找到，作为后备从原始数据源检索它。下面是一个例子:
```php
$cache
    ->get('foo')
    ->then(function ($result) {
        if ($result === null) {
            return getFooFromDb();
        }
        
        return $result;
    })
    ->then('var_dump');
```

首先尝试检索`foo`的值。注册一个回调函数，当结果值为`null`时，它将调用`getFooFromDb`。
`getFooFromDb`是一个函数(可以是任何PHP可调用的)，如果键在缓存中不存在，它将被调用。
`getFooFromDb`可以通过从数据库(或任何其他数据源)返回实际值的承诺来处理丢失的键。
因此，该链将在两种情况下正确地返回提供值。

### Fallback get and set

为了扩展 [`fallback get`](#fallback-get) 示例, 通常需要在从数据源获取值之后在缓存上设置该值。

```php
$cache
    ->get('foo')
    ->then(function ($result) {
        if ($result === null) {
            return $this->getAndCacheFooFromDb();
        }
        
        return $result;
    })
    ->then('var_dump');

public function getAndCacheFooFromDb()
{
    return $this->db
        ->get('foo')
        ->then(array($this, 'cacheFooFromDb'));
}

public function cacheFooFromDb($foo)
{
    $this->cache->set('foo', $foo);

    return $foo;
}
```

通过使用串联操作，您可以轻松地有条件地缓存从数据库中获取的值。

## 安装

推荐的安装这个库的方法是[通过Composer](https://getcomposer.org)。
[Composer 新手?](https://getcomposer.org/doc/00-intro.md)

该项目遵循[SemVer](https://semver.org/) ，
默认安装最新支持的版本:

```bash
$ composer require react/cache:^1.1
```

有关版本升级的详细信息，请参见[CHANGELOG](https://reactphp.org/cache/changelog.html) 。

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

MIT, see [LICENSE file](https://reactphp.org/cache/license.html).
