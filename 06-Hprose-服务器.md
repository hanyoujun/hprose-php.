# 概述

Hprose 2.0 for PHP 支持多种底层网络协议绑定的服务器，比如：HTTP 服务器，Socket 服务器和 WebSocket 服务器。

其中 HTTP 服务器支持在 HTTP、HTTPS 协议上通讯。

Socket 服务器支持在 TCP、Unix Socket 协议上通讯，并且支持全双工和半双工两种模式。

WebSocket 服务器支持在 ws、wss 协议上通讯。

其中，Hprose 2.0 的核心库提供了 HTTP 服务器和 Socket 服务器，基于 Swoole 的版本提供了 HTTP、Socket、WebSocket 服务器。另外还有 Yii2、Symfony、PSR7 版本的服务器。

为了清晰对比，这里列出一个表格：

功能列表          |    hprose-php     |    hprose-swoole   |     hprose-yii     |    hprose-symfony   |    hprose-psr7
:---------------:|:------------------:|:------------------:|:------------------:|:------------------:|:------------------:
  HTTP 服务器     | :white_check_mark: | :white_check_mark: | :white_check_mark: | :white_check_mark: | :white_check_mark: 
 Socket 服务器    | :white_check_mark: | :white_check_mark: |        :x:         |        :x:         |        :x:         
WebSocket 服务器  |        :x:         | :white_check_mark: |        :x:         |        :x:         |        :x:         
   命令行环境     | :white_check_mark: | :white_check_mark: |        :x:         |        :x:         | :white_check_mark: 
  非命令行环境    | :white_check_mark: |         :x:        | :white_check_mark: | :white_check_mark: | :white_check_mark: 
  推送服务    | :white_check_mark: | :white_check_mark: |        :x:         |        :x:         |        :x:              

尽管支持这么多不同的底层网络协议，但除了在对涉及到底层网络协议的参数设置上有所不同以外，其它的用法都完全相同。因此，我们在下面介绍 Hprose 服务器的功能时，若未涉及到底层网络协议的区别，就以 HTTP 服务器为例来进行说明。

# Server 与 Service 的区别

Hprose 的服务器端的实现，分为 `Service` 和 `Server` 两部分。

其中 `Service` 部分是核心功能，包括接收请求，处理请求，服务调用，返回应答等整个服务的处理流程。

而 `Server` 则主要负责启动和关闭服务器，它包括设置服务地址和端口，设置服务器启动选项，启动服务器，接收来自客户端的连接然后传给 `Service` 进行处理。

之所以分开，是为了更方便的跟已有的库和框架结合，例如：swoole, yii、symfony 等版本都是直接继承 `Service` 来实现自己的服务，之后再创建自己的 `Server` 用于服务设置和启动。

分开的另外一个理由是，`Server` 部分的实现是很简单的，有时候开发者可能会希望把 Hprose 服务结合到自己的某个服务器中去，而不是作为一个单独的服务器来运行，在这种情况下，也是直接使用 `Service` 就可以了。

当开发者没有什么特殊需求，只是希望启动一个独立的 Hprose 服务器时，那使用 `Server` 就是一个最方便的选择了。

# 创建服务器

创建服务器有多种方式，我们先从最简单的方式说起。

## 创建 HTTP 服务器

```php
use Hprose\Http\Server;

function hello($name) {
    return "Hello $name!";
}

$server = new Server();
$server->addFunction('hello');
$server->start();
```

该服务是基于 php-fpm 或者任何支持运行 PHP 的 Web 服务器（例如 Apache、IIS 等）。把它丢到配置好 PHP 的 Web 服务器上，然后在浏览器端打开该文件，你应该会看到以下输出：

>
```
Fa2{u#s5"hello"}z
```
>

如果你用过 Hprose 1.x，你会发现 2.0 版本多了一个 `u#`，这表示有一个名字叫 `#` 的方法，这是 Hprose 2.0 服务器端一个特殊的方法，用来产生一个唯一编号，客户端可以调用该方法得到一个唯一编号用来标识自己。Hprose 2.0 的客户端在使用推送功能时，如果没有手动指定 `id` 的情况下，会自动调用该方法来获取 `id`。

该方法的默认实现很简单，就是一个自增计数，所以一旦当服务器关闭之后重启，该方法会重新开始从 `0` 开始计数。这种方式可能不适用于某些场合，因此，你可能希望能够使用自己的实现来替换它，这是可以做到的。你只需要在发布方法时，将别名指定为 '#' 就可以覆盖默认实现了。

虽然上面对 '#' 这个方法介绍了这么多，然而该服务器并不支持推送服务。如果你真的需要推送服务，你需要创建一个独立的服务器。例如：

```php
use Hprose\Swoole\Server;

function hello($name) {
    return "Hello $name!";
}

$server = new Server("http://0.0.0.0:8086");
$server->addFunction('hello');
$server->start();
```

这是一个基于 Swoole 的 HTTP 的 Hprose 独立服务器。这代码跟上面的代码几乎一样。只是 `use` 的路径不同，`new Server` 时多了一个服务器地址。其它的都一模一样。

但是这段代码的运行方式跟上面那个服务器却完全不同。要运行该服务器，只需要在命令行中键入：

```
php HelloServer.php
```

上面假设跟这个文件保存为 `HelloServer.php`。你的服务就运行起来了，现在你的命令行会卡在那里不动了。但是你在浏览器中输入：`http://<IP>:8086`，其中 `<IP>` 是你运行这个程序的那台服务器的 IP 地址，如果跟你浏览器是同一台机器，IP 可以是 `127.0.0.1`，如果是不同的机器，你应该比我更清楚是多少，所以我就假设你输入正确了，然后你在浏览器端应该看到跟上面那个程序同样的输出（我就不再重复了）。

如果你的浏览器打不开，请检查你的地址是否输入正确，或者你是否开了防火墙把该服务给屏蔽了，或者你的网线是否插好了，诸如此类的问题我就不再一一列举并给出解决方案了。如果你解决不了，我也无能为力，所以请不要费劲地把此类问题提交到 [[issues|https://github.com/hprose/hprose-php/issues]] 中了，我真的帮不了你。

## 创建 TCP 服务器

```php
use Hprose\Socket\Server;

function hello($name) {
    return "Hello $name!";
}
$server = new Server("tcp://0.0.0.0:1314");
$server->addFunction('hello');
$server->start();
```

这是使用 Hprose 核心版本所提供的 Socket 服务器创建的 TCP 服务器。该服务的运行方式跟上面的 HTTP 独立服务器一样，在命令行中使用 php 命令执行就可以了。之后你就可以用客户端去调用它了，至于客户端如何使用，请参考 [[05. Hprose 客户端]] 一章。

该版本的 Socket 服务器没有使用 `pcntl` 之类的扩展，因此可以在 Windows 中使用。该服务是通过单线程异步方式实现的（类似于 node.js），因此该服务支持高并发，也支持推送服务，但是对于每一个服务方法来说，最好执行时间不要过长，因为这会阻塞整个服务。如果你确实有比较耗时的服务要执行，可以考虑开起子进程，借助消息队列将结果返回异步化等方法来自行解决。

当然你也可以考虑使用 Swoole 版本的 Socket 服务器，例如：

```php
use Hprose\Swoole\Server;

function hello($name) {
    return "Hello $name!";
}
$server = new Server("tcp://0.0.0.0:1314");
$server->addFunction('hello');
$server->start();
```

在基本方法的使用上，`Hprose\Swoole\Server` 和 `Hprose\Socket\Server` 是一样的，因此上面的代码中，只有 `use` 语句不同，其它的代码都相同。

对于上面的代码来说，`Hprose\Swoole\Server` 和 `Hprose\Socket\Server` 的性能是几乎一样的，看不出什么优势来。因为默认 Swoole 的 TCP 服务器是使用 Base 模式运行的，该方式跟 `Hprose\Socket\Server` 的方式是一致的。默认采用这种模式，是因为只有这种模式下才支持推送服务，而且该模式下服务编写简单，不需要考虑多进程数据通信问题，另外，新版本的 swoole 对 Base 模式也做了强化，提供了更多的设置和优化，因此 Hprose 默认采用这种模式。

swoole 的服务器还提供了一种进程模式，但是该模式下，多个进程因为不能共享内存，所以推送功能无法使用。如果你不需要推送服务，你可以在创建服务器时这样来指定进程模式：

```php
$server = new Server("tcp://0.0.0.0:1314", SWOOLE_PROCESS);
```

关于 swoole 的这两种模式可以参见 [[swoole 的文档|http://wiki.swoole.com/wiki/page/353.html]]，文档里介绍了 3 种模式，因为其中的线程模式，现在新版本的 swoole 已经不支持了，所以这里就不提了。

## 创建 UNIX Socket 服务器

```php
use Hprose\Socket\Server;

function hello($name) {
    return "Hello $name!";
}

$server = new Server("unix:/tmp/my.sock");
$server->addFunction('hello');
$server->start();
```

```php
use Hprose\Swoole\Server;

function hello($name) {
    return "Hello $name!";
}

$server = new Server("unix:/tmp/my.sock");
$server->addFunction('hello');
$server->start();
```

同样是这两种方式都可以，如何选择请随意。

这两个 Unix Socket 服务器，在启动时，稍微一些区别，对于 `Hprose\Socket\Server`，如果 `/tmp/my.sock` 文件已存在，服务器将不会启动，而是抛出异常。而对于 `Hprose\Swoole\Server` 则不管是否存在（哪怕有另一个服务正在运行），都会正常启动，不会报错。

## 创建 Web Socket 服务器

```php
use Hprose\Swoole\Server;

function hello($name) {
    return "Hello $name!";
}
$server = new Server("ws://0.0.0.0:8088");
$server->addFunction('hello');
$server->start();
```

目前，Web Socket 服务器只有 Swoole 版本支持。创建方式除了地址改为 web socket 的地址格式以外，其它都一样。这里就不多做解释了。

## 其它方式

Hprose 还提供了可以跟 Yii、Symfony、PSR7 等框架结合使用的 HTTP 服务器，这些都是作为单独项目模块提供的，这里就不多做介绍了。

# 服务设置

## debug 属性

该属性为 boolean 类型，默认值为 `false`。

用来设置服务器是否是工作在 debug 模式下，在该模式下，当服务器端发生异常时，将会将详细的错误堆栈信息返回给客户端，否则，只返回错误信息。

## isDebugEnabled 方法

用于获取 `debug` 属性的当前值。

## setDebugEnabled 方法

用于设置 `debug` 属性值。

## simple 属性

该属性为 boolean 类型，默认值为 `false`。

简单数据是指：null、数字（包括整数、浮点数）、Boolean 值、字符串、日期时间等基本类型的数据或者不包含引用的数组和对象。当该属性设置为 `true` 时，在进行序列化操作时，将忽略引用处理，加快序列化速度。但如果数据不是简单类型的情况下，将该属性设置为 `true`，可能会因为死循环导致堆栈溢出的错误。

简单的讲，用 JSON 可以表示的数据都是简单数据。但是对于比较复杂的 JSON 数据，设置 `simple` 为 `true` 可能不会加快速度，反而会减慢，比如对象数组。因为默认情况下，hprose 会对对象数组中的重复字符串的键值进行引用处理，这种引用处理可以对序列化起到优化作用。而关闭引用处理，也就关闭了这种优化。

因为不同调用的数据可能差别很大，因此，建议不要修改默认设置，而是针对某个调用进行单独设置。

## isSimple 方法

获取 `simple` 属性值。

## setSimple 方法

设置 `simple` 属性值。

## passContext 属性

该属性为 boolean 类型，默认值为 `false`。

该属性表示在调用中是否将 `$context` 自动作为最后一个参数传入调用方法。

你也可以针对某个服务函数/方法进行单独设置。

除非所有的服务方法的参数最后都定义了 `$context` 参数。否则，建议不要修改默认设置，而是针对某个服务函数/方法进行单独设置。

## isPassContext 方法

获取 `passContext` 属性值。

## setPassContext 方法

设置 `passContext` 属性值。

## errorDelay 属性

该属性为整型值，默认值为 10000，单位是毫秒。

该属性表示在调用执行时，如果发生异常，将延时一段时间后再返回给客户端。

在关闭该功能的情况下，如果某个服务因为编写错误抛出异常，客户端又反复重试该调用，可能会导致服务器不能正常处理其它业务请求而造成的假死机现象。使用该功能，可以避免这种问题发生。

如果你不需要该功能，设置为 0 就可以关闭它。

## getErrorDelay 方法

获取 `errorDelay` 的属性值。

## setErrorDelay 方法

设置 `errorDelay` 的属性值。

## errorTypes 属性

设置捕获错误的级别，默认值为 `error_reporting` 函数的返回值，即 PHP 的默认设置。

捕获到错误，警告、提示都会以异常的形式发送给客户端。

## getErrorTypes 方法

获取 `errorTypes` 的属性值。

## setErrorTypes 方法

设置 `errorTypes` 的属性值。

# 发布服务

hprose 为发布服务提供了多个方法，这些方法可以随意组合，通过这种组合，你所发布的服务将不会局限于某一个对象，或某一个类，而是可以将不同的函数和方法随意重新组合成一个服务。

## addFunction 方法

```php
public function addFunction($func[, $alias = ''[, array $options = array()]]);
```

该方法的功能是发布函数为远程调用。

`$func` 不仅仅可以是函数名，也可以是方法，例如：array("SomeClass", "somemethod"), array($someObj, "someMethod")，还可以是闭包（匿名函数），还可以是 callable 对象（就是实现了 `__invoke` 魔术方法的对象），还可以是生成器函数（就是 Hprose 支持的协程）。

`$alias` 是发布函数的别名，该别名是客户端调用时所使用的名字。如果 `$func` 本身是函数名，那么 `$alias` 参数可以省略，省略之后，发布的名称跟 `$func` 的名字一致。如果 `$func` 是方法，那么 `$alias` 省略之后，发布的名称跟 `$func` 中的方法名部分相同。其它情况下，不能省略 `$alias` 参数。别名中，你可以使用 `_` 分隔符。当客户端调用时，根据不同的语言，可以自动转换成 `.` 分隔的调用，或者 `->` 分隔的调用。在别名中不要使用 `.` 分隔符。

`$options` 是一个关联数组，它里面包含了一些对该服务函数的特殊设置，有以下设置项可以设置：

* mode
* simple
* oneway
* async
* passContext

### mode

该设置表示该服务函数返回的结果类型，它有4个取值，分别是：

* `Hprose\ResultMode::Normal`
* `Hprose\ResultMode::Serialized`
* `Hprose\ResultMode::Raw`
* `Hprose\ResultMode::RawWithEndTag`

`Hprose\ResultMode::Normal` 是默认值，表示返回正常的已被反序列化的结果。

`Hprose\ResultMode::Serialized` 表示返回的结果保持序列化的格式。

`Hprose\ResultMode::Raw` 表示返回原始数据。

`Hprose\ResultMode::RawWithEndTag` 表示返回带有结束标记的原始数据。 

这四种结果的形式在客户端的相关介绍中已有说明，这里不再重复。

不过要注意的是，这里的设置跟客户端的设置并没有直接关系，这里设置的是服务函数本身返回的数据格式，即使服务函数返回的是 `Hprose\ResultMode::RawWithEndTag` 格式的数据，客户端仍然可以以其它三种格式来接收数据。

该设置通常用于做高性能的缓存或代理服务器。我们在后面介绍 `addMissingFunction` 方法时再举例说明。

### simple

该设置表示本服务函数所返回的结果是否为简单数据。默认值与全局设置一致。前面在属性介绍中已经进行了说明，这里就不在重复。

### oneway

该设置表示本服务函数是否不需要等待返回值。当该设置为 `true` 时，调用会异步开始，并且不等待结果，立即返回 `null` 给客户端。默认值为 `false`。

### async

该设置表示本服务函数是否为异步函数，异步函数的最后一个参数是一个回调函数，用户需要在异步函数中调用该回调方法来传回返回值，例如：

```php
use Hprose\Http\Server;

function hello($name, $callback) {
    $callback("Hello $name!");
}

$server = new Server();
$server->addFunction('hello', array("async" => true));
$server->start();
```

### passContext

该设置与 `$server->passContext` 属性的功能相同。但在这里它是针对该服务函数的单独设置。例如：

```php
use Hprose\Socket\Server;

function hello($name, $context) {
    return "Hello $name! -- " . stream_socket_get_name($context->socket, true);
}

$server = new Server("tcp://0.0.0.0:1314");
$server->addFunction('hello', array("passContext" => true));
$server->start();
```

这里使用的是 `Hprose\Socket\Server`，所以 `$context->socket` 可以得到服务器接收到的客户端连接对象。要注意不同的服务器，$context 中包含的内容是不同的，要根据具体使用的服务器来确定用法。所以如果要编写通用的服务，要避免使用这些跟具体服务器有关的上下文属性。

`$context->clients` 这个属性上面包含了关于推送的方法，这些方法是通用的，不过也仅在独立服务器上通用，在 fpm、cgi 等模式下运行的 HTTP 服务器上不支持。

当 `passContext` 和 `async` 同时设置为 `true` 的时候，服务函数的 `$context` 参数应该放在 `$callback` 参数之前，例如：

```php
use Hprose\Socket\Server;

function hello($name, $context, $callback) {
    $callback("Hello $name! -- " . stream_socket_get_name($context->socket, true));
}

$server = new Server("tcp://0.0.0.0:1314");
$server->addFunction('hello', array("async" => true, "passContext" => true));
$server->start();
```

## addAsyncFunction 方法

```php
public function addAsyncFunction($func[, $alias = ''[, array $options = array()]]);
```

该方法与 `addFunction` 功能相同，但是 `async` 选项被默认设置为 `true`。也就是说，它是 `addFunction` 发布异步方法的简写形式。

## addMissingFunction 方法

```php
public function addMissingFunction($func[, array $options = array()]);
```

该方法用于发布一个用于处理客户端调用缺失服务的函数。缺失服务是指服务器端并没有明确发布的远程函数/方法。例如：

在服务器端没有发布 hello 函数时，在默认情况下，客户端调用该函数，服务器端会返回 `'Can't find this function hello().' 这样一个错误。

但是如果服务器端通过本方法发布了一个用于处理客户端调用缺失服务的 `$func` 函数，则服务器端会返回这个 `$func` 函数的返回值。

该方法还可以用于做代理服务器，例如：

```php
use Hprose\InvokeSettings;
use Hprose\Http\Client;
use Hprose\Socket\Server;
use Hprose\ResultMode;

$client = new Client("http://www.hprose.com/example/", false);
$settings = new InvokeSettings(array("mode" => ResultMode::RawWithEndTag));

$proxy = function($name, $args) use ($client, $settings) {
    return $client->invoke($name, $args, $settings);
};

$server = new Server("tcp://0.0.0.0:1314");
$server->debug = true;
$server->addMissingFunction($proxy, array("mode" => ResultMode::RawWithEndTag));
$server->start();
```

现在，客户端对这个服务器所发出的所有请求，都会通过 `$proxy` 函数转发到 `http://www.hprose.com/example/` 这个服务上，并把结果直接按照原始方式返回。

另外，这我们用的是同步客户端，如果用异步客户端，需要在调用 `$client->invoke` 之后，手动执行 `$client->loop()` 来开始调用，然后再把结果返回。

不过如果我们使用的是 Swoole 的 Http 客户端，就不需要手动调用 `loop` 方法了。所以用异步调用，首选 swoole 客户端。

如果用 swoole 异步客户端，`$client->invoke` 方法的返回值是一个 `promise` 对象，也就是说，服务函数/方法其实也可以直接返回 `promise` 对象，异步服务不一定非要用 callback 方式。

## addAsyncMissingFunction 方法

```php
public function addAsyncMissingFunction($func[, array $options = array()]);
```

该方法与 `addMissingFunction` 功能相同，但是 `async` 选项被默认设置为 `true`。也就是说，它是 `addMissingFunction` 发布异步方法的简写形式。

## addFunctions 方法

```php
public function addFunctions(array $funcs[, array $aliases = array()[, array $options = array()]]);
```

如果你想同时发布多个方法，可以使用该方法。

`$funcs` 是函数数组，数组元素必须为 callable 类型的对象。

`$aliases` 是别名数组，数组元素必须是字符串，并且需要与 `$funcs` 数组中每个相同键名的元素一一对应。

当 `$funcs` 中的函数全都是具名函数时，`$aliases` 可以省略。

`$options` 的选项值跟 `addFunction` 方法相同。

## addAsyncFunctions 方法

```php
public function addAsyncFunctions(array $funcs[, array $aliases = array()[, array $options = array()]]);
```
该方法与 `addFunctions` 功能相同，但是 `async` 选项被默认设置为 `true`。也就是说，它是 `addFunctions` 发布异步方法的简写形式。

## addMethod 方法

```php
public function addMethod($method, $scope[, $alias = ''[, array $options = array()]]);
```

该方法跟 `addFunction` 类似，它的功能是添加方法。

`$method` 是方法名，字符串类型。

`$scope` 是 `$method` 所在的类或对象。如果 `$method` 是静态方法，那么 `$scope` 应为类名，如果 `$method` 是实例方法，那么 `$scope` 应为对象。不要搞混，切记，切记。

`$alias` 是发布方法的别名，忽略时，跟 `$method` 的相同。

`$options` 的选项值跟 `addFunction` 方法相同。

## addAsyncMethod 方法

```php
public function addAsyncMethod($method, $scope[, $alias = ''[, array $options = array()]]);
```

该方法与 `addMethod` 功能相同，但是 `async` 选项被默认设置为 `true`。也就是说，它是 `addMethod` 发布异步方法的简写形式。

## addMissingMethod 方法

```php
public function addMissingMethod($method, $scope[, array $options = array()]);
```

该方法的功能与 `addMissingMethod` 类似。它们之前的区别跟 `addMethod` 和 `addFunction` 相同。这里就不详细介绍了。

## addAsyncMissingMethod 方法

```php
public function addAsyncMissingMethod($method, $scope[, array $options = array()]);
```

该方法与 `addMissingMethod` 功能相同，但是 `async` 选项被默认设置为 `true`。也就是说，它是 `addMissingMethod` 发布异步方法的简写形式。

## addMethods 方法

```php
public function addMethods($methods, $scope[, $aliases = array()[, array $options = array()]]);
```

该方法的功能与 `addFunctions` 类似。它们之前的区别跟 `addMethod` 和 `addFunction` 相同。这里就不详细介绍了。

## addAsyncMethods 方法

```php
public function addAsyncMethods($methods, $scope[, $aliases = array()[, array $options = array()]]);
```

## addInstanceMethods 方法

```php
public function addInstanceMethods($object[, $class = ''[, $aliasPrefix = ''[, array $options = array()]]]);
```

该方法用于发布 `$object` 对象上所在类上声明的所有 `public` 实例方法。

当指定了 `$class` 参数时，将只发布 `$class` 这一层上声明的所有 `public` 实例方法。

`$aliasPrefix` 是别名前缀，例如假设有一个 `$user` 对象，该对象上包含有 `add`，`del`，`update`，`query` 四个方法。那么当调用：

```php
$server->addInstanceMethods($user, '', 'user');
```

的方式来发布 `$user` 对象上的这四个方法后，等同于这样的发布：

```php
$server->addMethods(array('add'，'del'，'update'，'query'), $user,
                    array('user_add'，'user_del'，'user_update'，'user_query'));
```

即在每个发布的方法名之前都添加了一个 `user_` 的前缀。注意这里前缀和方法名之间是使用 `_` 分隔的。

当省略 `$aliasPrefix` 参数时，发布的方法名前不会增加任何前缀。

最后的 `$options` 选项值跟 `addFunction` 方法相同。

## addAsyncInstanceMethods 方法

```php
public function addAsyncInstanceMethods($object[, $class = ''[, $aliasPrefix = ''[, array $options = array()]]]);
```

该方法与 `addInstanceMethods` 功能相同，但是 `async` 选项被默认设置为 `true`。也就是说，它是 `addInstanceMethods` 发布异步方法的简写形式。

## addClassMethods 方法

```php
public function addClassMethods($class[, $scope = ''[, $aliasPrefix = ''[, array $options = array()]]]);
```

该方法用来发布 `$class` 上定义的所有静态方法。

`$scope` 是方法实际执行所在的类，通常 `$scope` 跟 `$class` 是等同的（默认值），不过你也可以设置 `$scope` 为 `$class` 的父类。

`$aliasPrefix` 作用跟 `addInstanceMethods` 的 `$aliasPrefix` 参数一样。

最后的 `$options` 选项值跟 `addFunction` 方法相同。

## addAsyncClassMethods 方法

```php
public function addAsyncClassMethods($class[, $scope = ''[, $aliasPrefix = ''[, array $options = array()]]]);
```

该方法与 `addClassMethods` 功能相同，但是 `async` 选项被默认设置为 `true`。也就是说，它是 `addClassMethods` 发布异步方法的简写形式。

## add 方法

上面如此之多的 `addXXX` 方法也许会把你搞晕，也许你不查阅本手册，都记不清该使用哪个方法来发布。

没关系，`add` 方法就是用来简化上面这些 `addXXX` 方法的。

`add` 方法不支持 `$options` 参数。其它参数你只要按照上面任何一个方法的参数来写，`add` 方法都可以自动根据参数的个数和类型判断该调用哪个方法进行发布，当你不需要设置 `$options` 参数时，它会大大简化你的工作量。

## addAsync 方法

该方法与 `add` 功能相同，但是 `async` 选项被默认设置为 `true`。也就是说，它是 `add` 发布异步方法的简写形式。

## remove 方法

该方法与 `add` 功能相反。使用该方法可以移除已经发布的函数，方法或者推送主题。该方法的参数为发布的远程方法的别名。注意：该别名是大小写敏感的。


## 在RPC方法中获取当前 $context 对象

在 addFunction 时给设置 passContext = true 参数可以在 RCP 回调的方法在最后一个参数传进去，但是这样做需要改动回调函数比较麻烦，那么可以在RPC回调方法里直接使用 `$context = Hprose\Service::getCurrentContext()` 获取到，如果是协程的，使用 `$context = Hprose\Service::getCurrentContextCo()` 方法获取。