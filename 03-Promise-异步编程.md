<a href="https://promisesaplus.com/">
    <img src="https://promisesaplus.com/assets/logo-small.png" alt="Promises/A+ logo"
         title="Promises/A+ 1.1 compliant" align="right" />
</a>

# 概述

PHP 的主要编程模式是同步方式，如果要在 PHP 中进行异步编程，通常是采用回调的方式，因为这种方式简单直接，不需要第三方库的支持，但缺点是当回调层层嵌套使用时，会严重影响程序的可读性和可维护性，因此层层回调的异步编程让人望而生畏。

回调的问题在 JavaScript 中更加明显，因为异步编程模式是 JavaScript 的主要编程模式。为了解决这个问题，JavaScript 社区提出了一套 Promise 异步编程模型。[Promise/A+](https://promisesaplus.com/)([中文版](http://www.ituring.com.cn/article/66566))是一个通用的、标准化的规范，它提供了一个可互操作的 then 方法的实现定义。Promise/A+ 规范的实现有很多，并不局限于 JavaScript 语言，它们的共同点就是都有一个标准的 then 方法，而其它的 API 则各不相同。

Hprose 2.0 为了更好的实现异步服务和异步调用，也为 PHP 提供了一套 Promise 异步模型实现。它基本上是参照 Promise/A+(中文版) 规范实现的。

Hprose 2.0 之前的版本提供了一组 `Future`/`Completer` 的 API，其中 `Future` 对象上也提供了 `then` 方法，但最初是参照 Dart 语言中的 `Future`/`Completer` 设计的。

而在 Hprose 2.0 版本中，我们对 `Future` 的实现做了比较大的改进，现在它既兼容 Dart 的 `Future`/`Completer` 使用方式，又兼容 [Promise/A+ 规范](https://promisesaplus.com/)，而且还增加了许多非常实用的方法。下面我们就来对这些方法做一个全面的介绍。

注意：下面的例子中，为了突出重点，代码中均省略了：

```php
<?php
require_once "vendor/autoload.php";
```

请读者自行脑补。

# 创建 Future/Promise 对象

Hprose 中提供了多种方法来创建 Future/Promise 对象。为了方便讲解，在后面我们不再详细区分 Future 对象和 Promise 对象实例的差别，统一称为 `promise` 对象。

## 使用 Future 构造器

### 创建一个待定（pending）状态 promise 对象

```php
use Hprose\Future;
$promise = new Future();
```

该 `promise` 对象的结果尚未确定，可以在将来通过 `resolve` 方法来设定其成功值，或通过 `reject` 方法来设定其失败原因。

### 创建一个成功（fulfilled）状态的 promise 对象

```php
use Hprose\Future;
$promise = new Future(function() { return 'hprose'; });
$promise->then(function($value) {
    var_dump($value);
});
```

该 `promise` 对象中已经包含了成功值，可以使用 `then` 方法来得到它。

### 创建一个失败（rejected）状态的 promise 对象

```php
use Hprose\Future;
$promise = new Future(function() { throw new Exception('hprose'); });
$promise->catchError(function($reason) {
    var_dump($reason);
});
```

该 `promise` 对象中已经包含了失败值，可以使用 `catchError` 方法来得到它。

上面的 `Future` 构造函数的参数可以是无参的函数、方法、闭包等，或者说只要是无参的 callable 对象就可以，不一定非要用闭包。

## 使用 Hprose\Future 名空间中的工厂方法

`Hprose\Future` 名空间内提供了 6 个工厂方法，它们分别是：

* `resolve`
* `value`
* `reject`
* `error`
* `sync`
* `promise`

其中 `resolve` 和 `value` 功能完全相同，`reject` 和 `error` 功能完全相同。

`resolve` 和 `reject` 这两个方法名则来自 ECMAScript 6 的 Promise 对象。

`value` 和 `error` 这两个方法名来自 Dart 语言的 `Future` 类。因为最初是按照 Dart 语言的 API 设计的，因此，这里保留了 `value` 和 `error` 这两个方法名。

`sync` 功能跟 `Future` 含参构造方法类似，但在返回值的处理上有所不同。

`promise` 方法跟 `Promise` 类的构造方法类似，但返回的是一个 `Future` 类型的对象，而 `Promise` 构造方法返回的是一个 `Promise` 类的对象，`Promise` 类是 `Future` 类的子类，但除了构造函数不同以外，其它都完全相同。

### 创建一个成功（fulfilled）状态的 promise 对象

```php
use Hprose\Future;
$promise = Future\value('hprose'); // 换成 Future\resolve('hprose') 效果一样
$promise->then(function($value) {
    var_dump($value);
});
```

使用 `value` 或 `resolve` 来创建一个成功（fulfilled）状态的 `promise` 对象效果跟前面用 `Future` 构造器创建的效果一样，但是写起来更加简单，不再需要把结果放入一个函数中作为返回值返回了。

### 创建一个失败（rejected）状态的 promise 对象

```php
use Hprose\Future;
$e = new Exception('hprose');
$promise = Future\error($e); // 换成 Future\reject($e) 效果一样
$promise->catchError(function($reason) {
    var_dump($reason);
});
```

使用 `error` 或 `reject` 来创建一个失败（rejected）状态的 `promise` 对象效果跟前面用 `Future` 构造器创建的效果也一样，但是写起来也更加简单，不再需要把失败原因放入一个函数中作为异常抛出了。

注意，这里的 `error`（或 `reject`）函数的参数并不要求必须是异常类型的对象，但最好是使用异常类型的对象。否则你的程序很难进行调试和统一处理。

### 通过 Future\sync 方法来创建 promise 对象

`Future` 上提供了一个：

```php
Future\sync($computation);
```

方法可以让我们同步的创建一个 `promise` 对象。

实际上，Hprose for PHP 的 `Future` 构造方法也是同步的，这一点跟 JavaScript 版本的有所不同。`sync` 函数跟 `Futrue` 构造方法区别在于结果上，通过 `Future` 构造方法的结果中如果包含生成器函数或者是生成器，则生成器函数和生成器将原样返回。而通过 `sync` 函数返回的生成器函数或生成器会作为协程执行之后，返回执行结果。

### 通过 Future\promise 方法来创建 promise 对象

该方法的参数跟 ECMAScript 6 的 `Promise` 构造器的参数相同，不同的是，使用该方法创建 `promise` 对象时，不需要使用 `new` 关键字。另外一点不同是，该方法创建的 `promise` 对象一定是 `Future` 的实例对象，而通过 `Promise` 构造器创建的 `promise` 对象是 `Promise` 实例对象。

```php
use Hprose\Future;

$p = Future\promise(function($resolve, $reject) {
    $a = 1;
    $b = 2;
    if ($a != $b) {
        $resolve('OK');
    }
    else {
        $reject(new Exception("$a == $b"));
    }
});
$p->then(function($value) {
    var_dump($value);
});
```

运行结果为：

>
```
string(2) "OK"
```
>

## 通过 Promise 构造方法来创建 promise 对象

```php
use Hprose\Promise;
$p = new Promise(function($reslove, $reject) { ... });
```

该构造方法的参数跟 `Future\promise` 函数的参数一致，这里就不再单独举例了。

`Promise` 构造方法的参数可以省略，省略时，行为跟 `Future` 构造方法省略参数一致。这里也不再单独举例。

## 通过 Completer 来创建 promise 对象

```php
use Hprose\Completer;

$completer = new Completer();
$promise = $completer->future();
$promise->then(function($value) {
    var_dump($value);
});
var_dump($completer->isCompleted());
$completer->complete('hprose');
var_dump($completer->isCompleted());
```

运行结果为：

>
```
bool(false)
string(6) "hprose"
bool(true)
```
>

`Future/Completer` 这套 API 来自 Dart 语言，首先通过 `Completer` 构造器创建一个 `completer` 对象，然后通过 `completer` 对象上的 `future` 方法返回 `promise` 对象。通过 `completer` 的 `complete` 方法可以设置成功值。通过 `completeError` 方法可以设置失败原因。通过 `isCompleted` 方法，可以查看当前状态是否为已完成（在这里，成功（fulfilled）或失败（rejected）都算完成状态）。

在 Hprose 2.0 之前的版本中，这是唯一可用的方法。但在 Hprose 2.0 中，该方式已经被其他方式所代替。仅为兼容旧版本而保留。

# Future 类上的方法

## then 方法

`then` 方法是 `Promise` 的核心和精髓所在。它有两个参数：`$onfulfill`, `$onreject`。这两个参数皆为 `callable` 类型。当它们不是 `callable` 类型时，它们将会被忽略。当 `promise` 对象状态为待定（pending）时，这两个回调方法都不会执行，直到 `promise` 对象的状态变为成功（fulfilled）或失败（rejected）。当 `promise` 对象状态为成功（fulfilled）时，`$onfulfill` 函数会被回调，参数值为成功值。当 `promise` 对象状态为失败（rejected）时，`$onreject` 函数会被回调，参数值为失败原因。

`then` 方法的返回值是一个新的 `promise` 对象，它的值由 `$onfulfill` 或 `$onreject` 的返回值或抛出的异常来决定。如果`$onfulfill` 或 `$onreject` 在执行过程中没有抛出异常，那么新的 `promise` 对象的状态为成功（fulfilled），其值为 `$onfulfill` 或 `$onreject` 的返回值。如果这两个回调中抛出了异常，那么新的 `promise` 对象的状态将被设置为失败（rejected），抛出的异常作为新的 `promise` 对象的失败原因。

同一个 `promise` 对象的 `then` 方法可以被多次调用，其值不会因为调用 `then` 方法而改变。当 `then` 方法被多次调用时，所有的 `$onfulfill`, `$onreject` 将按照原始的调用顺序被执行。

因为 `then` 方法的返回值还是一个 `promise` 对象，因此可以使用链式调用的方式实现异步编程串行化。

当 `promise` 的成功值被设置为另一个 `promise` 对象（为了区分，将其命名为 `promise2`)时，`then` 方法中的两个回调函数得到的参数是 `promise2` 对象的最终展开值，而不是 `promise2` 对象本身。当 `promise2` 的最终展开值为成功值时，`$onfulfill` 函数会被调用，当 `promise2` 的最终展开值为失败原因时，`$onreject` 函数会被调用。

当 `promise` 的失败原因被设置为另一个 `promise` 对象时，该对象会直接作为失败原因传给 `then` 方法的 `$onreject` 回调函数。因此最好不要这样做。

关于 `then` 方法的用法，这里不单独举例，您将在其它的例子中看到它的用法。

## done 方法

跟 `then` 方法类似，但 `done` 方法没有返回值，不支持链式调用，因此在 `done` 方法的回调函数中，通常不会返回值。

如果在 `done` 方法的回调中发生异常，会直接抛出，并且无法被捕获。

因此，如果您不是在写单元测试，最好不要使用 `done` 方法。

## fail 方法

该方法是 `done(null, $onreject)` 的简化方法。

如果您不是在写单元测试，最好不要使用 `fail` 方法。

## catchError 方法

```php
$promise->catchError($onreject);
```

该方法是 `then(null, $onreject)` 的简化写法。

```php
$promise->catchError($onreject, $test);
```

该方法第一个参数 `$onreject` 跟上面的相同，第二个参数 `$test` 是一个测试函数（`callable` 类型）。当该测试函数返回值为 `true` 时，`$onreject` 才会执行。

```php
use Hprose\Future;

$p = Future\reject(new OutOfRangeException());

$p->catchError(function($reason) { return 'this is a OverflowException'; },
               function($reason) { return $reason instanceof OverflowException; })
  ->catchError(function($reason) { return 'this is a OutOfRangeException'; },
               function($reason) { return $reason instanceof OutOfRangeException; })
  ->then(function($value) { var_dump($value);  });
```

输出结果为：

>
```
string(29) "this is a OutOfRangeException"
```
>

## resolve 方法

该方法可以将状态为待定（pending）的 `promise` 对象变为成功（fulfilled）状态。

该方法的参数值可以为任意类型。

## reject 方法

该方法可以将状态为待定（pending）的 `promise` 对象变为失败（rejected）状态。

该方法的参数值可以为任意类型，但通常只使用异常类型。

## inspect 方法

该方法返回当前 `promise` 对象的状态。

如果当前状态为待定（pending），返回值为：

```php
array('state' => 'pending')
```

如果当前状态为成功（fulfilled），返回值为：

```php
array('state' => 'fulfilled', 'value' => $promise->value)
```

如果当前状态为失败（rejected），返回值为：

```php
array('state' => 'rejected', 'reason' => $promise->reason);
```

## whenComplete 方法

有时候，你不但想要在成功（fulfilled）时执行某段代码，而且在失败（rejected）时也想执行这段代码，那你可以使用 `whenComplete` 方法。该方法的参数为一个无参回调函数。该方法执行后会返回一个新的 `promise` 对象，除非在回调函数中抛出异常，否则返回的 `promise` 对象的值跟原 `promise` 对象的值相同。

```php
use Hprose\Future;

$p1 = Future\resolve('resolve hprose');

$p1->whenComplete(function() {
    var_dump('p1 complete');
})->then(function($value) {
    var_dump($value);
});

$p2 = Future\reject(new Exception('reject thrift'));

$p2->whenComplete(function() {
    var_dump('p2 complete');
})->catchError(function($reason) {
    var_dump($reason->getMessage());
});

$p3 = Future\resolve('resolve protobuf');

$p3->whenComplete(function() {
    var_dump('p3 complete');
    throw new Exception('reject protobuf');
})->catchError(function($reason) {
    var_dump($reason->getMessage());
});
```

运行结果如下：

>
```
string(11) "p1 complete"
string(14) "resolve hprose"
string(11) "p2 complete"
string(13) "reject thrift"
string(11) "p3 complete"
string(15) "reject protobuf"
```
>

## complete 方法

该方法的回调函数 `oncomplete` 在不论成功还是失败的情况下都会执行，并且支持链式调用。相当于：`then(oncomplete, oncomplete)` 的简化写法。

## always 方法

该方法的回调函数 `oncomplete` 在不论成功还是失败的情况下都会执行，但不支持链式调用。相当于：`done(oncomplete, oncomplete)` 的简化写法。

如果您不是在写单元测试，最好不要使用 `always` 方法。

## fill 方法

将当前 `promise` 对象的值充填到参数所表示的 `promise` 对象中。

## tap 方法

```php
$promise->tap($onfulfilledSideEffect);
```

以下两种写法是等价的：

```php
$promise->then(function($result) use ($onfulfilledSideEffect) {
    call_user_func($onfulfilledSideEffect, $result);
    return result;
});

$promise->tap($onfulfilledSideEffect);
```

显然使用 `tap` 方法写起来更简单。

## spread 方法

```php
$promise->spread($onfulfilledArray);
```

以下两种写法是等价的：

```php
$promise->then(function($array) use ($onfulfilledArray) {
    return call_user_func_array($onfulfilledArray, $array);
});

$promise->spread($onfulfilledArray);
```

## each 方法

```php
$promise->each($callback);
```

如果 `promise` 对象中包含的是一个数组，那么使用该方法可以对该数组进行遍历。`$callback` 回调方法的格式如下：

```php
function callback(mixed $value[, mixed $key[, array $array]]);
```

后两个参数是可选的。

```php
use Hprose\Future;

function dumpArray($value, $key) {
  var_dump("a[$key] = $value");
}

$a1 = Future\value(array(2, Future\value(5), 9));
$a2 = Future\value(array('name' => Future\value('Tom'), 'age' => Future\value(18)));
$a1->each('dumpArray');
$a2->each('dumpArray');
```

输出结果为：

>
```
string(8) "a[0] = 2"
string(8) "a[1] = 5"
string(8) "a[2] = 9"
string(13) "a[name] = Tom"
string(11) "a[age] = 18"
```
>

## every 方法

```php
$promise->every($callback);
```

如果 `promise` 对象中包含的是一个数组，那么使用该方法可以遍历数组中的每一个元素并执行回调 `$callback`，当所有 `$callback` 的返回值都为 `true` 时，结果为 `true`，否则为 `false`。$callback 回调方法的格式如下：

```php
bool callback(mixed $value[, mixed $key[, array $array]]);
```

后两个参数是可选的。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

function isBigEnough($value) {
  return $value >= 10;
}

$a1 = Future\value(array(12, Future\value(5), 8, Future\value(130), 44));
$a2 = Future\value(array(12, Future\value(54), 18, Future\value(130), 44));
$dump($a1->every('isBigEnough'));   // false
$dump($a2->every('isBigEnough'));   // true
```

运行结果如下：

>
```
bool(false)
bool(true)
```
>

## some 方法

```php
$promise->some($callback);
```

如果 `promise` 对象中包含的是一个数组，那么使用该方法可以遍历数组中的每一个元素并执行回调 `$callback`，当任意一个 `$callback` 的返回值为 `true` 时，结果为 `true`，否则为 `false`。`$callback` 回调方法的格式如下：

```php
bool callback(mixed $value[, mixed $key[, array $array]]);
```

后两个参数是可选的。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

function isBigEnough($value) {
  return $value >= 10;
}

$a1 = Future\value(array(12, Future\value(5), 8, Future\value(130), 44));
$a2 = Future\value(array(1, Future\value(5), 8, Future\value(1), 4));
$dump($a1->some('isBigEnough'));   // true
$dump($a2->some('isBigEnough'));   // false
```

运行结果如下：

>
```
bool(true)
bool(false)
```
>

## filter 方法

```php
$promise->filter($callback, $preserveKeys = false);
```

如果 `promise` 对象中包含的是一个数组，那么使用该方法可以遍历数组中的每一个元素并执行回调 `$callback`，`$callback` 的返回值为 `true` 的元素所组成的数组将作为 `filter` 返回结果的 `promise` 对象所包含的值。当参数 `$preserveKeys` 为 `true` 时，结果数组中的 元素所对应的 `key` 保持原来的 `key`，否则将返回以 0 为起始下标的连续数字下标的数组。`$callback` 回调方法的格式如下：

```php
bool callback(mixed $value[, mixed $key[, array $array]]);
```

后两个参数是可选的。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

function isBigEnough($value) {
  return $value >= 8;
}

$a = Future\value(array(
    'Tom' => 8,
    'Jerry' => Future\value(5),
    'Spike' => 10,
    'Tyke' => 3
));

$dump($a->filter('isBigEnough'));
$dump($a->filter('isBigEnough', true));
```

运行结果为：

>
```
array(2) {
  [0]=>
  int(8)
  [1]=>
  int(10)
}
array(2) {
  ["Tom"]=>
  int(8)
  ["Spike"]=>
  int(10)
}
```
>

## map 方法

```php
$promise->map($callback);
```

如果 `promise` 对象中包含的是一个数组，那么使用该方法可以遍历数组中的每一个元素并执行回调 `$callback`，`$callback` 的返回值所组成的数组将作为 `map` 返回结果的 `promise` 对象所包含的值。`$callback` 回调方法的格式如下：

```php
mixed callback(mixed $value[, mixed $key[, array $array]]);
```

后两个参数是可选的。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

$a = Future\value(array(1, Future\value(4), 9));
$dump($a->map('sqrt'));
```

运行结果为：

>
```
array(3) {
  [0]=>
  float(1)
  [1]=>
  float(2)
  [2]=>
  float(3)
}
```
>

## reduce 方法

```php
$promise->reduce($callback, $initial = NULL);
```

如果 `promise` 对象中包含的是一个数组，那么使用该方法可以遍历数组中的每一个元素并执行回调 `$callback`，`$callback` 的第一个参数为 $initial 的值或者上一次调用的返回值。最后一次 $callback 的返回结果作为 `promise` 对象所包含的值。`$callback` 回调方法的格式如下：

```php
mixed callback(mixed $carry, mixed $item);
```

关于该方法的更多描述可以参见 PHP 手册中的 `array_reduce` 方法。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

$numbers = Future\value(array(Future\value(0), 1, Future\value(2), 3, Future\value(4)));

function add($a, $b) {
  return $a + $b;
}

$dump($numbers->reduce('add'));
$dump($numbers->reduce('add', 10));
$dump($numbers->reduce('add', Future\value(20)));
```

运行结果如下：
>
```
int(10)
int(20)
int(30)
```
>

## search 方法

```php
$promise->search($searchElement, $strict = false);
```

如果 `promise` 对象中包含的是一个数组，那么使用该方法可以在 `promise` 对象所包含的数组中查找 `$searchElement` 元素，返回值以 `promise` 对象形式返回，如果找到，返回的 `promise` 对象中将包含该元素对应的 `key`，否则为 `false`。当 `$strict` 为 `true` 时，使用 `===` 运算符进行相等测试。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

$numbers = Future\value(array(Future\value(0), 1, Future\value(2), 3, Future\value(4)));

$dump($numbers->search(2));
$dump($numbers->search(Future\value(3)));
$dump($numbers->search(true));
$dump($numbers->search(true, true));
```

运行结果如下：

>
```
int(2)
int(3)
int(1)
bool(false)
```
>

## includes 方法

```php
$promise->includes($searchElement, $strict = false);
```

该方法同 `search` 方法类似，只是在找到的情况下，仅仅返回包含 `true` 的 `promise` 对象。

## 魔术方法 __call 和 __get

`Future` 类上还定义了 __get 和 __call 这两个魔术方法，当 `promise` 对象中包含的值为 `object` 对象时，可以直接获取它的属性和调用它的方法。但是需要注意的是，返回的属性值是一个 `promise` 对象。调用方法的返回值也是一个 `promise` 对象，而且调用方法时，参数也可以是 `promise` 对象，即使原来的方法并不支持 `promise` 参数。因为在实际调用时，`__call` 会自动将 `promise` 的参数值转换为实际包含的值进行调用。

这在一定程度上可以使得异步代码在编写时看上去像是同步代码。

# Future 类上的常量和属性

## 常量
```php
    const PENDING = 0;
    const FULFILLED = 1;
    const REJECTED = 2;
```

这三个常量表示 `promise` 对象的状态。`PENDING` 表示结果待定。`FULFILLED` 表示成功。`REJECTED` 表示失败。

# Hprose\Future 名空间中的函数

## isFuture 函数

```php
bool isFuture(mixed $obj);
```

判断参数 $obj 是否为 `Future` 对象。

例如：

```php
use Hprose\Future;

var_dump(Future\isFuture(123));
var_dump(Future\isFuture(Future\value(123)));
```

运行结果为：

>
```
bool(false)
bool(true)
```
>

## toFuture 函数

```php
Future toFuture(mixed $obj);
```

如果 `$obj` 为 `Future` 对象则原样返回，否则返回 `Future\value($obj)`。

## toPromise 函数

```php
Future toPromise(mixed $obj);
```

该方法同 `toFuture` 函数类似。

如果 $obj 为生成器，则该方法将把生成器作为协程执行并以 `promise` 对象返回该协程的执行结果。

其他情况跟 `toFuture` 一致。

## all 函数

```php
Future all(mixed $array);
```

该方法的参数 `$array` 为数组或者值为数组的 `promise` 对象。该方法返回一个 `promise` 对象，该 `promise` 对象会在数组参数内的所有 `promise` 都被设置为成功（fulfilled）状态时，才被设置为成功（fulfilled）状态，其值为数组参数中所有 `promise` 对象的最终展开值组成的数组，其数组元素与原数组元素一一对应。

## join 函数

```php
Future join(mixed arg1[, mixed arg1[, ...]]);
```

该方法的功能同 `all` 方法类似，但它与 `all` 方法的参数不同，我们来举例看一下它们的差别：

```php
use Hprose\Future;

Future\all(array(1, Future\value(2), 3))->then(function($value) {
    var_dump($value);
});

Future\join(1, Future\value(2), 3)->then(function($value) {
    var_dump($value);
});
```

运行结果为：

>
```
array(3) {
  [0]=>
  int(1)
  [1]=>
  int(2)
  [2]=>
  int(3)
}
array(3) {
  [0]=>
  int(1)
  [1]=>
  int(2)
  [2]=>
  int(3)
}
```
>

## race 函数

```php
Future race(mixed $array);
```

该方法返回一个 `promise` 对象，这个 `promise` 在数组参数中的任意一个 `promise` 被设置为成功（fulfilled）或失败（rejected）后，立刻以相同的成功值被设置为成功（fulfilled）或以相同的失败原因被设置为失败（rejected）。

## any 函数

```php
Future any(mixed $array);
```

该方法是 `race` 函数的改进版。

对于 `race` 函数，如果输入的数组为空，返回的 `promise` 对象将永远保持为待定（pending）状态。

而对于 `any` 函数，如果输入的数组为空，返回的 `promise` 对象将被设置为失败状态，失败原因是一个 `RangeException` 对象。

对于 `race` 函数，数组参数中的任意一个 `promise` 被设置为成功（fulfilled）或失败（rejected）后，返回的 `promise` 对象就会被设定为成功（fulfilled）或失败（rejected）状态。

而对于 `any` 函数，只有当数组参数中的所有 `promise` 被设置为失败状态时，返回的 `promise` 对象才会被设定为失败状态。否则，返回的 `promise` 对象被设置为第一个被设置为成功（fulfilled）状态的成功值。

这两个函数通常跟计时器或者并发请求一起使用，用来获取最早完成的结果。

## settle 函数

```php
Future settle(mixed $array)
```

该方法返回一个 `promise` 对象，该 `promise` 对象会在数组参数内的所有 `promise` 都被设置为成功（fulfilled）状态或失败（rejected）状态时，才被设置为成功（fulfilled）状态，其值为数组参数中所有 `promise` 对象的 `inspect` 方法返回值，其数组元素与原数组元素一一对应。

例如：

```php
use Hprose\Future;

$p1 = Future\resolve(3);
$p2 = Future\reject(new Exception("x"));

Future\settle(array(true, $p1, $p2))->then('print_r');
```

输出结果为：

```
Array
(
    [0] => Array
        (
            [state] => fulfilled
            [value] => 1
        )

    [1] => Array
        (
            [state] => fulfilled
            [value] => 3
        )

    [2] => Array
        (
            [state] => rejected
            [reason] => Exception Object
                (
                    [message:protected] => x
                    [string:Exception:private] => 
                    [code:protected] => 0
...
```

>
注：上面输出中 `...` 表示省略掉的内容。
>

## run 函数

```php
Future run(callable $handler[, mixed $arg1[, mixed $arg2[, ...]]]);
```

`run` 方法的作用是执行 `$handler` 函数并返回一个包含执行结果的 `promise` 对象，`$handler` 的参数分别为 `$arg1`, `$arg2`, ...。参数可以是普通值，也可以是 `promise` 对象，如果是 `promise` 对象，则等待其变为成功（fulfilled）状态时再将其成功值代入 `handler` 函数。如果变为失败（rejected）状态，`run` 返回的 `promise` 对象被设置为该失败原因。如果参数中，有多个 `promise` 对象变为失败（rejected）状态，则第一个变为失败状态的 `promise` 对象的失败原因被设置为 `run` 返回的 `promise` 对象的失败原因。当参数中的 `promise` 对象都变为成功（fulfilled）状态时，`$handler` 函数才会执行，如果在 `$handler` 执行的过程中，抛出了异常，则该异常作为 `run` 返回的 `promise` 对象的失败原因。如果没有异常，则 `$handler` 函数的返回值，作为 `run` 返回的 `promise` 对象的成功值。

例如：

```php
use Hprose\Future;

function add($a, $b) {
    return $a + $b;
}

$p1 = Future\resolve(3);

Future\run('add', 2, $p1)->then('var_dump');
```

输出结果为：

>
```
int(5)
```
>

## wrap 函数

```php
mixed wrap(mixed $handler);
```

`run` 函数虽然可以将 `promise` 参数带入普通函数执行并得到结果，但是不方便复用。`wrap` 函数可以很好的解决这个问题。

`wrap` 函数的参数可以是一个 `callable` 对象，也可以是一个普通对象。

如果参数是一个 `callable` 数据（比如函数，方法），则返回值是一个闭包对象，它是一个包装好的函数，该函数的执行方式跟使用 `Future\run` 的效果一样。

如果参数是一个普通对象，则返回值是一个 `\Hprose\Future\Wrapper` 对象。你可以像存取源对象一样存取它，但是它上面的方法的执行方式跟使用 `Future\run` 的效果一样。

如果参数是一个 `callable` 对象，则返回值是一个 `\Hprose\Future\CallableWrapper` 对象。它跟 `\Hprose\Future\Wrapper` 对象类似，只是该对象本身也可以被直接作为函数调用，而且执行方式跟使用 `Future\run` 的效果一样。

例如：

```php
use Hprose\Future;

class Test {
    function add($a, $b) {
        return $a + $b;
    }
    function sub($a, $b) {
        return $a - $b;
    }
    function mul($a, $b) {
        return $a * $b;
    }
    function div($a, $b) {
        return $a / $b;
    }
}

$var_dump = Future\wrap('var_dump');

$test = Future\wrap(new Test());

$var_dump($test->add(1, Future\value(2)));
$var_dump($test->sub(Future\value(1), 2));
$var_dump($test->mul(Future\value(1), Future\value(2)));
$var_dump($test->div(1, 2));
```

该程序输出结果为：

>
```
int(3)
int(-1)
int(2)
float(0.5)
```
>

## each 函数

```php
void each(mixed $array, callable $callback);
```

参数 `$array` 可以是一个包含数组的 `promise` 对象，也可以是一个包含有 `promise` 对象的数组。

该函数对该数组中的每个元素的展开值进行遍历。返回值是一个 `promise` 对象。如果参数数组中的 `promise` 对象为失败（rejected）状态，则该方法返回的 `promise` 对象被设置为失败（rejected）状态，且设为相同失败原因。如果在 `$callback` 回调中抛出了异常，则该方法返回的 `promise` 对象也被设置为失败（rejected）状态，失败原因被设置为抛出的异常值。

`$callback` 回调方法的格式如下：

```php
function callback(mixed $value[, mixed $key[, array $array]]);
```

后两个参数是可选的。

```php
use Hprose\Future;

function dumpArray($value, $key) {
  var_dump("a[$key] = $value");
}

$a1 = array(2, Future\value(5), 9);
$a2 = array('name' => Future\value('Tom'), 'age' => Future\value(18));
Future\each($a1, 'dumpArray');
Future\each($a2, 'dumpArray');
```

输出结果为：

>
```
string(8) "a[0] = 2"
string(8) "a[1] = 5"
string(8) "a[2] = 9"
string(13) "a[name] = Tom"
string(11) "a[age] = 18"
```
>

## every 函数

```php
Future<bool> every(mixed $array, callable $callback);
```

参数 `$array` 可以是一个包含数组的 `promise` 对象，也可以是一个包含有 `promise` 对象的数组。

该函数可以遍历数组中的每一个元素并执行回调 `$callback`，当所有 `$callback` 的返回值都为 `true` 时，结果为 `true`，否则为 `false`。该函数返回值是一个 promise 对象。如果参数数组中的 promise 对象为失败（rejected）状态，则该方法返回的 promise 对象被设置为失败（rejected）状态，且设为相同失败原因。如果在 callback 回调中抛出了异常，则该方法返回的 promise 对象也被设置为失败（rejected）状态，失败原因被设置为抛出的异常值。

$callback 回调方法的格式如下：

```php
bool callback(mixed $value[, mixed $key[, array $array]]);
```

后两个参数是可选的。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

function isBigEnough($value) {
  return $value >= 10;
}

$a1 = array(12, Future\value(5), 8, Future\value(130), 44);
$a2 = array(12, Future\value(54), 18, Future\value(130), 44);
$a3 = Future\value($a1);
$a4 = Future\value($a2);
$dump(Future\every($a1, 'isBigEnough'));   // false
$dump(Future\every($a2, 'isBigEnough'));   // true
$dump(Future\every($a3, 'isBigEnough'));   // false
$dump(Future\every($a4, 'isBigEnough'));   // true
```

运行结果如下：

>
```
bool(false)
bool(true)
bool(false)
bool(true)
```
>

## some 函数

```php
Future<bool> some(mixed $array, callable $callback);
```

参数 `$array` 可以是一个包含数组的 `promise` 对象，也可以是一个包含有 `promise` 对象的数组。

该函数可以遍历数组中的每一个元素并执行回调 `$callback`，当任意一个 `$callback` 的返回值为 `true` 时，结果为 `true`，否则为 `false`。如果参数数组中的 promise 对象为失败（rejected）状态，则该方法返回的 promise 对象被设置为失败（rejected）状态，且设为相同失败原因。如果在 callback 回调中抛出了异常，则该方法返回的 promise 对象也被设置为失败（rejected）状态，失败原因被设置为抛出的异常值。

`$callback` 回调方法的格式如下：

```php
bool callback(mixed $value[, mixed $key[, array $array]]);
```

后两个参数是可选的。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

function isBigEnough($value) {
  return $value >= 10;
}

$a1 = array(12, Future\value(5), 8, Future\value(130), 44);
$a2 = array(1, Future\value(5), 8, Future\value(1), 4);
$a3 = Future\value($a1);
$a4 = Future\value($a2);
$dump(Future\some($a1, 'isBigEnough'));   // true
$dump(Future\some($a2, 'isBigEnough'));   // false
$dump(Future\some($a3, 'isBigEnough'));   // true
$dump(Future\some($a4, 'isBigEnough'));   // false
```

运行结果如下：

>
```
bool(true)
bool(false)
bool(true)
bool(false)
```
>

## filter 函数

```php
Future<array> filter(mixed $array, callable $callback);
```

参数 `$array` 可以是一个包含数组的 `promise` 对象，也可以是一个包含有 `promise` 对象的数组。

该函数可以遍历数组中的每一个元素并执行回调 `$callback`，`$callback` 的返回值为 `true` 的元素所组成的数组将作为 `filter` 返回结果的 `promise` 对象所包含的值。当参数 `$preserveKeys` 为 `true` 时，结果数组中的 元素所对应的 `key` 保持原来的 `key`，否则将返回以 0 为起始下标的连续数字下标的数组。如果参数数组中的 promise 对象为失败（rejected）状态，则该方法返回的 promise 对象被设置为失败（rejected）状态，且设为相同失败原因。如果在 callback 回调中抛出了异常，则该方法返回的 promise 对象也被设置为失败（rejected）状态，失败原因被设置为抛出的异常值。

`$callback` 回调方法的格式如下：

```php
bool callback(mixed $value[, mixed $key[, array $array]]);
```

后两个参数是可选的。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

function isBigEnough($value) {
  return $value >= 8;
}

$a1 = array(12, Future\value(5), 8, Future\value(130), 44);
$a2 = Future\value($a1);
$dump(Future\filter($a1, 'isBigEnough'));
$dump(Future\filter($a2, 'isBigEnough'));

$a3 = array('Tom' => 8, 'Jerry' => Future\value(5), 'Spike' => 10, 'Tyke' => 3);
$a4 = Future\value($a3);
$dump(Future\filter($a3, 'isBigEnough'));
$dump(Future\filter($a3, 'isBigEnough', true));
$dump(Future\filter($a4, 'isBigEnough', true));
```

运行结果为：

>
```
array(4) {
  [0]=>
  int(12)
  [1]=>
  int(8)
  [2]=>
  int(130)
  [3]=>
  int(44)
}
array(4) {
  [0]=>
  int(12)
  [1]=>
  int(8)
  [2]=>
  int(130)
  [3]=>
  int(44)
}
array(2) {
  [0]=>
  int(8)
  [1]=>
  int(10)
}
array(2) {
  ["Tom"]=>
  int(8)
  ["Spike"]=>
  int(10)
}
array(2) {
  ["Tom"]=>
  int(8)
  ["Spike"]=>
  int(10)
}
```
>

## map 函数

```php
Future<array> map(mixed $array, callable $callback);
```

参数 `$array` 可以是一个包含数组的 `promise` 对象，也可以是一个包含有 `promise` 对象的数组。

该函数可以遍历数组中的每一个元素并执行回调 `$callback`，`$callback` 的返回值所组成的数组将作为 `map` 返回结果的 `promise` 对象所包含的值。该函数返回值是一个 promise 对象。如果参数数组中的 promise 对象为失败（rejected）状态，则该方法返回的 promise 对象被设置为失败（rejected）状态，且设为相同失败原因。如果在 callback 回调中抛出了异常，则该方法返回的 promise 对象也被设置为失败（rejected）状态，失败原因被设置为抛出的异常值。

`$callback` 回调方法的格式如下：

```php
mixed callback(mixed $value[, mixed $key[, array $array]]);
```

后两个参数是可选的。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

$a = array(1, Future\value(4), 9);
$dump(Future\map($a, 'sqrt'));
```

运行结果为：

>
```
array(3) {
  [0]=>
  float(1)
  [1]=>
  float(2)
  [2]=>
  float(3)
}
```
>

## reduce 函数

```php
Future<mixed> reduce(mixed $array, callable $callback[, $initial = NULL]);
```

参数 `$array` 可以是一个包含数组的 `promise` 对象，也可以是一个包含有 `promise` 对象的数组。

该函数可以遍历数组中的每一个元素并执行回调 `$callback`，`$callback` 的第一个参数为 $initial 的值或者上一次调用的返回值。最后一次 $callback 的返回结果作为 `promise` 对象所包含的值。该函数返回值是一个 promise 对象。如果参数数组中的 promise 对象为失败（rejected）状态，则该方法返回的 promise 对象被设置为失败（rejected）状态，且设为相同失败原因。如果在 callback 回调中抛出了异常，则该方法返回的 promise 对象也被设置为失败（rejected）状态，失败原因被设置为抛出的异常值。

`$callback` 回调方法的格式如下：

```php
mixed callback(mixed $carry, mixed $item);
```

关于该方法的更多描述可以参见 PHP 手册中的 `array_reduce` 方法。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

$numbers = array(Future\value(0), 1, Future\value(2), 3, Future\value(4));

function add($a, $b) {
  return $a + $b;
}

$dump(Future\reduce($numbers, 'add'));
$dump(Future\reduce($numbers, 'add', 10));
$dump(Future\reduce($numbers, 'add', Future\value(20)));
```

运行结果如下：

>
```
int(10)
int(20)
int(30)
```
>

## search 方法

```php
Future<mixed> search(mixed $array, mixed $searchElement[, bool $strict = false]);
```

参数 `$array` 可以是一个包含数组的 `promise` 对象，也可以是一个包含有 `promise` 对象的数组。

该函数可以在 `promise` 对象所包含的数组中查找 `$searchElement` 元素，返回值以 `promise` 对象形式返回，如果找到，返回的 `promise` 对象中将包含该元素对应的 `key`，否则为 `false`。当 `$strict` 为 `true` 时，使用 `===` 运算符进行相等测试。该函数返回值是一个 promise 对象。如果参数数组中的 promise 对象为失败（rejected）状态，则该方法返回的 promise 对象被设置为失败（rejected）状态，且设为相同失败原因。

```php
use Hprose\Future;

$dump = Future\wrap('var_dump');

$numbers = array(Future\value(0), 1, Future\value(2), 3, Future\value(4));

$dump(Future\search($numbers, 2));
$dump(Future\search($numbers, Future\value(3)));
$dump(Future\search($numbers, true));
$dump(Future\search($numbers, true, true));
```

运行结果如下：

>
```
int(2)
int(3)
int(1)
bool(false)
```
>

## includes 函数

```php
Future<bool> includes(mixed $array, mixed $searchElement[, bool $strict = false]);
```

该方法同 `search` 方法类似，只是在找到的情况下，仅仅返回包含 `true` 的 `promise` 对象。

# Hprose\Promise 类和名空间

`Hprose\Promise` 类是 `Hprose\Future` 的子类，除了构造方法跟 `Hprose\Future` 有所不同之外，其它方法都完全一致。

`Hprose\Promise` 名空间下的方法也都跟 `Hprose\Future` 名空间下的方法相同。只是有一个函数名不同，即 `Hprose\Promise\isPromise`，但它的功能跟 `Hprose\Future\isFuture` 是完全一样的。

因此你也可以完全使用 `Hprose\Promise` 来代替 `Hprose\Future`。