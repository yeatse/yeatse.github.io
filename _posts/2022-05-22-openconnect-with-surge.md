---
layout: post
title: 'Apple Silicon 电脑上的 VPN 与 Surge 共存方案'
categories:
  - 笔记
date: 2022-05-22 00:31:00 +0800
tags:
  - Tools
---

时隔两年的又一次长期居家办公，不免又要长时间通过 VPN 访问公司内网，我个人的需求很简单，只有一条：

- 电脑上需要开启 Surge 无障碍上网，但同时对于公司内网域名要走 VPN，两者互不影响

之前参考 [这篇博客](https://blog.indigo.codes/2020/04/24/home-network-deployment/) 已经完成了整体的配置，具体思路就是使用 Surge 把公司内网流量转发到 Docker 容器，容器内部开启 OpenConnect 连 VPN。但是最近家里购置了一台 Mac Studio，Apple Silicon 什么都好，就是在一些特殊场景下兼容性堪忧，网络配置花了很久，所以在这里做个记录。

## Surge 规则配置

```
[Proxy]
🚇 VPN = snell, localhost, 8388, psk=123, obfs=http, version=3

[Proxy Group]
💼 公司内网 = select, DIRECT, 🚇 VPN

[Rule]
AND,((DOMAIN-KEYWORD,内网域名), (NOT,((PROCESS-NAME,com.docker.vpnkit)))),💼 公司内网
```

这里有几个细节点：
- 对于公司内网域名，可以根据情况手动选择是直连还是走 VPN，这样在家里或者公司都可以复用同一套配置
- 规则这里一定要排除掉 com.docker.vpnkit 进程的请求，否则在 Surge 的增强模式下会出现死循环的情况

## 本地 VPN 服务配置

参考上面的博客，按照 [openconnect-snell](https://hub.docker.com/r/dianqk/openconnect-snell) 里的说明起一个 Docker 实例就可以了——至少曾经可以这样。但是要想在 M1 上把实例跑起来，必须要重新发布一版 arm64 的镜像才行，具体来说就是要把原来的 Dockerfile 中引用的资源全部替换成 arm64 的，改动点见 [diff](https://github.com/Yeatse/openconnect-snell/commit/d9c6bed909c45fd3746ceebba4558b5be5276960)。

这里有一个坑点，就是 snell-server 依赖了 glibc，但是网络上根本找不到适配 arm64 的 alpine linux 的较新的 glibc 发布包（定语这么多找不到也正常），最终只能借助 [docker-glibc-builder](https://github.com/sgerrand/docker-glibc-builder.git) 自己在本机编译了一份，然后手动拷贝到镜像中。另外就是要解决各种系统库找不到的问题，alpine 下报错信息少得可怜，解决这个问题花了好几个晚上。

总而言之，镜像发布之后，开启 VPN 就比较方便了，新建一个名字叫 `runvpn.sh` 的脚本，内容如下：

```sh
docker run -d --privileged -p 8388:8388 -p 8388:8388/udp -e PSK="123" -e OC_HOST="..." -e OC_AUTH_GROUP="common-vpn" -e OC_PASSWD="..." -e OC_AUTH_CODE="$1" -e OC_USER="..." --name=openconnect-snell yeatse/openconnect-snell
```

使用时只需要执行以下命令即可：

```
sh ./runvpn.sh <动态验证码>
```