---
title: Leveldb源码解析第三篇【Table 收尾】
date: 2017-06-02 20:10:49
categories: Leveldb源码解析
tags: 
  - leveldb
  - C++
---

> 版权声明：本文为博主原创文章，未经博主允许不得转载。

前面介绍完了table的data block和filter block，今天就来讲table收一下尾，table还剩meta index block，index block，footer

这几个都比较简单，就一起介绍了

table.h中是用来解析一个table的，在没搞懂table是什么东西前直接解析有点困难，而了解table最快速的办法就是看怎么table怎么构造的，下面直接开始构造一个table，开整

本章涉及到源码文件有

```
include/leveldb/table.h
include/leveldb/table_builder.h
table/table.cc
table/table_builder.cc
table/format.h
```

``` c++
// table_builder.h
class TableBuilder {
 public:
  // 构造方法
  TableBuilder(const Options& options, WritableFile* file);

  // 析构方法
  ~TableBuilder();

  // 改变配置参数
  Status ChangeOptions(const Options& options);

  // 在 table 中添加一个key
  void Add(const Slice& key, const Slice& value);

  // 当一个data block达到阈值以后调用
  void Flush();

  // Return non-ok iff some error has been detected.
  Status status() const;

  // 当table大小达到阈值后调用，主要是讲meta block，index block等信息写入到文件中
  Status Finish();

  // Indicate that the contents of this builder should be abandoned.  Stops
  // using the file passed to the constructor after this function returns.
  // If the caller is not going to call Finish(), it must call Abandon()
  // before destroying this builder.
  // REQUIRES: Finish(), Abandon() have not been called
  void Abandon();

  // 返回有多少个键值对
  uint64_t NumEntries() const;

  // 返回文件大小
  uint64_t FileSize() const;

 private:
  // 返回操作是否成功
  bool ok() const { return status().ok(); }
  // 将block写入到编码到string中
  void WriteBlock(BlockBuilder* block, BlockHandle* handle);
  // 将编码后的block写入到文件中
  void WriteRawBlock(const Slice& data, CompressionType, BlockHandle* handle);

  // rep很重要，table的所有信息都放在这个结构体中，现在还不知道为什么这么设计，
  // 后面深入后在分析作者这么设计的目录
  struct Rep;
  Rep* rep_;

  // No copying allowed
  TableBuilder(const TableBuilder&);
  void operator=(const TableBuilder&);
};
```

``` c++
// table_builder.cc
// 先从存放table所有信息的结构体开始
struct TableBuilder::Rep {
  // 两个配置对象，暂时可以忽略
  Options options;
  Options index_block_options;
  // 最后table是要写入文件的，file对象就是用来干这个事的
  WritableFile* file;
  // offset记录table的偏移量，主要用于index block，后续会介绍
  uint64_t offset;
  // 每次操作的状态
  Status status;
  // 存放data block
  BlockBuilder data_block;
  // 存放index block
  BlockBuilder index_block;
  // 最后一个key
  std::string last_key;
  // table中有多个键值对
  int64_t num_entries;
  // table是否结束
  bool closed;          // Either Finish() or Abandon() has been called.
  // 上一章讲到的
  FilterBlockBuilder* filter_block;

  // 这个变量表示添加的key是不是data block的第一个key
  bool pending_index_entry;
  BlockHandle pending_handle;  // Handle to add to index block
  // data block添加到table中时会压缩block，这个变量是用来存储压缩后的内容
  std::string compressed_output;

  // rep构造方法
  Rep(const Options& opt, WritableFile* f)
      : options(opt),
        index_block_options(opt),
        file(f),
        offset(0),
        data_block(&options),
        index_block(&index_block_options),
        num_entries(0),
        closed(false),
        filter_block(opt.filter_policy == NULL ? NULL
                     : new FilterBlockBuilder(opt.filter_policy)),
        pending_index_entry(false) {
    index_block_options.block_restart_interval = 1;
  }
};

// 在table中添加一个key-value，也就是在table中的data block中添加一个key
void TableBuilder::Add(const Slice& key, const Slice& value) {
  // table的所有信息全部封装在了rep_中
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->num_entries > 0) {
    assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0);
  }
  // pending_index_entry表示添加的key是不是data block的第一个key
  if (r->pending_index_entry) {
    assert(r->data_block.empty());
    // 如果是的话，就要先得到上一个data block的最后一个key和当前data block的第一个key的一个中间key
    // 比如last_key为abcd，key为abcg，那么中间key为abce，也就是第一个不同的字符加1
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    // handle_encoding保留上一个data block的偏移信息
    std::string handle_encoding;
    // 将上一个data block的偏移信息编码到string中
    r->pending_handle.EncodeTo(&handle_encoding);
    // 在index_block中添加中间key和上一个data block的偏移信息
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    // pending_index_entry设置为false，表示后面在添加key，就不是第一个key了
     
    r->pending_index_entry = false;
  }

  if (r->filter_block != NULL) {
    // 在filter block的keys中添加key
    r->filter_block->AddKey(key);
  }
  // 给last_key赋上新值
  r->last_key.assign(key.data(), key.size());
  r->num_entries++;
  // 在data block中添加键值
  r->data_block.Add(key, value);
  // 得到当前data block的估值大小
  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  if (estimated_block_size >= r->options.block_size) {
    // 如果大于阈值的话就要刷新了
    Flush();
  }
}

// 当一个data block结束时，就会刷新一下
void TableBuilder::Flush() {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->data_block.empty()) return;
  assert(!r->pending_index_entry);
  // 将data block写入到文件中
  WriteBlock(&r->data_block, &r->pending_handle);
  if (ok()) {
    // 为下一个data block作准备
    r->pending_index_entry = true;
    r->status = r->file->Flush();
  }
  if (r->filter_block != NULL) {
    r->filter_block->StartBlock(r->offset);
  }
}

void TableBuilder::WriteBlock(BlockBuilder* block, BlockHandle* handle) {
  // File format contains a sequence of blocks where each block has:
  //    block_data: uint8[n]
  //    type: uint8
  //    crc: uint32
  assert(ok());
  Rep* r = rep_;
  // 当前得到的block里面只是存储了key-value，真正完成一个block还需要在后面写入重启点等信息
  raw是整个block（包括重启点）放到string里面的信息
  Slice raw = block->Finish();

  Slice block_contents;
  // data block是否需要压缩
  CompressionType type = r->options.compression;
  // TODO(postrelease): Support more compression options: zlib?
  switch (type) {
    case kNoCompression:
    // 不需要压缩的话直接向block的内容赋值给block_contents
      block_contents = raw;
      break;

    case kSnappyCompression: {
      // 如果需要压缩的话，会将压缩后的信息放到compressed
      std::string* compressed = &r->compressed_output;
      // 如果需要压缩，压缩率高于12.5%的话就压缩
      if (port::Snappy_Compress(raw.data(), raw.size(), compressed) &&
          compressed->size() < raw.size() - (raw.size() / 8u)) {
        block_contents = *compressed;
      } else {
        // 否则就直接不压缩了，并将type设置为不压缩
        // Snappy not supported, or compressed less than 12.5%, so just
        // store uncompressed form
        block_contents = raw;
        type = kNoCompression;
      }
      break;
    }
  }
  // 将block的内容放到table中
  WriteRawBlock(block_contents, type, handle);
  // 将压缩后的内容设置为空
  r->compressed_output.clear();
  // block也需要重置
  block->Reset();
}

void TableBuilder::WriteRawBlock(const Slice& block_contents,
                                 CompressionType type,
                                 BlockHandle* handle) {
  Rep* r = rep_;
  // 将当前block的偏移和block的内容放到handle中
  handle->set_offset(r->offset);
  handle->set_size(block_contents.size());
  // 将block的内容写入到文件中
  r->status = r->file->Append(block_contents);
  if (r->status.ok()) {
    // block的结尾标识，有5个字符
    char trailer[kBlockTrailerSize];
    // 第一个字符记录的是否压缩
    trailer[0] = type;
    // 算出block的crc值
    uint32_t crc = crc32c::Value(block_contents.data(), block_contents.size());
    crc = crc32c::Extend(crc, trailer, 1);  // Extend crc to cover block type
    // 将crc值放到后4个字符中
    EncodeFixed32(trailer+1, crc32c::Mask(crc));
    // 将结尾标识放到block后面
    r->status = r->file->Append(Slice(trailer, kBlockTrailerSize));
    if (r->status.ok()) {
      // r的偏移就要加上block的大小和结尾标识的大小了
      r->offset += block_contents.size() + kBlockTrailerSize;
    }
  }
}

Status TableBuilder::status() const {
  return rep_->status;
}


// 上述代码中都只是在文件中写入data block的内容，当table结束时就需要写入filter，index等信息了
Status TableBuilder::Finish() {
  Rep* r = rep_;
  Flush();
  assert(!r->closed);
  r->closed = true;
  // meta block就是filter block
  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

  // Write filter block
  // 将filter block不压缩，写入到文件中
  if (ok() && r->filter_block != NULL) {
    WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                  &filter_block_handle);
  }

  // Write metaindex block
  // meta index block只存了一个key-value，key是filter使用的算法名字，value是filter block的偏移信息
  // 作用是在table中快速定位到filter block
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != NULL) {
      // Add mapping from "filter.Name" to location of filter data
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    // TODO(postrelease): Add stats and other meta blocks
    // 将meta index block写入到file中
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  // Write index block
  // 
  if (ok()) {
    if (r->pending_index_entry) {
      // 如果最后一个key是最后data block的第一个key的话，需要将这个block的偏移写到index block中
      // 和Add函数类似
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }
    // 将index block写入到文件中
    WriteBlock(&r->index_block, &index_block_handle);
  }

  // Write footer
  if (ok()) {
    // footer里面记录了index block和meta index block的偏移信息
    Footer footer;
    // 将meta index block的偏移信息放在footer中
    footer.set_metaindex_handle(metaindex_block_handle);
    // 将index block的偏移信息放在footer中
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;
    // 将footer信息编码到string中
    footer.EncodeTo(&footer_encoding);
    // 然后将footer_encoding写入到文件中
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}
```

【作者：antonyxu   https://antonyxux.github.io/】
