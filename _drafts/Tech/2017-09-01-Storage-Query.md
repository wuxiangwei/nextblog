---
layout: post
title: 存储配置查询
date: 2017-09-01
author: wuxiangwei
categories: 技术 存储
tags: 技术 存储
---


查看RAID缓存：       
sudo megacli -LDGetProp -Cache -Lall -a0

查看磁盘调度策略：    
cat /sys/block/sda/queue/scheduler

