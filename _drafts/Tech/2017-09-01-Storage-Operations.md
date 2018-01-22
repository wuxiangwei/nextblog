---
layout: post
title: 存储运维知识点
date: 2017-09-20
author: wuxiangwei
categories: 技术 存储
tags: 技术 存储
---


坏盘，查看`/var/log/message`日志，存在下面的记录：    

```
Sep 20 03:03:52 10-165-0-41 kernel: [26617663.400353] sd 0:2:18:0: [sds] Unhandled sense code
Sep 20 03:03:52 10-165-0-41 kernel: [26617663.400359] sd 0:2:18:0: [sds]
Sep 20 03:03:52 10-165-0-41 kernel: [26617663.400361] Result: hostbyte=DID_OK driverbyte=DRIVER_SENSE
```
