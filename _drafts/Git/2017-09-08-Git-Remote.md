---
layout: post
title: Git Remote
date: 2017-09-08
author: wuxiangwei
categories: Git
tags: Git
---
<br>
远程主机由两部分构成：主机名和网址。

增：    
git remote add myceph git@github.com:wuxiangwei/ceph.git


删：    
git remote rm myceph


查：    
git remote [-v]


用：    
git pull myceph luminous     
git push myceph luminous:luminous      

<br>
应用场景：    
本地有份代码做了修改，打算提交到github自己的仓库。但本地代码不是从自己仓库里拉取的，此时可添加远程主机（自己的代码仓库），合并代码，再push到远程主机。





