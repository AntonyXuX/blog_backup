---
title: Leveldb源码解析第四篇【table添加key的流程】
date: 2017-06-04 16:48:45
categories: Leveldb源码解析
tags: 
  - leveldb
  - C++
---

> 版权声明：本文为博主原创文章，未经博主允许不得转载。

1. 添加一个key-value需要构造一个TableBuilder
2. 在构造TableBuilder时会构造一个Rep
  - Rep里面有BlockBuilder类型的data\_block和index\_block，还有FilterBlockBuilder类型的filter\_block
  - index\_block的重启点频率设置为1（默认是16）
  - filter\_block会调用StartBlock，传递的参数为0，这行貌似没啥用
3. 调用TableBuilder的Add函数
4. 首先判断这个key是不是data block的第一个key
5. 如果是，就要写入index block了，这里我们考虑一个极端情况，这个key是table的第一个key，那么再算上一个data block的最后一个key和当前key的中间key时，last\_key是为空的，在FindShortestSeparator中r->last\_key不会发生任何变化，也就是还是初始值，空值；上一个data block的偏移信息和大小都是0，那么index\_block中第一行的key为空，经过我的测试，任何一个字符串都不空大）
6. 在filter\_block中添加key，实际是在string类型的keys\_中追加key，然后在start\_中添加key的长度
7. 将当前key赋值给last\_key
8. data\_block中添加key-value，调用BlockBuilder::Add方法，将shared|non\_shared|value.size|key\_non\_shared\_data|value\_data放到buffer\_中
9. 如果data\_block的大小已经大于或等于阈值，那么就要刷新table了，这个地方data\_block的大小是什么计算出来的呢，buffer\_大小加上重启点个数乘以4再加上重启点个数占用的4个字节（这里的4个字节是大众机型，有可能有的机型uint32不是4个字节）
10. 如果data block小于阈值的话，那就继续上面的步骤，否则就开始flush吧；整个table可能当成是一个string，string先把所有的data block写完，然后在写其他东东  
  - 这里调用WriteBlock将data block放到一个长串中，调用的是block的Finish函数，长串中包含key-value、重启点和重启点个数；然后看是否要压缩这个长串，再将这个长串写入到文件中
  - 将r->pending\_handle的偏移设置为当前r的偏移（如果是第一个data block，那么现在r的偏移就是0），大小设置为写入的block长串的大小，然后写入长串，在写入五个字节的后缀，其中前4个字节是长串的crc信息，最后一个字节是这个长串有没有压缩
  - 将r的偏移设置为当前写入文件的长度
11. data\_block写完后开始构造filter\_block，也就是调用filter\_block的StartBlock函数，传入的参数是当前r的偏移，也就是此时r写入文件的大小
  - 算出要构造几个bitmap（在第2步中，构造rep的时候有调用StartBlock，但是没啥用，不知道为啥有这一步），构造几个bitmap就要调用几次GenerateFilter，如果是循环中的第一次调用，将当前result的大小加到filter\_offsets\_中（第一次是0），然后创建一个bitmap，bitmap最后一个字节存放的是一个key需要算多少次hash；并将bitmap放在filter的result\_中，result\_也是一个长字符串；接下来继续循环中的第二次，此时filter中的keys已经清空了，那么将此时result\_的大小添加到filter\_offsets\_中，后面还有循环的话也是这个做法
12. 如果table完成了的话，调用Finish函数
  - 先flush一下，将data block写入文件，同样的，继续构造bitmap
  - 再调用filter\_block的Finish函数，将filter\_block完结，调用一次GenerateFilter，将所有bitmap的大小写入到filter\_offsets\_中，将filter\_offsets\_中的偏移量放到result\_中，再将偏移量的个数写入到result\_中，最后追加一个字节放kFilterBaseLg
  - 将filter 的result\_写入到文件中，filter\_block\_handle的偏移存放r此时的偏移，也就是data block写完后指针指向的点，大小为result\_的大小，然后就和上面一样了，在写入5个字节，前四个是crc信息，最后一个是时候压缩
  - 写完filter后，开始写meta\_index\_block，meta\_index\_block是一个blockbuilder类型，只存放一个key-value，key的内容是filter的名字，value的信息是filter\_block\_handle的信息，也就是通过meta\_index\_block我可以快速得到所有data block的filter的信息，然后将meta\_index\_block写入到文件中
  - 接下来就是index\_block了，在index\_block中追加最后一个key，这个key要比last\_key大一丢丢，value的值是最后一个data block偏移和最后一个data block的大小
  - 将index\_block写入到文件中，最后加5个字节，和上面一样
13. 最后是footer，footer中存放了metaindex和index的偏移信息，将footer信息写入到文件中


【作者：antonyxu   https://antonyxux.github.io/ 】
