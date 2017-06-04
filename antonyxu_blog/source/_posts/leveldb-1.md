---
title: Leveldb源码解析第一篇【Data Block】
date: 2017-05-30 17:30:25
categories: Leveldb源码解析
tags: 
  - leveldb
  - C++
---

> 版权声明：本文为博主原创文章，未经博主允许不得转载。

`leveldb` 作为一个 `key-value` 数据库，它和 `redis` 的区别在于不仅没有把所有的数据放在内存中，而是把大部分数据放在了磁盘中
#### `leveldb` 存数据的流程
1. 先指定一块内存写数据（这块内存称为 `MemTable`），当占用的内存高于阈值后，将这块内存转为只读（这块只读内存称为 `Immutable MemTable`）
2. 同时开辟一块新的内存（ `MemTable`）来写数据
3. 然后异步将 `Immutable MemTable` 的数据添加到存到磁盘中

本文将从数据怎么写入磁盘开始讲起（第 `3` 步中的数据存储到磁盘），数据写入磁盘时会先将数据放在一个名叫 `table` 的结构中，`table` 中包含有很多个 `block`，`block` 是真正用来存储 `key-value` 的
当 `table` 的大小大于指定阈值时，就会把 `table` 中的数据写入到磁盘中
#### `table` 结构
``` xml
    <beginning_of_file>
    [data block 1]
    [data block 2]
    ...
    [data block N]
    [meta block 1]
    ...
    [meta block K]
    [metaindex block]
    [index block]
    [Footer]        (fixed size; starts at file_size - sizeof(Footer))
    <end_of_file>
```
> 第一眼看着不明觉厉，这是个什么鬼玩意呢

其实 `table` 就是用来存放多个 `data block` 的，那问题来了，下面一堆 `meta block` / `metaindex block` / `index block` 是什么鬼？

> 当我们在 `table` 里面查询一个 `key`，在没有索引的情况下，就需要将所有的 `data block` 遍历一遍，如果按照这种方式找的话，十亿级的数据存储找到猴年马月才能找到，怎样才能**高效**的找到我们需要查询的 `key` 呢，就靠上面所说的那堆鬼东西，这里我们按下不表，后面会详细介绍（挖坑1）

**上面说了这么多，只是为了让大家有个基本的概念，本章重点要讲解的是 `table` 中 `data block`**

###  data block
万丈高楼平地起，我们现在来介绍 `Leveldb` 中最基础的结构，也是真正存储 `KV` 的结构 `data block`

>搞懂 `data block` 需要阅读如下源码文件

``` c++
1 table/block.h             // [非常重要|难度:3级] block的结构
2 table/block.cc
3 table/block_build.h       // [非常重要|难度:3级] 用于构建 block
4 table/block_build.cc
5 include/leveldb/slice.h   // [非常重要|难度:1级] leveldb 中用到的 string 都封装成了 slice 
6 util/coding.h             // [非常重要|难度:2级] 类型转换，编码转换
7 util/coding.cc
```
#### slice.h （简单了解后可跳过）
先介绍最简单的 `slice`，实际就是把 `string` 封装了一下，它拥有两个私有变量和几个简单的处理函数

``` c++
 public:
  char operator[](size_t n) const;  // 符号重载[]，返回第n个字符
  void remove_prefix(size_t n);     // 删除前n个字符
  std::string ToString();           // 转成string
  int compare(const Slice& b);      // 比较字符串大小
  bool starts_with(const Slice& x); // 将this.data_前面x.size_个字符替换成x.data_

 private:
  const char* data_;    // 存储字符串
  size_t size_;         // 记录data_的长度
```
>涉及的小知识  （专为像我这样的新手准备，请看 【Leveldb源码解析新手补充篇】）
- 符号重载
- `size_t` 类型
- `inline` 内联函数
- 函数后面 `const` 有啥作用

#### coding.h 和 coding.cc
`coding` 这个类是用来类型转换和压缩空间的，比如声明一个 `uint32`，我们要分配4个字节的空间，如果这个 `uint32` 只是存储数字 `1`，那就太浪费了，一个字节就能搞定，浪费 `3` 个字节，而 `Leveldb` 中记录长度的 `uint` 变量非常多，不压缩的话就会浪费大量的空间

> 这里只贴在 `data block` 部分中用到的两个函数，更多关于 `coding` 类的源码分析请看 【Leveldb源码解析工具篇之coding类】

``` c++
// coding.h
// 将 char* 转换为 uint32_t
inline uint32_t DecodeFixed32(const char* ptr) {
/* 大小端模式，不同机器内存中存储字节的方式不同，这里有两个概念，高/低地址和高/低字节，举个栗子：如果我们要存储 0x12345678
   高/低字节：从高位到低位的字节依次是0x12、0x34、0x56和0x78
   高/低地址：大端模式 高位高地址，低位低地址；小端模式 低位高地址，高位低地址
   大端模式：高地址 0x78|0x56|0x34|0x12 低地址
   小端模式：高地址 0x12|0x34|0x56|0x78 低地址 */
// 在Leveldb中不管是什么模式存储，全部转为小端模式
  if (port::kLittleEndian) {
    // Load the raw bytes
    uint32_t result;
    // 小端模式不用说，直接存即可
    memcpy(&result, ptr, sizeof(result));  // 32位的int占几个字节就将ptr的几个字节拷贝到result中，直接写4不就好了吗？
    return result;
  } else {
    // 大端模式就要转一下了，低位的换到高位，高位的换到低位，因为是转成uint_32_t，所以只有4个字节
    // 也就是将 0x78563412 转为 0x12345678，(0x78|0x56 << 8|0x34 << 16|0x12 << 24)
    return ((static_cast<uint32_t>(static_cast<unsigned char>(ptr[0])))
        | (static_cast<uint32_t>(static_cast<unsigned char>(ptr[1])) << 8)
        | (static_cast<uint32_t>(static_cast<unsigned char>(ptr[2])) << 16)
        | (static_cast<uint32_t>(static_cast<unsigned char>(ptr[3])) << 24));
  }
}
```
``` c++
// 将数字v重新编码，然后放到dst中
void PutVarint32(std::string* dst, uint32_t v) {
  char buf[5];      // 这里为啥是5呢，请看 EncodeVarint32函数
  // 重新编码 v
  char* ptr = EncodeVarint32(buf, v);
  // 将重新编码后的 v 放到 dst 中
  dst->append(buf, ptr - buf);
}

/* 编码的具体实现，看着有点花眼，其实挺有意思的，耐心往下看，会有种脑洞大开的感觉，这么神奇的办法他们是怎么想到的
   编码的目的是为将int存储小数字时所浪费的空间节省下来，有可能节省1个字节，有可能节省2个字节，也有可能节省3个字节
   节省的空间不同，int编码后占用的空间也就不固定了
   如果直接将int放在string中，那么在string中解析这个int时只需读取4个字节，然后再转成int就是我们所要的int值
   我们为什么知道这地方要读取4个字节，因为一个int占4个字节，读取4个字节这个int就结束了
   但是编码后的int大小不固定，在string中解析编码后的int，怎么知道要读几个字节
   一个字节是8位，大神将一个字节分为两个部分，最高位和低7位，低7位用来存数据，最高位用来表示有没有结束，举个栗子
   值为300的int变量，二进制为0x100101100，在内存中实际只用两个字节，浪费两个字节
   在内存中是这么存的           |0|0|0|0|0|0|0|1| 0|0|1|0|1|1|0|0|
   而编码后在内存中是这么存的    |0|0|0|0|0|0|1|0| 1|0|1|0|1|1|0|0|
   区别在哪儿呢，就是在第8位中插入一位，设置为1，为什么要怎么做
   我们上面有讲，一个字节中7位用来存数据，1位表示是否结束，当我们读到一个字节时，判断最高位是否为1
   如果是1，说明编码后的int还没有结束，需要继续读下一个字节，直到读到字节的最高位为0为止
   PutVarint32 中定义的char数组长度为啥是5呢，在编码过程中，每个字节会拿出1bit来存储于是否结束标识
   如果有一个int变量的值本身就占据4个字节，再加上每个字节最高位来存储结束标识，4个字节肯定是不够的，所以定义长度为5的字符数组来存储编码后的int
*/ 
char* EncodeVarint32(char* dst, uint32_t v) {
  // 将 char* 强转为 unsigned char*，这里有个疑问，为啥不直接定义成unsigned char*？
  unsigned char* ptr = reinterpret_cast<unsigned char*>(dst);
  // 128是不是有点眼熟呀，2的7次方，二进制写法为0x10000000
  static const int B = 128;
  if (v < (1<<7)) {                 // v小于128，说明7bit能存储，那么一个字节就能搞定，将ptr赋值为v，返回ptr的下一个地址
    *(ptr++) = v;
  } else if (v < (1<<14)) {         // 如果v小于(1<<14)，说明要两个字节来存储
    *(ptr++) = v | B;               // v|B是将v的第8位设置为1，然后赋值给ptr，这里只会赋值一个字节
    *(ptr++) = v>>7;                // 将v右移7位，实际上是为了得到第二个字节，下面的以此类推
  } else if (v < (1<<21)) {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = v>>14;
  } else if (v < (1<<28)) {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = v>>21;
  } else {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = (v>>21) | B;
    *(ptr++) = v>>28;
  }
  return reinterpret_cast<char*>(ptr);
}
```
>涉及的小知识  （专为像我这样的新手准备，请看 【Leveldb源码解析新手补充篇】）
- 指针类型转换

#### block
上面讲解了一下在 `block` 中用的工具类，现在开始进入主题，介绍一下 `block` 的结构（`table` 和 `block` 的结构一定要牢牢记住了）
```
    [K-V Record 0] <------------------
    [K-V Record 1]                   |
    ...                              |
    [K-V Record 16] <--------        |
    [K-V Record 17]         |        |
    ...                     |        |
    [K-V Record n-1]        |        |
    [restart point 0]-------|--------|
    [restart point 1]-------|
    ...
    [restart point (n+15)/16]
    [restart point number]
```
- `K-V Record` 里面存储的是 `key value` 值，在 `block` 中存储的 `key-value` 是顺序存储的，所以会出现多个连续的 `key` 有部分前缀是相同的，`leveldb` 为了压缩存储，会压缩前缀（具体请往后看），一行记录在 `block` 具体存储格式为
```
    [key中相同部分大小][key中不同部分大小][value大小][key中不同部分值][value值]
```
- `restart point` 在前缀压缩时，只记录了 `key` 中不同部分的数据，相同部分的数据需要从上一个 `key` 中获取，如果要获取第100个 `key` ，那么就需要把前 `99` 个 `key` 全部遍历出来，这样做的话会使查找 `key` 时效率很低，`leveldb` 的做法是相邻的 `16` 个 `key` 做一次前缀压缩，`restart point` 就是记录每 `16` 个 `key` 中第一个 `key` 的偏移量，举个栗子
``` xml
    0  [key中相同部分大小][key中不同部分大小][value大小][key中不同部分值][value值]
    1  [key中相同部分大小][key中不同部分大小][value大小][key中不同部分值][value值]
    2  [key中相同部分大小][key中不同部分大小][value大小][key中不同部分值][value值]
    ...
    16 [key中相同部分大小][key中不同部分大小][value大小][key中不同部分值][value值]
    17 [key中相同部分大小][key中不同部分大小][value大小][key中不同部分值][value值]
```
这里有 `18` 行记录，假如存储数据如下：（中间记录忽略）
``` xml
    第0行   [helloantony : antonyvalue]
    第1行   [helloatom   : atomvalue  ]
    第2行   [helloboy    : boyvalue   ]
    ...
    第16行  [worldantony : antonyvalue]
    第17行  [worldatom   : atomvalue  ]  
```
在 `data block` 中真正存储的情况如下
``` xml
    [0][11][11][helloantony][antonyvalue]
    [6][ 3][ 9][tom][atomvalue]
    [5][ 3][ 8][boy][boyvalue]
    ...
    [0][11][11][worldantony][antonyvalue]
    [6][ 3][ 9][tom][atomvalue]
```
我们会看到第 `0` 行和第 `16` 行的 `key` 是完整保存的，这两行也是重启点所指向的点，为了方便理解，上面的结构中一条记录用一行显示，但实际存储中这些记录都是存储在一个长字符串中，重启点记录的是第 `0` 行和第 `16` 行在字符串中的偏移量，通过这些偏移就能能快速定位到这两个点，可以理解为索引
> 重启点记录的偏移量怎么计算会在代码中详细讲解（挖坑2）

``` c++
// block.h
class Block {
 public:
  // Initialize the block with the specified contents.
  explicit Block(const BlockContents& contents);    // 构造方法，explicit关键字是不允许隐式构造，具体请看新手篇

  ~Block();

  size_t size() const { return size_; }
  Iterator* NewIterator(const Comparator* comparator);  // 返回一个迭代器，后面详细介绍

 private:
  uint32_t NumRestarts() const;    // 返回重启点个数

  const char* data_;               // 这个就是我们上面说的长字符串
  size_t size_;                    // block大小，占据多少个字节
  uint32_t restart_offset_;        // 第一个restart point在data_中的偏移量
  bool owned_;                     // data_是不是堆分配的内存，如果是的话就需要手动回收

  // No copying allowed
  Block(const Block&);
  void operator=(const Block&);

  class Iter;                      // 内部类，用来迭代 key-value
};
```
构造函数中出现了一个结构体 `BlockContents`，这个结构体是在 `table/format.h` 中定义的
``` c++
struct BlockContents {
  Slice data;           // 存储数据的字符串，类似于block中的data_
  bool cachable;        // True iff data can be cached ，不太懂这个变量，后续补充
  bool heap_allocated;  // data是不是堆分配的内存，如果是的话就需要手动回收
};
```
`block` 构造方法
``` c++
// block.cc
Block::Block(const BlockContents& contents)
    : data_(contents.data.data()),
      size_(contents.data.size()),
      owned_(contents.heap_allocated) {
  if (size_ < sizeof(uint32_t)) {       //block中data_最后的restart_num就已经有4个字节的，如果size_还小于4个字节的话，只能说明这个块是坏的
    size_ = 0;  // Error marker
  } else { 
    // size_-sizeof(uint32_t)的意思是data_减去最后4个字节（最后4个字节记录着有多少个重启点），减去以后剩下部分大小就是数据大小加上重启点偏移数组大小
    // 如果减去的部分全部都是重启点偏移量数组，一个重启点偏移量占据4个字节，那么最多有(size_-sizeof(uint32_t)) / sizeof(uint32_t)个重启点
    size_t max_restarts_allowed = (size_-sizeof(uint32_t)) / sizeof(uint32_t);
    // block中存储的重启点比最大重启点数还大，只能说明这个块有问题
    // 是不是感觉这个地方的写法很怪，我对比了一下leveldb初版中在此处的代码，发现之前不是这么写的，我怀疑是这个地方出现过问题，所以改成这样的
    if (NumRestarts() > max_restarts_allowed) {
      // The size is too small for NumRestarts()
      size_ = 0;
    } else {
      // 第一个restart point在data_中的偏移量就很好计算了，总大小减去重启点偏移数组大小，再减去重启点个数占据的大小
      restart_offset_ = size_ - (1 + NumRestarts()) * sizeof(uint32_t);
    }
  }
}

```
`block` 构造方法说完，这里仅仅是 `block` 的结构，真正来来操作这个结构的类是 `blockbuilder`
``` c++
// blockbuilder.h
class BlockBuilder {
 public:
  explicit BlockBuilder(const Options* options);
  // 重置block
  void Reset();
  // 往block中添加一个key-value
  void Add(const Slice& key, const Slice& value);
  // block的大小已经超过限制后的操作
  Slice Finish();
  // 估算出当前block的大小
  size_t CurrentSizeEstimate() const;

  bool empty() const {
    return buffer_.empty();
  }

 private:
  const Options*        options_;     // leveldb的一些配置，比如block的大小等等
  std::string           buffer_;      // Destination buffer
  std::vector<uint32_t> restarts_;    // 记录重启点偏移量的数组
  int                   counter_;     // 计数器，用于每16个key-value，就要开始记录一个完整的key
  bool                  finished_;    // Has Finish() been called?
  std::string           last_key_;    // 记录最后一个key，用于压缩前缀用

  // No copying allowed
  BlockBuilder(const BlockBuilder&);
  void operator=(const BlockBuilder&);
};

}  // namespace leveldb
```
``` c++
BlockBuilder::BlockBuilder(const Options* options)
    : options_(options),
      restarts_(),
      counter_(0),
      finished_(false) {
  assert(options->block_restart_interval >= 1);
  restarts_.push_back(0);       // 第一个重启点就是第一个key，偏移量就是 0
}
// 重置就和构造方法差不多了
void BlockBuilder::Reset() {
  buffer_.clear();
  restarts_.clear();
  restarts_.push_back(0);       // First restart point is at offset 0
  counter_ = 0;
  finished_ = false;
  last_key_.clear();
}
// buffer_中存储的是key-value，加上重启点偏移数组大小，再加上重启点个数大小，就是block估算的大小
size_t BlockBuilder::CurrentSizeEstimate() const {
  return (buffer_.size() +                        // Raw data buffer
          restarts_.size() * sizeof(uint32_t) +   // Restart array
          sizeof(uint32_t));                      // Restart array length
}

// 划重点
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  Slice last_key_piece(last_key_);
  assert(!finished_);
  // options_->block_restart_interval 默认为16，counter_如果大于16，那就说明异常了
  assert(counter_ <= options_->block_restart_interval);
  // buffer_为空或添加的key比上一个key小都会异常（顺序存储）
  assert(buffer_.empty() // No values yet?
         || options_->comparator->Compare(key, last_key_piece) > 0);
  size_t shared = 0; // key中相同部分大小默认为0
  // 当计数器没有达到16时，说明没来到下一个重启点
  if (counter_ < options_->block_restart_interval) {
    // See how much sharing to do with previous string
    // 没有到下一个重启点话就可以压缩了，首先得到两个的最小长度
    const size_t min_length = std::min(last_key_piece.size(), key.size());
    // 得到相同部分大小
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {  // 到了新的重启点
    // Restart compression
    // 先在重启点偏移量数组中记录当前buffer_的大小，也就是偏移量（填坑2），然后重新计数
    restarts_.push_back(buffer_.size());
    counter_ = 0;
  }
  // 得到key中不同部分的长度，如果是新的重启点的话shared是0，不同部分就是整个key的长度
  const size_t non_shared = key.size() - shared;

  // Add "<shared><non_shared><value_size>" to buffer_
  // 无论是shard还是non_shared一般都不会很大，大部分情况下用一个字节就可以存储，
  // 而我们在定义的时候用的是size_t，直接将size_t放在buffer中，大部分情况下会浪费多个字节
  // 这里就用到了coding中的编码算法，具体分析请看前面coding类中的解析
  // 对照前文中讲到的格式分析
  // [key中相同部分大小][key中不同部分大小][value大小][key中不同部分值][value值]
  PutVarint32(&buffer_, shared);
  PutVarint32(&buffer_, non_shared);
  PutVarint32(&buffer_, value.size());

  // Add string delta to buffer_ followed by value
  buffer_.append(key.data() + shared, non_shared);
  buffer_.append(value.data(), value.size());
 
  /*
     这个地方有点意思，为啥不直接将key赋值给last_key_，这让我纳闷的很久，隐约可以看出是为了考虑效率？
     我把自己的测试结果讲解一下，如有不对还请大家指出
     last_key_是string类型，string中有两个成员变量，size和capacity，size表示当前string的长度是多少，capacity是string真实容量
     如果我们直接赋值的话string会先将之前存储的数据释放，然后调用隐式构造为last_key_重新分配空间，效率低
     如果我们调用resize的话，last_key_仅仅是将shared后面的字节截掉，capacity不会发生改变，也不会重新分配空间
     再将non_shared部分添加到last_key_中，最后的效果一样，但是效率却千差万别
  */ 
  last_key_.resize(shared);
  last_key_.append(key.data() + shared, non_shared);
  assert(Slice(last_key_) == key);
  counter_++;
}

// block的大小大于阈值后就会调用finish，就是将重启点偏移数组和重启点个数放到buffer中
Slice BlockBuilder::Finish() {
  // Append restart array
  for (size_t i = 0; i < restarts_.size(); i++) {
    PutFixed32(&buffer_, restarts_[i]);
  }
  PutFixed32(&buffer_, restarts_.size());
  finished_ = true;
  return Slice(buffer_);
}

```

【作者：antonyxu   https://antonyxux.github.io/】
