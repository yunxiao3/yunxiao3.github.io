---
title: 《漫谈LevelDB系列之：SkipList 》
description:  本文是漫谈LevelDB系列中的第一篇文章。主要是谈一谈我对LevelDB中核心数据结构SkipList的理解。在我看来这个SkipList在实现上其实做了非常多的取舍，以及特殊设计，是一个非常好的c++学习以及软件设计的样本。
date: Wed 16 Mar 2022 04:09:59
tags:
  - LevelDB
categories:
- LevelDB
---

​		其实从去年开始自己就在学习LevelDB了，源码也过了好几遍，但以我的记性看过一遍如果不输出点东西，很多理解肯定会很快抛诸脑后。刚好自己最近开始在基于LevelDB重构一个新的存储引擎，顺便写点东西记录自己体会到的LevelDB的设计理念。本系列的第一篇文章选择谈一谈memtable中的核心数据结构SkipList 。在我看来这个SkipList 在实现上其实做了非常多的取舍，以及特殊设计，是一个非常好的c++学习以及软件设计的样本。

## 1. 整体结构

![img](https://dblab-zju.feishu.cn/space/api/box/stream/download/asynccode/?code=MDZiYzU5YjJkMjgyNTQzMThkNDNhNGIwMDAyNDlkNWRfamkwZWpvb3ZKa1VMYlhoTHdHSGNEdk5NQ0tmRUxVZHNfVG9rZW46Ym94Y25YOWhWTFhpSXcxOWw2QjFiRXByQnFRXzE2NDc0MDc2MzY6MTY0NzQxMTIzNl9WNA)

我们可以看到LevelDB中的SkipList 其实和你别的地方看到的SkipList在结构上没有任何区别。如果对SkipList不了解可以先阅读几篇SkipList的介绍博客。

```c++
// Thread safety 
// -------------
// Writes require external synchronization, most likely a mutex.
// Reads require a guarantee that the SkipList will not be destroyed
// while the read is in progress.  Apart from that, reads progress
// without any internal locking or synchronization.

// Invariants:
// (1) Allocated nodes are never deleted until the SkipList is // destroyed.  This is trivially guaranteed by the code since we
// never delete any skip list nodes.
// (2) The contents of a Node except for the next/prev pointers are immutable after the Node has been linked into the SkipList.
// Only Insert() modifies the list, and it is careful to initialize a node and use release-stores to publish the nodes in one or
// more lists.
// ... prev vs. next pointer ordering ...

template<typename Key, class Comparator>
class SkipList {
  explicit SkipList(Comparator cmp, Arena* arena);
  void Insert(const Key& key);
  bool Contains(const Key& key) const;
  class Iterator {
     ....
  };
  ......
}
```

但是LevelDB中的SkipList却只有**Insert()、Contains()，**两个对外接口以及一个内部的**Iterator**。并不提供读取和删除操作。那么LevelDB为什么要这样设计以及这样做的好处是什么呢？本文尝试以LevelDB SkipList的增删读写操作为入口，谈一谈我对其设计的理解。

## 2. 删除操作

LevelDB中的SkipList并不提供删除操作，而是在为每个kv数据都增加一个有效和删除的标记位，把所有的删除操作都被转化成了写入操作，这样做的好处一方面是简化了代码的实现复杂度，更重要的是SkipList在并发的情况下，减少锁的使用提升性能。我们可以看下源码中最开头的几句关于 SkipList 线程安全的注释：

1. Write：在修改跳表时，需要在用户代码侧加锁。
2. Read：在访问跳表（查找、遍历）时，只需保证跳表不被其他线程销毁即可，不必额外加锁。

也就是说，用户侧只需要处理写写冲突就能保证线程安全。之所以不需要为读操作额外加锁是因为在实现时，LevelDB 做了以下假设（Invariants）：

1. 除非跳表被销毁，**跳表节点只会增加而不会被删除，因为跳表对外根本不提供删除接口。**

2. 被插入到跳表中的节点，除了 next 指针其他域都是不可变的，并且只有插入操作会改变跳表。

## 3. 读取操作

从源码中可以看到Contains只能判断SkipList 中是否有key，无法读取value的值。SkipList 所有的读取操作都通过Iterator来完成。之所以这样设计这其实是因为LevelDB在这里用到了**Iterator的设计模式：** 提供一种方法顺序访问一个聚合对象中各个元素, 而又不需暴露该对象的内部表示。

```c++
template <typename Key, class Comparator>
bool SkipList<Key, Comparator>::Contains(const Key& key) const {
  Node* x = FindGreaterOrEqual(key, nullptr);
  if (x != nullptr && Equal(key, x->key)) {
    return true;
  } else {
    return false;
  }
}
```

在leveldb的include/leveldb.h下定义了一个iterator基类，用于访问某层sst、某个sst内部kv以及某个memtable内部kv。SkipList中的所有数据的访问也是由iterator完成的，在memtable中点读数据时，需要先获取一个SkipList的iterator然后在使用seek二分查找目标数据，而访问整个levelDB中的数据时则需要将memtable中的iterator与sstable合并。这样做的好处是能将容器中访问数据的职能和其他职能分离开来，遵守了单一职能原则；增加新的聚合类时，只需继承iterator基类实现新的迭代器版本，遵守了开闭原则。而且我们可以看到SkipList的迭代器使用内部类实现，提高了封装性，利于获取其外部类的全部访问权限。

## 4. 插入操作

### 4.1 如何保证并发的正确性

在我个人看来LevelDB Skiplist实现的**复杂点**在于并发读写访问中如何在不加锁的情况下提供对读的正确性保证。而整个Skiplist设计的精髓也在于他的插入操作。

```c++
template <typename Key, class Comparator>
void SkipList<Key, Comparator>::Insert(const Key& key) {
  // 待做(opt): 由于插入要求外部加锁，因此可以使用 NoBarrier_Next 的 FindGreaterOrEqual 以提高性能
  Node* prev[kMaxHeight]; // 长度设定简单粗暴，直接取最大值（kMaxHeight = 12）肯定没错。
  Node* x = FindGreaterOrEqual(key, prev);
  // LevelDB 跳表要求不能插入重复数据
  assert(x == nullptr || !Equal(key, x->key));
  int height = RandomHeight(); // 随机获取一个 level 值
  if (height > GetMaxHeight()) { // GetMaxHeight() 为获取跳表当前
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;
    }
    // 此处不用为并发读加锁。因为并发读在（在另外线程中通过 FindGreaterOrEqual 中的 GetMaxHeight）
    // 读取到更新后跳表层数，但该节点尚未插入时也无妨。因为这意味着它会读到 nullptr，而在 LevelDB
    // 的 Comparator 设定中，nullptr 比所有 key 都大。因此，FindGreaterOrEqual 会继续往下找，
    // 符合预期。
    max_height_.store(height, std::memory_order_relaxed);

  x = NewNode(key, height);
  for (int i = 0; i < height; i++) {
    // 此句 NoBarrier_SetNext() 版本就够用了，因为后续 prev[i]->SetNext(i, x) 语句会进行强制同步。
    // 并且为了保证并发读的正确性，一定要先设置本节点指针，再设置原条表中节点（prev）指针
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
    prev[i]->SetNext(i, x);
  }
}
```

我们可以看到在Insert函数中Skiplist使用一个公用函数 FindGreaterOrEqual(const Key& key, Node** prev)，来找到待插入数据的前驱，然后随机生成一个高度的Node并将其插入到Skiplist。但是我们可以看到代码中**并没有使用锁来解决并发中的数据竞争问题**，而是自己定义了一堆原子操作来保证数据修改的原子性。

```c++
template <typename Key, class Comparator>
struct SkipList<Key, Comparator>::Node {
  explicit Node(const Key& k) : key(k) {}
  Key const key;
  Node* Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_acquire);
  }
  void SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_release);
  }
  // 不带内存屏障版本的访问器。内存屏障（Barrier）是个形象的说法，也即加一块挡板，阻止重排/进行同步
  Node* NoBarrier_Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_relaxed);
  }
  void NoBarrier_SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_relaxed);
  }
 private:
  // 指针数组的长度即为该节点的 level，next_[0] 是最低层指针.
  std::atomic<Node*> next_[1]; // 指向指针数组的指针
};
```

有读者可能会问为什么Skiplist不能直接赋值，而要设计这么复杂的操作呢？我们知道，编译器/CPU 在保在达到相同效果的情况下会按需（加快速度、内存优化等）对指令进行重排（乱序执行），这对单线程来说的确没什么。但是对于多线程，指令重排会使得多线程间代码执行顺序变的各种反直觉。以下面代码为例：

```c++
#include <iostream>
#include <thread>
using namespace std;
int a,b;
void func1() { a = 1; b = 2; }
void func2() { cout << a << b << endl; }
int main() {
    a = 0;
    b = 0;
    thread a(func1), b(func2);
    a.join();
    b.join();
    return 0;
}
```

该代码片段可能会打印出 `2 0`！原因就在于编译器/CPU 将 func1`()` 赋值顺序重排了或者将 func2`()` 中打印顺序重排了。因此如果不做显式同步，我们不能对多线程间代码执行顺序有任何假设。而LevelDB的Skiplist中主要涉及 C++11 中 atomic 标准库中新支持的几种 memory order，本文仅说明下用到的三种：

1. `std::memory_order_relaxed`：不对重排做限制，只保证相关共享内存访问的原子性。
2. `std::memory_order_acquire`: 用在 load 时，保证同线程中该 load 之后的对相关内存读写语句不会被重排到 load 之前，并且其他线程中对同样内存用了 store release 都对其可见。
3. `std::memory_order_release`：用在 store 时，保证同线程中该 store 之后的对相关内存的读写语句不会被重排到 store 之前，并且该线程的所有修改对用了 load acquire 的其他线程都可见。

```c++
template <typename Key, class Comparator>
int SkipList<Key, Comparator>::RandomHeight() {
  // 每次以 1/4 的概率增加层数
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

### 4.2 如何确定插入节点的高度

插入时除了要保证并发的正确性，另外一个**关键点**在于每个节点插入时，如何确定新插入节点的层数，以使跳表满足概率均衡，进而提供高效的查询性能,即 `RandomHeight` 的实现：

```c++
emplate <typename Key, class Comparator>
int SkipList<Key, Comparator>::RandomHeight() {
  // 每次以 1/4 的概率增加层数
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

我们可以看到LevelDB 的采样较为稀疏，每四个采一个，且层数上限为 kMaxHeight = 12。具体实现过程即为构造 P = 3/4 的几何分布的过程。结果为所有节点中：3/4 的节点为 1 层节点，3/16 的节点为 2 层节点，3/64 的节点为 3 层节点，依次类推。这样平均每个节点含指针数 1/P = 4/3 = 1.33 个。选 p = 1/4 相比 p = 1/2 是用时间换空间，即只牺牲了一些常数的查询效率，换取平均指针数的减少，从而减小内存使用。论文中也推荐 p = 1/4 ，除非读速度要求特别严格。在此情况下，可以最多支持 n = (1/p)^kMaxHeight 个节点的情况，查询时间复杂度不退化。如果选 kMaxHeight = 12，则跳表最多可以支持 n = 4 ^ 12 = 16M 个值。

## 5. 迭代操作

```c++
template <typename Key, class Comparator> 
inline void SkipList<Key, Comparator>::Iterator::Prev() { 
// 相比在节点中额外增加一个 prev 指针，我们使用从头开始的查找定位其 prev 节点  
    assert(Valid()); 
    node_ = list_->FindLessThan(node_->key); 
    if (node_ == list_->head_) { node_ = nullptr; } 
}
```

LevelDB 中Skiplist的Iterator::Prev()设计也非常有意思。我们可以看出该迭代器没有为每个节点增加一个额外的 prev 指针以进行反向迭代，而是用了选择使用FindLessThan操作来反向遍历。这样设计可能是作者认为SkipList前向遍历情况相对较少，而且在原有的leveldb设计中memtable大小通常为2MB大小，内存是相对稀缺的的。因此选择用时间换空间的取舍，即使遍历全部数据由于memtable通常不大性能损失也是相对较小的。

## 6. 小结

跳表本质上是对链表的一种优化，通过逐层跳步采样的方式构建索引，以加快查找速度。使用概率均衡的思路，确定新插入节点的层数，使其满足集合分布，在保证相似的查找效率简化了插入实现。LevelDB 的跳表实现还专门对多线程读做了优化，但写仍需外部加锁。在实现思路上，整体采用时间换空间的策略，优化内存使用。