# Redis学习

## 概念

* **Redis**是互联网技术领域使用最为广泛的**存储中间件**
* **特点：**
  * 支持**数据持久化**，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用。
  * Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。
  * Redis支持数据的备份，即**master-slave模式**的数据备份。
* **优势：**
  * 性能高
  * 丰富数据类型
  * Redis的所有操作都是**原子性**的，意思就是要么成功执行要么失败完全不执行。
  * 丰富的特性

## 数据类型

### 基础数据结构

#### **string 简单动态字符串** 

* Redis最基本的数据类型，一个键对应一个值，一个键值最大存储512MB。

* **实现方式：**

  * 背景：

    * 首先C语言中并没有字符串类型，要实现的话只能使用char[]来实现，但是使用字符数组必须先给变量分配足够的空间，否则会溢出，分配多了又可能造成浪费

    * 如果要获取字符串的长度，就需要遍历字符数组，时间复杂度高O(n)

    * 字符串的长度更改会对字符数组的内存进行重新分配

    * C语言的 \0 是字符串的标志结束位，如果存储图片音频等多媒体文件的时候，存在二进制安全问题

      * **二进制安全**：该方法的参数可以包含任何字符，方法会公平的对待数据流的每个字符，不特殊处理其中某一个字符，包括特殊字符。

      * ~~~c
        // C语言中字符串是以特殊字符“\0”来作为字符串的结束标识
        String str = "0123456789\0123456789";
        // 对于上述字符串 使用如下方法 得到字符串长度为10
        strlen(str)=10
        // strlen()方法 特殊处理了 '/0' 所以不是二进制安全方法
        ~~~

  * SDS特点：

    * 无需担心内存溢出的问题，如果需要就对SDS进行扩容
    * 定义了len属性，获取字符串长度时间复杂度O(1)
    * 通过“空间预分配” 和“惰性空间释放”，防止多次重分配内存
    * 判断字符串是否结束是len属性

* string 类型是**二进制安全**的。意思是 redis 的 string 可以包含任何数据。比如jpg图片或者序列化的对象。

  * **Redis中的二进制安全：**

    * Redis 3.2 之前的SDS主要是通过**C语言中结构体**类型确定的

      * ~~~c
        struct sdshdr {  
            //buf中已占用字节数
            int len;  
            //buf剩余可用字节数
            int free;  
            //实际保存字符串的字符数组
            char buf[];  
        }; 
        ~~~

      * 由于有长度的统计变量len的存在，读写字符串时不依赖“\0”终止符，保证了二进制安全

      * Redis保存的字符串对外暴露的是数组的长度指针，而不是结构体的指针，上层可以像操作普通字符串一样操作SDS。

* **杜绝缓冲区溢出：**

  * 此时需要给s1 追加一个“boy”， 如果是C字符串，忘记了在追加之前先给s1 分配空间，此时追加将导致 s2的值被意外的修改。 而使用 sds则不会有这个问题。 因为其封装好的函数，会在追加数据之前先检查 空间是否够用，如果不够用就扩容。
  * ![https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/%E7%BC%93%E5%86%B2%E5%8C%BA%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/%E7%BC%93%E5%86%B2%E5%8C%BA%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA.png)

* **内存分配：**

  * 当给sds的值追加一个字符串，而当前的剩余空间不够时，就会触发sds的扩容机制。**扩容采用了空间预分配的优化策略**，即分配空间的时候：如果sds 值大小< 1M ,则增加一倍； 反之如果>1M , 则当前空间加1M作为新的空间。
  * 当sds的字符窜缩短了，sds的buf内会多出来一些空间，这个空间并不会马上被回收，而是暂时留着以防再用的时候进行多余的内存分配。**这个是惰性空间释放的策略**

* **应用场景：**

  * 统计网站访问数量
  * 当前在线人数

* **实现方式：**

  * String在redis内部存储默认就是一个字符串，被redisObject所引用。
  
  * ~~~c
    typedef struct redisObject {
        // 类型
        unsigned type:4;
        // 编码
        unsigned encoding:4;
        // 对象最后一次被访问的时间
        unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
        // 引用计数
        int refcount;
        // 指向实际值的指针
        void *ptr;
        
    } robj;
    ~~~
  
  * ![https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/SDS%E6%A8%A1%E5%9E%8B.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/SDS%E6%A8%A1%E5%9E%8B.png)

#### **list 简单字符串列表**

* Redis的列表允许用户从序列的两端推入或者弹出元素，列表由多个字符串值组成的有序可重复的序列，是链表结构，可以充当栈和队列的角色，适合存储对象。

* **特点：**

  * 向列表两端添加元素的时间复杂度为0(1)，获取越接近两端的元素速度就越快
  * List中可以包含的最大元素数量是4294967295
  * 列表中的元素是有序的，可以通过索引下标来获取某个元素或者某个范围内的元素列表
  * 列表中的元素是可以重复的

* **应用场景：**

  * 最新消息排行榜 
  * 消息队列 以完成多程序之间的消息交换。可以用push操作将任务存在list中（生产者），然后线程在用pop操作将任务取出进行执行。

* **实现方式：**

  * **List是一个双向链表**

    * ~~~c
      struct listNode {
        // 指向前一个节点的指针
        listNode * prev;
        // 指向后一个节点的指针
        listNode * next;
        // 本节点的值
        void * value;
      } listNode;
      
      struct list {
        // 队列第一个节点的指针
        listNode * head;
        // 队列最后一个节点的指针
        listNode * tail;
        // 队列中元素的数量
        unsigned long len;
        // 节点值复制函数
        void *(*dup) ( void * ptr );
        // 释放一个节点
        void (*free) ( void * ptr );
        // 两个节点对比
        void (*match) ( void * ptr, void * key );
      } list;
      ~~~

    * **List获取长度 O(1)**

    * **当list键里包含的元素较少、并且每个元素要么是小整数要么是长度较小的字符串时，redis将会用ziplist作为list键的底层实现。**

#### 题外话1 zipList

**zipList 压缩表：**

* **ziplist**是**list键、hash键以及zset键**的底层实现之一（**3.0之后list键已经不直接用ziplist和linkedlist作为底层实现了，取而代之的是quicklist**）

* **ziplist**是由**一系列特殊编码的连续内存块组成的顺序存储结构**，类似于数组，ziplist在内存中是连续存储的，但是不同于数组，为了节省内存 ziplist的**每个元素所占的内存大小可以不同**（数组中叫元素，ziplist叫节点entry，下文都用“节点”），每个节点可以用来存储一个整数或者一个字符串。
* **ziplist** 是为了**节省内存**而设计出来的一种数据结构。**ziplist**是由一系列特殊编码组成的连续内存块的顺序型数据结构，一个 **ziplist** 可以包含任意多个 **entry**，而每一个 **entry** 又可以保存一个字节数组或者一个整数值。
  * **ziplist** 作为一种列表，其和普通的双端列表，如 **linkedlist** 的最大区别就是 **ziplist**并不存储前后节点的指针，而 **linkedlist** 一般每个节点都会维护一个指向前置节点和一个指向后置节点的指针。那么 **zipList** 不维护前后节点的指针，它又是如何寻找前后节点的呢？
    * `ziplist` 虽然不维护前后节点的指针，但是它却维护了上一个节点的长度和当前节点的长度，然后每次通过长度来计算出前后节点的位置。既然涉及到了计算，那么相对于直接存储指针的方式肯定有性能上的损耗，这就是一种典型的用「时间来换取空间」的做法。因为每次读取前后节点都需要经过计算才能得到前后节点的位置，所以会消耗更多的时间，而在 `Redis` 中，一个指针是占了 `8` 个字节，但是大部分情况下，如果直接存储长度是达不到 `8` 个字节的，所以采用存储长度的设计方式在大部分场景下是可以节省内存空间的。

**zipList布局：**

* ![](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/20210530162914153.png)
* **zlbytes**: 32 位无符号整型，记录 ziplist **整个结构体的占用空间大小**。当然了也包括 zlbytes 本身。
  * 这个结构有个很大的用处，**就是当需要修改 ziplist 时候不需要遍历即可知道其本身的大小**。 这个 SDS 中记录字符串的长度有相似之处。
* **zltail**: 32 位无符号整型, 记录整个 ziplist 中最后一个 entry 的偏移量。所以在尾部进行 POP 操作时候不需要先遍历一次。
* **zllen**: 16 位无符号整型, 记录 entry 的数量， 所以只能表示 2^16。但是 Redis 作了特殊的处理：当实体数超过 2^16 ,该值被固定为 2^16 - 1。 所以这种时候要知道所有实体的数量就必须要遍历整个结构了。
* **entry**: 真正存数据的结构。
* zlend: 8 位无符号整型, 固定为 255 。为 ziplist 的结束标识。

**Entry**

* **Entry的组成：**
  * **prelen**：前一个entry的存储大小 ，**目的是为了方便从后向前遍历**
    * 记录前一个 entry 的长度。若前一个 entry 的长度小于 254 , 则使用 1 个字节的 8 位无符号整数来表示。
    * 若前一个 entry 长度大于等于 254，则使用 5 个字节来表示。第 1 个字节固定为 254 (FE) 作为标识，剩余 4 字节则用来表示前一个 entry 的实际大小。
  * **encoding**：**数据的编码形式，分辨是数字还是字符串，同时记录长度**
  * **data**：实际存储的数据
* ziplist在内存中是**高度紧凑的连续存储**，这意味着它起始对修改并不友好，**如果要对ziplist做修改类的操作，那就需重新分配新的内存来存储新的ziplist**，代价很大

**总结：**

* ziplist 为了节省内存，采用了紧凑的连续存储。所以在**修改操作**下并不能像一般的链表那么容易，需要**从新分配新的内存**，然后复制到新的空间。
* ziplist 是一个**双向链表**，可以在时间复杂度为O(1)从下头部、尾部进行pop或push。
* 可能会出现**连锁更新**现象。
  * **连锁更新：**前面说过，每个节点的previous_entry _length 属性都记录了前一个节点的长度：_
    * _(1)如果前一节点的长度小于254 字节，那么previ ous_ entry_length 属性需要用1字节长的空间来保存这个长度值。
    * (2)如果前一节点的长度大于等于254 字节，那么previous entry length 属性需要用5 字节长的空间来保存这个长度值。
      * 如果我们将一个长度大于等于 254 字节的新节点 new 设置为压缩列表的表头节点，那么麻烦的事情来了，由于previous entry length大小不够用(1->5B)，后面所有的节点可能都要重新分配内存大小。因为连锁更新在最坏情况下需要对压缩列表执行 N 次空间重分配操作， 而每次空间重分配的最坏复杂度为 O(N) ， 所以连锁更新的最坏复杂度为 O(N^2) 。

#### 题外话2 quickList

* 上次说到ziplist每次变更的时间复杂度都非常高，因为必须要重新生成一个新的ziplist来作为更新后的list，如果一个list非常大且更新频繁，那就会给redis带来非常大的负担。如何既保留ziplist的空间高效性，又能不让其更新复杂度过高？ redis的作者给出的答案就是quicklist。

* 其实说白了就是把**ziplist和普通的双向链表结合**起来。每个**双链表节点中保存一个ziplist**，然后**每个ziplist中存一批list中的数据**(具体ziplist大小可配置)，这样既可以**避免大量链表指针带来的内存消耗**，也可以**避免ziplist更新导致的大量性能损耗**，将大的ziplist**化整为零**。

  * 双向链表便于在表的两端进行push和pop操作，但是它的内存开销比较大。首先，它在每个节点上除了要保存数据之外，还要额外保存两个指针；其次，双向链表的各个节点是单独的内存块，地址不连续，节点多了容易产生内存碎片。
  * ziplist由于是一整块连续内存，所以存储效率很高。但是，它**不利于修改操作**，每次数据变动都会引发一次内存的realloc。特别是当ziplist长度很长的时候，一次realloc可能会导致大批量的数据拷贝，进一步降低性能。

* **quickList总体实现：**

  * ![https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/20190506153148861.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/20190506153148861.png)
  * **quickList实现问题：**
    * **quick如何处理zipList的长度？**
      * 从**存储效率**上讲：
        * **每个quicklist节点上的ziplist越短，则内存碎片越多**。内存碎片多了，有可能在内存中产生很多无法被利用的小碎片，从而降低存储效率。这种情况的极端是每个quicklist节点上的ziplist只包含一个数据项，这就蜕化成一个普通的双向链表了。
        * **每个quicklist节点上的ziplist越长，则为ziplist分配大块连续内存空间的难度就越大**。有可能出现内存里有很多小块的空闲空间（它们加起来很多），但却找不到一块足够大的空闲空间分配给ziplist的情况。这同样会降低存储效率。这种情况的极端是整个quicklist只有一个节点，所有的数据项都分配在这仅有的一个节点的ziplist里面。这其实蜕化成一个ziplist了。
      * 对此，Redis提供一个**list-max-ziplist-size**参数进行调整：
        * 当取**正值**的时候，表示按照数据项个数来限定每个quicklist节点上的ziplist长度。
          * 比如，当这个参数配置成5的时候，表示每个quicklist节点的ziplist最多包含5个数据项。
        * 当取**负值**的时候，表示按照占用字节数来限定每个quicklist节点上的ziplist长度。
          * -5: 每个quicklist节点上的ziplist大小不能超过64 Kb。（注：1kb => 1024 bytes）
          * -4: 每个quicklist节点上的ziplist大小不能超过32 Kb。
          * -3: 每个quicklist节点上的ziplist大小不能超过16 Kb。
          * -2: 每个quicklist节点上的ziplist大小不能超过8 Kb。（-2是Redis给出的默认值）
          * -1: 每个quicklist节点上的ziplist大小不能超过4 Kb。
    * **题外话：Redis会对中间部分（不容易访问的数据）进行压缩 采用LZF压缩算法（一种无损压缩算法）**
      * **当列表很长的时候，最容易被访问的很可能是两端的数据，中间的数据被访问的频率比较低（访问起来性能也很低）**。如果应用场景符合这个特点，那么list还提供了一个选项，能够把中间的数据节点进行压缩，从而进一步节省内存空间。
      * Redis的配置参数**list-compress-depth**就是用来完成这个设置的。
        * 这个参数表示**一个quicklist两端不被压缩的节点个数**。
          * 注：这里的节点个数是指quicklist双向链表的节点个数，而不是指ziplist里面的数据项个数。实际上，**一个quicklist节点上的ziplist，如果被压缩，就是整体被压缩的。**
        * 参数list-compress-depth的取值含义如下：
          - 0: 是个特殊值，表示都不压缩。这是Redis的默认值。
            - 由于0是个特殊值，很容易看出quicklist的头节点和尾节点总是不被压缩的，以便于在表的两端进行快速存取。
          - 1: 表示quicklist两端各有1个节点不压缩，中间的节点压缩。
          - 2: 表示quicklist两端各有2个节点不压缩，中间的节点压缩。
          - 3: 表示quicklist两端各有3个节点不压缩，中间的节点压缩。
          - 依此类推…

* **quickList内部实现：**

  * ~~~c
    typedef struct quicklistNode {
        struct quicklistNode *prev;
        struct quicklistNode *next;
        unsigned char *zl;           /* quicklist节点对应的ziplist */
        unsigned int sz;             /* ziplist的字节数 */
        unsigned int count : 16;     /* ziplist的item数*/
        unsigned int encoding : 2;   /* 数据类型，RAW==1 or LZF==2 */
        unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
        unsigned int recompress : 1; /* 这个节点以前压缩过吗？ */
        unsigned int attempted_compress : 1; /* node can't compress; too small */
        unsigned int extra : 10; /* 未使用到的10位 */
    } quicklistNode;
    ~~~

    * prev: 指向链表前一个节点的指针。
    * next: 指向链表后一个节点的指针。
    * zl: 数据指针。如果当前节点的数据没有压缩，那么它指向一个ziplist结构；否则，它指向一个quicklistLZF结构。
    * sz: 表示zl指向的ziplist的总大小（包括zlbytes, zltail, zllen,
      zlend和各个数据项）。需要注意的是：如果ziplist被压缩了，那么这个sz的值仍然是压缩前的ziplist大小。
    * count: 表示ziplist里面包含的数据项个数。这个字段只有16bit。稍后我们会一起计算一下这16bit是否够用。
    * encoding:表示ziplist是否压缩了（以及用了哪个压缩算法）。目前只有两种取值：2表示被压缩了（而且用的是LZF压缩算法），1表示没有压缩。
    * quicklistLZF结构表示一个被压缩过的ziplist。其中：
      - sz: 表示压缩后的ziplist大小。
      - compressed: 是个柔性数组（flexible array member），存放压缩后的ziplist字节数组。

  * ~~~c
    typedef struct quicklist {
        quicklistNode *head;        /* 头结点 */
        quicklistNode *tail;        /* 尾结点 */
        unsigned long count;        /* 在所有的ziplist中的entry总数 */
        unsigned long len;          /* quicklist节点总数 */
        int fill : QL_FILL_BITS;              /* 16位，每个节点的最大容量 */
        unsigned int compress : QL_COMP_BITS; /* 16位，quicklist的压缩深度，0表示所有节点都不压缩，否则就表示从两端开始有多少个节点不压缩 */
        unsigned int bookmark_count: QL_BM_BITS;  /*4位，bookmarks数组的大小，bookmarks是一个可选字段，用来quicklist重新分配内存空间时使用，不使用时不占用空间*/
        quicklistBookmark bookmarks[];
    } quicklist;
    ~~~

    * head: 指向头节点（左侧第一个节点）的指针。
    * tail: 指向尾节点（右侧第一个节点）的指针。
    * count: 所有ziplist数据项的个数总和。
    * len: quicklist节点的个数。
    * fill: 16bit，ziplist大小设置，存放list-max-ziplist-size参数的值。
    * compress: 16bit，节点压缩深度设置，存放list-compress-depth参数的值。

* **quickList操作：**

  * **quickList头尾插入：**
    * 如果头节点（或尾节点）上ziplist大小没有超过限制（即_quicklistNodeAllowInsert返回1），那么新数据被直接插入到ziplist中（调用ziplistPush）。
    * 如果头节点（或尾节点）上ziplist太大了，那么新创建一个quicklistNode节点（对应地也会新创建一个ziplist），然后把这个新创建的节点插入到quicklist双向链表中（调用_quicklistInsertNodeAfter）。
  * **quickList任意位置插入：**
    * 当插入位置所在的ziplist大小没有超过限制时，直接插入到ziplist中就好了；
    * 当插入位置所在的ziplist大小超过了限制，但插入的位置位于ziplist两端，并且相邻的quicklist链表节点的ziplist大小没有超过限制，那么就转而插入到相邻的那个quicklist链表节点的ziplist中；
    * 当插入位置所在的ziplist大小超过了限制，但插入的位置位于ziplist两端，并且相邻的quicklist链表节点的ziplist大小也超过限制，这时需要新创建一个quicklist链表节点插入。
    * 对于插入位置所在的ziplist大小超过了限制的其它情况（主要对应于在ziplist中间插入数据的情况），则需要把当前ziplist分裂为两个节点，然后再其中一个节点上插入数据。
  * **quickList头尾删除：**
    * 先从头部或尾部节点的ziplist中把对应的数据项删除，如果在删除后ziplist为空了，那么对应的头部或尾部节点也要删除。删除后还可能涉及到里面节点的解压缩问题。

#### **hash 字典**

* 哈希类型是指键值对里的value本身存储的也是一个个的KV键值对，类似于python中的dict和java中的map集合。
  * hash_value={undefined{field1，value1}，...{fieldN，valueN}}	
    * hkey-->hvalue
    * hvalue{k1:v1 ,k2:v2 ,k3:v3...}
  
* **Hash内部实现**

  * 其内部键值对**小于512个**时，内部结构采用zipList实现；

    * ![https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/190457129-5daf1f99b0061_fix732.jpg](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/190457129-5daf1f99b0061_fix732.jpg)

  * 否则内部采用**字典dict**实现

    * ~~~c
      typedf struct dict{
          dictType *type;//和特定类型键值对相关的函数；
          void *privdata;//上述特定函数的可选参数；
          dictht ht[2];//两张hash表 
          int rehashidx;//rehash索引，字典没有进行rehash时，此值为-1
          unsigned long iterators; //正在迭代的迭代器数量
      }dict;
      ~~~
      
      * type 是一个指向 dict.h/dictType 结构的指针，保存了一系列用于操作特定类型键值对的函数；
      * privdata 保存了需要传给上述特定函数的可选参数；
      * ht[2] 两个hash表，使用两个hash表的作用之后会说明。
      * rehashidx 用于标记rehash的进度，若当前这个值为-1，则表示字典没有在执行rehash操作。
      * Iterators 记录正在迭代的迭代器的数量。
      
    * ![https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/20201208211414684.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/20201208211414684.png)
    
    * ~~~c
      typedf struct dictht{
          dictEntry **table;//存储数据的数组 二维
          unsigned long size;//数组的大小
          unsigned long sizemask;//哈希表的大小的掩码，用于计算索引值，总是等于size-1
          unsigned long used; 哈希表中中元素个数
      }dictht;
      ~~~
    
      * table 是一个二维数组，第一维度数组表示hash表的槽位，第二个维度是每一个槽对应的链表。因为是采用拉链法来解决冲突的，所以存在相同槽位的数据，会以链表的形式连接在一起。
      * size 表示数组的大小，也就是槽位的数量。
      * sizemask 哈希表的大小的掩码，用于计算索引值。
      * used 记录hash表中实际存放元素的个数。
    
    * ~~~c
      typedf struct dictEntry{
          void *key;//键
          union{
              void val;
              unit64_t u64;
              int64_t s64;
              double d;
          }v;//值
          struct dictEntry *next；//指向下一个节点的指针
      }dictEntry;
      ~~~
    
      * ![https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/2237074585-5daf1f994e36a_fix732.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/2237074585-5daf1f994e36a_fix732.png)
    
    * Hash处理hash冲突采用的方法为**拉链法**
    
      * ![https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/1924496847b55736e579e27cf0e3f163.jpg](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/1924496847b55736e579e27cf0e3f163.jpg)

* **Hash的扩容与缩容**

  * hash表的扩容是为了减少hash冲突的概率，当hash表中的数据逐渐增多的时候，会导致冲突的概率增大，从而导致每个槽位下的链表的长度会变长。那么就会影响到查询的效率了。
  * 而hash表的缩容是为了减少空间的消耗。Redis的数据是保存在内存当中的，若一个hash表占用很大的空间，里面的数据却很少，那么是极度的浪费，所以需要缩容操作。
  * **扩容与缩容：**
    * 如果执行**扩展**操作，会基于原哈希表创建一个大小等于 ht[0].used*2n 的哈希表（也就是每次扩展都是根据原哈希表**已使用的空间扩大一倍**创建另一个哈希表）。
    * 相反如果执行的是**收缩**操作，每次收缩是根据**已使用空间缩小一倍**创建一个新的哈希表。
      * 重新利用上面的哈希算法，计算索引值，然后将键值对放到新的哈希表位置上。
      * 所有键值对都迁徙完毕后，释放原哈希表的内存空间。
    * **扩容原则：**
      * 服务器目前没有执行 BGSAVE（异步保存数据库数据到磁盘） 命令或者 BGREWRITEAOF 命令，并且负载因子大于等于1。
      * 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且负载因子大于等于5。
        * ps：负载因子 = 哈希表已保存节点数量 / 哈希表大小。
  * **rehash**
    * java当中需要新建一个hash表，然后一次性的将旧表里的数据进行rehash到新的hash表当中，之后在释放掉原油的hash表。而这一过程的时间复杂度达到了**O(n)**。
    * Redis为单线程方式执行请求，其无法接受O(n)的复杂度，其采用的是**渐进式rehash**。
      * 首先当需要扩容或者缩容的时候，会根据上面提到的规则，在dictht[1]当中分配足够的空间。
      * 然后在dict当中维护一个变量，也就是前面提到的rehashidx，用于标记rehash的进度，将其初始化为0，表示rehash正式开始。
      * rehash进行期间，每次对字典执行添加、删除、查找或者更新操作的时候，除了执行指定的操作之外，还会顺带将dictht[0] hash表当中在rehashidx索引上的所有键值对进行rehash到dictht[1]当中，当一次rehash工作完成之后，会将rehashidx的值+1。
      * 同时在循环时间事件serverCron当中，会调用rehash相关函数，在1ms的时间内，进行rehash处理，每次仅处理少量的转移任务（100个元素）
        随着字典操作的不断执行，最终在某个时间点上，dictht[0]当中所有的键值对都会被rehash到dictht[1]当中，此时将rehashidx属性值设置为-1，表示rehash操作已经完成，将dictht[0]重新赋值dictht[1],接着清空dictht[1]。
    * **在rehash过程中CRUD操作过程**
      * 增加操作：直接将数据添加到dictht[1]当中
      * 修改操作：首先寻找数据在不在dictht[0]当中，若存在，则修改，否则去dictht[1]当中去找，若存在则修改。
      * 删除操作：和上面修改一样，先要定位到元素所在哪个hash表当中，然后执行删除操作。
      * 查找操作：和上面寻找的过程一样。
    * **优势与劣势：**
      * **优点**：采用了**分而治之**的思想，将 rehash 操作分散到每一个对该哈希表的操作上以及定时函数上，避免了集中式rehash 带来的性能压力。
      * **缺点**：在 rehash 的时间内，需要保存两个 hash 表，对内存的占用稍大，而且如果在 redis 服务器本来内存满了的时候，突然进行 rehash 会造成大量的 key 被抛弃。
  
* **应用场景**：

  * 存储对象：缓存对象信息：e.g.用户信息，商品信息...
  * 购物车：每个用户的购物车都是一个哈希表,这个哈希表存储了商品ID与商品订购数量之间的映射。在商品的订购数量出现变化时,我们操作Redis哈希对购物车进行更新:
    * ``` HSET uid pid num```
  * 计数器：记录网站每天，每月，每年访问的数量
    * ```HINCRBY blogName Month num```
  * 记录点赞数，评论数...

#### Set 无序集合

* ~~~c
  typedf struct inset{
      uint32_t encoding;//编码方式 有三种 默认 INSET_ENC_INT16
      uint32_t length;//集合元素个数
      int8_t contents[];//实际存储元素的数组 
                        //元素类型并不一定是ini8_t类型，柔性数组不占intset结构体大小，并且数组中的元
                        //素从小到大排列
  }inset;
  #define INTSET_ENC_INT16 (sizeof(int16_t))   //16位，2个字节，表示范围-32,768~32,767
  #define INTSET_ENC_INT32 (sizeof(int32_t))   //32位，4个字节，表示范
                                               //围-2,147,483,648~2,147,483,647
  #define INTSET_ENC_INT64 (sizeof(int64_t))   //64位，8个字节，表示范
  //围-9,223,372,036,854,775,808~9,223,372,036,854,775,807
  ~~~

* 底层使用了intset和hashtable两种数据结构存储的，intset我们可以理解为数组，hashtable就是普通的哈希表（key为set的值，value为null）。

  * set的底层存储intset和hashtable是存在编码转换的，使用**intset**存储必须满足下面两个条件，否则使用hashtable，条件如下：
    * 结合对象保存的所有元素都是整数值
    * 集合对象保存的元素数量不超过512个
  * intset可能会随着数据的添加而改变它的数据编码：
    1. 最开始，新创建的intset使用占内存最小的INTSET_ENC_INT16（两字节）作为数据编码。
    2. 每添加一个新元素，则根据元素大小决定是否对数据编码进行升级。
  * **intset的升级与降级**
    * 如下图为一个保存了五个元素intset，且ecoding编码为INTSET_ENC_INT16(两字节,所以该编码能保存的范围为![-2^{15}](https://private.codecogs.com/gif.latex?-2%5E%7B15%7D)~![2^{15}-1](https://private.codecogs.com/gif.latex?2%5E%7B15%7D-1))
      * ![https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/20190702175024479.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/20190702175024479.png)
      * 我们插入一个 32768，由于 32768 = ![2^{15}](https://private.codecogs.com/gif.latex?2%5E%7B15%7D) 已经超出了INTSET_ENC_INT16表示范围，所以这时会引起集合的升级。值得注意的是 集合升级不仅把当前元素升级为32位，数组其他元素也会升级为32位(四个字节)，如下图：
      * ![https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/20190702175059274.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/Redis/20190702175059274.png)
    * **整数集合不支持降级操作**，一旦对数组进行了升级，编码就会一直保持升级后的状态。如上图，当我们把32768删除以后该数组还是INTSET_ENC_INT32编码。
  * **zipList与intset的区别**
    * ziplist可以存储任意二进制串，而intset只能存储整数。
    * ziplist是无序的，而intset是**从小到大有序**的。因此，在ziplist上查找只能遍历，而在intset上可以进行**二分查找**，性能更高。
    * ziplist可以对每个数据项进行不同的变长编码（每个数据项前面都有数据长度字段len），而intset只能整体使用一个统一的编码（encoding）。

* **应用场景：**

  * 统计独立信息
  * 共同好友...

#### 题外话3  跳跃表

* skipList本质上是一个list，有序链表

* 背景：

  * 我们先来看一个有序链表，如下图（最左侧的灰色节点表示一个空的头结点）：
    * ![https://img-blog.csdnimg.cn/20200427221324575.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/20200427221324575.png)
  * 在这样一个链表中，如果我们要查找某个数据，那么需要从头开始逐个进行比较，直到找到包含数据的那个节点，或者找到第一个比给定数据大的节点为止（没找到）。也就是说，时间复杂度为**O(n)**。同样，当我们要插入新数据的时候，也要经历同样的查找过程，从而确定插入位置
    * 假如我们每相邻两个节点增加一个指针，让指针指向下下个节点，如下图：
    * ![https://img-blog.csdnimg.cn/20200427221348478.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/20200427221348478.png)
    * 这样所有新增加的指针连成了一个新的链表，但它包含的节点个数只有原来的一半（上图中是7, 19, 26）。现在当我们想查找数据的时候，可以先沿着这个新链表进行查找。当碰到比待查数据大的节点时，再回到原来的链表中进行查找。比如，我们想查找23，查找的路径是沿着下图中标红的指针所指向的方向进行的：
    * ![https://img-blog.csdnimg.cn/20200427221407406.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/20200427221407406.png)
      * 23首先和7比较，再和19比较，比它们都大，继续向后比较但23和26比较的时候，比26要小，因此回到下面的链表（原链表），与22比较
      * 23比22要大，沿下面的指针继续向后和26比较
      * 23比26小，说明待查数据23在原链表中不存在，而且它的插入位置应该在22和26之间
  * 在这个新的三层链表结构上，如果我们还是查找23，那么沿着最上层链表首先要比较的是19，发现23比19大，接下来我们就知道只需要到19的后面去继续查找，从而一下子跳过了19前面的所有节点。可以想象，当链表足够长的时候，这种多层链表的查找方式能让我们跳过很多下层节点，大大加快查找的速度
    * skiplist正是受这种多层链表的想法的启发而设计出来的。实际上，按照上面生成链表的方式，上面每一层链表的节点个数，是下面一层的节点个数的一半，这样查找过程就非常类似于一个二分查找，使得查找的时间复杂度可以降低到**O(log n)**。但是，这种方法在插入数据的时候有很大的问题。**新插入一个节点之后，就会打乱上下相邻两层链表上节点个数严格的2:1的对应关系。\**如果要维持这种对应关系，就必须把新插入的节点后面的所有节点（也包括新插入的节点）重新进行调整，这会让时间复杂度重新\**退化成O(n)**。删除数据也有同样的问题。

* **skipList的解决方式：**

  * skiplist为了避免这一问题，它不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是**为每个节点随机出一个层数**(level)。比如，一个节点随机出的层数是3，那么就把它链入到第1层到第3层这三层链表中。为了表达清楚，下图展示了如何通过一步步的插入操作从而形成一个skiplist的过程

    * ![https://img-blog.csdnimg.cn/20200427221625808.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hlbGxvd29ybGRfcHR0,size_16,color_FFFFFF,t_70](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/20200427221625808.png)

  * 从上面skiplist的创建和插入过程可以看出，**每一个节点的层数（level）是随机出来的**，而且新插入一个节点不会影响其它节点的层数。因此，插入操作只需要修改插入节点前后的指针，而不需要对很多节点都进行调整。这就**降低了插入操作的复杂度**。实际上，这是skiplist的一个很重要的特性，这让它在插入性能上明显优于平衡树的方案。

    * 刚刚创建的这个skiplist总共包含4层链表，现在假设我们在它里面依然查找23，下图给出了查找路径：
    * ![https://img-blog.csdnimg.cn/20200427221646349.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hlbGxvd29ybGRfcHR0,size_16,color_FFFFFF,t_70](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/20200427221646349.png)

  * **skipList生成随机数的方式：**

    * 首先，每个节点肯定都有第1层指针（每个节点都在第1层链表里）

    * 如果一个节点有第i层(i>=1)指针（即节点已经在第1层到第i层链表中），那么它有第(i+1)层指针的**概率为p**

    * 节点最大的层数不允许超过一个最大值，记为**MaxLevel**

    * ~~~c
      randomLevel()
          level := 1
          // random()返回一个[0...1)的随机数
          while random() < p and level < MaxLevel do
              level := level + 1
          return level
      ~~~

      * 在Redis中 **p = 1/4**  **MaxLevel = 32**

  * **skipList与BTree，hashTable的区别**

    * skipList与各种平衡树的元素是**有序的**，而哈希不是，所以哈希表**不适合做范围查找（查找两点之间的值）**
    * 在做范围查找的时候，平衡树比skiplist操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现
    * 平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而**skiplist的插入和删除只需要修改相邻节点的指针**，操作简单又快速
    * 从内存占用上来说，skiplist比平衡树更灵活一些。一般来说，平衡树每个节点包含2个指针（分别指向左右子树），而skiplist每个节点包含的指针数目平均为1/(1-p)，具体取决于参数p的大小。如果像Redis里的实现一样，取p=1/4，那么平均每个节点包含1.33个指针，比平衡树更有优势
    * **查找单个key，skiplist和平衡树的时间复杂度都为O(log n)**，大体相当；而哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近O(1)，性能更高一些。所以我们平常使用的各种Map或dictionary结构，大都是基于哈希表实现的
    * 从算法实现难度上来比较，skiplist比平衡树要简单得多

#### zset 有序集合

* 有序集合类型 (Sorted Set或ZSet) 相比于集合类型多了一个排序属性 score（分值），对于有序集合 ZSet 来说，每个存储元素相当于有两个值组成的，一个是有序结合的元素值，一个是排序值。有序集合保留了集合不能有重复成员的特性(分值可以重复)，但不同的是，有序集合中的元素可以排

  ![image-20220125092925101](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220125092925101.png)

* **内部实现：**

  * zset由**zipList**或**skipList跳跃表**组成
    * 数据较少时，采用**zipList**，需满足下面两个条件
      * 有序集合保存元素个数要小于128个
      * 有序集合保存所有元素成员长度都必须小于64字节
    * 否则采用**skipList**实现

* **应用场景：**

  * **排行榜系统：**
    * 添加用户赞
      * ``` zadd user:ranking arcticle1 10```
    * 取消用户赞（将赞数从榜单中减1）
      * ```zincrby user:ranking arcticle1 -1```
    * 查看某篇文章的赞数
      * ```zscore user:ranking arcticle1```
    * 获取赞数最多的十篇文章
      * ```zrevrange user:ranking 0 9```
  * **电话号码排序**

## 持久化

* 在实际开发中，为了**保证数据的完整性**，防止数据丢失，我们除了在原有的传统数据库保存数据的同时，最好是再用redis持久化再保存一次数据。如果仅仅是使用redis而不进行持久化配置的话，当redis重启之后，并不能达到保存数据的目的。因此开始redis持久化是很必要的步骤。

#### RDB（半持久化）

*  此模式是**Redis默认的备份方式**，通过**快照**方式，将数据从内存写入磁盘中，如果Redis异常退出，下次启动则从打快照的这个时间节点来恢复此之前的数据，根据数据量大小、结构、服务器性能不同，通常将记录1千万个string类主键、大小为1GB的快照文件载入内存中需要20~30s
*  **Redis实现快照的过程**：
  * Redis使用fork函数复制一份当前进程（父进程）的副本（子进程）；
  * 父进程继续接收并处理客户端发来的命令，而子进程开始将内存中的数据写入硬盘中的临时文件；
    * 在父进程fork出子进程时，它们共享内存中的数据，当父进程接受命令请求要修改某片数据时，为了使子进程数据不受影响，这时Redis会有一种策略，就是写时复制（copy-on-write）
      * 原理是：在父进程要改动某片数据时，操作系统会把此片数据先copy一份给子进程，以保证子进程的内存数据不受影响，所以新的rdb文件就是父进程fork子进程时那一刻的内存数据。
  * 当子进程写入完所有数据后会用该临时文件替换旧的 RDB 文件，至此一次快照操作完成。
    * dump.rdb文件可以进行压缩（rdbcompression），节省占用空间、方便传输，也可以禁用压缩节省cpu工作负载。除了自动执行快照，还可以通过手动save和bgsave来执行快照，两者区别是，save是由主进程来进行快照操作，会阻塞其它请求，bgsave是通过fork子进程来操作。
* **配置RDB方式**
  * 打开配置文件**redis.conf**
  * 找到**save**，如图：
    * ![https://img-blog.csdn.net/20180427154856258?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xKRlBIUA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70](https://img-blog.csdn.net/20180427154856258?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xKRlBIUA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
    * **含义：**这是配置文件默认的策略，他们之间的关系是或，每隔900秒，在这期间变化了至少一个键值，做快照。或者每三百秒，变化了十个键值做快照。或者每六十秒，变化了至少一万个键值，做快照。
    * 按需修改配置文件即可
* **手动快照**
  * 如果没有触发自动快照,需要对redis执行手动快照的操作,使用SAVE或BGSAVE都是执行手动快照的命令
  * 区别：前者是由主进程进行快照,会阻塞其他应用,后者是通过fork子进程进行快照操作
* **RDB方式的优劣**
  * **优势：**
    * RDB 是一个非常**紧凑**（compact）的文件，它保存了 Redis 在某个时间点上的数据集。 这种文件非常适合用于进行备份： 比如说，你可以在最近的 24 小时内，每小时备份一次 RDB 文件，并且在每个月的每一天，也备份一个 RDB 文件。 这样的话，即使遇上问题，也可以随时将数据集还原到不同的版本。RDB 非常适用于**灾难恢复**（disaster recovery）：它只有一个文件，并且内容都非常紧凑，可以（在加密后）将它传送到别的数据中心，或者亚马逊 S3 中。RDB 可以最大化 Redis 的性能：父进程在保存 RDB 文件时唯一要做的就是 fork 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘 I/O 操作。**RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快**。
  * **劣势：**
    * **通过RDB方式实现持久化，一旦Redis异常退出，就会丢失最后一次快照以后更改的所有数据**，此时需要开发者根据具体的应用场合，通过组合设置自动快照条件的方式来将可能发生的数据损失控制在能接受的范围内。

#### AOF（日志）

* redis会将每一个收到的写命令都通过write函数追加到文件中(默认是 appendonly.aof)。AOF就可以做到全程持久化，只需要在配置文件中开启（默认是no不开启的）,开启AOF之后，（默认是：每秒fsync一次）
* **AOF过程：**
  * Redis 执行 fork() ，现在同时拥有父进程和子进程。
  * 子进程开始将新 AOF 文件的内容写入到临时文件。对于所有新执行的写入命令，父进程一边将它们累积到一个内存缓存中，一边将这些改动追加到现有 AOF 文件的末尾：
  *  这样即使在重写的中途发生停机，现有的 AOF 文件也还是安全的。当子进程完成重写工作时，它给父进程发送一个信号，父进程在接收到信号之后，将内存缓存中的所有数据追加到新 AOF 文件的末尾。
  * 现在 Redis 原子地用新文件替换旧文件，之后所有命令都会直接追加到新 AOF 文件的末尾。
* **AOF持久化配置**
* **同时启用RDB和AOF，进行恢复时，默认AOF文件的优先级高于RDB文件，即会使用AOF文件进行恢复**
* **AOF模式的优劣：**
  * **优势：**
    * **数据安全性相对较高**，根据使用的fsync策略（fsync是同步内存中redis所有已经修改的文件到存储设备），默认是appendfsync everysec，即每秒执行一次fsync，在这种配置下，redis仍然可以保持良好的性能，并且就算发生故障停机，最多指挥丢失一秒钟的数据（fsync会在后台线程执行，所以主线程可以继续努力的处理命令请求）
    * 即使出现宕机现象，也不会丢失破坏文件中已经存在的内容。然而如果本次操作只是写入了一半数据就出现了系统崩溃问题，不用担心，在redis下一次启动之前，可以通过redis-check-aof工具来解决数据一致性的问题
  * **劣势：**
    * aof文件的大小要大于rdb格式的文件
    * aof在恢复大数据集时的速度比rdb的恢复速度要慢
    * 根据fsync策略不同，aof速度可能会慢于rdb
    * bug出现的可能性更多

#### RDB与AOF的选择

* 如果主要充当缓存功能，或者可以承受数分钟的数据丢失，通常生产环境一般只需启用rdb即可，此选项也是默认值
* 如果数据需要持久化保存，一点不能丢失，可以选择同时开启rdb和aof，一般不建议只开启aof

## 缓存一致性

* 什么是**一致性**？

  * **强一致性：**
    * 要求系统写入什么，读出来的也会是什么，用户体验好，但实现起来往往对系统的性能影响大
  * **弱一致性：**
    * 系统在写入成功后，不承诺立即可以读到写入的值，也不承诺多久之后数据能够达到一致，
       但会尽可能地保证到某个时间级别（比如秒级别）后，数据能够达到一致状态
  * **最终一致性：**
    * 最终一致性是弱一致性的一个特例，系统会保证在一定时间内，能够达到一个数据一致的状态。

* 缓存可以提升性能、缓解数据库压力，但是使用缓存也会导致数据**不一致性** 的问题。一般我们是如何使用缓存呢？**有三种经典的缓存使用模式：**

  * **Cache-Aside Pattern 边缘缓存模式** 

    * **读流程**

      * ![https://img-blog.csdnimg.cn/20210719145448810.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bW1lcl9maXNo,size_16,color_FFFFFF,t_70](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/20210719145448810.png)
      * 首先去缓存中取 没有从数据库中读 然后更新缓存数据 同时返回数据

    * **写流程**

      * ![https://img-blog.csdnimg.cn/20210719145903403.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/20210719145903403.png)

      * **缓存失效**

        * 该情况下，当请求需要更新数据库数据的时候，缓存中的值需要被删除掉（删除掉就表示旧值不可用了），当下次该key被再次查询到就去数据库中查出最新的数据，在Spring中可实现如下：

        * ~~~java
          @CacheEvict("default", key="#search.keyword)
          public Record updateRecordForSearch(Search search)
          ~~~

      * **缓存更新**

        * 缓存数据也可以在数据库更新的时候被更新，从而在一次操作中让之后的查询有更快的查询体验和更好的数据一致性，在Spring中可实现如下：

        * ~~~java
          @CachePut("default", key="#search.keyword)
          public Record updateRecordForSearch(Search search)
          ~~~

      * **为了应对不用类型的数据需要，有以下缓存加载策略可被选择：**

        * **使用时加载缓存**：当需要使用缓存数据时，就从数据库中把它查询出来，第一次查询之后，接下来的请求都能从缓存中查询到数据。
        * **预加载缓存**：在项目启动的时候，预加载类似“国家信息、货币信息、用户信息，新闻信息”等不是经常变更的数据。

  * **Read-Through/Write-through 读写穿透**

    * **Read/Write-Through** 模式中，服务端把缓存作为主要数据存储。应用程序跟数据库缓存交互，都是通过**抽象缓存层** 完成的。

    * **Read-Through**

      * ![https://img-blog.csdnimg.cn/20210719151726918.png](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/20210719151726918.png)
      * 这个简要流程跟**Cache-Aside** 很像，其实**Read-Through** 就是多了一层**Cache-Provider** 而已，不同点在于程序不需要再去管理从哪去读数据（缓存还是数据库）。相反它会直接从缓存中读数据，该场景下是缓存去决定从哪查询数据。当我们比较两者的时候这是一个优势因为它会让程序代码变得更简洁。
      * ![https://img-blog.csdnimg.cn/202107191505594.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bW1lcl9maXNo,size_16,color_FFFFFF,t_70](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/202107191505594.png)

    * **Write-Through**

      * 当发生写请求时，也是由**缓存抽象层** 完成数据源和缓存数据的更新。**Write-behind**下所有的写操作都经过缓存，每次我们向缓存中写数据的时候，缓存会把数据持久化到对应的数据库中去，且这两个操作都在一个事务中完成。因此，只有两次都写成功了才是最终写成功了。这的确带来了一些写延迟但是它保证了数据一致性。

      * 同时，因为程序只和缓存交互，编码会变得更加简单和整洁，当你需要在多处复用相同逻辑的时候这点变的格外明显。

      * 当使用`Write-Through`的时候一般都配合使用`Read-Through`。

        `Write-Through`适用情况有：

        - 需要频繁读取相同数据
        - 不能忍受数据丢失（相对`Write-Behind`而言）和数据不一致

        **`Write-Through`的潜在使用例子是银行系统。**

  * **Write-behind 异步缓存写入**

    * **Write-behind** 跟Read-Through/Write-Through有相似的地方，都是由**Cache Provider** 来负责缓存和数据库的读写。它们又有个很大的不同：**Read/Write-Through** 是同步更新缓存和数据的，**Write-Behind** 则是只更新缓存，不直接更新数据库，通过**批量异步** 的方式来更新数据库
    * ![https://img-blog.csdnimg.cn/20210719150945998.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1bW1lcl9maXNo,size_16,color_FFFFFF,t_70](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/20210719150945998.png)
    * 此模式**对一致性要求高的系统要谨慎使用**，但是它适合**频繁写**的场景。

## 常见问题及处理方案

#### 缓存雪崩

* **问题**
  * 当缓存服务器重启或者大量缓存集中在某一个时间段失效，这样在失效的时候，也会给后端系统(比如DB)带来很大压力。
* **解决**
  * **批量往redis存数据的时候，把每个key的失效时间加上个随机数，这样就能保证数据不会在同一个时间大面积失效。**

#### 缓存穿透

* **问题**
  * 一般的缓存系统，都是按key去缓存查询。如果不存在相应的value，就去DB中查询。如果key对应的value一定不存在，但key的并发请求量很大，就会对后端系统造成很大压力，**因为每次都绕开了缓存直接查询DB**。
* **解决**
  * **在接口层增加校验**，不合法的参数直接返回。不相信任务调用方，根据自己提供的API接口规范来，作为被调用方，要考虑可能任何的参数传值。
  * 在缓存查不到，DB中也没有的情况，可以**将对应的key的value写为null**，或者其他特殊值写入缓存，同时将过期失效时间设置短一点，以免影响正常情况。这样是可以防止反复用同一个ID来暴力攻击。
  * 正常用户是不会这样暴力功击，只有是恶意者才会这样做，可以在网关NG作一个配置项，为每一个IP设置访问阀值。
  * **高级用户布隆过滤器**（Bloom Filter),这个也能很好地防止穿透。原理就是利用高效的数据结构和算法快速判断出你这个Key是否在DB中存在，不存在你return就好了，存在你就去查了DB刷新KV再return。

#### 缓存击穿

* **问题**
  * 与雪崩类似，但是又不一样。雪崩是因为大面积缓存失效，请求全打到DB；而击穿是指一个key是热点，并发请求全都集中访问此key,而当此key过期瞬间，持续的并发就击穿缓存，全都打在DB上。进而又引发雪崩的问题。
* **解决**
  * 设置热点key不过期。或者加上互斥锁。
