---
layout: post
title: Ceph Monitor Compact
date: 2017-08-31
author: wuxiangwei
category: [Ceph]
tags: Ceph
---


为什么Compact？    
执行Ceph Monitor的compact命令能够减少leveldb对磁盘空间的占用，compact命令实际调用了Leveldb的compact操作。Leveldb compact主要针对以下两种情况：第一种是覆盖写，由于Leveldb采用追加写的方式添加KV表项，所以在不同Level中可能存在相同Key不同Value。如果对该Key执行compact，能够删除该Key对应的旧Value，只保留最新的Value，从而回收磁盘空间；第二种情况是表项删除，Leveldb删除表项也以追加写的方式是向db添加一条删除标记，没有立即删除数据。此时compact，可以将删除标记以及Key对应的所有Value删除，从而回收磁盘空间。

Ceph Monitor插入表项到Leveldb可以分为两种方式：第一种方式是按照版本号递增，例如pgmap表项，一个版本一条表项；另一种是直接覆盖，例如`pgmap_pg`，一个pg一条表项，每次更新该表项都是覆盖写，旧表项直接失效。第一种方式插入的表项，通过版本号仍然可以读取出来。

伴随表项插入的是表项删除问题。如果只插入不删除，那么db就会越来越大，最后挤爆磁盘，挂死进程。以第二种覆盖方式插入的表项，只要执行一次compact操作就能够回收磁盘。以第一种版本递增方式插入的表项，需要手动执行表项删除操作。对这类表项的删除操作，Ceph采用了两种不同的策略：一种是定时检查表项数量，如果超出阈值，则删除多余表项；另一种是针对Paxos表的，每次执行一轮Paxos后检查一次表项数量是否超出阈值，超出则删除。

回到Leveldb删除表项的机制，虽然Ceph删除了表项，但还没回收磁盘空间，回收磁盘依赖Compact操作。Leveldb提供两种compact操作，一种是Leveldb后台自动触发的compact操作，另一种是用户手动触发的compact操作。在数据量小的情况，可以直接依赖自动compact来回收磁盘；在数据量大的情况，自动compact操作无法及时回收磁盘空间，会导致db磁盘占用一直变大。Ceph提供了一个`mon_compact_on_trim`的配置选项，以选择是否在删除表项时触发手动Compact操作，默认为True。


# Paxos表


# Log表

Ceph Audit日志会记录到两个地方，一个是Monitor本地日志，另一个是每个Monitor节点的Leveldb数据库。记录到Leveldb要经历2个过程，第一个过程是在产生日志的地方将日志发送到Monitor节点，第二个过程是日志内容将作为Proposal经由Paxos同步到每个Monitor节点。如果系统中有监控程序不中断地来获取集群状态，那么将会产生大量本地日志，同时也会加大Monitor的负担。   

```cpp
void Monitor::handle_command() {
 (cmd_is_rw ? audit_clog->info() : audit_clog->debug())
    << "from='" << session->inst << "' "
    << "entity='" << session->entity_name << "' "
    << "cmd=" << m->cmd << ": dispatch";
}
```
Audit日志实现涉及到下面2个概念：(1) LogClientTemp，每记录一条日志就实例化一个对象，对象析构时把日志内容传递给LogChannel；(2) LogChannel 根据配置可以将日志内容写到多个地方，例如写到syslog、graylog或者发送给Monitor。同时，每条日志都会被忽略优先级记录到本地log文件。
