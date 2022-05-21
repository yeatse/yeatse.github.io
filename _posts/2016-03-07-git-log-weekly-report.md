---
layout: post
title: 使用git log命令自动生成周报
tags:
  - git
id: 278
categories:
  - 笔记
date: 2016-03-07 20:03:56 +0800
---

*2016-3-15 update: 项目已提交至GitHub，地址<https://github.com/yeatse/git-log-weekly-report/>*

---

最近公司主管突然让我们每周末发周报了，具体就是总结一下上周都干了什么之类的。从小学开始就经常不交作业的我心里是一百个不愿意，但是这个是跟绩效挂钩的，人总不能跟钱过不去嘛，没办法只能养成习惯去写周记了。

当然，作为一个懒惰的程序员，每周让我拿出半个小时的时间去整理周报这种事我是不会去做的。好在现在的项目是用git作为版本控制工具的，用git log命令的话，只要一行就可以在屏幕输出日志内容了：

```bash
$ git log --author="yeatse" --format="%cd : %s" --since=last.Monday --reverse --no-merges --date=format:'%F %T'
```

具体的参数意义在[git-log Documentation](https://git-scm.com/docs/git-log)上可以找到(其中date format选项[需要升级git版本到v2.6.0rc以上才有效](https://github.com/git/git/commit/aa1462cc3d3b0c4c8ad6a60aaf31e0f3a424162d))。输出的结果大概是这个样子：

```
2016-02-29 22:04:54 : Update elctron-prebuilt version. Fix #11.
2016-02-29 22:49:51 : Optimize user's avatar display in Linux.
2016-02-29 23:03:20 : Disable zooming in the app.
2016-03-01 00:23:32 : Introduce travis CI.
2016-03-01 00:28:14 : Fix tar-all chmod.
```

<!-- more -->

不过即使这样，每次还要打这么一长串命令，而且输出的结果需要手动编辑一下才能看，实在不够偷懒，最后还是在[GitHub](https://github.com/pkyeck/git-log-by-day)上找到了一个合适的轮子，自己魔改了一下，大致符合了自己的需求：

```bash
# Generates git changelog grouped by day and output it to file
#
# optional parameters
# -a    to filter by author
# -s    to select start date
# -e    to select end date
# -o    to save it to file
# -r    to specify the repository path

NEXT=$(date +%F)
SINCE="last.Monday"
UNTIL=$NEXT
AUTHOR=$(git config user.email)
OUTPUT="$(date +%F).log"

while getopts "a:s:e:o:r:" arg
do
  case $arg in
    a)
      AUTHOR=$OPTARG
      ;;
    s)
      SINCE=$OPTARG
      ;;
    e)
      UNTIL=$OPTARG
      ;;
    o)
      OUTPUT=$OPTARG
      ;;
    r)
      REPO=$OPTARG
      ;;
    ?)
      echo "unknown argument"
      exit 1
      ;;
  esac
done

(
git -C "${REPO}" log --author="${AUTHOR}" --since="${SINCE}" --until="${UNTIL}" --format="%cd" --date=short | sort -u | while read DATE ; do
  GIT_PAGER=$(git -C "${REPO}" log --no-merges --reverse --format="* %s" --since="${DATE} 00:00:00" --until="${DATE} 23:59:59" --author="${AUTHOR}")
  if [ ! -z "$GIT_PAGER" ]
  then
    echo "[${DATE}]"
    echo "${GIT_PAGER}"
    echo
  fi
done
) > $OUTPUT

echo "log is written to ${OUTPUT}"
```

使用方法也很简单，`chmod +x`之后直接执行脚本，日志就会输出到`yyyy-MM-dd.log`文件里啦。
日志默认是以当前git config的邮箱来过滤作者的，具体的可选参数列表在脚本开头可以找到。拿来试了GitHub上一个项目，结果如下：

```
[2016-02-29]
* Update elctron-prebuilt version. Fix #11.
* Optimize user's avatar display in Linux.
* Disable zooming in the app.

[2016-03-01]
* Introduce travis CI.
* Fix tar-all chmod.
* Introduce gitter.
* Update docs for the new release.
* Fix travis deployment files.
* Fix wrong regex which might cause stickers not showing in group chat.
* Force travis work.
* Update travis config and wrong regex.

[2016-03-04]
* Remove unnecessary dev tools.
* Take control of quit event. Press Cmd+Q to quit, and Cmd+W to hide window.

[2016-03-05]
* Modify implementation of listening message change, which caused scrolling jank. Fix #66.
* Revert <q>修正消息统计</q>
* Take control over reload. Useful when session timeout and needs refresh.
* Improve experience on the Linux platform. Specifically fix wrong app icon, add tray icon, add menu and override shortcuts.
```

稍微润色下就可以交差啦，好评好评。

