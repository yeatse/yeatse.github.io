---
layout: post
title: WP Author主题的一些手动调整
id: 181
categories:
  - 笔记
date: 2015-10-18 14:09:41
tags:
  - Ops
---

最近发现了个wordpress主题[Author](https://www.competethemes.com/author/)，装上去还是挺好看的，对移动版适配得还可以，但是一些设置项要激活的话需要购买pro版Orz。不过直接改style.css还是可以的，就是升级之后手动改过的文件都会还原，所以在这里做个记录，万一以后忘了呢。至于为什么我一定要去升级。。。强迫症呗~

首先是文章标题字体的设定，默认字体对于中文来说太大了。在style.css里搜索`post-title`，应该有好几个结果，把它们的`font-size`那项都改成2em。

然后是默认摘要字数的设定，默认为25个字。在functions.php里搜索`ct_author_custom_excerpt_length`函数，把返回值改成75就行。

以上，没啥技术含量，先记着吧。

