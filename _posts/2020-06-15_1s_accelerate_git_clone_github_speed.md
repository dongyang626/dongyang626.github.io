---  
layout: post  
title: 1s 提升Git clone Github 速度  
subtitle: 简单、快速  
date: 2020-06-15  
author: Dongyang  
header-img: img/post-bg-cook.jpg  
catalog: true  
tags:  
- Git  
---  
  
这类操作性质的文章网上很多，本来不想重复造轮子的。  
奈何很多方法，1费时间，2可能还不好用。  
所以还是写一下亲测简单又好用的方法（该方法来自知乎，文末会附链接）。  
  
该方法只需一步：  
将clone地址中的 github.com 替换为github的Mirror: github.com.cnpmjs.org 即可  
（该方法同样适用于对github网站的访问）  
Eg：你本来打算clone  
> git clone https://github.com/dongyang626/dongyang626.github.io.git  
  
只需将地址改为：  
> git clone https://github.com.cnpmjs.org/dongyang626/dongyang626.github.io.git  
  
速度即可成倍提升 :)  
  
注意：  
很多Git仓库，更偏向支持SSH方式clone， 像BitBucket已经不再支持HTTP的方式了，  
所以有些情况下使用HTTP可能会报错，网速没问题的尽量使用SSH方式。另，我对SSH中的github.com进行替换，发现会导致time  
out 无法clone（没有进一步调查该问题）。  
  
知乎原文作者：Don.hub  
原文链接：[https://www.zhihu.com/question/27159393/answer/1117219745](https://www.zhihu.com/question/27159393/answer/1117219745)
