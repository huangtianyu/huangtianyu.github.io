---
title: 通过adb shell命令查看当前交互的Activity名称
date: 2018-01-08 20:46:09
categories: Android
tags: adb查看交互Activity名称
---
#### 需求
在做android逆向的时候，有时候会需要知道当前的界面处于哪个Activity，这时候就可以使用adb shell命令来查看当前与用户交互的Activity名称。
#### 原文地址
先给出原文地址：
http://stackoverflow.com/questions/11549366/print-the-current-back-stack-in-the-log/26424943#26424943
#### 方法
有如下几种方法可以获取：
##### 方法一：
``` 
adb shell dumpsys activity activities | sed -En -e '/Running activities/,/Run #0/p'  
```
![方法一](http://img.blog.csdn.net/20170110121929007?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
其中TaskRecord即为查询到的记录。其中com.sina.weibo为包名，.VisitorMainTabActivity为对应的Activity名称。
##### 方法二：
```
adb shell dumpsys activity | grep -i run  
```
查询结果为：
![方法二](http://img.blog.csdn.net/20170110122114759?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
##### 方法三：
```
adb shell dumpsys activity | grep "mFoc"  
```
查询结果为：
![方法三](http://img.blog.csdn.net/20170110122247214?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
其中mFocusedActivity就是当前和用户交互的Activity。
如果在Windows下使用时，则先通过adb shell进入到adb shell里，然后把adb shell去了，再将余下的复制到$后面进行执行，例如：
![在shell下执行grep语句](http://img.blog.csdn.net/20170110122816847?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHR5MTA1MzI0MDEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
这样就不会提示：“grep”不是内部或外部命令，也不是可运行查询了