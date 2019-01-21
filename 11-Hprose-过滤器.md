# 简介

有时候，我们可能会希望在远程过程调用中对通讯的一些细节有更多的控制，比如对传输中的数据进行加密、压缩、签名、跟踪、协议转换等等，但是又希望这些工作能够跟服务函数/方法本身可以解耦。这个时候，Hprose 过滤器就是一个不错的选择。

Hprose 过滤器是一个接口，它有两个方法：

```php
interface Filter {
    public function inputFilter($data, stdClass $context);
    public function outputFilter($data, stdClass $context);
}
```

其中 `inputFilter` 的作用是对输入数据进行处理，`outputFilter` 的作用是对输出数据进行处理。

`$data` 参数就是输入输出数据，它是 `string` 类型的。这两个方法的返回值也是 `string` 类型的数据，它表示已经处理过的数据，如果你不打算对数据进行修改，你可以直接将 `$data` 参数作为返回值返回。

`$context` 参数是调用的上下文对象，我们在服务器和客户端的介绍中已经多次提到过它。

# 执行顺序

不论是客户端，还是服务器，都可以添加多个过滤器。假设我们按照添加的顺序把它们叫做 `filter1`, `filter2`, ... `filterN`。那么它们的执行顺序是这样的。

## 在客户端的执行顺序

```
+------------------- outputFilter -------------------+
| +-------+      +-------+                 +-------+ |
| |filter1|----->|filter2|-----> ... ----->|filterN| |---------+
| +-------+      +-------+                 +-------+ |         v
+----------------------------------------------------+ +---------------+
                                                       | Hprose Server |
+-------------------- inputFilter -------------------+ +---------------+
| +-------+      +-------+                 +-------+ |         |
| |filter1|<-----|filter2|<----- ... <-----|filterN| |<--------+
| +-------+      +-------+                 +-------+ |
+----------------------------------------------------+
```

## 在服务器端的执行顺序

```
                  +-------------------- inputFilter -------------------+
                  | +-------+                 +-------+      +-------+ |
        +-------->| |filterN|-----> ... ----->|filter2|----->|filter1| |
        |         | +-------+                 +-------+      +-------+ |
+---------------+ +----------------------------------------------------+
| Hprose Client |                                                     
+---------------+ +------------------- outputFilter -------------------+
        ^         | +-------+                 +-------+      +-------+ |
        +---------| |filterN|<----- ... <-----|filter2|<-----|filter1| |
                  | +-------+                 +-------+      +-------+ |
                  +----------------------------------------------------+
```

# 跟踪调试

有时候我们在调试过程中，可能会需要查看输入输出数据。用抓包工具抓取数据当然是一个办法，但是使用过滤器可以更方便更直接的显示出输入输出数据。

**LogFilter.php**
```php
use Hprose\Filter;

class LogFilter implements Filter {
    public function inputFilter($data, stdClass $context) {
        error_log($data);
        return $data;
    }
    public function outputFilter($data, stdClass $context) {
        error_log($data);
        return $data;
    }
}
```

**Server.php**
```php
use Hprose\Socket\Server;

function hello($name) {
    return "Hello $name!";
}

$server = new Server('tcp://0.0.0.0:1143/');
$server->addFunction('hello')
       ->addFilter(new LogFilter())
       ->start();
```

**Client.php**
```php
use Hprose\Client;

$client = Client::create('tcp://127.0.0.1:1143/', false);
$client->addFilter(new LogFilter());
var_dump($client->hello("world"));
```

上面的服务器和客户端代码我们省略了包含路径。请自行脑补，或者直接参见 [[examples 目录|https://github.com/hprose/hprose-php/tree/master/examples/src]]里面的例子。

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

上面输出操作我们用了 `error_log` 函数，并不是因为我们要输出的内容是错误信息，而是因为 Hprose 内部在服务器端过滤了 `echo`、`var_dump `这些用 `ob_xxx` 操作可以过滤掉的输出信息，因为不过滤这些信息，一旦服务器有输出操作，就会造成客户端无法正常运行。所以，我们这里用 `error_log` 函数来进行输出。你也可以换成别的你喜欢的方式，只要不会被 `ob_xxx` 操作过滤掉就可以了。下面的例子我们同样使用这个函数来作为输出。

# 压缩传输

上面的例子，我们只使用了一个过滤器。在本例中，我们展示多个过滤器组合使用的效果。

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

**SizeFilter.php**
```php
use Hprose\Filter;

class SizeFilter implements Filter {
    private $message;
    public function __construct($message) {
        $this->message = $message;
    }
    public function inputFilter($data, stdClass $context) {
        error_log($this->message . ' input size: ' . strlen($data));
        return $data;
    }
    public function outputFilter($data, stdClass $context) {
        error_log($this->message . ' output size: ' . strlen($data));
        return $data;
    }
}
```

**Server.php**
```php
use Hprose\Socket\Server;

$server = new Server('tcp://0.0.0.0:1143/');
$server->addFilter(new SizeFilter('Non compressed'))
       ->addFilter(new CompressFilter())
       ->addFilter(new SizeFilter('Compressed'))
       ->addFunction(function($value) { return $value; }, 'echo')
       ->start();
```

**Client.php**
```php
use Hprose\Client;

$client = Client::create('tcp://127.0.0.1:1143/', false);
$client->addFilter(new SizeFilter('Non compressed'))
       ->addFilter(new CompressFilter())
       ->addFilter(new SizeFilter('Compressed'));

$value = range(0, 99999);
var_dump(count($client->echo($value)));
```

然后分别启动服务器和客户端，就会看到如下输出：

**服务器输出**
>
```
Compressed input size: 216266
Non compressed input size: 688893
Non compressed output size: 688881
Compressed output size: 216245
```
>

客户端输出
>
```
Non compressed output size: 688893
Compressed output size: 216266
Compressed input size: 216245
Non compressed input size: 688881
int(100000)
```
>

在这个例子中，压缩我们使用了 PHP 内置的 gzip 算法，运行前需要确认你开启了这个扩展（一般默认就是开着的）。

加密跟这个类似，这里就不再单独举加密的例子了。

# 运行时间统计

有时候，我们希望能够对调用执行时间做一个统计，对于客户端来说，也就是客户端调用发出前，到客户端收到调用结果的时间统计。对于服务器来说，就是收到客户端调用请求到要发出调用结果的这一段时间的统计。这个功能，通过过滤器也可以实现。

**StatFilter.php**
```php
use Hprose\Filter;

class StatFilter implements Filter {
    private function stat(stdClass $context) {
        if (isset($context->userdata->starttime)) {
            $t = microtime(true) - $context->userdata->starttime;
            error_log("It takes $t s.");
        }
        else {
            $context->userdata->starttime = microtime(true);
        }
    }
    public function inputFilter($data, stdClass $context) {
        $this->stat($context);
        return $data;
    }
    public function outputFilter($data, stdClass $context) {
        $this->stat($context);
        return $data;
    }
}
```

**Server.php**
```php
use Hprose\Socket\Server;

$server = new Server('tcp://0.0.0.0:1143/');
$server->addFilter(new StatFilter())
       ->addFunction(function($value) { return $value; }, 'echo')
       ->start();
```

**Client.php**
```php
use Hprose\Client;

$client = Client::create('tcp://127.0.0.1:1143/', false);
$client->addFilter(new StatFilter());

$value = range(0, 99999);
var_dump(count($client->echo($value)));
```

然后分别启动服务器和客户端，就会看到如下输出：

**服务器输出**
>
```
It takes 0.028308868408203 s.
```
>

**客户端输出**
>
```
It takes 0.03558087348938 s.
int(100000)
```
>

最后让我们把这个这个运行时间统计的例子跟上面的压缩例子结合一下，可以看到更详细的时间统计。

**Server.php**
```php
use Hprose\Socket\Server;

$server = new Server('tcp://0.0.0.0:1143/');
$server->addFilter(new StatFilter())
       ->addFilter(new SizeFilter('Non compressed'))
       ->addFilter(new CompressFilter())
       ->addFilter(new SizeFilter('Compressed'))
       ->addFilter(new StatFilter());
       ->addFunction(function($value) { return $value; }, 'echo')
       ->start();
```

**Client.php**
```php
use Hprose\Client;

$client = Client::create('tcp://127.0.0.1:1143/', false);
$client->addFilter(new StatFilter())
       ->addFilter(new SizeFilter('Non compressed'))
       ->addFilter(new CompressFilter())
       ->addFilter(new SizeFilter('Compressed'))
       ->addFilter(new StatFilter());

$value = range(0, 99999);
var_dump(count($client->echo($value)));
```

然后分别启动服务器和客户端，就会看到如下输出：

**服务器输出**
>
```
Compressed input size: 216266
Non compressed input size: 688893
It takes 0.0014259815216064 s.
It takes 0.031302928924561 s.
Non compressed output size: 688881
Compressed output size: 216245
It takes 0.055199861526489 s.
```
>

**客户端输出**
>
```
Non compressed output size: 688893
Compressed output size: 216266
It takes 0.023594856262207 s.
It takes 0.082166910171509 s.
Compressed input size: 216245
Non compressed input size: 688881
It takes 0.083509922027588 s.
int(100000)
```
>

在这里，我们可以看到客户端和服务器端分别输出了三段用时。

服务器端输出：

第一个 `0.0014259815216064 s` 是解压缩输入数据的时间。

第二个 `0.031302928924561 s` 是第一个阶段用时 + 反序列化 + 调用 + 序列化的总时间。

第三个 `0.055199861526489 s` 是前两个阶段用时 + 压缩输出数据的时间。

客户端输出：

第一个 `0.023594856262207 s` 是压缩输出数据的时间。

第二个 `0.082166910171509 s` 是第一个阶段用时 + 从客户端调用发出到服务器端返回数据的总时间。

第三个 `0.083509922027588 s` 是前两个阶段用时 + 解压缩输入数据的时间。

# 协议转换

Hprose 过滤器的功能不止于此，如果你对 Hprose 协议本身有所了解的话，你还可以直接在过滤器中对输入输出数据进行解析转换。

在 Hprose for PHP 中已经提供了现成的 JSONRPC、XMLRPC 的过滤器。使用它，你可以将 Hprose 服务器变身为 Hprose + JSONRPC + XMLRPC 三料服务器。也可以将 Hprose 客户端变身为 JSONRPC 客户端或 XMLRPC 客户端。

**Hprose + JSONRPC + XMLRPC 三料服务器**
```php
use Hprose\Socket\Server;
use Hprose\Filter\JSONRPC;
use Hprose\Filter\XMLRPC;

function hello($name) {
    return "Hello $name!";
}

$server = new Server('tcp://0.0.0.0:1143/');
$server->addFunction('hello')
       ->addFilter(new JSONRPC\ServiceFilter())
       ->addFilter(new XMLRPC\ServiceFilter())
       ->addFilter(new LogFilter())
       ->start();
```

实现一个三料服务器就这么简单，只需要添加一个协议转换的 `ServiceFilter` 实例对象就可以了。而且这个服务器可以同时接收 Hprose 和 JSONRPC、XMLRPC 三种请求。

**JSONRPC 客户端**
```php
use Hprose\Client;
use Hprose\Filter\JSONRPC;

$client = Client::create('tcp://127.0.0.1:1143/', false);
$client->addFilter(new JSONRPC\ClientFilter())
       ->addFilter(new LogFilter());

var_dump($client->hello("world"));
```

**XMLRPC 客户端**
```php
use Hprose\Client;
use Hprose\Filter\XMLRPC;

$client = Client::create('tcp://127.0.0.1:1143/', false);
$client->addFilter(new XMLRPC\ClientFilter())
       ->addFilter(new LogFilter());

var_dump($client->hello("world"));

```

客户端也是同样的简单，只需要添加一个协议转换的 `ClientFilter` 实例对象，Hprose 客户端就马上变身为对应协议的客户端了。不过需要注意一点，跟服务器不同，添加了 `JSONRPC\ClientFilter` 的客户端，是一个纯 JSONRPC 客户端，这个客户端只能跟 JSONRPC 服务器通讯，不能再跟纯 Hprose 服务器通讯了，但是跟 Hprose + JSONRPC 的双料服务器通讯是没问题的。

上面的程序我们先执行服务器，然后分别执行两个客户端，结果为：

服务器输出

```xml
{"jsonrpc":"2.0","method":"hello","params":["world"],"id":1}
{"id":1,"jsonrpc":"2.0","result":"Hello world!"}
<?xml version="1.0" encoding="iso-8859-1"?>
<methodCall>
<methodName>hello</methodName>
<params>
 <param>
  <value>
   <string>world</string>
  </value>
 </param>
</params>
</methodCall>

<?xml version="1.0" encoding="utf-8"?>
<params>
<param>
 <value>
  <string>Hello world!</string>
 </value>
</param>
</params>
```

JSONRPC 客户端输出

```json
{"jsonrpc":"2.0","method":"hello","params":["world"],"id":1}
{"id":1,"jsonrpc":"2.0","result":"Hello world!"}
string(12) "Hello world!"
```

XMLRPC 客户端输出

```xml
<?xml version="1.0" encoding="iso-8859-1"?>
<methodCall>
<methodName>hello</methodName>
<params>
 <param>
  <value>
   <string>world</string>
  </value>
 </param>
</params>
</methodCall>

<?xml version="1.0" encoding="utf-8"?>
<params>
<param>
 <value>
  <string>Hello world!</string>
 </value>
</param>
</params>

string(12) "Hello world!"
```

Hprose 过滤器的功能很强大，除了上面这些用法之外，你还可以结合服务器事件来实现更为复杂的功能。不过这里就不再继续举例说明了。