#### Redis数据结构之字典

字典是一种保存键值对的抽象数据结构。在字典中，一个键可以和一个值进行关联，这些关联的键和值就称为键值对。

Redis的数据库就是通过字典作为其底层实现的。对数据库的增删改查都是建立在字典的操作之上。

#####  1.字典的实现

Redis的字典使用hash表作为底层实现

##### 1.1哈希表

Redis字典所使用的哈希表结构定义如下

```c
typedef struct dictht{
	//哈希表数组
	dictEntry **table;
	
	//哈希表大小
	unsigned long size;
	
	//哈希表大小掩码，用于计算索引值
	//总是等于size-1
	unsigned long sizemask;
	
	//该哈希表已有节点的数量
	unsigned long used;
}dictht;
```

 dictEntry结构保存着一个键值对。size属性记录了哈希表大小，也即是table数组的大小。

used属性指的是当前哈希表目前已有节点的数量。

##### 1.2哈希表节点

哈希表节点用dictEntry结构表示，每个dictEntry结构都保存着一个键值对。

```c
typedef struct dictEntry{
	//键
	void *key;
	
	//值
	union{
		void *val;
		uint64_tu64;
		int64_ts64;
	}v;
	
	//指向下个哈希表节点，形成链表
	struct dictEntry *next;
}dicEntry;
```

其中键值对的值可以是一个指针，或者是uint64_t整数，又或者是一个int64_t整数。

next属性是指向另一个哈希表节点的指针，将多个哈希值相同的键值对连接在一起，以此来解决键冲突问题。

##### 1.3字典

Redis中字典由dict.h/dict结构表示:

```c
typedef struct dict{
	//类型特定函数
	dictType *type;
	
	//私有数据
	void *privdata;
	
	//哈希表
	dictht ht[2];
	
	//rehash索引
	//当rehash不在进行时，值为-1
	int trehashidx;/*rehashing not in processing if rehashidx = -1;*/
	
}dict;
```

其中type属性和privdata属性是针对不同类型的键值对，为创建多态字典而设置的：

type属性是一个指向dictType结构的指针，每个 dictType结构表存了一簇操作特定类型键值对的函数，redis

会为用途不同的字典设置不同类型的特定函数。

```c
typedef struct dictType{
	//计算哈希值的函数
	unsigned int (*hashFunction)(const void *key);
	
	//复制键的函数
    void *(*keyDup)(void *privdata, const void *key;)
    
    //复制值的函数
    void *(*valDup)(void *privdata, const void *obj;)
       
    ....   
}
```

ht属性是一个包含了两个项的数组，数组中每一项都是dictht哈希表。一般情况下只使用ht[0],在rehash的时候才会使用ht[1].

##### 2哈希算法

Redis计算哈希值和索引值的方法如下：

```c
#使用字典设置的哈希函数，计算键key的哈希值
hash = dict->type->hashfunction（key）；
```

```
#使用hash表的sizemask属性和哈希值计算出索引值
#根据使用情况的不同，ht[x]可以是ht[0]或者ht[1]
index = hash & dict -> ht[x].sizemask;
```

例如，我们要将一个键值对k0和v0添加到字典里面：

1. 计算键k0的hash值，hash = dict->type->hashFunction(k0);
2. 假设计算所得的hash值为8，那么程序会继续使用语句：index = hash&dict->ht[0].sizemask = 8&3 = 0;
3. 计算出键k0的索引值为0，这表示包含键值对k0和v0的节点被放置到哈希表数组的索引0位置上 。

> MurmurHash算法：Redis使用Murmurhash2

##### 3解决键冲突

键冲突的定义:当两个或者以上的键被分配到了hash表数组的同一个索引上，我们称这些键发生了冲突。

Redis的哈希表使用链地址法解决键冲突。其中由于dictEntry没有指向表尾的指针，新节点采取头插法。

##### 4rehash

为了让哈希表的负载因子在一个合理范围内，程序需要对哈希表进行扩展和收缩。

rehash的操作：

1. 为字典的ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量。
   - 扩展操作，ht[1]的大小等于第一个ht[0].used*2的2^n
   - 收缩操作，ht[1]的大小等于 第一个ht[0].used的2^n

   2.将保存在ht[0]上的所有键值对rehash到ht[1]上,rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上。

  3.将ht[0]包含的所有键值对都迁移到ht[1]上，释放ht[0]，将ht[1]变成ht[0],并且创建一个新的ht[1].

哈希表的负载因子计算：

load_factor = ht[0].used / ht[0].size

##### 5渐进式rehash

扩展或者收缩得到哈希表需要将ht[0]里面的所有键值对rehash到ht[1]中，但是rehash的过程并不是一次性完成的，rehash的动作是分多次，渐进式地完成的。

详细步骤如下：

1. 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个hash表。
2. 在字典中维持一个索引计数器变量rehashidx,将其值设为0，表示rehash工作开始。
3. rehash进行期间，每次对字典进行增删改查时，除了执行指定操作，还顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]上。
4. 随着字典操作的不断执行，最终在某个时间点ht[0]完全被rehash到ht[1].
   

在进行渐进式rehash的过程中，字典会同时使用ht[0]和ht[1]两个哈希表，所以在渐进式rehash进行期间，字典的增删改查需要在两个表上进行，例如，在ht[0]中未找到，继续到ht[1]里面进行查找。