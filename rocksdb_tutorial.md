# RocksDB 0基础代码走读教程

## 1. 前言

RocksDB是由Facebook开发和维护的持久化键值存储库，基于Google的LevelDB构建。它采用Log-Structured-Merge-Database (LSM)设计，特别适合在闪存和RAM存储上存储数据。本教程旨在为0基础用户提供RocksDB的代码走读指南，从基础概念到深入架构，逐步解析RocksDB的核心组件和实现原理，并最终针对未解决的issue提出可行的解决方案。

## 2. RocksDB基本架构

### 2.1 LSM树原理

LSM树（Log-Structured Merge Tree）是RocksDB的核心数据结构，它的设计理念是将随机写入转换为顺序写入，从而提高写入性能。LSM树由以下部分组成：

- **MemTable**：内存中的有序数据结构，用于接收新写入的数据
- **Immutable MemTable**：当MemTable达到一定大小后，会被转换为Immutable MemTable，不再接受写入
- **SST文件**：Immutable MemTable会被刷新到磁盘，形成SST文件
- **Compaction**：后台进程会定期将SST文件合并，优化读取性能

### 2.2 RocksDB整体架构

RocksDB的整体架构包括以下核心组件：

- **DB**：主数据库接口，提供Put、Get、Delete等操作
- **MemTable**：内存中的数据结构，通常使用SkipList实现
- **WAL**：Write-Ahead Log，用于崩溃恢复
- **SST文件**：磁盘上的有序数据文件
- **Compaction**：合并SST文件的过程
- **TableCache**：缓存SST文件的元数据
- **BlockCache**：缓存SST文件中的数据块

## 3. MemTable实现

### 3.1 MemTable接口

MemTable是RocksDB中用于接收新写入数据的内存数据结构，定义在[memtable.h](file:///workspace/db/memtable.h)中：

```cpp
class MemTable {
 public:
  MemTable(const InternalKeyComparator& comparator, 
           Allocator* allocator, 
           const Options& options);
  
  // 添加键值对
  void Add(SequenceNumber seq, ValueType type, 
           const Slice& key, const Slice& value);
  
  // 查找键值对
  bool Get(const LookupKey& key, std::string* value, Status* s);
  
  // 生成迭代器
  Iterator* NewIterator(const ReadOptions& read_options);
  
  // 获取内存使用量
  size_t ApproximateOffsetOf(const Slice& key);
  size_t MemTableSize();
  
  // 转换为Immutable MemTable
  void MarkImmutable();
  bool IsImmutable() const;
};
```

### 3.2 SkipList实现

MemTable内部通常使用SkipList作为底层数据结构，定义在[skiplist.h](file:///workspace/memtable/skiplist.h)中：

```cpp
template <typename Key, class Comparator>
class SkipList {
 private:
  struct Node;

 public:
  // 创建一个新的SkipList
  explicit SkipList(Comparator cmp, Arena* arena);
  
  // 插入键
  void Insert(const Key& key);
  
  // 查找键
  bool Contains(const Key& key) const;
  
  // 生成迭代器
  class Iterator {
   public:
    // 移动到第一个元素
    void Seek(const Key& target);
    // 移动到最后一个元素
    void SeekToFirst();
    // 移动到下一个元素
    void Next();
    // 检查是否到达末尾
    bool Valid() const;
    // 获取当前键
    const Key& key() const;
  };
};
```

### 3.3 MemTable与WAL的关系

当数据写入MemTable时，会先写入WAL（Write-Ahead Log），以确保数据在崩溃后能够恢复。WAL的实现位于[log_writer.cc](file:///workspace/db/log_writer.cc)和[log_reader.cc](file:///workspace/db/log_reader.cc)中。

## 4. SST文件格式和管理

### 4.1 SST文件格式

SST文件（Sorted String Table）是RocksDB中存储在磁盘上的有序数据文件，定义在[format.h](file:///workspace/table/format.h)中：

```cpp
class BlockHandle {
 public:
  // 块的偏移量
  uint64_t offset() const { return offset_; }
  // 块的大小
  uint64_t size() const { return size_; }
};

class Footer {
 public:
  // metaindex block的句柄
  BlockHandle metaindex_handle() const;
  // index block的句柄
  BlockHandle index_handle() const;
};
```

### 4.2 BlockBasedTable

RocksDB默认使用BlockBasedTable作为SST文件的实现，定义在[block_based_table_factory.h](file:///workspace/table/block_based/block_based_table_factory.h)中：

```cpp
class BlockBasedTableFactory : public TableFactory {
 public:
  // 创建表读取器
  Status NewTableReader(const TableReaderOptions& table_reader_options, 
                        std::unique_ptr<RandomAccessFileReader>&& file, 
                        uint64_t file_size, 
                        std::unique_ptr<TableReader>* table_reader) const override;
  
  // 创建表构建器
  TableBuilder* NewTableBuilder(const TableBuilderOptions& table_builder_options, 
                                WritableFileWriter* file) const override;
};
```

## 5. Compaction机制

### 5.1 Compaction类型

RocksDB支持多种Compaction策略，包括：

- **Level Compaction**：将数据分层存储，每层数据量呈指数增长
- **Universal Compaction**：根据文件大小和分数选择要合并的文件
- **FIFO Compaction**：按照文件创建时间顺序删除旧文件

### 5.2 Compaction实现

Compaction的核心实现位于[compaction.h](file:///workspace/db/compaction/compaction.h)和[compaction_job.h](file:///workspace/db/compaction/compaction_job.h)中：

```cpp
class Compaction {
 public:
  // 获取输入文件
  const std::vector<FileMetaData*>& inputs(int level) const;
  
  // 获取输出level
  int output_level() const { return output_level_; }
  
  // 检查是否是手动压缩
  bool IsManualCompaction() const { return is_manual_; }
};

class CompactionJob {
 public:
  // 准备压缩
  Status Prepare();
  
  // 运行压缩
  Status Run();
  
  // 安装压缩结果
  Status Install();
};
```

## 6. 关键算法和数据结构

### 6.1 布隆过滤器

布隆过滤器是RocksDB中用于快速判断键是否存在的算法，实现位于[bloom_impl.h](file:///workspace/util/bloom_impl.h)中：

```cpp
class FastLocalBloomImpl {
 public:
  // 添加哈希值
  static inline void AddHash(uint32_t h1, uint32_t h2, uint32_t len_bytes, 
                             int num_probes, char* data);
  
  // 检查哈希值是否可能存在
  static inline bool HashMayMatch(uint32_t h1, uint32_t h2, uint32_t len_bytes, 
                                 int num_probes, const char* data);
};
```

### 6.2 跳表

跳表是MemTable的底层数据结构，提供了O(log n)的查找和插入性能：

```cpp
template <typename Key, class Comparator>
class SkipList {
 private:
  struct Node {
    Key key;
    Node* next_[1];
  };
  
  Node* head_;
  int max_height_;
  Comparator comparator_;
  Arena* arena_;
};
```

## 7. 未解决的Issue分析

### 7.1 IODispatcher内存泄漏问题

在分析RocksDB代码时，我们发现了一个IODispatcher内存泄漏的问题。该问题发生在`ReadIndex()`方法中，当从`ReadSet`中移动块值时，没有释放相关的预取内存，导致当设置了`max_prefetch_memory_bytes`时，后续的预取会被阻塞。

问题代码位于[io_dispatcher_imp.cc](file:///workspace/util/io_dispatcher_imp.cc)中的`ReadIndex`方法：

```cpp
Status ReadSet::ReadIndex(size_t block_index, CachableEntry<Block>* out) {
  // ...
  // Case 3: Block needs synchronous read (pending or never-dispatched blocks).
  // No ReleaseMemory() needed here because blocks reaching this path never had
  // TryAcquireMemory() called — they were either pending prefetch or skipped
  // during SubmitJob. block_sizes_[block_index] may be > 0 (set during
  // SubmitJob for all uncached blocks) but that does not imply memory was
  // acquired.
  RemoveFromPending(block_index);

  Status s = SyncRead(block_index);
  if (s.ok()) {
    *out = std::move(pinned_blocks_[block_index]);
    num_sync_reads_++;
  }
  return s;
}
```

### 7.2 解决方案

我们的解决方案是在`ReadIndex`方法的同步读取分支中添加内存释放逻辑，确保当块值被移动时，相关的预取内存也被释放：

```cpp
Status ReadSet::ReadIndex(size_t block_index, CachableEntry<Block>* out) {
  // ...
  // Case 3: Block needs synchronous read (pending or never-dispatched blocks).
  // No ReleaseMemory() needed here because blocks reaching this path never had
  // TryAcquireMemory() called — they were either pending prefetch or skipped
  // during SubmitJob. block_sizes_[block_index] may be > 0 (set during
  // SubmitJob for all uncached blocks) but that does not imply memory was
  // acquired.
  RemoveFromPending(block_index);

  Status s = SyncRead(block_index);
  if (s.ok()) {
    *out = std::move(pinned_blocks_[block_index]);
    // Release memory accounting for prefetched blocks. After moving the value
    // out, ReleaseBlock() and the destructor check pinned_blocks_.GetValue()
    // which will be null, so they won't release memory again.
    if (block_index < block_sizes_.size() && block_sizes_[block_index] > 0) {
      if (auto dispatcher_data = dispatcher_data_.lock()) {
        dispatcher_data->ReleaseMemory(block_sizes_[block_index]);
      }
      block_sizes_[block_index] = 0;
    }
    num_sync_reads_++;
  }
  return s;
}
```

## 8. PR提交指南

### 8.1 准备PR

1. **fork仓库**：在GitHub上fork RocksDB仓库
2. **克隆仓库**：`git clone https://github.com/your-username/rocksdb.git`
3. **创建分支**：`git checkout -b fix-io-dispatcher-memory-leak`
4. **提交修改**：`git add util/io_dispatcher_imp.cc && git commit -m "Fix memory leak in IODispatcher ReadIndex()"`
5. **推送分支**：`git push origin fix-io-dispatcher-memory-leak`

### 8.2 创建PR

1. 在GitHub上打开你的fork仓库
2. 点击"Pull request"按钮
3. 填写PR标题和描述，说明问题和解决方案
4. 点击"Create pull request"按钮

### 8.3 代码审查

在PR创建后，RocksDB的维护者会进行代码审查。在审查过程中，你可能需要：

1. 回答审查者的问题
2. 根据审查者的建议修改代码
3. 确保所有测试通过

## 9. 总结

通过本教程，我们从0基础开始，逐步深入了解了RocksDB的核心架构和实现原理。我们分析了：

- RocksDB的基本架构和LSM树原理
- MemTable的实现和SkipList数据结构
- SST文件格式和BlockBasedTable实现
- Compaction机制和策略
- 布隆过滤器等关键算法
- IODispatcher内存泄漏问题及其解决方案

希望本教程能够帮助你快速理解RocksDB的内部工作原理，掌握其核心概念和实现细节，从而能够有效地使用和扩展RocksDB。