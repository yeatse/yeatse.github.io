---
layout: post
title: '有关QObject::deleteLater()的一些事'
tags:
  - Qt
id: 81
categories:
  - 笔记
date: 2015-03-18 22:12:07 +0800
---

最近在做一个文件批量上传的工具，要实现暂停继续、断点续传等功能。利用Qt自带的QtNetwork模块，完成这些需求并没有费多少周章，主要思路就是将文件分块，然后用while循环依次传输。具体实现代码比较复杂，简化了一下大致是这样子的：

```c++
int chunkUpload(UploadItem *item, const qint64 &chunkSize)
{
    Q_ASSERT(item != 0);

    // ...

    QNetworkRequest request;
    QNetworkReply* reply;
    QHttpMultiPart* multiPart = new QHttpMultiPart(QHttpMultiPart::FormDataType);
    QBuffer* buffer = new QBuffer(multiPart);
    QEventLoop loop;
    QTimer timer;

    // ... append fields to multiPart ...

    request.setUrl(UPLOAD_URL);
    reply = networkManager->post(request, multiPart);
    // 保证reply销毁的时候multiPart和buffer也会随之销毁。
    multiPart->setParent(reply);

    // 加入一个消息循环，reply传输完成前进程会一直阻塞在这里；并设置一个定时器，如果传输超时则强制跳出消息循环
    timer.setSingleShot(true);
    timer.setInterval(NetworkTimeout);
    connect(reply, SIGNAL(uploadProgress(qint64,qint64)), &timer, SLOT(start()));
    connect(reply, SIGNAL(downloadProgress(qint64,qint64)), &timer, SLOT(start()));
    connect(&timer, &QTimer::timeout, &loop, &QEventLoop::quit);
    connect(reply, &QNetworkReply::finished, &loop, &QEventLoop::quit);
    timer.start();
    loop.exec();

    reply->deleteLater();
    if (reply->isRunning())
        reply->abort();

    if (reply->error() != QNetworkReply::NoError)
        return -1;

    // ... deal with the reply ...

    return 0;
}

void upload()
{
    while (NOT_FINISHED) {
        chunkUpload(item, CHUNK_SIZE);
    }
}
```

程序编译运行过程很顺利，测试的时候也没发现什么问题。但后来我随手上传了一个1G大小的文件，发现每次文件上传到70%左右的时候程序就崩溃了，小文件就没这个问题。急忙打开任务管理器，这才发现上传文件的时候，程序内存占用会随着上传进度的增加而增加，上传1G文件的时候内存最多会吃到1.5G，这时候程序申请不到更多内存了，我又没做检查，当然就会崩溃掉。

<!-- more -->

限制上传文件大小这种事我是不会做的，毕竟一个上传工具占用内存比PS都高实在不科学。注意到文件上传完成之后内存会立即回到正常值，显然原因并不是我忘记释放内存而是内存释放不及时，这样看来唯一可疑的地方就是上面`chunkUpload`函数里面的`reply->deleteLater()`那一句了吧。于是我写了个方法监听reply的销毁时机，果然每一块上传完成之后reply没有销毁，直到文件全部上传完毕之后才输出一大堆“I'm destroyed...”的信息。

根据Qt文档的说明，`QObject::deleteLater()`并没有将对象立即销毁，而是向主消息循环发送了一个event，下一次主消息循环收到这个event之后才会销毁对象。我在这里使用deleteLater只是因为Qt文档里推荐这么做而已，其他并没多想。是这样的话一切都说得通了，因为`chunkUpload`函数是在一个while循环里，程序还没来得及处理这个event就立即进行下一块传输了，传输过程中生成的`QNetworkReply`以及它关联的`QBuffer`、`QHttpMultiPart`当然也就来不及删除了。崩溃原因找到了。你不就是来不及处理销毁对象的event嘛，手动让你处理下不就行了？于是修改`upload`函数代码如下：

```c++
void upload()
{
    while (NOT_FINISHED) {
        chunkUpload(item, CHUNK_SIZE);
        qApp->processEvents(); // 让主程序把消息队列中的QEvent处理完
    }
}
```

编译、运行，内存占用依然没有改变，看样子加这一行用处不大。再次查询`QObject::deleteLater()`的文档，发现这样一句话：

> ...for the object to be deleted, the control must return to the event loop from which deleteLater() was called.

这么说来，`deleteLater`销毁`QObject`的唯一时机就是程序返回主消息循环以后了呢。无奈只能放弃`deleteLater`。考虑到每次return前都要手动delete亦或是使用goto语句实在都不够优雅，所以利用Qt自带的`QScopedPointer`，修改`chunkUpload`函数如下：

```c++
int chunkUpload(UploadItem *item, const qint64 &chunkSize)
{
    Q_ASSERT(item != 0);

    // ...

    QNetworkRequest request;
    QScopedPointer&lt;QNetworkReply> reply;
    QHttpMultiPart* multiPart = new QHttpMultiPart(QHttpMultiPart::FormDataType);
    QBuffer* buffer = new QBuffer(multiPart);
    QEventLoop loop;
    QTimer timer;

    // ... append fields to multiPart ...

    request.setUrl(UPLOAD_URL);
    reply.reset(networkManager->post(request, multiPart));
    multiPart->setParent(reply.data());

    // ... event loop ...

    if (reply->isRunning())
        reply->abort();

    if (reply->error() != QNetworkReply::NoError)
        return -1;

    // ... deal with the reply ...

    return 0;
}
```

问题解决。

