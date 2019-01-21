# 普通 Socket 服务器

该服务器实现是基于 PHP 内置的 Stream 实现的，它支持以下属性和方法：

## settings 属性

该属性的设置值为即 `stream_context_create` 的 `$options` 参数的值。其具体设置可以参见：[[PHP 官方手册——上下文（Context）选项和参数|http://www.php.net/manual/zh/context.php]] ，当使用 TCP 服务器时，可以参见 [[套接字上下文选项|http://php.net/manual/zh/context.socket.php]] 进行设置，当使用 SSL 服务器时，可以参见 [[SSL 上下文选项|http://php.net/manual/zh/context.ssl.php]] 进行设置。

## set 方法

用于设置 `settings` 属性。多次设置可以被合并到 `settings` 属性中。

## noDelay 属性

设置为 `true` 后, TCP 连接发送数据时，会关闭 Nagle 合并算法，立即发往客户端连接。默认值即为 `true`。

## isNoDelay 方法

获取 `noDelay` 的属性值。

## setNoDelay 方法

设置 `noDelay` 的属性值。

## keepAlive 属性

开启 Socket 的 keepAlive 监测机制。默认为 `true`。

## isKeepAlive 方法

获取 `keepAlive` 的属性值。

## setKeepAlive 方法

设置 `keepAlive` 的属性值。

## readBuffer 属性

该属性表示读取缓冲区大小，默认值为 8192。

## getReadBuffer 方法

获取 `readBuffer` 的属性值。

## setReadBuffer 方法

设置 `readBuffer` 的属性值。

## writeBuffer 属性

该属性表示写入缓冲区大小，默认值为 8192。

## getWriteBuffer 方法

获取 `writeBuffer` 的属性值。

## setWriteBuffer 方法

设置 `writeBuffer` 的属性值。

## defer 方法

```php
function $server->defer(callable $callback);
```

延后执行一个 PHP 函数。底层会在 IO 事件循环完成后执行此函数。此方法的目的是为了让一些 PHP 代码延后执行，程序优先处理 IO 事件。

## after 方法

```php
function $server->after($delay, callable $callback);
```

在指定的时间后执行函数 `$callback`。

`after` 方法是一个一次性定时器，执行完成后就会销毁。

`$delay` 指定延时时间，单位为毫秒。

`$callback` 为时间到期后所执行的函数。`$callback` 函数不接受任何参数。

## tick 方法

```php
function $server->tick($delay, callable $callback);
```

在指定的时间后周期性的执行函数 `$callback`。

`tick` 方法是一个周期性定时器，每次执行完成后，会重新计时。

`$delay` 指定延时时间，单位为毫秒。

`$callback` 为时间到期后所执行的函数。`$callback` 函数不接受任何参数。

## clear 方法

```php
function $server->clear($id);
```

用于清除 `after` 和 `tick` 计时器。参数 `$id` 为 `after` 和 `tick` 方法的返回值。

## addListener 方法

用于添加新的监听地址。新增的监听地址必须为同一类型的。

# Swoole 的 Socket 服务器

该服务器实现是基于 `swoole_server` 实现的，它支持以下属性和方法：

## server 属性

只读属性，它是底层的 `swoole_server` 对象。你可以通过它来调用 swoole 服务器的功能。

## settings 属性

用于设置 swoole 服务器运行时的各项参数。具体有哪些设置，可参见 swoole 的官方文档，不过需要注意，有一些关于协议解析的选项参数不要设置。

## set 方法

用于设置 `settings` 的属性值。多次设置可以合并。在服务器启动之后，该方法就不能再调用了。

## noDelay 属性

设置为 `true` 后, TCP 连接发送数据时，会关闭 Nagle 合并算法，立即发往客户端连接。默认值即为 `true`。

## isNoDelay 方法

获取 `noDelay` 的属性值。

## setNoDelay 方法

设置 `noDelay` 的属性值。

## on 方法

用于设置 swoole 的服务事件。

## addListener 方法

添加新的监听地址，添加的地址必须为相同的类型。

## listen 方法

添加新的监听地址，并返回监听的服务端口对象，在该对象上进行设置后，可以实现不同类型服务的监听。