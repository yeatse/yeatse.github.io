---
layout: post
title: 让Qt的无边框窗口支持拖拽、Aero Snap、窗口阴影等特性
tags:
  - Qt
  - Windows
id: 55
categories:
  - 笔记
date: 2015-03-01 18:48:23 +0800
---

环境：Desktop Qt 5.4.1 MSVC2013 32bit

需要的库：`dwmapi.lib` 、`user32.lib`

需要头文件：`<dwmapi.h>` 、`<windowsx.h>`

在要处理的`QWidget` 构造函数中，添加以下两行：

```c++
setWindowFlags(Qt::Window | Qt::FramelessWindowHint);
SetWidgetBorderless(this);
```

`SetWidgetBorderless`的实现如下：

```c++
void SetWidgetBorderless(const QWidget *widget)
{
#ifdef Q_OS_WIN
    HWND hwnd = reinterpret_cast<HWND>(widget->winId());

    const LONG style = ( WS_POPUP | WS_CAPTION | WS_SYSMENU | WS_MINIMIZEBOX | WS_MAXIMIZEBOX | WS_THICKFRAME | WS_CLIPCHILDREN );
    SetWindowLongPtr(hwnd, GWL_STYLE, style);

    const MARGINS shadow = {1, 1, 1, 1};
    DwmExtendFrameIntoClientArea(hwnd, &shadow);

    SetWindowPos(hwnd, 0, 0, 0, 0, 0, SWP_FRAMECHANGED | SWP_NOMOVE | SWP_NOSIZE);
#endif
}
```

这个函数的作用是给无边框窗口加上阴影、Aero Snap以及其他动画特效。

<!-- more -->

这时窗口还无法手动更改大小，需要更改的话，需要自己实现一个`QAbstractNativeEventFilter` 类，内容如下：

```c++
class NativeEventFilter : public QAbstractNativeEventFilter
{
public:
    bool nativeEventFilter(const QByteArray &eventType, void *message, long *result) Q_DECL_OVERRIDE
    {
#ifdef Q_OS_WIN
        if (eventType != "windows_generic_MSG")
            return false;

        MSG* msg = static_cast<MSG*>(message);
        QWidget* widget = QWidget::find(reinterpret_cast<WId>(msg->hwnd));
        if (!widget)
            return false;

        switch (msg->message) {
        case WM_NCCALCSIZE: {
            *result = 0;
            return true;
        }

        case WM_NCHITTEST: {
            const LONG borderWidth = 9;
            RECT winrect;
            GetWindowRect(msg->hwnd, &winrect);
            long x = GET_X_LPARAM(msg->lParam);
            long y = GET_Y_LPARAM(msg->lParam);

            // bottom left
            if (x >= winrect.left && x < winrect.left + borderWidth &&
                    y < winrect.bottom && y >= winrect.bottom - borderWidth)
            {
                *result = HTBOTTOMLEFT;
                return true;
            }

            // bottom right
            if (x < winrect.right && x >= winrect.right - borderWidth &&
                    y < winrect.bottom && y >= winrect.bottom - borderWidth)
            {
                *result = HTBOTTOMRIGHT;
                return true;
            }

            // top left
            if (x >= winrect.left && x < winrect.left + borderWidth &&
                    y >= winrect.top && y < winrect.top + borderWidth)
            {
                *result = HTTOPLEFT;
                return true;
            }

            // top right
            if (x < winrect.right && x >= winrect.right - borderWidth &&
                    y >= winrect.top && y < winrect.top + borderWidth)
            {
                *result = HTTOPRIGHT;
                return true;
            }

            // left
            if (x >= winrect.left && x < winrect.left + borderWidth)
            {
                *result = HTLEFT;
                return true;
            }

            // right
            if (x < winrect.right && x >= winrect.right - borderWidth)
            {
                *result = HTRIGHT;
                return true;
            }

            // bottom
            if (y < winrect.bottom && y >= winrect.bottom - borderWidth)
            {
                *result = HTBOTTOM;
                return true;
            }

            // top
            if (y >= winrect.top && y < winrect.top + borderWidth)
            {
                *result = HTTOP;
                return true;
            }

            return false;
        }
        default:
            break;
        }

        return false;
#else
        return false;
#endif
    }
};
```

然后在窗口创建之前，使用`QApplication::installNativeEventFilter` 方法把监听器注册给主程序。

要手动移动窗口位置的话，还要重截`QWidget::mousePressEvent` 方法：

```c++
void MyWidget::mousePressEvent(QMouseEvent *ev)
{
    QWidget::mousePressEvent(ev);
    if (!ev->isAccepted()) {
#ifdef Q_OS_WIN
        ReleaseCapture();
        SendMessage(reinterpret_cast<HWND>(winId()), WM_SYSCOMMAND, SC_MOVE + HTCAPTION, 0);
    }
#endif
}
```

在实际操作时，有时还需要添加最大化和关闭按扭，这时正常调用`QWidget::showMaximized()`和`QWidget::close()` 等Qt自带方法即可。

最终实现效果大概是这样：

[![frameless_window_preview](/assets/images/2015/frameless_window_preview-300x138.png)](/assets/images/2015/frameless_window_preview.png)

缩放动画、阴影什么的和Windows本地窗口一样。

参考链接：[https://github.com/deimos1877/BorderlessWindow](https://github.com/deimos1877/BorderlessWindow)


