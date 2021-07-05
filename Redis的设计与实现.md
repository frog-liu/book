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

    ![img](C:/Users/qqq/AppData/Local/Temp/企业微信截图_16254702169746.png)

    ​	当前服务器没有执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1，进行拓展操作；当前服务器如果有执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5，进行拓展操作。

    ​	当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作。

    -   rehash前的字典表

    ![img](C:/Users/qqq/AppData/Local/Temp/企业微信截图_16254703485553.png)

    -   计算拓展后ht[1]的空间，ht[0].used * 2  = 4 * 2 = 8, 8刚好是2的3次方，ht[1]分配空间之后如图：

    ![img](C:/Users/qqq/AppData/Local/Temp/企业微信截图_16254706356397.png)

    -   将ht[0]里的键值对rehash到ht[1]里面，如图：

    ![img](C:/Users/qqq/AppData/Local/Temp/企业微信截图_16254707251599.png)

    -   ht[0]设置为ht[1]，ht[1]设置为空白的哈希表，如图：

    ![img](C:/Users/qqq/AppData/Local/Temp/企业微信截图_16254708757557.png)

3.  其他

    -   Redis使用MurmurHash2算法计算键的哈希值
    -   Redis使用**链地址法**解决哈希冲突，放在链表头

