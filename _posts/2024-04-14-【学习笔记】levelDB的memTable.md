---
layout: post
title: 【学习笔记】levelDB的memTable
date:   2024-04-14
categories: study
---
这周继续学习levelDB的源码，参考了源码本身以及各大博主所发布的博客，写了这篇学习笔记。这次着重关注于其比较核心的“存储层”部分中的内存表Memtable，这是其内存数据库的一个模块。  
## Memtable类的简介
Memtable本身的数据结构是跳表（Skiplist），它的类的定义中自带了一个比较器（Comparator）。每次写操作时，LevelDB会将“编码后”的数据插入Memtable，编码后的数据结构的相关解释可见下图：  
![my alternate text](/assets/LookupKey.png) 

<p align="center">
图1：编码后的LookUpKey
</p>

因此，Memtable的写入操作，也即Skiplist的插入操作，上一篇博客中已经介绍过很详细的Skiplist数据结构的创建、插入和删除等操作的原理与实现了，这里就不再赘述了。  
以下是Memtable这个类的主要的定义代码： 

```cpp  
class MemTable {
 private:

  struct KeyComparator {        /* 用于作key比较的一个结构体类型 */
    const InternalKeyComparator comparator;
    explicit KeyComparator(const InternalKeyComparator& c) : comparator(c) {}
    int operator()(const char* a, const char* b) const;
  };

  //menTable本身的表的数据结构，实际上是Skiplist（跳表）结构
  typedef SkipList<const char*, KeyComparator> Table; 
  KeyComparator comparator_;
  int refs_; // 引用计数
  Arena arena_; // 内存管理的对象
  Table table_; // menTable本身所代表的表

public:
  /* 估计大约的内存使用情况 */
  size_t ApproximateMemoryUsage();

  /* 自带的迭代器 */
  Iterator* NewIterator();

  /* 加入memTable一个内容 */
  void Add(SequenceNumber seq, ValueType type, const Slice& key, const Slice& value);

  /* 取得memTable的一个内容 */
  bool Get(const LookupKey& key, std::string* value, Status* s);
};

```
在 LevelDB 的内部结构中，memTable 扮演了非常重要的角色。其主要作用如下：  

#### 写入缓存：  
LevelDB 的写操作首先会将数据写入到 memTable 中。这样，新写入的数据会先保存在内存中，而不是直接写入到磁盘上。这种设计可以大大提高写操作的性能，因为内存的读写速度远远快于磁盘。  
#### 合并与排序：  
在 memTable 中，数据并不是直接按照键值顺序存储的。但是，当 memTable 达到一定大小或者需要进行持久化操作时，它会将数据按照键值顺序进行排序，并合并成一个新的、有序的、不可变的数据结构（通常称为 SSTable）。这个过程有助于优化后续的读取操作，因为数据已经是按照键值顺序排列的。  
#### 减少磁盘 I/O：  
通过将新写入的数据暂时保存在 memTable 中，LevelDB 可以减少频繁的磁盘 I/O 操作。仅当memTable 达到一定大小或者需要进行持久化时，才会触发磁盘写入操作。这种批处理的方式可以大大提高系统的整体性能。  
#### 支持并发写入：  
LevelDB 支持并发写入操作，而 memTable 的设计使得这种并发写入成为可能。每个写入操作都可以在其自己的 memTable 中进行，而不需要等待其他写入操作完成。这种设计使得 LevelDB 能够更好地处理高并发的写入场景。  