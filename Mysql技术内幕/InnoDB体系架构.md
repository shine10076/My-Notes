#### InnoDB体系架构

![avatar](https://upload-images.jianshu.io/upload_images/5304392-451053fb97782ac9.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/766/format/webp)

InnoDB是一个单进程多线程的模型。

InnoDB存储引擎分为多个内存块，可以认为这些内存块组成了一个大的内存池，负责：

1. 维护所有进程/线程需要访问的多个内部数据结构。

2. 缓存磁盘中的数据，方便快速的读取，同时对磁盘文件的数据修改之前在这里缓存

3. 重做日志(redo log)缓冲

   ....

后台线程的作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存是最近的数据。

##### 1.多线程

1. ###### masterThread

   核心后台线程，负责将缓冲池中数据异步刷新到磁盘，保证数据一致性。包括脏页刷新，合并插入缓冲等

2. ###### IO Thread

   InnoDB大量使用了AIO(异步IO)来处理**写IO请求**，极大提升数据库性能，IO Thread的主要工作就是负责这些IO请求的回调。

3. ###### Pruge Thread

   事务被提交后，其undolog可能不再需要，因此Pruge Thread来回收已经使用并分配的undo页。

4. ###### Page Cleaner Thread

   Page Cleaner Thread作用是将脏页刷新放到单独的线程中操作。

   > 脏页：所谓脏数据是指内存中的数据没有刷新到非易失存储设备上，包括但不限于更新、插入和删除等操作都会产生脏数据。

   

##### 2.内存

1. ###### 缓冲池

   ​	InnoDB存储引擎基于磁盘存储，其中的记录按照页的方式进行管理，由于CPU速度和磁盘速度差距大，基于磁盘的数据库系统通常使用缓冲池来提升整体性能。

   ​	缓冲池是一块内存区域，在数据库中进行读取页的操作，首先将磁盘中读到的页存放到缓冲池中，下一次再读取相同页时，判断是否在缓冲池中，在直接读取该页，否则从磁盘读。

   ​	对于数据库中页修改操作，首先修改缓冲池中的页，然后再以一定的频率刷新回磁盘。

2. ###### LRU List Free List和Flush List

   ​	数据库中缓冲池通过LRU(Latest Recent Used)来管理内存的。但InnoDB对LRU算法做了一定的改进，新增了一个midpoint位置，新读取到的点放入midPoint位置。（默认为5/8处）

   ​	FreeList:在数据库刚启动时，LRU列表是空的，此时现在FreeList存放页

   ​	FlushList:脏页存放于Flush列表和LRU列表中，Flush列表用于管理脏页刷新回磁盘。

3. ###### 重做日志缓冲

   InnoDB引擎首先将redo log存放到缓冲区，然后按照一定的频率刷新到重做日志文件。

##### 3checkPoint技术

​	checkPoint解决以下问题

- 缩短数据库恢复时间

- 缓冲池不够用时将脏页刷新回磁盘

- 重做日志不可用时，刷新脏页

  