---
layout: post
title: 存储问题FAQ
date: 2018-01-20
author: wuxiangwei
categories: 存储
tags: 存储
---

* Kramdown table of content
{:toc, toc}

# 磁盘问题 #

## 坏盘 ##

检查`/var/log/syslog`中对应磁盘的日志记录。


```
Sep 20 03:03:52 10-165-0-41 kernel: [26617663.400353] sd 0:2:18:0: [sds] Unhandled sense code
Sep 20 03:03:52 10-165-0-41 kernel: [26617663.400359] sd 0:2:18:0: [sds]
Sep 20 03:03:52 10-165-0-41 kernel: [26617663.400361] Result: hostbyte=DID_OK driverbyte=DRIVER_SENSE
```

## 慢盘 ##

检查磁盘监控，是否出现长时间util为100%。

## 查看RAID缓存 ##

sudo megacli -LDGetProp -Cache -Lall -a0

## 查看调度策略 ##

cat /sys/block/sda/queue/scheduler

# 内存问题 #

## swap ##

swap导致卡住。
存储应该关闭swap。

# 网络问题 #

## 带宽测试 ##

iperf


