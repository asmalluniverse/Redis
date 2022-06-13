### 0.官网地址
1. 官网地址  
英文：http://www.redis.io/   

中文：http://www.redis.cn/   

### 1.概览
Redis有以下这五种基本类型：
1. String（字符串）
2. Hash（哈希）
3. List（列表）
4. Set（集合）
5. zset（有序集合）
它还有三种特殊的数据结构类型
1. Geospatial
2. Hyperloglog
3. Bitmap

![redis数据结构](/图片/redis数据结构.png)

### 2.redis中数据接口介绍
##### 2.1 String
String是Redis最基础的数据结构类型，它是二进制安全的，可以存储图片或者序列化的对象，值最大存储为512M，常用命令如下：  
![String命令](/图片/String命令.png)

1. 官网命令地: http://www.redis.cn/commands.html#string
2. 应用场景：
共享Session、分布式锁、限流器、计数器。  
3. 实现结构
C语言的字符串是char[]实现的，而Redis使用SDS（simple dynamic string） 封装，sds源码如下：
```java
struct sdshdr{
  unsigned int len; // 标记buf的长度
  unsigned int free; //标记buf中未使用的元素个数
  char buf[]; // 存放元素的坑
}
```
在SDS的结构中会保存，存储的实际内容、已经使用的空间以及空闲的空间大小。那么就可以O(1)的时间复杂度获取到字符串长度，空间换时间的思想。  

**SDS特点：可动态扩展内存、二进制安全、快速遍历字符串 和与传统的C语言字符串类型兼容。**

- SDS数据结构的优缺点：
sds数据结构，采用空间预分配和惰性空间释放来提升效率，缺点就是耗费内存。
**空间预分配**：当一个sds被修改成更长的buf时，除了会申请本身需要的内存外，还会额外申请一些空间。  
**惰性空间**：当一个sds被修改成更短的buf时，并不会把多余的内存还回去，而是会保存起来。  
**总结**：这种设计的核心思想就是空间换时间，只要 free还剩余足够的空间时，下次string变长的时候，不会像系统申请内存。  

##### 2.2 List
列表（list）类型是用来存储多个有序的字符串，一个列表最多可以存储2^32-1个元素。常用命令如下：
![List命令](/图片/List命令.png)

1. 官网地址：http://www.redis.cn/commands.html#list

2. 应用场景：消息队列，文章列表,抢红包

3. list插入和弹出数据
![list插入和弹出数据](/图片/list插入和弹出数据.png)

4. 实现结构：ziplist（压缩列表）、linkedlist（链表）
在版本3.2之前，Redis 列表list使用两种数据结构作为底层实现（ziplist+linkedlist），之后是通过quickList进行实现。  

因为双向链表占用的内存比压缩列表要多，所以当创建新的列表键时，列表会优先考虑使用压缩列表，并且在有需要的时候，才从压缩列表实现转换到双向链表实现。可以认为quickList，是ziplist和linkedlist二者的结合；quickList将二者的优点结合起来。  

**压缩列表转化成双向链表条件**  
创建新列表时redis默认使用redis_encoding_ziplist编码，当以下任意一个条件被满足时，列表会被转换成redis_encoding_linkedlist编码：
1. 试图往列表新添加一个字符串值，且这个字符串的长度超过server.list_max_ziplist_value（默认值为 64 ）。
2. ziplist 包含的节点超过 server.list_max_ziplist_entries （默认值为 512 ）。
注意：这两个条件是可以修改的，在 redis.conf 中： `?	list-max-ziplist-value 64` ,`list-max-ziplist-entries 512` 这两个参数进行修改。

5. 压缩列表ziplist
压缩列表ziplist是为Redis节约内存而开发的。没有维护双向指针:prev next；而是存储上一个entry的长度和当前entry的长度，通过长度推算下一个元素在什么地方。
牺牲读取的性能，获得高效的存储空间，因为(简短字符串的情况)存储指针比存储entry长度 更费内存。这是典型的“时间换空间”。  
  
ziplist是由一系列特殊编码的内存块构成的列表(像内存连续的数组，但每个元素长度不同)，一个 ziplist可以包含多个节点（entry）。  
ziplist 将表中每一项存放在前后连续的地址空间内，每一项因占用的空间不同，而采用变长编码。当元素个数较少时，Redis用ziplist来存储数据，当元素个数超过某个值时，链表键中会把 ziplist 转化为 linkedlist，字典键中会把 ziplist 转化为 hashtable。  
由于内存是连续分配的，所以遍历速度很快。在3.2之后，ziplist被quicklist替代。但是仍然是zset底层实现之一。
![ziplist存储结构](/图片/ziplist存储结构.png)

6. quickList
可以认为quickList，是ziplist和linkedlist二者的结合；quickList将二者的优点结合起来。quickList是一个ziplist组成的双向链表。每个节点使用ziplist来保存数据。本质上来说，quicklist里面保存着一个一个小的ziplist。结构如下：
![quickList数据结构](/图片/quickList数据结构.png)

7. 这两种存储方式的优缺点
- 双向链表linkedlist便于在表的两端进行push和pop操作，在插入节点上复杂度很低，但是它的内存开销比较大。首先，它在每个节点上除了要保存数据之外，还要额外保存两个指针；其次，双向链表的各个节点是单独的内存块，地址不连续，节点多了容易产生内存碎片。
-	ziplist存储在一段连续的内存上，所以存储效率很高。但是，它不利于修改操作，插入和删除操作需要频繁的申请和释放内存。特别是当ziplist长度很长的时候，一次realloc可能会导致大批量的数据拷贝(连锁更新)。

##### 2.3 Hash 
在Redis中，哈希类型是指v（值）本身又是一个键值对（k-v）结构,简单使用举例：`hset key field value 、hget key field`。

1. 内部编码：ziplist（压缩列表） 、hashtable（哈希表） 
redis的哈希对象的底层存储可以使用ziplist（压缩列表）和hashtable。当hash对象可以同时满足以下两个条件时，哈希对象使用ziplist编码。  
- 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节
- 哈希对象保存的键值对数量小于512个

![hash存储结构](/图片/hash存储结构.png)

2. 应用场景：缓存用户信息等, 缓存购物车信息, 缓存商品详情, 做计数器。

3. 注意点：如果开发使用hgetall，哈希元素比较多的话，可能导致Redis阻塞，可以使用hscan。而如果只是获取部分field，建议使用hmget。

4. rehash操作：每个字典有两个hash表，一个平时使用，一个rehash的时候使用。rehash是渐进式的，分别是由以下触发：
- serveCron定时检测迁移。
- 每次kv变更的时候（新增、更新）的时候顺带rehash。
Redis 扩容采用的是渐进式Hash的方式进行扩容。原来的hash表是h[0]，然后加倍size生成一个新的hash表h[1]，然后在定时任务中，渐渐地将h[0]中的内容迁移到h[1] 中。
执行下面的操作时，内部的实现步骤：   
读操作：先去 h[0] 中找，找不到再去 h[1] 中去找。  
写操作：直接写在 h[1] 中  
hash冲突：采用单向链表的方式解决hash冲突，新的冲突元素会被放到链表的表头。  

##### 2.4 Set
集合（set）类型也是用来保存多个的字符串元素，但是不允许重复元素，结构如下：
![set存储结构](/图片/set存储结构.png)

1. 常用命令：
![set命令](/图片/set命令.png)
2. 内部编码：intset（整数集合）、hashtable（哈希表）
![set存储结构](/图片/set存储结构.png)

3. 应用场景：用户标签、生成随机数抽奖、社交需求。

set的底层为了实现内存的节约，会根据集合的类型而采用不同的数据结构来保存，当集合的元素都是整型且数量不多时会采用整数集合来存储。
##### 2.5 ZSet
已排序的字符串集合，同时元素不能重复
1. 常用命令：
![zset常用命令](/图片/zset常用命令.png)

2. 底层内部编码：ziplist（压缩列表）、skiplist（跳跃表）

![zset存储结构](/图片/zset存储结构.png)  
![跳表](/图片/跳表.png)  
3. 应用场景：排行榜，社交需求（如用户点赞），热点话题。

- 跳跃表是Redis特有的数据结构，就是在链表的基础上，增加多级索引提升查找效率。
- 跳跃表支持平均 O（logN）,最坏 O（N）复杂度的节点查找，还可以通过顺序性操作批量处理节点。
