---
layout: post
title: 在Qt工程中使用quazip库
tags:
  - Qt
  - Windows
id: 38
categories:
  - 笔记
date: 2015-02-28 18:34:55 +0800
---

环境：Desktop Qt 5.4.1 MSVC2013 32bit

1.  编译zlib库
    打开`Visual Studio Tools\VS2013 x86 本机工具命令提示`，转到zlib-1.2.8目录，执行命令：

	```bash
	nmake -f win32\Makefile.msc
	```

	得到zlib1.dll和zlib.lib文件。

2.  编译quazip库
    打开`Qt 5.4 32-bit for Desktop(MSVC 2013)`，首先执行`C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\vcvarsall.bat`文件配置好环境，然后转到`quazip-0.7.1\quazip`目录。执行命令：

	```bash
	qmake LIBS+=-L..\..\zlib-1.2.8 LIBS+=-lzlib INCLUDEPATH+=..\..\zlib-1.2.8
	nmake
	```

	得到`quazip.dll`和`quazip.lib`文件。

3.  引用quazip库
    在工程目录下新建`zlib`和`quazip`文件夹，把`zlib.h`和`zconf.h`复制到`zlib`目录，把quazip源码中所有.h文件以及第2步获得的库文件复制到`quazip`目录，然后在.pro文件中添加以下命令

	```bash
	win32 {
		INCLUDEPATH += $$PWD\zlib $$PWD\quazip
			LIBS += -L$$PWD\quazip -lquazip
	}
    ```

之后可按文档描述正常使用。

