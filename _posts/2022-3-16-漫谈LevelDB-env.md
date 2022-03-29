---
title: 《漫谈LevelDB系列之：Env家族》
description:  本文是漫谈LevelDB系列中的第二篇文章。主要是谈一谈我对LevelDB中核心数据工具类Env的理解。在我看来LevelDB中的Env家族，对底层细节做了非常好的封装使得LevelDB可以很轻松的跨平台运行。是一个非常好的c++学习以及软件设计的样本。
date: Wed 23 Mar 2022 04:09:59
tags:
  - LevelDB
categories:
- LevelDB
---

​		其实从去年开始自己就在学习LevelDB了，源码也过了好几遍，但以我的记性看过一遍如果不输出点东西，很多理解肯定会很快抛诸脑后。刚好自己最近开始在基于LevelDB重构一个新的存储引擎，顺便写点东西记录自己体会到的LevelDB的设计理念。本系列的第一篇文章选择谈一谈LevelDB中Env家族的设计和实现。在我看来这个Env对底层细节做了非常好的封装使得LevelDB可以很轻松的跨平台运行。是一个非常好的c++学习以及软件设计的样本。

![img](https://dblab-zju.feishu.cn/space/api/box/stream/download/asynccode/?code=NDIxYWY3NWJmMDIxYWY4YzJlZGU1N2Y2NmZhOWY4OGRfcW9VUmFvTUlMaDJhUkZrRGF3YUxYdXVjRGo0Y1lLUjBfVG9rZW46Ym94Y256UWNSZXdjSG9zYlZjenBhVTJtTldjXzE2NDg1NTc2NzE6MTY0ODU2MTI3MV9WNA)

## 1. 整体设计

![img](https://dblab-zju.feishu.cn/space/api/box/stream/download/asynccode/?code=YmQzNDc3OTg5NjE2Mzg0MDZiY2M5NGU1NDU4ZmM3YWNfa2FjVUF2SXNyQWxUSHM1UTZnb0xVQW13VUhwUzBZWXFfVG9rZW46Ym94Y25uZ1FaTjVVZDM0cFRMYjVJT2ZXZmdiXzE2NDg1NTc2NzE6MTY0ODU2MTI3MV9WNA)

由于Leveldb是跨平台的，因此LevelDB使用Env封装了所有跨平台的内容，屏蔽了OS层面的差异性。其成员函数如图一所示：主要为Leveldb提供文件操作，以及线程管理的功能。透过其类的成员函数的修饰，我们可以看出，该类就是一个纯虚类，需要在POSIX和Windows环境下进行具体的实现。不过好像1.20版本就已经没有windows下的实现了可能是c++11统一了很多标准吧。因此本文接下来只分析Env在POSIX下的实现。

```c++
// An implementation of Env that forwards all calls to another Env.
// May be useful to clients who wish to override just part of the
// functionality of another Env.
class LEVELDB_EXPORT EnvWrapper : public Env {
  // Return the target to which this Env forwards all calls.
  Env* target() const { return target_; }
  // The following text is boilerplate that forwards all methods to target().
  Status NewSequentialFile(const std::string& f, SequentialFile** r) override {
    return target_->NewSequentialFile(f, r);
  }
  Status NewRandomAccessFile(const std::string& f,
                             RandomAccessFile** r) override {
    return target_->NewRandomAccessFile(f, r);
  }
 .....
 }
```

我们可以看到Leveldb为了解决跨平台中API差异对实现所造成的复杂性，使用了设计模式中的代理设计模式[3]，定义了一个EnvWrapper将所有的实现转发到target_的实现。虽然这里直接使用虚函数也是能完成这样的需求的。但是若要扩展功能，Wrapper能提供了比继承更具弹性的代替方案[2]。

```c++
template<typename EnvType>
class SingletonEnv {
 public:
......
  Env* env() { return reinterpret_cast<Env*>(&env_storage_); }
 private:
  // 使用了std::aligned_storage来指定对象的对齐方式 
  typename std::aligned_storage<sizeof(EnvType), alignof(EnvType)>::type
      env_storage_;
#if !defined(NDEBUG)
  static std::atomic<bool> env_initialized_;
#endif  // !defined(NDEBUG)
};

// 使用了单例模式生成唯一的PosixDefaultEnv
using PosixDefaultEnv = SingletonEnv<PosixEnv>;

Env* Env::Default() {
  static PosixDefaultEnv env_container;
  return env_container.env();
}
```

而当Leveldb需要生成一个默认的Env实例的时候，则使用了单例模式，这应该是为了确保Leveldb只有单个Env对象被创建，避免环境的错乱。非常值得注意的是Leveldb在具体的实现方式中使用了std::aligned_storage来指定对象的对齐方式。不太确定在这里对齐内存的目的是什么，可能作者考虑到PosixDefaultEnv 调用非常频繁，起到一种加速作用？ [c++ - What is the purpose of std::aligned_storage? - Stack Overflow](https://stackoverflow.com/questions/50271304/what-is-the-purpose-of-stdaligned-storage)

## 2. 文件操作

Leveldb中所有的数据都是以SStable的格式持久化在磁盘上的，例如每一个sst其实都是一个2MB（不同版本不一样）文件。而为了更好的读写sst，log等文件，Leveldb定义了RandomAccessFile、SequentialFile、WritableFile等多个类来对Leveldb中的文件进行抽象。

### 2.1 只读文件

当leveldb需要读取一个sst的时候，会将该sst的文件路径作为参数传递给NewRandomAccessFile 并由得到一个对应的RandomAccessFile。我们可以看到RandomAccessFile中只有一个Read接口，而生成的其具体的实现则有两种，当mmap_limiter_资源不足的时候返回一个PosixRandomAccessFile，否则优先返回一个PosixMmapReadableFile。

```c++
// A file abstraction for randomly reading the contents of a file.
class LEVELDB_EXPORT RandomAccessFile {
 public:
  RandomAccessFile() = default;
  .....
  virtual ~RandomAccessFile();
  // 只有一个Read接口
  virtual Status Read(uint64_t offset, size_t n, Slice* result,
                      char* scratch) const = 0;
};

Status NewRandomAccessFile(const std::string& filename,
                             RandomAccessFile** result) override {
  .....
    // 当mmap_limiter_资源不足的时候返回一个PosixRandomAccessFile
    if (!mmap_limiter_.Acquire()) {
      *result = new PosixRandomAccessFile(filename, fd, &fd_limiter_);
      return Status::OK();
    }
    // 否则优先返回一个PosixMmapReadableFile
 .....
   if (mmap_base != MAP_FAILED) {
     *result = new PosixMmapReadableFile(
         filename, reinterpret_cast<char*>(mmap_base), file_size,
         &mmap_limiter_);
   }
}
```

之所以要优先使用PosixMmapReadableFile的原因在于：1. mmap可以避免数据在内核空间以及用户空间的拷贝。2. mmap可以利用OS提供的缓存（page cache）对于一些被重复读取的数据能够起到一个加速的作用[4]。但mmap最近好像被喷的很惨，为此数据库邻域的著名网红老师andy还特意写了一篇论文发在cidr上[1]。论文中mmap被喷的原因在于他实际上是把数据库中的内存管理交给了OS，但是OS其实并不知道数据库中的那些数据应该保留那些应该换出，这使得在内存资源有限的情况下，OS可能会把一些不应该换出的page换出，造成严重的性能下降。因此目前的大多数DB都是自己维护一个buffer pool来管理buffer data。虽然leveldb也有自己的cache，但是那个cache的粒度block级别的，和这里所说的page cache并不一样。不过leveldb的优点就是简洁易懂，目前更多的是个学习样本，使用mmap实现起来相对更简单。而且对性能有要求的一般都用Rocksdb了，我想这也是leveldb一直没有跟进的原因吧。

```c++
// 由于是随机读PosixRandomAccessFile中的Read使用pread来实现
class PosixRandomAccessFile final : public RandomAccessFile {
  Status Read(uint64_t offset, size_t n, Slice* result,
              char* scratch) const override {
........
    ssize_t read_size = ::pread(fd, scratch, n, static_cast<off_t>(offset));
    *result = Slice(scratch, (read_size < 0) ? 0 : read_size);
........
  }
}
// 而PosixMmapReadableFile 中则直接用偏移地址即可
class PosixMmapReadableFile final : public RandomAccessFile {
  Status Read(uint64_t offset, size_t n, Slice* result,
              char* scratch) const override {
    if (offset + n > length_) {
      *result = Slice();
      return PosixError(filename_, EINVAL);
    }
    *result = Slice(mmap_base_ + offset, n);
    return Status::OK();
  }
}
```

但是Leveldb中的一些文件如Log文件只只需要顺序读即可，为此Leveldb定义了一个SequentialFile 的抽象类，其具体实现PosixSequentialFile 使用了read、lseek两个库函数来完成Read和Skip的操作。Leveldb之所以在有了RandomAccessFile以后还设计了一个SequentialFile，在我看来应该书处于性能的考量，毕竟pread在顺序读上可能比不过read(没做实验不过pread应该多一次seek的操作)，一次性读的话mmap不太确定。

```c++
// A file abstraction for reading sequentially through a file
class LEVELDB_EXPORT SequentialFile {
 public:
  // REQUIRES: External synchronization
  virtual Status Read(size_t n, Slice* result, char* scratch) = 0;
  // Skip "n" bytes from the file. This is guaranteed to be no
  // slower that reading the same data, but may be faster
  // REQUIRES: External synchronization
  virtual Status Skip(uint64_t n) = 0;
};
// Instances of this class are thread-friendly but not thread-safe, as required
// by the SequentialFile API.
class PosixSequentialFile final : public SequentialFile {
  Status Read(size_t n, Slice* result, char* scratch) override {
........ 
      ::ssize_t read_size = ::read(fd_, scratch, n);
........
      *result = Slice(scratch, read_size);
........
    return status;
  }
  Status Skip(uint64_t n) override {
    if (::lseek(fd_, n, SEEK_CUR) == static_cast<off_t>(-1)) {
      return PosixError(filename_, errno);
    }
    return Status::OK();
  }
};
```

### 2.2 可写文件

相对于只读文件，可写文件的设计和抽象显得更为复杂,不过好在Leveldb中的所有数据都是顺序写入的，并且更新也都是非原地更新，因此WritableFile 只有一个Append接口来对写入数据。

```c++
class LEVELDB_EXPORT WritableFile {
 public:
  virtual Status Append(const Slice& data) = 0;
  virtual Status Close() = 0;
  virtual Status Flush() = 0;
  virtual Status Sync() = 0;
};
```

而在其具体实现PosixWritableFile 中做了一些优化，例如在写文件的时候可以先写到kWritableFileBufferSize大小的buff中，然后在flush到磁盘。这样在写入一些小数据操作时，性能提升是巨大的。但是数据写到buffer中断电以后可能丢失，因此还增加了Flush、Sync两个接口来持久化数据。TODO(yunxiao)......

### 2.3 资源限制类Limiter

```c++
// Helper class to limit resource usage to avoid exhaustion.
// Currently used to limit read-only file descriptors and mmap file usage
// so that we do not run out of file descriptors or virtual memory, or run into
// kernel performance problems for very large databases.
class Limiter {
....
  Limiter(int max_acquires) : acquires_allowed_(max_acquires) {}
  bool Acquire() {....}
  void Release() {....}
  //使用atomic原子变量，保证线程安全
  std::atomic<int> acquires_allowed_; 
};
// Set by EnvPosixTestHelper::SetReadOnlyMMapLimit() and MaxOpenFiles().
int g_open_read_only_file_limit = -1;
// Up to 1000 mmap regions for 64-bit binaries; none for 32-bit.
constexpr const int kDefaultMmapLimit = (sizeof(void*) >= 8) ? 1000 : 0;
// Can be set using EnvPosixTestHelper::SetReadOnlyMMapLimit.
int g_mmap_limit = kDefaultMmapLimit;
```

在只读文件一章讲到可以打开的mmap文件数量是有限的，在Leveldb中64-bit机器下默认是1000个，而在32位机器上是0。之所以这样设计，我觉得可能是因为Leveldb中的sst通常较小，当Leveldb中有大量的数据且随机读的情况下，如果不限制mmap文件的使用，可能会很快就耗尽os的page cache，将原本有效的page cache都给换出，对系统造成巨大的抖动。特别是32位的机器最大内存也就4GB，因此在32位的机器上干脆就不用mmap来随机读取数据了。

[1]  https://db.cs.cmu.edu/papers/2022/cidr2022-p13-crotty.pdf

[2] [比继承更有弹性的装饰者模式](https://blog.csdn.net/weixin_44778952/article/details/105960757)

[3] [设计模式之代理模](https://blog.csdn.net/nobody_1/article/details/86549762)

[4] [认真分析mmap：是什么 为什么 怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)