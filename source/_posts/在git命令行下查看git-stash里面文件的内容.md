---
title: ' 在git命令行下查看git stash里面文件的内容'
date: 2018-01-08 20:51:31
categories: Git
tags: 查看git bash内容
---
#### 需求
在使用git的时候往往会保存一些东西，在保存的时候使用的就是git stash，强大的git使得保存修改和恢复修改变的很容易，但有时候时间久了不记得stash里面的内容是什么了。
#### 原英文地址
通过在stackflow里面查找，找到了一个好的方法。其网址是：
http://stackoverflow.com/questions/10725729/git-see-whats-in-a-stash-without-applying-stash 
#### 方法
当利用git stash pop弹出来会有些耗费时间，这时可以使用git stash show来查看stash里面保存的内容。
在git bash上可以使用git --help stash来查看git stash命令的用法，当在stash后加show时，官方给出的介绍如下：

show stash

Show the changes recorded in the stash as a diff between the stashed state and its original parent. When no <stash> is given, shows the latest one. By default, the command shows the diffstat, but it will accept any format known to git diff(e.g., git stash show -p stash@{1} to view the second most recent stash in patch form). You can use stash.showStat and/or stash.showPatch config variables to change the default behavior.

翻译如下：显示修改在stash状态与原版本之间的不同变化。当没有<stash>给定时，显示最新stash的变化。默认情况下，命令显示diffstat，它可以接受任何已知的git diff格式（例如，git stash show -p stash@{1}是查看第二最近stash的变化）。你可以使用stash.showstat和/或stash.showpatch配置变量来改变默认的行为。
#### 总结
也就是使用git stash show -p stash@{1}来查看stash的内容变化。