---
layout: post
title: Leveldb Write
date: 2017-09-06
author: wuxiangwei
categories: Leveldb
tags: Leveldb
---

* Kramdown table of contents
{:toc .toc}

Leveldb是线程安全的，允许同个进程的多个线程不用加锁就读写数据库。

# 同步

同步1，先处理队首元素。

```cpp
Status DBImpl::Write() {
    MutexLock l(&mutex_);
    writers_.push_back(&w);
    while (!w.done && &w != writers_.front()) {
        w.cv.Wait();
    }
}
```
所有写操作先被放入`writers_`队列，然后等待，直到自己要么成为队首被合并掉才允许进行后续操作。注意，Wait时会先解锁`mutex_`，唤醒后会重新加锁。


同步2，等待compact以减少Level0的文件数量。

```cpp
Status DBImpl::MakeRoomForWrite() {
    while (true) {
        if (allow_delay && 
            versions_->NumLevelFiles(0) >= config::kL0_SlowdownWritesTrigger) {
            mutex_.Unlock();
            env_->SleepForMicroseconds(1000);
            allow_delay = false;
            mutex_.Lock();
        }
    }
}
```
解锁等待一段时间，这段时间内阻塞在`writers_`队列中的写操作因为不满足队首的条件仍然无法继续后续操作。

同步3，解锁写Log、写Memtable。    
同步4，加锁将写操作从`writers_`队列中移除。    

从同步的角度来看，所有写操作都变成串行执行，所有写线程都阻塞等待。    
既然所有写操作线程都会阻塞，那么这过程中几次释放锁的目的是什么呢？让其它线程，主要考虑Compact线程或者读操作线程。

# 合并

如果队首元素为非空写操作，Leveldb在写Log前会将`writers_`队列中的多个写操作合并成一个写操作，详细内容参考BuildBatchGroup函数。合并的条件是，合并后的写操作大小不能超过1MB，如果队首大小小于128K，那么总大小不能超过1MB+128KB。

通过合并写操作，多个写线程的写请求可以在一次写操作中被处理掉。

合并操作对`TEST_CompactMemTable`的影响。考虑如下场景：    
假设`writers_`队列中有两个写操作w1，w2，其中w2为写空操作，同时假设`imm_`为空。    
如果w1将w2合并写入`mem_`，同时`imm_`为仍然为空。那么`TEST_CompactMemTable`仍然会返回，但它写空操作并没有将它前面的数据刷入文件。
