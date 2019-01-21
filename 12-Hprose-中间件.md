# 简介

Hprose 过滤器的功能虽然比较强大，可以将 Hprose 的功能进行扩展。但是有些功能使用它仍然难以实现，比如缓存。

为此，Hprose 2.0 引入了更加强大的中间件功能。Hprose 中间件不仅可以对输入输出的数据进行操作，它还可以对调用本身的参数和结果进行操作，甚至你可以跳过中间的执行步骤，或者完全由你来接管中间数据的处理。

Hprose 中间件跟普通的 HTTP 服务器中间件有些类似，但又有所不同。

Hprose 中间件在客户端服务器端都支持。

Hprose 中间件分为两种：

* 调用中间件
* 输入输出中间件

另外，输入输出中间件又可以细分为 `beforeFilter` 和 `afterFilter` 两种，但它们本质上没有什么区别，只是在执行顺序上有所区别。

# 执行顺序

Hprose 中间件的顺序执行是按照添加的前后顺序执行的，假设添加的中间件处理器分别为：`handler1`, `handler2` ... `handlerN`，那么执行顺序就是 `handler1`, `handler2` ... `handlerN`。

不同类型的 Hprose 中间件和 Hprose 其它过程的执行流程如下图所示：

```
+------------------------------------------------------------------+
|                 +-----------------batch invoke----------------+  |
|       +------+  | +-----+   +------+       +------+   +-----+ |  |
|       |invoke|  | |begin|   |invoke|  ...  |invoke|   | end | |  |
|       +------+  | +-----+   +------+       +------+   +-----+ |  |
|           ^     +---------------------------------------------+  |
|           |                            ^                         |
|           |                            |                         |
|           v                            v                         |
| +-------------------+        +------------------+                |
| | invoke middleware |        | batch middleware |                |
| +-------------------+        +------------------+                |
|           ^                            ^                         |
|           |     +---------------+      |                         |
|           +---->| encode/decode |<-----+                         |
|                 +---------------+                                |
|                         ^                                        |
|                         |                                        |
|                         v                                        |
|           +--------------------------+                           |
|           | before filter middleware |                           |
|           +--------------------------+                           |
|                         ^                                        |
|                         |       _  _ ___  ____ ____ ____ ____    |
|                         v       |__| |__] |__/ |  | [__  |___    |
|                    +--------+   |  | |    |  \ |__| ___] |___    |
|                    | filter |                                    |
|                    +--------+     ____ _    _ ____ _  _ ___      |
|                         ^         |    |    | |___ |\ |  |       |
|                         |         |___ |___ | |___ | \|  |       |
|                         v                                        |
|            +-------------------------+                           |
|            | after filter middleware |                           |
|            +-------------------------+                           |
+------------------------------------------------------------------+
                                  ^                                 
                                  |                                 
                                  |                                 
                                  v                                 
+------------------------------------------------------------------+
|           +--------------------------+                           |
|           | before filter middleware |                           |
|           +--------------------------+                           |
|                         ^                                        |
|                         |        _  _ ___  ____ ____ ____ ____   |
|                         v        |__| |__] |__/ |  | [__  |___   |
|                    +--------+    |  | |    |  \ |__| ___] |___   |
|                    | filter |                                    |
|                    +--------+    ____ ____ ____ _  _ ____ ____   |
|                         ^        [__  |___ |__/ |  | |___ |__/   |
|                         |        ___] |___ |  \  \/  |___ |  \   |
|                         v                                        |
|            +-------------------------+                           |
|            | after filter middleware |                           |
|            +-------------------------+                           |
|                         ^                                        |
|                         |                                        |
|                         v                                        |
|                 +---------------+                                |
|    +----------->| encode/decode |<---------------------+         |
|    |            +---------------+                      |         |
|    |                    |                              |         |
|    |                    |                              |         |
|    |                    v                              |         |
|    |            +---------------+                      |         |
|    |            | before invoke |-------------+        |         |
|    |            +---------------+             |        |         |
|    |                    |                     |        |         |
|    |                    |                     |        |         |
|    |                    v                     v        |         |
|    |          +-------------------+    +------------+  |         |
|    |          | invoke middleware |--->| send error |--+         |
|    |          +-------------------+    +------------+            |
|    |                    |                     ^                  |
|    |                    |                     |                  |
|    |                    v                     |                  |
|    |            +--------------+              |                  |
|    |            | after invoke |--------------+                  |
|    |            +--------------+                                 |
|    |                    |                                        |
|    |                    |                                        |
|    +--------------------+                                        |
+------------------------------------------------------------------+
```

# 调用中间件

调用中间件的形式为：

```php
function(string $name, array &$args, stdClass $context, Closure $next) {
    ...
    $result = $next($name, $args, $context);
    ...
    return $result;
}
```

`$name` 是调用的远程函数/方法名。

`$args` 是调用参数，引用传递的数组。

`$context` 是调用上下文对象。

`$next` 表示下一个中间件。通过调用 `$next` 将各个中间件串联起来。

在调用 `$next` 之前的操作在调用发生前执行，在调用 `$next` 之后的操作在调用发生后执行，如果你不想修改返回结果，你应该将 `$next` 的返回值作为该中间件的返回值返回。

另外，对于服务器端来说，`$next` 的返回值 `$result` 总是 `promise` 对象。对于客户端来说，如果客户端是异步客户端，那么 `$next` 的返回值 `$result` 是 `promise` 对象，如果客户端是同步客户端，`$next` 的返回值 `$result` 是实际结果。

## 跟踪调试

我们来看一个例子：

**LogHandler.php**
```php
use Hprose\Future;

$logHandler = function($name, array &$args, stdClass $context, Closure $next) {
    error_log("before invoke:");
    error_log($name);
    error_log(var_export($args, true));
    $result = $next($name, $args, $context);
    error_log("after invoke:");
    if (Future\isFuture($result)) {
        $result->then(function($result) {
            error_log(var_export($result, true));
        });
    }
    else {
        error_log(var_export($result, true));
    }
    return $result;
};
```

**Server.php**
```php
use Hprose\Socket\Server;

function hello($name) {
    return "Hello $name!";
}

$server = new Server('tcp://0.0.0.0:1143/');
$server->addFunction('hello');
$server->debug = true;
$server->addInvokeHandler($logHandler);
$server->start();
```

**Client.php**
```php
use Hprose\Client;

$client = Client::create('tcp://127.0.0.1:1143/', false);
$client->addInvokeHandler($logHandler);
var_dump($client->hello("world"));
```

然后分别启动服务器和客户端，就会看到如下输出：

**服务器输出**

>
```
before invoke:
hello
array (
  0 => 'world',
)
after invoke:
'Hello world!'
```
>

**客户端输出**

>
```
before invoke:
hello
array (
  0 => 'world',
)
after invoke:
'Hello world!'
string(12) "Hello world!"
```
>

这个例子中是使用的同步客户端，所以我们在 `$logHandler` 中判断了返回结果是同步结果还是异步结果，并根据结果的不同做了不同的处理。

如果我们的中间是单独为异步客户端或者服务器编写的，则不需要做这个判断。甚至我们还可以使用协程的方式来简化代码（需要 PHP 5.5+ 才可以）。

下面，我们来把上面的例子翻译成一个只针对异步客户端和服务器的使用协程方式编写的日志中间件：

**coLogHandler.php**
```php
$coLogHandler = function($name, array &$args, stdClass $context, Closure $next) {
    error_log("before invoke:");
    error_log($name);
    error_log(var_export($args, true));
    $result = (yield $next($name, $args, $context));
    error_log("after invoke:");
    error_log(var_export($result, true));
};
```

客户端和服务器的代码以及运行结果这里就省略了，如果你写的正确，运行结果跟上面的是一致的。

这个例子看上去要简单清爽的多，在这个例子中，我们使用 `yield` 关键字调用了 `$next` 方法，因为后面没有再次调用 `yield`，所以这个返回值也是该协程的返回值。

注意，不要用 `Future\wrap` 来包装这个协程，包装之后，不支持引用参数传递。

## 缓存调用

我们再来看一个实现缓存调用的例子，在这个例子中我们也使用了上面的日志中间件，用来观察我们的缓存是否真的有效。

**CacheHandler.php**
```php
class CacheHandler {
    private $cache = array();
    function handle($name, array &$args, stdClass $context, Closure $next) {
        if (isset($context->userdata->cache)) {
            $key = hprose_serialize($args);
            if (isset($this->cache[$name])) {
                if (isset($this->cache[$name][$key])) {
                    return $this->cache[$name][$key];
                }
            }
            else {
                $this->cache[$name] = array();
            }
            $result = $next($name, $args, $context);
            $this->cache[$name][$key] = $result;
            return $result;
        }
        return $next($name, $args, $context);
    }
}
```

**Client.php**
```php
use Hprose\Client;
use Hprose\InvokeSettings;

$cacheSettings = new InvokeSettings(array("userdata" => array("cache" => true)));
$client = Client::create('tcp://127.0.0.1:1143/', false);
$client->addInvokeHandler(array(new CacheHandler(), 'handle'));
$client->addInvokeHandler($logHandler);
var_dump($client->hello("cache world", $cacheSettings));
var_dump($client->hello("cache world", $cacheSettings));
var_dump($client->hello("no cache world"));
var_dump($client->hello("no cache world"));
```

我们的服务器仍然使用上面例子中的服务器。在确保服务器已启动的情况下，我们运行客户端，可以看到它们分别输出以下结果：

**服务器输出**
>
```
before invoke:
hello
array (
  0 => 'cache world',
)
after invoke:
'Hello cache world!'
before invoke:
hello
array (
  0 => 'no cache world',
)
after invoke:
'Hello no cache world!'
before invoke:
hello
array (
  0 => 'no cache world',
)
after invoke:
'Hello no cache world!'
```
>

**客户端输出**
>
```
before invoke:
hello
array (
  0 => 'cache world',
)
after invoke:
'Hello cache world!'
string(18) "Hello cache world!"
string(18) "Hello cache world!"
before invoke:
hello
array (
  0 => 'no cache world',
)
after invoke:
'Hello no cache world!'
string(21) "Hello no cache world!"
before invoke:
hello
array (
  0 => 'no cache world',
)
after invoke:
'Hello no cache world!'
string(21) "Hello no cache world!"
```
>

我们看到输出结果中 `'cache world'` 的日志只被打印了一次，而 `'no cache world'` 的日志被打印了两次。这说明 `'cache world'` 确实被缓存了。

在这个例子中，我们用到了 `userdata` 设置项和 `$context->userdata`，通过 `userdata` 配合 Hprose 中间件，我们就可以实现自定义选项功能了。

# 输入输出中间件

输入输出中间件可以完全代替 Hprose 过滤器。使用输入输出中间件还是使用 Hprose 过滤器完全看开发者喜好。

输入输出中间件的形式为：

```php
function(string $request, stdClass $context, Closure $next) {
    ...
    $result = $next($request, $context);
    ...
    return $result;
}
```

`$request` 是原始请求数据，对于客户端来说它是输出数据，对于服务器端来说，它是输入数据。该数据的类型为 `string` 类型对象。

`$context` 是调用上下文对象。

`$next` 表示下一个中间件。通过调用 `$next` 将各个中间件串联起来。

`$next` 的返回值 `$response` 是返回的响应数据。对于客户端来说，它是输入数据。对于服务器端来说，它是输出数据。跟调用中间一样，服务器和异步客户端返回的 `$response` 是一个 `promise` 对象，而同步客户端返回的是是一个 `string` 数据。

## 跟踪调试

下面我们来看一下 Hprose 过滤器中的跟踪调试的例子在这里如何实现。

**logHandler2.php**
```php
use Hprose\Future;

$logHandler2 = function($request, stdClass $context, Closure $next) {
    error_log($request);
    $response = $next($request, $context);
    Future\run('error_log', $response);
    return $response;
};
```

**Server.php**
```php
use Hprose\Socket\Server;

function hello($name) {
    return "Hello $name!";
}

$server = new Server('tcp://0.0.0.0:1143/');
$server->addFunction('hello');
$server->debug = true;
$server->addBeforeFilterHandler($logHandler2);
$server->start();
```

**Client.php**
```php
use Hprose\Client;

$client = Client::create('tcp://127.0.0.1:1143/', false);
$client->addBeforeFilterHandler($logHandler2);
var_dump($client->hello("world"));
```

然后分别启动服务器和客户端，就会看到如下输出：

**服务器输出**
>
```
Cs5"hello"a1{s5"world"}z
Rs12"Hello world!"z
```
>

**客户端输出**
>
```
Cs5"hello"a1{s5"world"}z
Rs12"Hello world!"z
string(12) "Hello world!"
```
>

这个结果跟使用 Hprose 过滤器的例子的结果一模一样。

但是我们发现，这里使用 Hprose 中间件要写的代码比起 Hprose 过滤器来要多一些。主要原因是在 Hprose 中间件中，`$next` 的返回值为 `promise` 对象，需要异步处理，而 Hprose 过滤器只需要同步处理就可以了。在这个例子中，我们是直接使用 `Future\run` 来处理异步结果的。

另外，因为这个例子中，我们没有使用过滤器功能，因此使用 `addBeforeFilterHandler` 方法或者 `addAfterFilterHandler` 方法添加中间件处理器效果都是一样的。

但如果我们使用了过滤器的话，那么 `addBeforeFilterHandler` 添加的中间件处理器的 `$request` 数据是未经过过滤器处理的。过滤器的处理操作在 `$next` 的最后一环中执行。`$next` 返回的响应 `$response` 是经过过滤器处理的。

如果某个通过 `addBeforeFilterHandler` 添加的中间件处理器跳过了 `$next` 而直接返回了结果的话，则返回的 `$response` 也是未经过过滤器处理的。而且如果某个 `addBeforeFilterHandler` 添加的中间件处理器跳过了 `$next`，不但过滤器不会执行，而且在它之后使用 `addBeforeFilterHandler` 所添加的中间件处理器也不会执行，`addAfterFilterHandler` 方法所添加的所有中间件处理器也都不会执行。

而 `addAfterFilterHandler` 添加的处理器所收到的 `$request` 都是经过过滤器处理以后的，但它当中使用 `$next` 方法返回的  `$response` 是未经过过滤器处理的。

下面，我们在来看一个结合了压缩过滤器和输入输出缓存中间件的例子。

## 压缩、缓存、计时

**CompressFilter.php**
```php
use Hprose\Filter;

class CompressFilter implements Filter {
    public function inputFilter($data, stdClass $context) {
        return gzdecode($data);
    }
    public function outputFilter($data, stdClass $context) {
        return gzencode($data);
    }
}
```

上面的代码跟 Hprose 过滤器一章的压缩代码完全相同。

**SizeHandler.php**
```php
class SizeHandler {
    private $message;
    public function __construct($message) {
        $this->message = $message;
    }
    public function asynchandle($request, stdClass $context, Closure $next) {
        error_log($this->message . ' request size: ' . strlen($request));
        $response = (yield $next($request, $context));
        error_log($this->message . ' response size: ' . strlen($response));
    }
    public function synchandle($request, stdClass $context, Closure $next) {
        error_log($this->message . ' request size: ' . strlen($request));
        $response = $next($request, $context);
        error_log($this->message . ' response size: ' . strlen($response));
        return $response;
    }
}
```

**StatHandler.php**
```php
class StatHandler {
    private $message;
    public function __construct($message) {
        $this->message = $message;
    }
    public function asynchandle($name, array &$args, stdClass $context, Closure $next) {
        $start = microtime(true);
        yield $next($name, $args, $context);
        $end = microtime(true);
        error_log($this->message . ': It takes ' . ($end - $start) . ' s.');
    }
    public function synchandle($name, array &$args, stdClass $context, Closure $next) {
        $start = microtime(true);
        $response = $next($name, $args, $context);
        $end = microtime(true);
        error_log($this->message . ': It takes ' . ($end - $start) . ' s.');
        return $response;
    }
}
```

**StatHandler2.php**
```php
class StatHandler2 {
    private $message;
    public function __construct($message) {
        $this->message = $message;
    }
    public function asynchandle($request, stdClass $context, Closure $next) {
        $start = microtime(true);
        yield $next($request, $context);
        $end = microtime(true);
        error_log($this->message . ': It takes ' . ($end - $start) . ' s.');
    }
    public function synchandle($request, stdClass $context, Closure $next) {
        $start = microtime(true);
        $response = $next($request, $context);
        $end = microtime(true);
        error_log($this->message . ': It takes ' . ($end - $start) . ' s.');
        return $response;
    }
}
```

这三个中间件，我们给它们分别编写了同步和异步处理程序。异步我们用的是协程方式，看上去跟同步一样简单。

**CacheHandler2.php**
```php
class CacheHandler2 {
    private $cache = array();
    function handle($request, stdClass $context, Closure $next) {
        if (isset($context->userdata->cache)) {
            if (isset($this->cache[$request])) {
                return $this->cache[$request];
            }
            $response = $next($request, $context);
            $this->cache[$request] = $response;
            return $response;
        }
        return $next($request, $context);
    }
}
```

缓存中间件因为不涉及到对结果的判断，所以同步和异步写法是一样的。

**Server.php**
```php
use Hprose\Socket\Server;

$server = new Server('tcp://0.0.0.0:1143/');
$server->addFunction(function($value) { return $value; }, 'echo')
       ->addBeforeFilterHandler(array(new StatHandler2("BeforeFilter"), 'asynchandle'))
       ->addBeforeFilterHandler(array(new SizeHandler("compressedr"), 'asynchandle'))
       ->addFilter(new CompressFilter())
       ->addAfterFilterHandler(array(new StatHandler2("AfterFilter"), 'asynchandle'))
       ->addAfterFilterHandler(array(new SizeHandler("Non compressed"), 'asynchandle'))
       ->addInvokeHandler(array(new StatHandler("Invoke"), 'asynchandle'))
       ->start();
```

在这里我们看到了发布方法，添加中间件，添加过滤器都支持链式调用。

**Client.php**
```php
use Hprose\Client;
use Hprose\InvokeSettings;

$cacheSettings = new InvokeSettings(array("userdata" => array("cache" => true)));

$client = Client::create('tcp://127.0.0.1:1143/', false);
$client->addBeforeFilterHandler(array(new CacheHandler2(), 'handle'))
       ->addBeforeFilterHandler(array(new StatHandler2('BeforeFilter'), 'synchandle'))
       ->addBeforeFilterHandler(array(new SizeHandler('Non compressed'), 'synchandle'))
       ->addFilter(new CompressFilter())
       ->addAfterFilterHandler(array(new StatHandler2('AfterFilter'), 'synchandle'))
       ->addAfterFilterHandler(array(new SizeHandler('compressed'), 'synchandle'))
       ->addInvokeHandler(array(new StatHandler("Invoke"), 'synchandle'));

$value = range(0, 99999);
var_dump(count($client->echo($value, $cacheSettings)));
var_dump(count($client->echo($value, $cacheSettings)));
```

客户端添加过滤器和中间件也支持链式调用。

分别启动服务器和客户端，就会看到如下输出：

**服务器输出**
>
```
compressedr request size: 216266
Non compressed request size: 688893
Invoke: It takes 0.0062549114227295 s.
Non compressed response size: 688881
AfterFilter: It takes 0.039386034011841 s.
compressedr response size: 216245
BeforeFilter: It takes 0.066082954406738 s.
```
>

**客户端输出**
>
```
Non compressed request size: 688893
compressed request size: 216266
compressed response size: 216245
AfterFilter: It takes 0.068840026855469 s.
Non compressed response size: 688881
BeforeFilter: It takes 0.093698024749756 s.
Invoke: It takes 0.10785388946533 s.
int(100000)
Invoke: It takes 0.012754201889038 s.
int(100000)
>
```

我们可以看到两次的执行结果都出来了，但是中间件的输出内容只有一次。原因就是第二次执行时，缓存中间件将缓存的结果直接返回了。因此后面所有的步骤就都略过了。

通过这个例子，我们可以看出，将 Hprose 中间件和 Hprose 过滤器结合，可以实现非常强大的扩展功能。如果你有什么特殊的需求，直接使用 Hprose 无法实现的话，就考虑一下是否可以添加几个 Hprose 中间件和 Hprose 过滤器吧。