Hprose 服务器端提供了几个事件，它们分别是：

* `onBeforeInvoke`
* `onAfterInvoke`
* `onSendError`

这三个事件所有的 Hprose 服务器都支持。

* `onSendHeader`

这个事件仅 HTTP 服务器支持。

* `onAccept`
* `onClose`

这两个事件 Socket 和 WebSocket 服务器支持。

这些事件是以属性方式提供的，只需要将事件函数赋值给这些属性即可。

例如：

```php
$server->onBeforeInvoke = function($name, $args, $byref, \stdClass $context) {
    ...
}
$server->onAfterInvoke = function($name, $args, $byref, $result, \stdClass $context) {
    ...
}
$server->onSendError = function($error, \stdClass $context) {
    ...
}
```

对于 `onBeforeInvoke`，`onAfterInvoke` 和 `onSendError` 这三个事件属性的事件处理函数允许有返回值。

`onBeforeInvoke` 和 `onAfterInvoke` 可以返回一个 `promise` 对象，当该 `promise` 对象变为失败（`rejected`）状态时，将会返回失败原因作为返回给客户端的错误信息。

`onBeforeInvoke`，`onAfterInvoke` 和 `onSendError` 都可以直接返回一个 `Error` 实例对象来作为返回给客户端的错误信息。

`onBeforeInvoke`，`onAfterInvoke` 和 `onSendError` 还可以通过抛出异常来返回错误信息给客户端。

## `onBeforeInvoke` 事件

该事件在调用执行前触发，该事件的处理函数形式为：

```php
function($name, &$args, $byref, \stdClass $context) { ... }
```

参数 `$name` 是服务函数/方法名。
参数 `$args` 是调用的参数数组，可以声明为引用参数。
参数 `$byref` 表示是否是引用参数传递。
参数 `$context` 是该调用的上下文参数。

如果在该事件中抛出异常、返回错误对象、或者返回一个失败（`rejected`）状态的 `promise` 对象。则不再执行服务函数/方法。

## `onAfterInvoke` 事件

该事件在调用执行后触发，该事件的处理函数形式为：

```php
function($name, &$args, $byref, &$result, \stdClass $context) { ... }
```

参数 `$name` 是服务函数/方法名。
参数 `$args` 是调用的参数数组，可以声明为引用参数。
参数 `$byref` 表示是否是引用参数传递。
参数 `$result` 是调用执行的结果，可以声明为引用参数。
参数 `$context` 是该调用的上下文参数。

如果在该事件中抛出异常、返回错误对象、或者返回一个失败（`rejected`）状态的 `promise` 对象。则不再返回结果 `$result`，而是将错误信息返回给客户端。

## `onSendError` 事件

该事件在服务端发生错误时触发，该事件的处理函数形式为：

```php
function(&$error, \stdClass $context) { ... }
```

如果在该事件中抛出异常、返回错误对象。则该错误会替代原来的错误信息返回给客户端。

$error 参数可以声明为引用参数，在事件中可以对 $error 进行修改。

当服务器与客户端之间发生网络中断性的错误时，仍然会触发该事件，但是不会有错误信息发送给客户端。

## `onSendHeader` 事件

该事件在服务器发送 HTTP 头时触发，该事件的处理函数形式为：

```php
function(\stdClass $context) { ... }
```

如果在该事件中抛出异常，则不再执行后序操作，直接返回异常信息给客户端。

## `onAccept` 事件

该事件在 Socket 或 WebSocket 服务器接受客户端连接时触发，该事件的处理函数形式为：

```php
function(\stdClass $context) { ... }
```

如果在该事件中抛出异常，则会断开跟该客户端的连接。

## `onClose` 事件

该事件在 Socket 或 WebSocket 服务器跟客户端之间的连接关闭时触发，该事件的处理函数形式为：

```php
function(\stdClass $context) { ... }
```

该事件中抛出异常不会对服务器和客户端有任何影响。

## `onError` 事件

该事件仅被 `Hprose\Socket\Server` 所支持，其它服务器不支持，该事件在服务器与客户端发生通讯错误，无法将错误发送给客户端时触发。该事件的处理函数形式为：

```php
function($error, \stdClass $context) { ... }
```

该事件中抛出异常不会对服务器和客户端有任何影响。
