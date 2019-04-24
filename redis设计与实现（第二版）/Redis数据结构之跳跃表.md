### Redis数据结构之跳跃表

##### 1.跳跃表的定义

跳跃表是一种有序数据结构，通过每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

支持平均O(N)，最坏时间复杂度为O(N)复杂度的节点查找，还可以通过顺序性操作批量处理节点。

**跳跃表在Redis里的用处：一是实现有序集合，另一个是在集群节点中用作内部数据结构。**

##### 2.跳跃表的实现

Redis的跳跃表由zskiplistNode和zskiplist两个结构定义。跳跃表结构如下

![redis跳跃表](D:\Learning\学习笔记\笔记图片\redis跳跃表.jpg)

左边的是zskiplist结构，该结构包含以下属性：

- header:指向跳跃表的表头节点。
- tail：指向跳跃表的表尾节点。
- level：记录目前跳跃表中节点最大的层数。（不包含表头节点）
- length：记录跳跃表的长度，跳跃表目前的节点数量。（不包含表头节点）

其余四个结构是zskiplistNode结构，包含以下属性：

- level：节点用L1，L2等字样标记节点的各个层，L1代表第一层，L2第二层以此类推，每层包含两个属性：**前进指针和跨度**，前进指针用于访问表尾方向的其他节点，而跨度是前进指针指向节点和当前节点的距离。
- backward(后退指针)：节点中用BW字样标记节点的后退指针，指向位于当前节点的前一个节点。
- score：跳跃表节点按分值由大到小排列。
- obj：成员对象，指向节点保存的成员对象。

##### 2.1跳跃表节点

redis.h/zskiplistNode

```c
typedef struct zskiplistNode{
    //层
    struct zskiplistLevel{
        //前进指针
        struct zskiplistNode *forward;
        //跨度
        unsigned int span;
    }level[],
    //后退指针
    struct zskiplistNode *backward;
    //分支
    double score;
    //成员对象,
    robj *obj;
}zskiplistNode;
```

###### 层

每次创建一个新的跳跃表节点时，程序都会根据幂次定律生成一个0~32之间的数

> 幂次定律：越大的数生成的概率越小。

###### 前进指针

用于访问表头到表尾的节点

###### 跨度

其用途是用于计算节点在链表中的位置。

###### 后退指针

每次只能后退一个节点。

###### 分值和成员

分值用于节点间的排序。

成员对象指向一个字符串，字符串对象存放着一个SDS值。这个值代表着具体的对象类型。

##### 2.2跳跃表实现

redis.h/zskiplist

```c
typedef struct zskiplist{
  //表头节点和表尾节点
  struct zskiplistNode *header, *tail;
  //节点数量
  int length;
  //表中层数最大的节点的层数
  int level;
}zskiplist;
```

##### 3总结

跳跃表是有序集合的底层实现之一(另一个是压缩列表)。

跳跃表由zskiplist和zskiplistNode两个节点实现。

每个跳跃表节点的层高是1~32之间的随机数。

多个节点可以包含相同的分值，但每个节点的成员对象是唯一的。

跳跃表中节点按分值大小排序，分值相同的按对象大小排序。