---
layout: post
title: Beyond Compare 4 for Mac 无限试用
tags:
  - macOS
id: 229
categories:
  - 笔记
date: 2016-02-18 16:03:47
---

转自[https://www.puteulanus.com/archives/677](https://www.puteulanus.com/archives/677)，略有修改，修复了Source Tree等软件的兼容问题。

终端命令：

```bash
$ cd /Applications/Beyond\ Compare.app/Contents/MacOS/
$ mv BCompare BCompare.real
```

新建BCompare文件，内容：

```bash
#!/bin/bash
rm "/Users/$(whoami)/Library/Application Support/Beyond Compare/registry.dat"
"`dirname "$0"`"/BCompare.real $@
```

保存之后执行：

```bash
$ chmod +x BCompare
```

完。


