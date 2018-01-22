---
layout: post
title: OpenStack Operations
date: 2017-09-06
author: wuxiangwei
categories: OpenStack
tags: OpenStack
---

* Kramdown table of content
{:toc, toc}

# Cinder

查看卷：    
cinder list

创建卷，单位为GB：    
cinder create --display-name $volumename --volume-type ceph  $volumesize

删除卷：    
cinder delete $volumename

挂载卷：    
nova volume-attach $vmname $volumeid auto --force True


卸载卷：    
nova volume-detach $vmname $volumeid


# Nova

查看虚机：    
nova list

