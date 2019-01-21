Hprose for PHP 有一个 pecl 扩展：https://github.com/hprose/hprose-pecl

该扩展包含了 Hprose 序列化和反序列化部分的 C 语言实现，安装它之后可以有效的提高 Hprose 的性能。但是它并不包括 Hprose 远程调用服务器和客户端的实现，因此，单纯安装它并不能代替 Hprose for PHP。

## Windows 安装方式

可以直接从 pecl 官网：https://pecl.php.net/package/hprose/1.6.5/windows 下载你所需要的 Windows 版本的 dll，安装即可。

## Linux 安装方式

可以直接通过：

```
pecl install hprose
```

方式安装，也可以下载源码之后，使用 phpize 安装。

## Mac 安装方式

可以直接通过：

```
brew install phpXX-hprose
```

来安装。其中 XX 表示 PHP 的版本号。

也可以通过 pecl 命令安装，或者下载源码之后，通过 phpize 方式安装。