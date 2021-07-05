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
