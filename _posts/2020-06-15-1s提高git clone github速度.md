---


---

<hr>
<h2 id="layout-----posttitle------1s提高国内git-clone-github-速度subtitle---轻松解决clone-速度缓慢问题.date-------2020-06-15author-----dongyangheader-img-imgpost-bg-cook.jpgcatalog-truetags-git-">layout:     post<br>
title:      1s提高国内Git clone github 速度<br>
subtitle:   轻松解决clone 速度缓慢问题.<br>
date:       2020-06-15<br>
author:     Dongyang<br>
header-img: img/post-bg-cook.jpg<br>
catalog: true<br>
tags: git<br>
-</h2>
<p>这类操作性质的文章网上很多，本来不想重复造轮子的。 奈何很多方法，1费时间，2可能还不好用。<br>
所以还是写一下亲测简单又好用的方法（该方法来自知乎，文末会附链接）。</p>
<p>该方法只需一步：<br>
将clone地址中的 <a href="http://github.com">github.com</a> 替换为github的Mirror: <a href="http://github.com.cnpmjs.org">github.com.cnpmjs.org</a> 即可 （该方法同样适用于对github网站的访问）<br>
Eg：你本来打算clone</p>
<blockquote>
<p>git clone <a href="https://github.com/dongyang626/dongyang626.github.io.git">https://github.com/dongyang626/dongyang626.github.io.git</a></p>
</blockquote>
<p>只需将地址改为：</p>
<blockquote>
<p><a href="https://github.com.cnpmjs.org/dongyang626/dongyang626.github.io.git">https://github.com.cnpmjs.org/dongyang626/dongyang626.github.io.git</a></p>
</blockquote>
<p>速度即可成倍提升  :)</p>
<p>注意：<br>
很多Git仓库，更偏向支持SSH方式clone， 像BitBucket已经不再支持HTTP的方式了， 所以有些情况下使用https可能会报错，还是网速没问题的尽量使用ssh方式。另，我对ssh中的github.com进行替换，发现会导致time out 无法clone（没有进一步调查该问题）。</p>
<p>知乎原文作者：Don.hub<br>
原文链接：<a href="https://www.zhihu.com/question/27159393/answer/1117219745">https://www.zhihu.com/question/27159393/answer/1117219745</a></p>

