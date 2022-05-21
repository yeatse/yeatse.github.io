---
layout: post
title: Jetpack启动时返回register_http_failed错误
id: 201
categories:
  - 笔记
date: 2015-11-08 22:47:46
tags:
  - Tools
---

在iPhone上安装了wordpress的移动客户端，里边有个统计的功能，需要安装Jetpack插件包才能实现：
[![IMG_0448](/assets/images/2015/IMG_0448-360x640.png)](/assets/images/2015/IMG_0448.png)

于是顺着它的意思安装了，结果启动的时候提示我网络连接不上，说我主机有什么地方配置错误了。
[![IMG_0449](/assets/images/2015/IMG_0449-360x640.jpg)](/assets/images/2015/IMG_0449.jpg)

于是google了一下，查到[一篇文章](http://ben.lobaugh.net/blog/49601/quick-fix-for-jetpack-register_http_request_failed)说只要把Jetpack强制设定为使用http（而不是https）连接就好了，试了下果然可行。

具体做法就是在`wp-config.php`中加上这么一行：

```php
define( 'JETPACK_CLIENT__HTTPS', 'NEVER' );
```

至于为什么要这样做，可能是我vps上搭了shadowsocks，占用了443端口的缘故吧。

