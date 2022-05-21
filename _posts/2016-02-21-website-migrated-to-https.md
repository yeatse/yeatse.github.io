---
layout: post
title: 本站已迁移至https
tags:
  - Tools
id: 254
categories:
  - 笔记
date: 2016-02-21 23:03:33
---

使用的是`Let's Encrypt`这个免费的证书签发服务，按照[这里](https://imququ.com/post/letsencrypt-certificate.html)的教程一步步照着来，很快就完成了。

迁移过程总体来说比较顺利，只是遇到了两个不大不小的坑。一个是域名的跳转问题，迁移完成以后对于所有`http`的请求都需要返回`https`开头的地址，同时因为`Let's Encrypt`的证书不支持wildcard域名，所以还要把域名都301跳转到证书里包含的域名上，不然浏览器会弹证书错误。研究了几个小时，找到了一个比较完美的方案，`nginx.conf`具体内容大概像这样：

```bash
server
{
    listen 80 default_server;
    server_name _;
    return 301 https://yeatse.com$request_uri;
}

server
{
    listen 443 ssl;
    server_name yeatse.com;
    # ...
}
```

另外一个坑是之前文章里的图片都是`http`开头的，因为`WordPress`的图片都是以绝对地址插入的。这个倒是没什么好办法，手动一张张改吧，好在我比较懒，之前写的文章不多。

迁移完成之后，又按照[《本博客 Nginx 配置之安全篇》](https://imququ.com/post/my-nginx-conf-for-security.html)和[《本博客 Nginx 配置之性能篇》](https://imququ.com/post/my-nginx-conf-for-wpo.html)两篇文章的说明修改了下nginx配置，之后也到[https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/)测试了下，也拿到了一个`A+`，洋洋得意中。。。

[![screenshot](/assets/images/2016/捕获-640x386.png)](/assets/images/2016/捕获.png)
