---
title: "使用typora和hugo构建博客"
author: "kzl"
tags: ["markdown", "typora"]
date: 2019-03-20T16:46:30+08:00

typora-root-url: G:/kzltime/static

---



## 设置typora的工具的相对路径



hugo的基本目录为：

![](/assets/1553071713273.png)







其中static是静态文件目录，编译的时候会把所有的资源拷贝走，所以typora设置的时候以这个目录为基准。

如下图：

![1553071826744](/assets/1553071826744.png)



设置`typora-root-url`的路径，然后在typora的设置中，选择使用相对路径

![1553072094421](/assets/1553072094421.png)



保存。这样写文章的时候，本地的图片显示也正确（基于typora-toor-url的相对路径），同时服务器端的路径也正确(基于static的路径)

