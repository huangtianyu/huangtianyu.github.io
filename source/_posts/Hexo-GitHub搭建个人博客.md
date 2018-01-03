---
title: Hexo+GitHub搭建个人博客
date: 2017-12-27 15:19:06
categories: Hexo
tags: Hexo搭建
---
### 什么是Hexo
Hexo是一个轻量级的博客，使用Markdown解析文章。上传到后台的是静态的网页，因而加载速度快。
### Hexo和jekyll两者对比
Hexo的主题更多，更好看。
### 安装使用Hexo
1. 申请github账号
免费申请，直接到[GitHub官网](https://github.com)上面申请。具体步骤参考：[GitHub的注册与使用（详细图解）](http://blog.csdn.net/p10010/article/details/51336332)
2. 安装Git，Node.js工具
3. Node.js安装好后执行以下命令
	$ npm install hexo-cli -g 
   * npm是Node.js安装时自带的类库,是目前全球最大的类库之一,比Maven仓库还大,类似CentOS的yum源,Mac OX中brew的软件库
   * 通过npm install可以直接安装基于Node.js的所有插件
4. 创建站点
* 新建文件夹 mkdir blog
* 初始化站点 hexo init blog
* 安装npm插件支持 npm install
* 启动站点 hexo server
通过上面就可以在http://localhost:4000/ 上查看生成的站点了
