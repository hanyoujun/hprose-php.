
# 概述

Hprose 2.0 for PHP 客户端相比 1.x 版本有了比较大的改动。

核心版本除了提供了客户端的基本实现的基类以外，还提供了 HTTP 客户端和 Socket 客户端。这两个客户端都可以创建为同步或异步客户端。这两个客户端既可以在命令行环境下使用，也可以在 php-fpm 或其他 PHP 环境下使用。

另外 [[swoole 版本|https://github.com/hprose/hprose-swoole]] 提供了纯异步的 HTTP 客户端，Socket 客户端和 WebSocket 客户端。Swoole 客户端只能在命令行环境下使用。

其中 HTTP 客户端支持跟 HTTP、HTTPS 绑定的 Hprose 服务器通讯。

Socket 客户端支持跟 TCP、Unix Socket 绑定的 Hprose 服务器通讯，并且支持全双工和半双工两种模式。

WebSocket 客户端支持跟 ws、wss 绑定的 Hprose 服务器通讯。

为了清晰对比，这里列出一个表格：

功能列表          |    hprose-php     |    hprose-swoole
:---------------:|:------------------:|:-----------------:
   同步调用       | :white_check_mark: |        :x:  
   异步调用       | :white_check_mark: | :white_check_mark:  
  HTTP 客户端     | :white_check_mark: | :white_check_mark:  
 Socket 客户端    | :white_check_mark: | :white_check_mark: 
WebSocket 客户端  |        :x:         | :white_check_mark:
   命令行环境     | :white_check_mark: | :white_check_mark: 
  非命令行环境    | :white_check_mark: |         :x: 

尽管支持这么多不同的底层网络协议，但除了在对涉及到底层网络协议的参数设置上有所不同以外，其它的用法都完全相同。因此，我们在下面介绍 Hprose 客户端的功能时，若未涉及到底层网络协议的区别，就以 HTTP 客户端为例来进行说明。

# 创建客户端

创建客户端有两种方式，一种是直接使用构造器，另一种是使用工厂方法 create。

## 使用构造器创建客户端

`Hprose\Client` 是一个抽象类，因此它不能作为构造器直接使用。如果你想创建一个具体的底层网络协议绑定的客户端，你可以将它作为父类，至于如何实现一个具体的底层网络协议绑定的客户端，这已经超出了本手册的内容范围，这里不做具体介绍，有兴趣的读者可以参考 `Hprose\Http\Client`、`Hprose\Socket\Client` 和 `Hprose\Swoole\WebSocket\Client` 等底层网络协议绑定客户端的实现源码。

`Hprose\Http\Client`、`Hprose\Socket\Client` 这两个类是可以直接使用的构造器。它们分别对应 HTTP 客户端、Socket 客户端。

创建方式如下：

```php
$client = new \Hprose\Http\Client([$uriList = null[, $async = true]]);
```

`[]` 内的参数表示可选参数。

当两个参数都省略时，创建的客户端是未初始化的异步客户端，后面需要使用 `useService` 方法进行初始化，这是后话，暂且不表。

第 1 个参数 `$uriList` 是服务器地址，该服务器地址可以是单个的地址字符串，也可以是由多个地址字符串组成的数组。当该参数为多个地址字符串组成的数组时，客户端会从这些地址当中随机选择一个作为服务地址。因此需要保证这些地址发布的都是完全相同的服务。

第 2 个参数 `$async` 表示是否是异步客户端，在 Hprose 2.0 for PHP 中，默认创建的都是异步客户端，这是因为 Swoole 客户端只支持异步，为了可以方便的在普通客户端和 Swoole 客户端之间切换，所以默认设置为异步。异步客户端在进行远程调用时，返回值为一个 `promise` 对象。而同步客户端在进行远程调用时，返回值为实际返回值（或者抛出异常）。客户端创建之后，该类型不能被更改。

例如：

**创建一个同步的 HTTP 客户端**
```php
$client = new \Hprose\Http\Client('http://hprose.com/example/', false);
```

**创建一个同步的 TCP 客户端**
```php
$client = new \Hprose\Socket\Client('tcp://127.0.0.1:1314', false);
```

**创建一个异步的 Unix Socket 客户端**
```php
$client = new \Hprose\Socket\Client('unix:/tmp/my.sock');
```

**创建一个异步的 WebSocket 客户端**
```php
$client = new \Hprose\Swoole\WebSocket\Client('ws://127.0.0.1:8080/');
```

>
注意：如果要使用 swoole 客户端，需要在 composer.json 加入对 `hprose/hprose-swoole` 的引用。且 Swoole 客户端不支持第二个参数。
>

另外，如果创建的是 Swoole 的客户端，还有更简单的方式：

**同样创建一个异步的 WebSocket 客户端**
```php
$client = new \Hprose\Swoole\Client('ws://127.0.0.1:8080/');
```

也就是说，只需要使用 `Hprose\Swoole\Client`，就可以创建所有 Swoole 支持的客户端了，Hprose 可以自动根据服务器地址的 scheme 来判断客户端类型。


## 通过工厂方法 `create` 创建客户端

```php
$client = \Hprose\Client::create($uriList = null,[ $async = true]);
```

`create` 方法与构造器函数的参数一样，返回结果也一样。但是第一个参数 `$uriList` 不能被省略。

使用 `create` 方法更加方便，因此，除非在创建客户端的时候，不想指定服务地址，否则，应该优先考虑使用 `create` 方法来创建客户端。

`\Hprose\Client::create` 支持创建 Hprose 核心库上的客户端，`Hprose\Swoole\Client::create` 支持创建 swoole 的客户端。例如：

**创建一个同步的 HTTP 客户端**
```php
$client = \Hprose\Client::create('http://hprose.com/example/', false);
```

**创建一个同步的 TCP 客户端**
```php
$client = \Hprose\Client::create('tcp://127.0.0.1:1314', false);
```

**创建一个异步的 Unix Socket 客户端**
```php
$client = \Hprose\Client::create('unix:/tmp/my.sock');
```

**创建一个异步的 WebSocket 客户端**
```php
$client = \Hprose\Swoole\Client::create('ws://127.0.0.1:8080/');
```

## 注册自己的客户端实现类

如果你自己创建了一个客户端实现，你可以通过：
```php
Client::registerClientFactory($scheme, $clientFactory);
```

或者
```php
Client::tryRegisterClientFactory($scheme, $clientFactory);
```

这两个静态方法来注册自己的客户端实现。

注册之后，你就可以使用 `create` 方法来创建你的客户端对象了。

`registerClientFactory` 方法会覆盖原来已注册的相同 `$scheme` 的客户端类，`tryRegisterClientFactory` 方法不会覆盖。

## uri 地址格式

### HTTP 服务地址格式

HTTP 服务地址与普通的 URL 地址没有区别，支持 `http` 和 `https` 两种协议，这里不做介绍。

### WebSocket 服务地址格式

除了协议从 `http` 改为 `ws`（或 `wss`） 以外，其它部分与 `http` 地址表示方式完全相同，这里不再详述。

### TCP 服务地址格式

```
<scheme>://<ip>:<port>
```

`<ip>` 是服务器的 IP 地址，也可以是域名。

`<port>` 是服务器的端口号，hprose 的 TCP 服务没有默认端口号，因此不可省略。

`<scheme>` 表示协议，它可以为以下取值：

* `tcp`
* `ssl`
* `sslv2`
* `sslv3`
* `tls`

`tcp` 表示 tcp 协议，地址可以是 ipv6 地址，也可以是 ipv4 地址。

`ssl`, `sslv2`, `sslv3` 和 `tls` 表示安全的 tcp 协议。如有必要，可设置客户端安全证书。

### Unix Socket 服务地址格式

```
unix:<path>
```

其中 `<path>` 是绝对路径（以 `/` 开头）。例如：

```
unix:/tmp/my.sock
```

# 事件

## onError 事件

该事件为 `callable` 类型，默认值为 `NULL`。

当客户端采用回调方式进行调用时，并且回调函数没有参数，如果发生异常，该事件会被调用。该事件的回调函数格式为：

```
function onError($name, $e);
```

`$name` 是字符串类型，`$e` 在 PHP 5 中 为 `Exception` 类型或它的子类型对象，在 PHP 7 中是 `Throwable` 接口的实现类对象。

## onFailswitch 事件

该属性为 `callable` 类型，默认值为 `NULL`。

当调用的 `failswitch` 属性设置为 `true` 时，如果在调用中出现网络错误，进行服务器切换时，该事件会被触发。该事件的回调函数格式为：

```
function onFailswitch($client);
```

`$client` 即客户端对象。

# 属性

## uri 属性

只读属性。字符串类型，表示当前客户端所调用的服务地址。

## filters 属性

只读属性。数组类型，表示当前客户端上添加的过滤器列表。

## timeout 属性

整数类型，默认值为 `30000`，单位是毫秒（ms）。表示客户端在调用时的超时时间，如果调用超过该时间后仍然没有返回，则会以超时错误返回。

## failswitch 属性

布尔类型。默认值为 `false`。表示当前客户端在因网络原因调用失败时是否自动切换服务地址。当客户端服务地址仅设置一个时，不管该属性值为何，都不会切换地址。

## failround 属性

整数类型，只读属性。初始值为 `0`。当调用中发生服务地址切换时，如果服务列表中所有的服务地址都切换过一遍之后，该属性值会加 `1`。你可以根据该属性来决定是否更新服务列表。更新服务列表可以使用 `setUriList` 方法。

## idempotent 属性

布尔类型，默认值为 `false`。表示调用是否为幂等性调用，幂等性调用表示不论该调用被重复几次，对服务器的影响都是相同的。幂等性调用在因网络原因调用失败时，会自动重试。如果 `failswitch` 属性同时被设置为 `true`，并且客户端设置了多个服务地址，在重试时还会自动切换地址。

## retry 属性

整数类型，默认值为 `10`。表示幂等性调用在因网络原因调用失败后的重试次数。只有 `idempotent` 属性为 `true` 时，该属性才有作用。

## byref 属性

布尔类型，默认值为 `false`。表示调用是否为引用参数传递。当设置为引用参数传递时，服务器端会传回修改后的参数值（即使没有修改也会传回）。因此，当不需要该功能时，设置为 `false` 会比较节省流量。

## simple 属性

布尔类型，默认值为 `false`。表示调用中所传输的数据是否为简单数据。

简单数据是指：null、数字（包括整数、浮点数）、Boolean 值、字符串、日期时间等基本类型的数据或者不包含引用的数组和对象。当该属性设置为 `true` 时，在进行序列化操作时，将忽略引用处理，加快序列化速度。但如果数据不是简单类型的情况下，将该属性设置为 `true`，可能会因为死循环导致堆栈溢出的错误。

简单的讲，用 `JSON` 可以表示的数据都是简单数据。但是对于比较复杂的 `JSON` 数据，设置 `simple` 为 `true` 可能不会加快速度，反而会减慢，比如对象数组。因为默认情况下，hprose 会对对象数组中的重复字符串的键值进行引用处理，这种引用处理可以对序列化起到优化作用。而关闭引用处理，也就关闭了这种优化。

因为不同调用的数据可能差别很大，因此，建议不要修改默认设置，而是针对某个调用进行单独设置。

# 方法

## close 方法

关闭客户端。它会在析构方法中被自动调用，因此，你通常不需要手动调用它。

## getUriList 方法

获取服务器列表。

## setUriList 方法

设置服务器列表。

## getTimeout 方法

获取 timeout 属性值。

## setTimeout 方法

设置 timeout 属性值。

## getRetry 方法

获取 retry 属性值。

## setRetry 方法

设置 retry 属性值。

## isIdempotent 方法

获取 idempotent 属性值。

## setIdempotent 方法

设置 idempotent 属性值。

## isFailswitch 方法

获取 failswitch 属性值。

## setFailswitch 方法

设置 failswitch 属性值。

## getFailround 方法

获取 failround 属性值。

## isByref 方法

获取 byref 属性值。

## setByref 方法

设置 byref 属性值。

## isSimple 方法

获取 simple 属性值。

## setSimple 方法

设置 simple 属性值。

## getFilter 方法

获取添加的第一个过滤器对象。

## setFilter 方法

设置一个过滤器对象，如果原来已经设置或添加了过滤器，该方法会先清除掉之前的设置，然后在设置参数所指定的过滤器。

## addFilter 方法

添加一个过滤器。跟 `setFilter` 只能设置一个过滤器不同，通过 `addFilter` 方法，可以添加多个过滤器。

## removeFilter 方法

删除指定的过滤器。如果没有找到返回 false，删除成功返回 true。

## useService 方法

```php
function useService([$uriList = array()[, $namespace = '']]);
```

该方法两个参数都是可选的。

该方法返回值为一个 `\Hprose\Proxy` 对象。该对象是远程服务调用代理对象，你可以在上面直接调用远程方法。

当设置了 `$uriList` 参数时，跟上面的功能相同，但是会替换当前的 `$uriList` 设置。`$uriList` 可以是单个地址，也可以是一个服务地址列表。

参数 `$namespace` 是远程服务的名称空间，它本质上是方法名的别名前缀。例如，服务器端发布的方法名是：`user_add`，`user_remove`，`user_update`, `user_get`。那么可以这样使用：

```php
$userService = $client->useService($uri, 'user');
$userId = $userService->add(array('name' => 'Tom', 'age' => 18));
$userService->update($userId, array('age' => 14));
$user = $userService->get($userId);
$userService->remove($userId);
```

上面的例子中 `$userService` 是 `useService` 方法返回的一个远程服务代理对象。因为在调用 `useService` 方法时指定了 `user` 这个名称空间，所以在 `$userService` 对象上调用远程方法时，就不再加 `user_` 这个前缀了。代理对象在发起调用时，会自动帮你加上。

>
*注意：`$userService` 和 `useService` 方法看上去很像，但他们拼写并不相同，意义也不相同，它们之间没有关系，没有关系，没有关系【重要的事情说三遍】。*
>

## invoke 方法

```php
function invoke($name[, array $args = array()[, $callback = null[, InvokeSettings $settings = null]]])
```

该方法是客户端最核心的方法，但是你可能永远也不会直接使用它，虽然你可能不用，但我还是要对它做一下介绍。

`$name` 是远程函数/方法名。

`$args` 是方法参数。如果没有参数就是空数组（当然你也可以不写该参数，因为默认值就是空数组），有一个参数就是有一个值的数组，有两个参数就是有两个值的数组。如果是异步客户端，`$args` 中可以包含 `promise` 对象的参数值。但是同步客户端的同步调用不可以包含 `promise` 对象的参数。另外，请注意，$args 只支持位置参数，不支持命名参数。所以**不要**试图使用字符串键名来作为命名参数的名称，它是不管用的，不管用的，不管用的……

`$callback` 是回调函数。这个是传统的异步调用方式。不管你创建的是同步客户端，还是异步客户端，使用回调函数的方式进行异步调用都是支持的。另外，如果指定了该参数，`$args` 参数中也可以包含 `promise` 对象的参数值，哪怕是同步客户端。但是你真的愿意使用回调函数层层回调吗？真的愿意吗？不愿意吗？愿意吗？不管你愿意还是不愿意，我还是需要介绍一下回调函数的格式：

```php
function callback();
function callback($result);
function callback($result, $args);
function callback($result, $args, $error);
```

回调函数支持这四种格式， 在使用 `invoke` 方法时，`$callback` 只要是 `callable` 类型就可以，不管是函数，方法，还是闭包和可执行对象，都可以。但是如果是用方法名直接调用，`$callback` 必须是闭包类型。

下面介绍一下参数的意义

**参数**   | **解释**
:-------:  | :----------
`$result`  | 服务器端返回的结果，如果没有结果，该值为 `NULL`，如果没有 `$error` 参数，并且调用发生异常，该值为异常对象。
`$args`    | 调用的参数，如果没有参数，该值为空数组。
`$error`   | 如果调用发生异常，该值为异常对象，否则该值为 `NULL`。

最后一个参数 `$settings` 是 Hprose 2.0 新加的。1.x 版本中在 `$callback` 后面还有一堆几个参数用户设置调用的特殊属性，但是 Hprose 2.0 为每个调用增加了许多特殊属性设置，所以再用位置参数的方式，用户恐怕就要晕了。所以，这里牺牲了兼容性，改成了使用一个 `$settings` 参数来存储所有的特殊属性设置。

`$settings` 参数是一个 `\Hprose\InvokeSettings` 类型的对象。`\Hprose\InvokeSettings` 是个什么东？让我们来看一下：

## InvokeSettings 类

该类的构造参数是一个数组。你可以通过这个数组来设置调用的特殊属性，下面来看一下支持哪些设置：

* `mode`
* `byref`
* `simple`
* `failswitch`
* `timeout`
* `idempotent`
* `retry`
* `oneway`
* `userdata`

下面来分别介绍一下这些设置的意义：

### mode

该设置表示结果返回的类型，它有4个取值，分别是：

* `Hprose\ResultMode::Normal`
* `Hprose\ResultMode::Serialized`
* `Hprose\ResultMode::Raw`
* `Hprose\ResultMode::RawWithEndTag`

`Hprose\ResultMode::Normal` 是默认值，表示返回正常的已被反序列化的结果。

`Hprose\ResultMode::Serialized` 表示返回的结果保持序列化的格式。

`Hprose\ResultMode::Raw` 表示返回原始数据。

`Hprose\ResultMode::RawWithEndTag` 表示返回带有结束标记的原始数据。

这样说明也许有些晦涩，让我们来看一个例子就清楚了：

```php
use Hprose\Client;
use Hprose\InvokeSettings;
use Hprose\ResultMode;

$client = Client::create('http://hprose.com/example/', false);

var_dump($client->hello("World", new InvokeSettings(array('mode' => ResultMode::Normal))));
var_dump($client->hello("World", new InvokeSettings(array('mode' => ResultMode::Serialized))));
var_dump($client->hello("World", new InvokeSettings(array('mode' => ResultMode::Raw))));
var_dump($client->hello("World", new InvokeSettings(array('mode' => ResultMode::RawWithEndTag))));
```

该程序执行结果如下：

>
```
string(11) "Hello World"
string(16) "s11"Hello World""
string(17) "Rs11"Hello World""
string(18) "Rs11"Hello World"z"
```
>

眼尖的同学一定发现了，我们这里并没有使用 `invoke` 方法，而是直接使用远程方法名 `hello` 作为本地方法名来直接使用，参数值 `World`，也没有放在数组里，我们前面说过了，你可能永远都不会直接使用 `invoke` 方法，就是因为你可以直接这样调用，这比使用 `invoke` 方法简单直观的多。

`$callback` 参数在这里也省略了，因为我们这里是使用同步调用，所以不能有它。使用 `invoke` 方法也是通过省略它来进行同步调用的。而且，一旦习惯了 Hprose 2.0 的写法，即使是异步调用，你也不会再使用这个参数。所以，就让我们忽略它啊，后面我们也不再提它了。

你可能会有这样的疑问，为啥不直接用数组来表示这些特殊设置，而要在数组外面再套一个 `new InvokeSettings` 呢？这样写起来很麻烦不是吗？确实是麻烦了一点，但是这也是不得已而为之，因为数组也是远程调用可以传递的参数类型，所以，如果用数组，调用可分不清这个数组是设置，还是传给服务器的参数，就会把它作为远程调用参数传给服务器了，这显然不是我们期望的。所以，只好包上一层 `InvokeSettings` 的包装，这样客户端就知道，这个设置不是要传给服务器的参数了。

不过这里可以透露一个小秘密，如果你是使用方法名进行远程调用（也就是说，你不是直接使用 `invoke` 方法），而且使用了 `$callback` 参数（我们这里不是为了提它，只是不得不提它），那么在 `$callback` 参数后面的设置确实可以使用一个普通数组，而不需要套一个 `new InvokeSettings` 的包装。不过为了避免你用顺了手，你最好还是不要这样玩。就当我没提好了。

### byref

该设置表示调用是否为引用参数传递方式。例如：

```php
<?php
require_once "../../vendor/autoload.php";

use Hprose\Client;
use Hprose\InvokeSettings;

$client = Client::create('http://hprose.com/example/', false);

$weeks = array(
    'Monday' => 'Mon',
    'Tuesday' => 'Tue',
    'Wednesday' => 'Wed',
    'Thursday' => 'Thu',
    'Friday' => 'Fri',
    'Saturday' => 'Sat',
    'Sunday' => 'Sun'
);

$args = array($weeks);
$client->invoke('swapKeyAndValue', $args, new InvokeSettings(array('byref' => true)));
var_dump($args[0]);

$client->swapKeyAndValue($weeks, function($result, $args) {
    var_dump($args[0]);
}, array('byref' => true));
```

运行结果为：

>
```
array(7) {
  ["Mon"]=>
  string(6) "Monday"
  ["Tue"]=>
  string(7) "Tuesday"
  ["Wed"]=>
  string(9) "Wednesday"
  ["Thu"]=>
  string(8) "Thursday"
  ["Fri"]=>
  string(6) "Friday"
  ["Sat"]=>
  string(8) "Saturday"
  ["Sun"]=>
  string(6) "Sunday"
}
array(7) {
  ["Mon"]=>
  string(6) "Monday"
  ["Tue"]=>
  string(7) "Tuesday"
  ["Wed"]=>
  string(9) "Wednesday"
  ["Thu"]=>
  string(8) "Thursday"
  ["Fri"]=>
  string(6) "Friday"
  ["Sat"]=>
  string(8) "Saturday"
  ["Sun"]=>
  string(6) "Sunday"
}
```
>

注意：同步调用需要使用 `invoke` 方法才支持引用参数传递。异步调用要用回调方式才支持引用参数传递。

### simple

该设置表示本次调用中所传输的参数是否为简单数据。前面在属性介绍中已经进行了说明，这里就不在重复。

### failswitch

该设置表示当前调用在因网络原因失败时是否自动切换服务地址。

### timeout

该设置表示本次调用的超时时间，如果调用超过该时间后仍然没有返回，则会以超时错误返回。

### idempotent

该设置表示本次调用是否为幂等性调用，幂等性调用在因网络原因调用失败时，会自动重试。

### retry

该设置表示幂等性调用在因网络原因调用失败后的重试次数。只有 `idempotent` 设置为 `true` 时，该设置才有作用。

### oneway

该设置表示当前调用是否不等待返回值。当该设置为 `true` 时，请求发送之后，并不等待服务器返回结果，而是直接返回 `null`。

### userdata

该属性是一个对象，它用于存放一些用户自定义的数据。这些数据可以通过 `$context` 对象在整个调用过程中进行传递。当你需要实现一些特殊功能的 Filter 或 Handler 时，可能会用到它。

上面这些设置除了可以作为 `settings` 参数的属性传入以外，还可以在远程方法上直接进行属性设置，这些设置会成为 `settings` 参数的默认值。例如上面引用参数传递的例子还可以写成这样：

```php
use Hprose\Client;

$client = Client::create('http://hprose.com/example/', false);

$weeks = array(
    'Monday' => 'Mon',
    'Tuesday' => 'Tue',
    'Wednesday' => 'Wed',
    'Thursday' => 'Thu',
    'Friday' => 'Fri',
    'Saturday' => 'Sat',
    'Sunday' => 'Sun'
);

$args = array($weeks);

$client->swapKeyAndValue["byref"] = true;
$client->swapKeyAndValue($weeks, function($result, $args) {
    var_dump($args[0]);
});
```

但是要注意，这样设置的属性，只有用远程方法名直接调用才有效，使用 `invoke` 方法调用时无效。

## 直接用远程方法名调用

上面在介绍 `invoke` 方法时，我们已经在例子中看到过直接用远程方法名调用的用法了。这里再补充一点高级的。

当你的服务器发布了多个对象，并且每个对象上都有一组自己的方法，还有可能有重名的方法，因此，你可能会为每个对象添加一个名称空间（namespace），它在 Hprose 中是以别名前缀的方式实现的。

例如：服务器端发布了一个 `user` 对象，一个 `order` 对象。上面都有 `add`, `update`, `remove` 和 `get` 方法。发布 `user` 对象时，添加了一个 `user` 名空间，发布 `order` 对象时，添加了一个 `order` 名空间。

这样服务器端发布的方法名实际上是：`user_add`, `user_update`, `user_remove`，`user_get`, `order_add`, `order_update`, `order_remove` 和 `order_get`。

客户端可以直接用这些方法名进行调用，例如：

```php
$userid = $client->user_add(array('name' => 'Tom', 'age' => 18));
$client->user_remove($userid);
```

还可以这样调用：

```php
$userid = $client->user->add(array('name' => 'Tom', 'age' => 18));
$client->user->remove($userid);
```

这两种调用方式都是可以的，后一种方法看上去更清晰，而且还可以简化为：

```php
$user = $client->user;
$userid = $user->add(array('name' => 'Tom', 'age' => 18));
$user->remove($userid);
```

## 链式调用

上面我们讲了很多同步调用，异步调用用的也是传统的回调方式。下面我们来讲讲 Hprose 2.0 新增的 `promise` 方式的异步调用。

对于 Hprose 异步客户端来说，使用 `invoke` 方法或者直接使用远程方法名调用，返回的结果是一个 `promise` 对象。因此，它可以进行链式调用，例如：

```php
<?php
require_once "../../vendor/autoload.php";

use Hprose\Client;

$client = Client::create('http://hprose.com/example/');

$client->sum(1, 2)
       ->then(function($result) use ($client) {
            return $client->sum($result, 3);
       })
       ->then(function($result) use ($client) {
            return $client->sum($result, 4);
       })
       ->then(function($result) {
            var_dump($result);
       });
```

该程序的结果为：

>
```
10
```
>

当你有很多回调的时候，这种方式要比层层回调的异步方式清晰的多。但是，这还不算最简单的。

## 更简单的顺序调用

前面我们讲过，当使用方法名调用时，远程调用的参数本身也可以是 `promise` 对象。

因此，上面的链式调用还可以直接简化为：

```php
use Hprose\Client;

$client = Client::create('http://hprose.com/example/');

$client->sum($client->sum($client->sum(1, 2), 3), 4)
       ->then(function($result) {
            var_dump($result);
       });
```

这比上面的链式调用更加直观。但是还可以更简单，例如：

```php
use Hprose\Client;
use Hprose\Future;

$client = Client::create('http://hprose.com/example/');

$var_dump = Future\wrap('var_dump');
$sum = $client->sum;

$var_dump($sum($sum($sum(1, 2), 3), 4));
```

远程方法可以直接作为一个闭包对象返回，之后可以单独调用。然后在加上用 `wrap` 函数包装的 `var_dump`。

现在代码看上去完全是同步的啦。

这种方式看上去是同步的，但是实际上却可以并行异步执行，尤其是当一个调用的参数依赖于其它几个调用的结果时候，例如：

```php
use Hprose\Client;
use Hprose\Future;

$client = Client::create('http://hprose.com/example/');

$var_dump = Future\wrap('var_dump');
$sum = $client->sum;

$r1 = $sum(1, 3, 5, 7, 9);
$r2 = $sum(2, 4, 6, 8, 10);
$r3 = $sum($r1, $r2);
$var_dump($r1, $r2, $r3);
```

这个程序的运行结果为：

>
```
int(25)
int(30)
int(55)
```
>

该程序虽然是异步执行，但是书写方式却是同步的，不需要写任何回调。

而且这里还有一个好处，`$r1` 和 `$r2` 两个调用的参数之间没有依赖关系，是两个相互独立的调用，因此它们将会并行执行，而 `$r3` 的调用依赖于 `$r1` 和 `$r2`，因此 `$r3` 会等 `$r1` 和 `$r2` 都执行结束后才会执行。也就是说，它不但保证了有依赖关系的调用会根据依赖关系顺序执行，而且对于没有依赖的调用还能保证并行执行。

这是回调方式和链式调用方式都很难做到的，即使可以做到，也会让代码变得晦涩难懂。

这也是 Hprose 2.0 最大的改进之一。