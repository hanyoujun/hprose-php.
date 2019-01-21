# 普通 HTTP 服务器

## crossDomain 属性

该属性用于设置是否允许浏览器客户端跨域调用本服务。默认值为 `false`。当设置为 `true` 时，自动开启 CORS 跨域支持。当配合 `addAccessControlAllowOrigin` 和 `removeAccessControlAllowOrigin` 这两个方法时，还可以做细粒度的跨域设置。

## isCrossDomainEnabled 方法

获取 `crossDomain` 的属性值。

## setCrossDomainEnabled 方法

设置 `crossDomain` 的属性值。

## p3p 属性

该属性用于设置是否允许 IE 浏览器跨域设置 Cookie。默认为 `false`。

## isP3PEnabled 方法

获取 `p3p` 的属性值。

## setP3PEnabled 方法

设置 `p3p` 的属性值。

## get 属性

该属性用于设置是否接受 GET 请求。默认为 `true`。如果你不希望用户使用浏览器直接浏览器服务器函数列表，你可以将该属性设置为 `false`。

## isGetEnabled 方法

获取 `get` 的属性值。

## setGetEnabled 方法

设置 `get` 的属性值。

## addAccessControlAllowOrigin 方法

添加允许跨域的地址。

## removeAccessControlAllowOrigin 方法

删除允许跨域的地址。

# Swoole 的 HTTP 服务器

Swoole 的 HTTP 服务器出了包含普通服务器提供的上面那些设置和方法以外，还有以下几个特殊属性和方法：

## server 属性

只读属性，它是底层的 `swoole_http_server` 对象。你可以通过它来调用 swoole 服务器的功能。

## settings 属性

用于设置 swoole 服务器运行时的各项参数。具体有哪些设置，可参见 swoole 的官方文档，不过需要注意，有一些关于协议解析的选项参数不要设置。

## set 方法

用于设置 `settings` 的属性值。多次设置可以合并。在服务器启动之后，该方法就不能再调用了。

## on 方法

用于设置 swoole 的服务事件。

## addListener 方法

添加新的监听地址，添加的地址必须为相同的类型。

## listen 方法

添加新的监听地址，并返回监听的服务端口对象，在该对象上进行设置后，可以实现不同类型服务的监听。

# Swoole 的 WebSocket 服务器

Swoole 的 WebSocket 服务器跟 Swoole 的 HTTP 服务器所包含的属性和方法是一样的。而且 Swoole 的 WebSocket 服务器是双料服务器，同时支持 Web Socket 通讯和 HTTP 通讯，所以不论是 WebSocket 客户端还是 HTTP 客户端都可以访问该服务器。

## server 属性

只读属性，它是底层的 `swoole_websocket_server` 对象。你可以通过它来调用 swoole 服务器的功能。
