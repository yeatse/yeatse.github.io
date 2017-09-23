---
title: 关于Android VideoView导致的内存泄漏的问题
tags:
  - Android
  - Java
  - VideoView
id: 186
categories:
  - 笔记
date: 2015-11-01 02:32:35
---

最近在做的项目里要实现在Android中播放视频这么一个需求。功能本身并不复杂，因为赶时间图省事，没有用Android底层的MediaPlayer API，直接用了谷歌封装好的`VideoView`组件：

```xhtml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="fill_parent"
                android:layout_height="fill_parent"
                android:background="@android:color/black">

    <RelativeLayout
        android:id="@+id/videoContainer"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent">

        <VideoView
            android:id="@+id/videoView"
            android:layout_width="fill_parent"
            android:layout_height="fill_parent"
            android:layout_centerInParent="true"/>

        <View
            android:id="@+id/placeHolder"
            android:layout_width="fill_parent"
            android:layout_height="fill_parent"
            android:background="@android:color/black"/>

    </RelativeLayout>

    <!-- ...... -->

</RelativeLayout>
```

像这样在XML里声明一下，然后调用`setVideoPath`方法传入视频路径，`start`、`pause`和`stopPlayback`方法控制播放，基本功能就做好啦。可喜可贺，可喜可贺。

但是在实机运行时，我发现即使关闭了含有VideoView的Activity，它申请到的内存也不会被释放。多次进入这个页面再关闭的话，程序占用的内存越来越多，用Android Studio自带的Java Heap Dump功能可以看到这些Activity都没有被回收掉，显然在某些地方出现了内存泄漏。经过一番仔细排查，问题定位到了`VideoView.setVideoPath`方法上，屏蔽掉这一行，Activity就可以正常被回收；把这一行加回来，问题又出现了。

<!-- more -->

这个方法是用来给VideoView传视频url的，当然不能删掉了事，于是开google，看到有人在天国的google code上[讨论过这个问题](https://code.google.com/p/android/issues/detail?id=152173)。大体意思是说，VideoView内部的`AudioManager`会对Activity持有一个强引用，而`AudioManager`的生命周期比较长，导致这个Activity始终无法被回收，这个bug直到2015年5月才被谷歌修复。因为Activity是通过VideoView的构造函数传给AudioManager的，所以回复里有人提出了一个workaround：不要用XML声明VideoView，改用代码动态创建，创建的时候把全局的Context传给VideoView，而不是Activity本身。试了一下果然可行：

```java
videoView = new VideoView(getApplicationContext());
RelativeLayout.LayoutParams params = new RelativeLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT);
params.addRule(RelativeLayout.CENTER_IN_PARENT);
videoView.setLayoutParams(params);
((RelativeLayout)findViewById(R.id.videoContainer)).addView(videoView, 0);
```

这样做到底还是感觉不太优雅，且不说动态创建和XML声明混到一起后期难维护，直接传全局Context总觉得程序会在哪个魔改rom上就崩掉了。考虑到内存泄漏的原因出在AudioManager上，所以只针对它作处理就足够了。于是把VideoView放回XML里，重写`Activity.attachBaseContext`方法：

```java
@Override
protected void attachBaseContext(Context newBase)
{
    super.attachBaseContext(new ContextWrapper(newBase)
    {
        @Override
        public Object getSystemService(String name)
        {
            if (Context.AUDIO_SERVICE.equals(name))
                return getApplicationContext().getSystemService(name);

            return super.getSystemService(name);
        }
    });
}
```

重新调试，为了方便跟踪回收事件我在`finalize`方法中加了一行log，关闭Activity之后调用`System.gc()`，果然有回收的log输出，问题解决。

