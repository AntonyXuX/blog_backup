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
    // 如果n很小的话，误判率会很高，
    if (bits < 64) bits = 64;

    size_t bytes = (bits + 7) / 8;
    bits = bytes * 8;

    const size_t init_size = dst->size();
    dst->resize(init_size + bytes, 0);
    dst->push_back(static_cast<char>(k_));  // Remember # of probes in filter
    char* array = &(*dst)[init_size];
    for (int i = 0; i < n; i++) {
      // Use double-hashing to generate a sequence of hash values.
      // See analysis in [Kirsch,Mitzenmacher 2006].
      uint32_t h = BloomHash(keys[i]);
      const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
      for (size_t j = 0; j < k_; j++) {
        const uint32_t bitpos = h % bits;
        array[bitpos/8] |= (1 << (bitpos % 8));
        h += delta;
      }
    }
  }

  virtual bool KeyMayMatch(const Slice& key, const Slice& bloom_filter) const {
    const size_t len = bloom_filter.size();
    if (len < 2) return false;

    const char* array = bloom_filter.data();
    const size_t bits = (len - 1) * 8;

    // Use the encoded k so that we can read filters generated by
    // bloom filters created using different parameters.
    const size_t k = array[len-1];
    if (k > 30) {
      // Reserved for potentially new encodings for short bloom filters.
      // Consider it a match.
      return true;
    }

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

const FilterPolicy* NewBloomFilterPolicy(int bits_per_key) {
  return new BloomFilterPolicy(bits_per_key);
}
```
 