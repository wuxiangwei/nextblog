---
layout: post
title: Ceph Operations
date: 2017-09-04
author: wuxiangwei
categories: Ceph
tags: Ceph
---

* Kramdown table of contents:
{:toc, toc}

查看自启动：    
`sudo chkconfig --list | grep ceph`

查看OSD所在主机：    
`ceph osd find 123`

# Luminous v12.2.0 版本


修改源，参考[CEPH-DEPLOY SETUP](http://docs.ceph.com/docs/master/start/quick-start-preflight/#debian-ubuntu)。




```
sudo systemctl stop ceph.target
sudo systemctl stop ceph-osd@{osd-num}
```




