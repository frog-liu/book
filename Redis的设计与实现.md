[TOC]

## Redis的设计与实现

##### SDS (simple dynamic string)

1.  结构定义

    ```c
    struct sdshdr {
        // 记录buf数组中已经使用的字节数量 = 等于SDS保存的字符串长度
    	int len;
        // 记录buf数组中未使用的字节数量
        int free;
        // 字节数组，用于保存字符串，以空字符结尾，但是不计入数量
        char buf[];
    }
    ```

2.  SDS的空间分配策略

    -   **空间预分配**：如果SDS进行拼接操作，buf的预留空间不够，则自动进行内存重分配操作；buf扩容后的大小，以SDS的长度而定，如果拼接后SDS的len小于1MB，则buf拓展为2len + 1，即len = free；如果len > 1MB, 则 free = 1MB。

    -   **惰性空间释放**：如果对SDS进行截取操作，则SDS不会对减少buf的空间，而是增加SDS的free值。

        **空间预分配**和**惰性空间释放**可以减少SDS拼接或截取操作导致的内存重分配操作。

3.  SDS相对于C语言字符串的优点

    -   额外记录字符串长度，可以使得获取字符串长度时间复杂度降为O(1);
    -   SDS的空间分配策略可以杜绝缓冲区溢出以及减少空间重分配操作；
    -   支持任意格式的二进制的数据，也支持数据里面存在空字符串；

##### 链表

1.  结构定义

    -   链表节点

    ```c
    typedef struct listNode {
        // 前置节点
        struct listNode *prev;
        // 后置节点
        struct listNode *next;
        // 节点的值
        void *value;
    }
    ```

    -   链表

    ```c
    typedef struct list {
        // 表头节点
        listNode *head;
        // 表尾节点
        listNode *tail;
        // 链表所包含的节点数量
        unsigned long len;
        // 节点值复制函数
        void *(*dup)(void *ptr);
        // 节点值释放函数
        void (*free)(void *ptr);
        // 节点值对比函数
        int (*match)(void *ptr, void *key);
    }
    ```

2.  特点

    -   双向：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的时间复杂度为O(1);
    -   无环：表头节点的prev和表尾节点的next都为null，链表遍历以null为终止条件；
    -   包含表头节点、表尾节点、链表长度
    -   多态：链表节点使用void*指针来保存节点值，并且通过list的dup、free、match三个属性为节点值设置类型特定函数，所有链表可以用来保存各种不同类型的值；

##### 字典

1.  结构定义

    -   哈希表节点

    ```c
    typedef struct dictEntry {
        // 键
        void *key;
        // 值
        union{
            void *val;
            unit64_tu64;
            int64_ts64;
        } v;
        struct dictEntry *next;
    }
    ```

    -   哈希表

    ```c
    typedef struct dictht {
        // 哈希表数组
        dictEntry **table;
        // 哈希表大小
        unsigned long size;
        // 哈希表大小掩码=size-1
        unsigned long sizemask;
        // 该哈希表已有节点的数量
        unsigned long used;
    }
    ```

    -   字典

    ```c
    typedef struct dict {
        // 类型待定函数
        dictType *type;
        // 私有数据
        void *privdata;
        // 哈希表 常使用ht[0],ht[1]进行rehash时会使用
        dictht ht[2];
        // rehash索引;当rehash不在进行时，值为-1
        int rehashidx;
    }
    ```

    -   数据字典类型

    ```c
    typedef struct dictType {
        // 计算哈希值的函数
        unsigned int (*hashFunction)(const void *key);
        // 复制键的函数
        void *(*keyDup)(void *privdata, const void *key);
        // 复制值的函数
        void *(*valDup)(void *privdata, const void *obj);
        // 对比键的函数
        int (*keyCompare)(void *privdata, const void *key1, const void *key2);
        // 删除键的函数
        void (*keyDestructor)(void *privdata, void *key);
        // 删除值得函数
        void (*valDestructor)(void *privdata, void *obj);
    }
    ```

2.  rehash

    ![img](https://i.loli.net/2021/07/05/j5dWXxDs12cSvyK.png)

    ​	当前服务器没有执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1，进行拓展操作；当前服务器如果有执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5，进行拓展操作。

    ​	当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作。

    -   rehash前的字典表

    ![img](https://i.loli.net/2021/07/05/3Ncyp7xSEIkOfbJ.png)

    -   计算拓展后ht[1]的空间，ht[0].used * 2  = 4 * 2 = 8, 8刚好是2的3次方，ht[1]分配空间之后如图：

    ![img](https://i.loli.net/2021/07/05/zY4r6bQHLIwWBx2.png)

    -   将ht[0]里的键值对rehash到ht[1]里面，如图：

    ![img](https://i.loli.net/2021/07/05/NyfkY849bmeR2iA.png)

    -   ht[0]设置为ht[1]，ht[1]设置为空白的哈希表，如图：

    ![img](https://i.loli.net/2021/07/05/yazZoVcx7vHGJiP.png)

    -   渐进式rehash

        (1)为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个表；

        (2)在字典中维持一个索引计数器变量rehashidx，设置为0，表示rehash开始；

        (3)在rehash期间，每次对字典进行增删改查操作时，顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash至ht[1]，当rehash工作完成后，程序将rehashidx值增一；

        (4)完成所有rehash操作，程序将rehashidx属性值设置为-1，表示rehash操作已经完成；

        渐进式rehash可以保证在进行rehash的同时，不影响用户操作。

3.  其他

    -   Redis使用MurmurHash2算法计算键的哈希值
    -   Redis使用**链地址法**解决哈希冲突，放在链表头

##### 跳表

1.  结构定义

    -   跳表节点

        ```c
        typedef struct zskiplistNode {
            // 层
            struct zskiplistLevel {
                // 前进指针
                struct zskiplistNode *forward;
                // 跨度
                unsigned int span;
            } level[];
            // 后退节点
            struct zskiplistNode *backward;
            // 分值
            double score;
            // 成员对象
            robj *obj;
        } zskiplistNode;
        ```

    -   跳表

        ```c
        typedef struct zskiplist {
            // 表头节点和表尾节点
            struct zskiplistNode *head, *tail;
            // 表中节点的数量
            unsigned long length;
            // 表中层数最大的节点层数
            int level;
        } zskiplist;
        ```

##### 整数集合

​	当一个集合数量不多且只包含整数数值元素时，Redis就会使用整数集合。

1.  结构定义

    ```c
    typedef struct intset {
        // 编码方式
        uint32_t encoding;
        // 元素数量
        uint32_t length;
        // 保存元素的数组--从小到大排列且不重复
        int8_t contents[];
    } intset;
    ```

2.  升级操作

    contents数组支持int8_t类型数据，当集合添加int16_t类型数据时，会进行升级操作：

    -   根据新元素的类型，拓展整数集合底层数组的空间大小，并未新元素分配空间；
    -   将底层数组现有的元素全部转换为新元素的类型，并维持底层数组的有序性不变；
    -   将新元素添加到底层数组里面

##### 压缩列表

1.  组成

    | zlbytes | zltail | zllen | entry1 | entry2 | ...... | entryN | zlend |
    | ------- | ------ | ----- | ------ | ------ | ------ | ------ | ----- |

    | 名称    | 类型     | 长度  | 说明                                                         |
    | ------- | -------- | ----- | ------------------------------------------------------------ |
    | zlbytes | unit32_t | 4节点 | 记录整个压缩表占用的内存字节数：对压缩列表进行内存重新分配或者计算zlend位置时使用 |
    | zltail  | unit32_t | 4节点 | 记录表尾节点距离压缩列表起始位置有多少字节：快速定位表尾节点地址 |
    | zllen   | unit16_t | 2节点 | 当值小于UNIT16_MAX时，记录节点数量；                         |
    | entryX  | 列表节点 | 不定  | 压缩列表包含的各个节点                                       |
    | zlend   | unit8_t  | 1节点 | 特殊值0xFF，标记压缩列表的末端                               |

2.  压缩列表节点的组成

    | previous_entry_length | encoding | content |
    | --------------------- | -------- | ------- |

    | 名称                  | 字节数  | 说明                                                         |
    | --------------------- | ------- | ------------------------------------------------------------ |
    | previous_entry_length | 1或5    | 记录前一个节点的长度：可以根据当前节点的起始地址计算前一个节点的起始地址 |
    | encoding              | 1、2或5 | 记录content数组数据类型和长度                                |
    | content               | 不定    | 节点的值                                                     |

##### 对象
