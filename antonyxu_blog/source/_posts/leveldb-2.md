---
title: Leveldb源码解析第二篇【Meta Block】
date: 2017-06-01 19:35:05
categories: Leveldb源码解析
tags: 
  - leveldb
  - C++
---

> 版权声明：本文为博主原创文章，未经博主允许不得转载。

上一章中详细讲解了 `table` 中的 `data block` 的结构以及涉及的源码，本章中将讲解 `table` 结构中的 `meta block`

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

> 先说说 `meta block` 在 `table` 中的作用

一个meta block对应一个data block，meta block的作用是快速判断对应的data block中是否存在某个key，详情可以搜索“Bloom Filter”

原理是这样的，首先需要定义一个大的bitmap，实际就是一个字符串，bitmap中初始时每一位都是0，当往data block中添加key时，会根据这个key值算出一组hash值，hash值对bitmap位数取模后将bitmap中对应的位置设置为1；当需要查询data block中是否存在某个key时，只需通过这个key计算一组hash值，然后查看hash值在bitmap中对应的位置的值是否为1，只有有一个位置不为1，说明这个data block中不存在这个key

上面所说的算法中会存在hash冲突，如果bitmap中已经存了两个key

``` xml
    key1 计算出的位置为 [1,3]
    key2 计算出的位置为 [2,4]
```

当我们要查询的 `key3` 计算出来的位置为[1,4]时，在 `bitmap` 中[1,4]两个位置都是1，这个只能说明data block中有可能存在key3，所以说bitmap是用来快速判断key不在data block中，我们需要做的是使出现误判的概率降到最低，可以得到一个公式

假设一个key要对应k个hash值，总共有n个key，bitmap的位数为m，那么出现误判的概率为

```
(1-(1-1/m)^(kn))^k
```

上面的公式是怎么得到的呢？

假设我们现在要在bitmap中判断某个key是否存在，先要算出k个hash值，而这k个hash值对应的位置上面恰恰都有1的概率是多少呢

假设一个位置上面恰恰为1的概率为p，那么k个位置上面都为1的概率为p^k

p要怎么得到呢？

p代表的是一个位置上面恰恰为1的概率，我们可以先得到这个位置不为1的概率为q，那么，p=（1-q）

q要怎么得到呢？

q代表的是一个位置上面不为1的概率，说明n个key在计算k个hash的时候都没有落在这个点上，一次没有落在这个点上的概率为1-1/m，kn次没有落在这个点上的概率为(1-1/m)^(kn)，那么kn次落在这个点上的概率为1-(1-1/m)^(kn)，一共有k个点，k个点都是1的概率就为(1-(1-1/m)^(kn))^k

> 好多年不搞数学，上面的公式解释的好痛苦（-_-!!!）

为了保证误判的概率最低，如果m和n固定的话，可以得到k的最优解为k=m/n*ln2，我也不知道怎么算出来的，网上抄的（-_-!!!）

上面说了这么多理论，接下来要开始撸代码啦

> 搞懂 `meta block` 需要阅读如下源码文件
``` c++
1 table/filter_block.h             // [非常重要|难度:4  级] filter_block的结构
2 table/filter_block.cc

3 include/leveldb/filter_policy.h       // [重要|难度:2级] 过滤策略
4 util/bloom.cc                         // [重要|难度:2级] 过滤策略具体实现
```

#### filter_policy.h
先介绍 `filter_policy`，中文翻译为过滤策略，这个地方只是定义了一个接口，用户可以重写这个接口

``` c++
class FilterPolicy {
 public:
  virtual ~FilterPolicy();
  virtual const char* Name() const = 0;       //返回当前策略的名字

  // dst就是上面讲的bitmap，n表示一个key在bitmap中占多少位，这个函数就是通过传进来的keys，来构建一个bitmap
  virtual void CreateFilter(const Slice* keys, int n, std::string* dst) const = 0;

  // bitmap中有没有key
  virtual bool KeyMayMatch(const Slice& key, const Slice& filter) const = 0;
};

// Return a new filter policy that uses a bloom filter with approximately
// the specified number of bits per key.  A good value for bits_per_key
// is 10, which yields a filter with ~ 1% false positive rate.
//
// Callers must delete the result after any database that is using the
// result has been closed.
//
// Note: if you are using a custom comparator that ignores some parts
// of the keys being compared, you must not use NewBloomFilterPolicy()
// and must provide your own FilterPolicy that also ignores the
// corresponding parts of the keys.  For example, if the comparator
// ignores trailing spaces, it would be incorrect to use a
// FilterPolicy (like NewBloomFilterPolicy) that does not ignore
// trailing spaces in keys.
extern const FilterPolicy* NewBloomFilterPolicy(int bits_per_key);
}
```

#### bloom.cc
`filter_policy` 的具体实现

``` c++
class BloomFilterPolicy : public FilterPolicy {
 private:
  size_t bits_per_key_;   // 前文中讲到的n/m，bitmap的位数除以key的个数
  size_t k_;              // 前文中讲到的k，一个key在bitmap中要算出多少个hash

 public:
  explicit BloomFilterPolicy(int bits_per_key)
      : bits_per_key_(bits_per_key) {
    // 构造方法，直接给bits_pre_key_赋值
    // 根据前文讲到的k的最优解，k=n/m * ln2，ln2=~ 0.69，所以k的最优解为bits_per_key*0.69
    // We intentionally round down to reduce probing cost a little bit
    k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
    // k_肯定不能小于1，如果k_太大的话，每次计算hash会消耗计算资源，划不来，这个地方对误判率和计算资源做了权衡，k_最大为30
    if (k_ < 1) k_ = 1;
    if (k_ > 30) k_ = 30;
  }

  // 过滤策略的名称
  virtual const char* Name() const {
    return "leveldb.BuiltinBloomFilter2";
  }

  // dst就是上面讲的bitmap，n表示一个key在bitmap中占多少位，这个函数就是通过传进来的keys，来构建一个bitmap
  virtual void CreateFilter(const Slice* keys, int n, std::string* dst) const {
    // 计算出bitmap有多少位
    size_t bits = n * bits_per_key_;

    // For small n, we can see a very high false positive rate.  Fix it
    // by enforcing a minimum bloom filter length.
    // 如果n很小的话，误判率会很高，所以设置了一个最小值，64位，也就是8个字节
    if (bits < 64) bits = 64;

    // 加上7然后除以8，只要不是整除，就会进一位
    size_t bytes = (bits + 7) / 8;
    // 算出实际的bitmap有多少位
    bits = bytes * 8;

    const size_t init_size = dst->size();
    // 把bitmap放到dst后面，并用0填充，dst存放不止一个bitmap
    dst->resize(init_size + bytes, 0);
    // 将k_添加到dst的末尾，k_表示的是一个key算多少次hash
    dst->push_back(static_cast<char>(k_));  // Remember # of probes in filter
    // init_size是dst的初始长度，array指向当前bitmap的起始位置
    char* array = &(*dst)[init_size];
    // 又到了划重点的时间，这个地方就是将key转为hash值，然后将bitmap对应的点这是为1
    // n是key的个数
    for (int i = 0; i < n; i++) {
      // Use double-hashing to generate a sequence of hash values.
      // See analysis in [Kirsch,Mitzenmacher 2006].
      // 关于bloomHash这里就不多解释了，有时间我会专门写一篇关于hash的文章
      // 这里只需要知道BloomHash就是闯进去一个key，返回一个hash值，h就是返回的hash值
      uint32_t h = BloomHash(keys[i]);
      // (h >> 17) | (h << 15) 这个和hash算法有关，通过一系列的位运算来随机种子
      const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
      // k_表示的是一个key算多少次hash，如果用BloomHash来算hash的话，计算量太大，这里用到了一个简单的方法
      // 就是前几行中得到的随机种子再与h相加
      for (size_t j = 0; j < k_; j++) {
        // 得到要修改的位在第几个字节上
        const uint32_t bitpos = h % bits;
        // 取模 然后位移 再或上 定位到的字节
        array[bitpos/8] |= (1 << (bitpos % 8));
        h += delta;
      }
    }
  }

// 判断bitmap中有没有key
  virtual bool KeyMayMatch(const Slice& key, const Slice& bloom_filter) const {
    const size_t len = bloom_filter.size();
    if (len < 2) return false;

    const char* array = bloom_filter.data();
    // len-1是因为bitmap的最后一位存放的是k_，也就是一个key要计算多少次hash
    const size_t bits = (len - 1) * 8;

    // 得到最后一个自己里面存放的值
    const size_t k = array[len-1];
    if (k > 30) {
      // k的计算方法是,k=m/n*ln2，m表示bitmap的位数，n表示key的个数，如果k大于30，说明key很少
      // 既然key很少，遍历一遍data block也不会很耗资源
      // 但是得到30多个hash的话就比较费资源了，那还不如认为当前data block中有可能有这个key
      return true;
    }

    // 下面的代码就和CreateFilter中的差不多了，只要找到一个hash值对应的点不为1，那就说明key在这个data block中不存在
    uint32_t h = BloomHash(key);
    const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
    for (size_t j = 0; j < k; j++) {
      const uint32_t bitpos = h % bits;
      if ((array[bitpos/8] & (1 << (bitpos % 8))) == 0) return false;
      h += delta;
    }
    return true;
  }
};
}

// 暂时还不知道有啥用，后面补充
const FilterPolicy* NewBloomFilterPolicy(int bits_per_key) {
  return new BloomFilterPolicy(bits_per_key);
}
```
 
上面介绍一下bloom的具体实现，现在来介绍在table中的filter block怎么构建了，这个地方的命名我个人觉得有点奇怪，block.h中介绍的是block的结构，真正来构建一个block的类是block_builder，而filter_block.h就是用来构建filter_block的

filter_block.h中有两个类，一个是构建一个filter_block的，还有一个是来解析filter_block的

``` c++
// filter_block.h
// filter block构建类，一个FilterBlockBuilder对象并不是只存放一个
class FilterBlockBuilder {
 public:
  explicit FilterBlockBuilder(const FilterPolicy*);

  // 开始构建filter block
  void StartBlock(uint64_t block_offset);
  // 在table添加key的时候，有一个地方会专门存放key，当一个data block完成时，就会用存储的key来构建filter block
  void AddKey(const Slice& key);
  // 当一个filter block构建完成后，还需要做一些处理
  Slice Finish();

 private:
 // 真正构建filter block的地方
  void GenerateFilter();

  // 上面讲了那么多就是说的这个，里面有k_、bits_per_key_、filter block中是否存在key的方法和创建filter block的方法
  const FilterPolicy* policy_;
  // 把key通过追加的方式全部放在keys_中
  std::string keys_;              // Flattened key contents
  // 存放了每个key的索引，和keys_一起用可以得到每个key
  std::vector<size_t> start_;     // Starting index in keys_ of each key
  // 真正放bitmap的地方，table里面所有data block的bitmap全部都在这里面哟
  std::string result_;            // Filter data computed so far
  // 临时存放keys的地方，后面会提到
  std::vector<Slice> tmp_keys_;   // policy_->CreateFilter() argument
  // 和start_类似，用来存放每个bitmap的索引，其实就是在result_中的起始地址，和result一起使用可以得到具体的bitmap
  std::vector<uint32_t> filter_offsets_;

  // 只允许引用，不允许隐式构造
  FilterBlockBuilder(const FilterBlockBuilder&);
  void operator=(const FilterBlockBuilder&);
};

// filter block解析类
class FilterBlockReader {
 public:
 // REQUIRES: "contents" and *policy must stay live while *this is live.
  FilterBlockReader(const FilterPolicy* policy, const Slice& contents);
  bool KeyMayMatch(uint64_t block_offset, const Slice& key);

 private:
  const FilterPolicy* policy_;
  const char* data_;    // Pointer to filter data (at block-start)
  const char* offset_;  // Pointer to beginning of offset array (at block-end)
  size_t num_;          // Number of entries in offset array
  size_t base_lg_;      // Encoding parameter (see kFilterBaseLg in .cc file)
};
```

``` c++
// filter_block.cc
static const size_t kFilterBaseLg = 11;
static const size_t kFilterBase = 1 << kFilterBaseLg;

// 构造方法，policy_里面有k_、bits_per_key_、filter block中是否存在key的方法和创建filter block的方法
FilterBlockBuilder::FilterBlockBuilder(const FilterPolicy* policy)
    : policy_(policy) {
}

// 开始构建filter block，上一章我理解错了，不是一个data block构建一个filter block，而是2k的数据构建一个filter block
/* 
如果这一块的逻辑让我来写的话，逻辑应该是这样的
当一个data block构建完成后，调用StartBlock方法，分配一个的bitmap，
通过policy_里面的CreateFilter方法将这个data block里面的keys转换成bitmap的对应的位；
上面的是写入bitmap，怎么读取这个bitmap呢？
一个data block对应一个bitmap，如果要读bitmap，就需要知道table中bitmap对应的是第k个data block
当遍历table里面的data block时，就需要记录k值
然后得到第k个bitmap
*/
/*
上面是我个人的逻辑，再看看大神的逻辑
上面我们是通过得到是第k个data block从而得到bitmap
大神的方法通过data block在table中的偏移量来得到是第几个bitmap，方法如下
直观上理解是每2k的data block就分配一个bitmap，通过data block的偏移量，然后除以2，就可以得到是第几个bitmap了
那问题了，data block的大小阈值是2k，但并不是说data block大小就是2k，如果data block中存放了一个7k的key-value,
那在得到bitmap的时候怎么将这个data block分成多个2k呢？
【这里我们补充一个变量filter_offsets_，它是用来存放每个bitmap的偏移量，如果要得到第n个bitmap，
只需filter_offsets_[n]，得到第n个bitmap的偏移，然后result_+filter_offsets_[n]，就可以得到bitmap了】
还是上面7k的例子，如果除以2，需要分配3个bitmap，实际上第一个bitmap就是这个data block的索引了
那剩下的两个bitmap呢？剩下的两个bitmap是空的，这两个bitmap在filter_offsets_中指向的还是第一个bitmap的位置
相当于在filter_offsets_中占了两个位置，实际存放bitmap的result_啥也没存
如果第1个data block大小是7k，第2个data block大小是5k
那么第1个data block在table中的偏移量为7，7/2=3,在filter_offsets_中会有3条记录，不过存放的都是第一个bitmap的偏移
第2个data block在table中的偏移量为12，12/2=5,在filter_offsets_中会加2条记录，2条记录存放的都是第二个bitmap的偏移
现在得到第1个data block的bitmap，偏移量/2=3，filter_offsets_的第3条记录指向的是第1个bitmap的偏移，获取成功
得到第2个data block的bitmap，偏移量/2=5，filter_offsets_的第5条记录指向的是第2个bitmap的偏移，获取成功
*/
void FilterBlockBuilder::StartBlock(uint64_t block_offset) {
// filter_index bitmap索引个数
uint64_t filter_index = (block_offset / kFilterBase);
assert(filter_index >= filter_offsets_.size());
// 只要bitmap索引个数比当前索引个数大，就一直构建filter block，实际只有第一次循环的时候在result_中放了一个bitmap
// 其他循环都只是在filter_offsets_中存放第一次循环中得到的bitmap的偏移
  while (filter_index > filter_offsets_.size()) {
    GenerateFilter();
  }
}

// 将key放在keys_中，将key的大小放在start_中，构建filter data的时候用到
void FilterBlockBuilder::AddKey(const Slice& key) {
  Slice k = key;
  start_.push_back(keys_.size());
  keys_.append(k.data(), k.size());
}

// finish是在table构建时调用的
Slice FilterBlockBuilder::Finish() {
  if (!start_.empty()) {
    GenerateFilter();
  }

  // Append array of per-filter offsets
  // 当所有bitmap全部构建完成后，将所有的偏移量放到result_中
  const uint32_t array_offset = result_.size();
  for (size_t i = 0; i < filter_offsets_.size(); i++) {
    PutFixed32(&result_, filter_offsets_[i]);
  }

  // 最后再将所有bitmap的大小放到result_的后面
  PutFixed32(&result_, array_offset);
  // 将 2k 这个数字放在最后
  // 那么在解析result_的时候，读取最后一个字节，表示data block按多少k分割
  // 读取倒数第5到倒数第2个字节（size），表示所有bitmap的大小
  // result_+size就可以字节偏移到记录偏移点的地方，这有就可以一步一步解析了
  result_.push_back(kFilterBaseLg);  // Save encoding parameter in result
  
  return Slice(result_);
}

// 划重点
void FilterBlockBuilder::GenerateFilter() {
  // 得到当前block中key的个数
  const size_t num_keys = start_.size();
  // 如果没有key，说明之前已经创建过bitmap了，现在只需要把当前偏移量放到filter_offsets_中
  if (num_keys == 0) {
    // Fast path if there are no keys for this filter
    filter_offsets_.push_back(result_.size());
    return;
  }

  // Make list of keys from flattened key structure
  // start_存放的是所有key的起始位置，下面一步是把结束的位置也放进去
  start_.push_back(keys_.size());  // Simplify length computation
  // tmp_keys_上面再定义的时候没有解释，这里详细解释，就是用来存放当前data block中的key的，不过是以slice的形式存储的
  tmp_keys_.resize(num_keys);
  for (size_t i = 0; i < num_keys; i++) {
    const char* base = keys_.data() + start_[i];
    size_t length = start_[i+1] - start_[i];
    tmp_keys_[i] = Slice(base, length);
  }

  // Generate filter for current set of keys and append to result_.
  // 记录result_的大小，实际是记录下一个filter block的起始位置
  filter_offsets_.push_back(result_.size());
  // 得到filter block后放在result_里面
  policy_->CreateFilter(&tmp_keys_[0], static_cast<int>(num_keys), &result_);

  tmp_keys_.clear();
  keys_.clear();
  start_.clear();
}
```

```  c++
// 下面就是filter  block的解析过程了
FilterBlockReader::FilterBlockReader(const FilterPolicy* policy,
                                     const Slice& contents)
    : policy_(policy),
      data_(NULL),
      offset_(NULL),
      num_(0),
      base_lg_(0) {
  size_t n = contents.size();
  // contents的大小不可能小于5，最后一个字节记录kFilterBaseLg，然后4个字节记录所有bitmap的大小
  if (n < 5) return;  // 1 byte for base_lg_ and 4 for start of offset array
  // 这个在上面说过，最后一个字节是kFilterBaseLg
  base_lg_ = contents[n-1];
  // 然后得到倒数第5到倒数第2个字节，转为uint32，就可以得到所有bitmap的大小
  uint32_t last_word = DecodeFixed32(contents.data() + n - 5);
  if (last_word > n - 5) return;
  data_ = contents.data();
  // offset_就是记录偏移量的位置了
  offset_ = data_ + last_word;
  // num_记录有多少个偏移量，总大小减去最后5个字节，然后再减去bitmap的大小，剩下的就全是记录偏移量了
  // 一个偏移量占4个字节，(n - 5 - last_word) / 4 就表示有多少个偏移量了
  num_ = (n - 5 - last_word) / 4;
}

// 通过block的偏移量来判断 key 在不在这个block中
bool FilterBlockReader::KeyMayMatch(uint64_t block_offset, const Slice& key) {
  // block_offset 往右移11位，意思就是除以2k，也就是得到是第几个bitmap
  uint64_t index = block_offset >> base_lg_;
  if (index < num_) {
    // 然后再通过bitmap的偏移量得到真正的bitmap
    uint32_t start = DecodeFixed32(offset_ + index*4);
    uint32_t limit = DecodeFixed32(offset_ + index*4 + 4);
    if (start <= limit && limit <= static_cast<size_t>(offset_ - data_)) {
      Slice filter = Slice(data_ + start, limit - start);
      // 判断bitmap中有没有key
      return policy_->KeyMayMatch(key, filter);
    } else if (start == limit) {
      // Empty filters do not match any keys
      return false;
    }
  }
  return true;  // Errors are treated as potential matches
}
```

【作者：antonyxu   https://antonyxux.github.io/】
