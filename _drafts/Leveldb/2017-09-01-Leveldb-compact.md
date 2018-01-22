---
layout: post
title: Leveldb Compact
date: 2017-09-01
author: wuxiangwei
categories: Leveldb
tags: Leveldb
---

* Kramdown table of contents
{:toc .toc}


# 编译

默认make，不编译测试程序。
查看Makefile文件，按需编译。
make check编译结束自动运行测试用例。

```shell
check: all $(PROGRAMS) $(TESTS)                                                                     
       for t in $(TESTS); do echo "***** Running $$t"; ./$$t || exit 1; done
```

# 源码分析

## CompactMemTable

CompactMemTable通过写空表项来实现，将空表项前面的数据全部刷入磁盘。空表项前面的表项可能存在于`imm_`或者`mem_`中，甚至可能连数据还没写入到`mem_`中。
CompactMemTable先写空表项，然后阻塞等待，直到`imm_`中没有数据为止，具体函数调用过程如下：

```
DBImpl::TEST_CompactMemTable() --> DBImpl::Write() --> DBImpl::MakeRoomForWrite(true)
```
Write函数加锁等待，直到所有它前面的Write请求完成为止，即保证空表项前面的数据至少要写入到`mem_`中。    
MakeRoomForWrite函数：    

1. 如果`imm_`中有数据，一直等待，直到`imm_`的数据被compact掉为止。先保证`imm_`中的数据落盘；
2. 如果level0的文件数量超过阈值，一直等待，直到level0的文件被compact掉，低于阈值为止；
3. 此时，可以保证`imm_`中没有数据；
4. 新建Log文件；
5. 将`imm_`指向`mem_`指向的数据，将`mem_`数据迁移到`imm_`中。开始准备`mem_`数据落盘；
6. 为`mem_`新建一个MemTable；
7. 调度Compaction，让原`mem_`中的数据落盘，在`TEST_CompactRange`中会一直等待`imm_`为空，也就是原`mem_`数据落盘；
8. 返回。

总之，`TEST_CompactMemTable`会先写条空表项再将所有空表项前面的数据刷入文件，不管这些数据原本在`imm_`中还是在`mem_`中。简单来说，主要通过3个步骤实现：(1) 等待空表项前面的数据至少写入`mem_`后才开始真正执行写空表项的操作；(2) 等待当前`imm_`中的数据全部写入文件后才继续后续操作；(3) 将当前的`mem_`数据迁移到`imm_`中，调度Compaction，等待这部分数据全部写入文件后才返回。

## CompactRange

```
// 投递compact任务
DBImpl::CompactRange(begin, end) --> DBImpl::TEST_CompactRange(level, begin, end) --> DBImpl::MaybeScheduleCompaction() --> PosixEnv::Schedule()

// compact线程
// 如果队列为空，等待；否则，取出任务，执行任务函数。
// compact的任务函数为DBImpl::BGWork，任务参数为DBImpl实例。
PosixEnv::BGThread() --> DBImpl::BGWork() --> DBImpl::BackgroundCall() --> DBImpl::BackgroundCompaction() --> DoCompactionWork()
// 如果手动compact，调用VersionSet::CompactRange；否则调用VersionSet::PickCompaction()，获得1个Compaction实例。
```

分次Compact。如果一次Compact的Range太大，分成多次Compact，不过对Level0除外。Level0允许存在重复Key，如果分次Compact，就会导致Key回退到旧版本。     

```cpp
void DBImpl::BackgroundCompaction() {
    if (is_manual) {
        c = versions_->CompactRange(m->level, m->begin, m->end);
        m->done = (c == NULL);  // 标记此次compact未完成
        if (c != NULL) {
            // 此次compact的结束点，下次compact的起点
            manual_end = c->input(0, c->num_input_files(0) - 1)->largest;
        }
    } 

    if (is_manual) {
        manual_compaction_ = NULL;
    }
}

Compaction* VersionSet::CompactRange() {
    if (level > 0) {
        const uint64_t limit = MaxFileSizeForLevel(level);
        uint64_t total = 0;
        for (size_t i = 0; i < inputs.size(); i++) {
            uint64_t s = inputs[i]->file_size;
            total += s;
            if (total >= limit) {
                inputs.resize(i + 1);  // 丢弃i+1后面的文件
                break;
            }
        }
    }
}
```


打印日志
"Compacting %d@%d + %d@%d files"
文件数@层号，L层的文件数@L层 + L+1层的文件数@L+1层。

# FAQ

## 每Level的大小

```
mem 4MB
immem 4MB
0 10MB
1 10MB
2 100MB
3 1G
4 10G
5 100G
6 无穷大
```
Level0和Level1都是10M，其余Level，每增加一层增加10倍，参考MaxBytesForLevel函数。

## Key大小比较

```c
inline int Slice::compare(const Slice& b) const {
  const int min_len = (size_ < b.size_) ? size_ : b.size_;
  int r = memcmp(data_, b.data_, min_len);
  if (r == 0) {
    if (size_ < b.size_) r = -1;
    else if (size_ > b.size_) r = +1;
  }
  return r;
}
```
按字节比较。

## Slice构造函数不能传入临时变量

```c
class Slice {
private:
    const char* data_;
}

{
    leveldb::Slice start(Key(0));
    leveldb::Slice end(Key(10));
    db_->CompactRange(&start, &end);
}
```
Slice::data_直接引用了构造函数传入的数据，如果传入的是临时变量，临时变量销毁后data_将指向一个未知位置。
查看Leveldb的LOG文件可以发现Manual compact 的开始和结束Key不正确。


## v1.5版本显示日志问题

```shell
2017/08/30-23:12:35.013862 7f36641ff700 Manual compaction at level-0 from 'paxos .. 'paxos; will stop at (end)
2017/08/30-23:12:35.015385 7f36641ff700 Manual compaction at level-1 from 'paxos .. 'paxos; will stop at 'paxos
```
日志中手动compaction的开始和结束位置相同，都为paxos。正确的位置应该为“paxos012345”，由于Key中存在一个0，所以字符串被阶段了。

## v1.5版本compact问题

为避免单次CompactRange的范围过大，Leveldb将一次CompactRange拆分成多次执行。但Level0是个例外，Level0的Key-range不能拆分。因为Level0的不同sst文件允许包含同个Key的不同Value，如果拆分，将导致Key的当前。
