---
layout: post
title: Qt实现LRC歌词的解析
tags:
  - Qt
id: 142
categories:
  - 笔记
date: 2015-06-08 02:16:12
---

网易云音乐的歌词格式共分三种：不含时间标签的纯文本格式、每一句歌词带有一个时间标签的格式，以及每一个字都带有一个时间标签的卡拉OK格式。对于第一种格式，只需把源文本按照换行符`\n`分割开来即可，而第三种格式自己写的程序里用不到所以先不管它，所以这里只讨论第二种格式。

按照[维基百科的说明](http://zh.wikipedia.org/wiki/LRC%E6%A0%BC%E5%BC%8F)，标准的LRC格式歌词，每一行开头都有一个形如`[mm:ss.xx]`形式的时间标签，实际上有的歌词每一行开头可能有多个时间标签，表示这一句可能会在不同时间点重复多次。以此为标准，实现解析的思路如下：

1.  查找第一个形如[mm:ss.xx]的时间标签。如果找不到，就把文本按照换行符简单分割，完成解析；找到的话就记录标签的位置并执行步骤2。
2.  查找下一个时间标签的位置。如果找不到，就把最后一个时间标签之后的全部文本作为最后一句歌词，保存到结果中，执行步骤4；找到的话就记录标签的位置并执行步骤3。
3.  比较当前标签和上一个标签的位置。如果是紧挨的话，表示这两个标签代表同一句歌词，重复步骤2；如果不是紧挨的，就把两个标签之间的文本作为歌词保存到结果中，然后重复步骤2。
4.  把得到的结果按照时间顺序重新排列，完成解析。

<!-- more -->

每一行歌词都可以抽象为一个包含时间和文本的结构体，解析的结果使用`QList`维护，最终代码如下：

```c++
class LyricLine
{
public:
    LyricLine(int time, QString text):time(time), text(text){}

    int time;
    QString text;
};

QList<LyricLine*> mLines;
bool mHasTimer;

bool lyricTimeLessThan(const LyricLine* line1, const LyricLine* line2)
{
    return line1->time < line2->time;
}

bool LyricLoader::processContent(const QString &content)
{
    if (!mLines.isEmpty()) {
        qDeleteAll(mLines);
        mLines.clear();
        mHasTimer = false;
        emit lyricChanged();
    }

    const QRegExp rx("\\[(\\d+):(\\d+(\\.\\d+)?)\\]"); //用来查找时间标签的正则表达式

    // 步骤1
    int pos = rx.indexIn(content);
    if (pos == -1) {
        QStringList list = content.split('\n', QString::SkipEmptyParts);
        foreach (QString line, list)
            mLines.append(new LyricLine(0, line));

        mHasTimer = false;
    }
    else {
        int lastPos;
        QList<int> timeLabels;
        do {
            // 步骤2
            timeLabels << (rx.cap(1).toInt() * 60 + rx.cap(2).toDouble()) * 1000;
            lastPos = pos + rx.matchedLength();
            pos = rx.indexIn(content, lastPos);
            if (pos == -1) {
                QString text = content.mid(lastPos).trimmed();
                foreach (const int& time, timeLabels)
                    mLines.append(new LyricLine(time, text));
                break;
            }
            // 步骤3
            QString text = content.mid(lastPos, pos - lastPos);
            if (!text.isEmpty()) {
                foreach (const int& time, timeLabels)
                    mLines.append(new LyricLine(time, text.trimmed()));
                timeLabels.clear();
            }
        }
        while (true);
        // 步骤4
        qStableSort(mLines.begin(), mLines.end(), lyricTimeLessThan);
        mHasTimer = true;
    }

    if (!mLines.isEmpty()) {
        emit lyricChanged();
        return true;
    }

    return false;
}
```

播放音乐的时候，需要监听播放进度的变化，根据当前的时间点找出对应的歌词显示出来，代码如下：

```c++
int LyricLoader::getLineByPosition(const int &millisec, const int &startPos) const
{
    if (!mHasTimer || mLines.isEmpty())
        return -1;

    int result = qBound(0, startPos, mLines.size());
    while (result < mLines.size()) {
        if (mLines.at(result)->time > millisec)
            break;

        result++;
    }
    return result - 1;
}
```

这里`startPos`是上一行歌词的行号，从它开始查找是为了提高效率。具体显示歌词的话就很简单了，按照具体需求使用`ListView`或者`QLabel`都可以。

