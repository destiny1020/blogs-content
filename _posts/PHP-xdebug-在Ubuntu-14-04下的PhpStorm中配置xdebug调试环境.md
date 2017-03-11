---
title: '[PHP+xdebug] 在Ubuntu 14.04下的PhpStorm中配置xdebug调试环境'
date: 2015-03-27 16:52:00
categories: ['工具, 库与框架', LAMP]
tags: [Ubuntu, LAMP, PHP, xdebug]
---

**在配置过程中参考了一些文章, 中英文的都有.. 但是都不能完整地解决这个问题. 经过一些折腾终于可以调试了, 现记录如下, 希望对后来人有所帮助.**

## 安装xdebug

### 得到本地PHP配置信息

在终端中运行: 

```bash
php -i > outputphp.txt
```

然后将得到的txt文件中的信息拷贝并复制到http://xdebug.org/wizard.php 这个页面提供的一个textarea中. 然后点击下方的Analyze按钮, 它会自动帮你解析你本地的PHP环境信息从而得到你需要下载的xdebug版本和相关配置指令.

<!-- More -->

为了进行下面步骤，还需要安装php5-dev依赖包：

```bash
sudo apt-get install php5-dev
```

### 得到需要下载的版本和相关指令

比如, 我得到的信息如下:

1. 下载 xdebug-2.3.2.tgz (下载地址直接点击生成的链接)
2. 解压缩文件： `tar -xvzf xdebug-2.3.2.tgz`
3. 运行: `cd xdebug-2.3.2`
4. 运行: `phpize (`See the FAQ if you don't have phpize.
	部分输出如下所示:
	Configuring for:
	...
	Zend Module Api No:      20121212
	Zend Extension Api No:   220121212
如果没有以上输出, 那么代表你的phpize有问题. 参考FAQ.

5. 运行: `./configure`
6. 运行: `make`
7. 运行: `sudo cp modules/xdebug.so /usr/lib/php5/20121212`

以上有些步骤也许需要sudo.

### 向php.ini中添加配置项

```bash
sudo vim /etc/php5/cli/php.ini

zend_extension = /usr/lib/php5/20121212/xdebug.so
xdebug.remote_host = 127.0.0.1
xdebug.remote_enable = 1
xdebug.remote_port = 9000
xdebug.remote_handler = dbgp
xdebug.remote_mode = req
```

如非必要, 以上的配置项不需要修改. 之前我就是想当然的将remote_port那一项修改成了我的应用在Server上的端口号, 导致无法调试. 花了好些时间才定位到是这里的问题.

到这里, xdebug就安装成功了. 可以通过php --version命令进行验证:

> PHP 5.5.9-1ubuntu4.7 (cli) (built: Mar 16 2015 20:47:39)  Copyright
> (c) 1997-2014 The PHP Group Zend Engine v2.5.0, Copyright (c)
> 1998-2014 Zend Technologies
>     with Xdebug v2.3.2, Copyright (c) 2002-2015, by Derick Rethans
>     with Zend OPcache v7.0.3, Copyright (c) 1999-2014, by Zend Technologies

可以发现输出中已经存在了Xdebug的信息.

## 安装 Xdebug extension helper
在主流的浏览器上都有xdebug的扩展助手插件, 能够帮助你方便的打开或者关闭调试功能, 为什么需要这个插件, 可以参考[这篇文章](https://confluence.jetbrains.com/display/PhpStorm/Zero-configuration+Web+Application+Debugging+with+Xdebug+and+PhpStorm)中的4, 5, 6小节(是英文的, 有兴趣的同学可以自行查阅)

以Chrome为例, 在这里找到插件的安装地址:
https://chrome.google.com/webstore/detail/xdebug-helper/eadndfjplgieldjbigjakmdgkmoaaaoc?hl=en
如果打不开, 可以参考[这篇文章](http://blog.csdn.net/matraxa/article/details/39836159), 介绍了如何利用插件的ID进行离线下载, 毕竟现在Google的服务全面被墙.....

Xdebug helper的插件ID是: eadndfjplgieldjbigjakmdgkmoaaaoc

安装完毕之后, 打开该插件的options, 设置IDEKey为PhpStorm.

## 配置PhpStorm

终于到最后一步了, 这一步很简单.
就是勾选**Run**菜单下的**Start Listening for PHP Debug Connections**.

然后在你需要调试的地方打个断点, 最后在浏览器中输入PHP脚本的地址就可以了. 注意要启用之前安装的Xdebug Helper.
启用的方法是:

![这里写图片描述](http://img.blog.csdn.net/20150327164848455)

OK, 开心地进行调试吧!!!

![这里写图片描述](http://img.blog.csdn.net/20150327165100053)


### 原理示意图(从xdebug的官网上引用的)
![这里写图片描述](http://xdebug.org/images/docs/dbgp-setup.gif)

### 主要参考文章:
https://confluence.jetbrains.com/display/PhpStorm/Zero-configuration+Web+Application+Debugging+with+Xdebug+and+PhpStorm
http://icephoenix.us/php/how-to-setup-local-php-debugging-with-phpstorm-and-xdebug/
http://xdebug.org/docs/remote#starting





