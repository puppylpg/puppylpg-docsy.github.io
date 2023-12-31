---
layout: post
title: "Innodb - 页"
date: 2022-01-14 00:01:01 +0800
categories: mysql innodb
tags: mysql innodb
---

之前介绍了[Innodb - 行]({% post_url 2022-01-13-innodb-row %})，行是一种紧凑存放列值的结构，而行是放在页page里的。

1. Table of Contents, ordered
{:toc}

行组成页，**页是innodb管理存储空间的基本单位**。

这一点非常好理解，和虚拟内存使用页式管理一模一样。当需要一条记录的时候，绝对不是只从磁盘读那一条数据！那样会非常浪费！

> 磁盘读写这么慢，好不容易定位到了地方，结果你就只读了一条？？？

根据局部性原理，一次从磁盘读取一块内容到内存才是最合理的：没什么额外的读取开销，而想读取的下一条记录大概率也在这一块里面，那么想访问它的时候，它已经在内存里了，直接省了一次读取磁盘的开销。

正是基于这个思想：
- 磁盘是按块读取的；
- 虚拟内存是按页加载到内存的（一般是4KB）；
- innodb也是这么做的：把很多条记录放到一页，一次读取一页到内存里。**一页大小为16KB。**

> innodb的页和虚拟内存的页没什么关系，只不过二者都需要从磁盘加载到内存，所以有相似的设计理念，又恰好都用了“页”这个名字。

# 行怎么在页内排列
## 物理上按插入顺序排列
页的主要部分就是用来放数据的，而数据是以行为单位的，所以行就按照实际插入顺序一条接一条排列在数据部分。

由行的compact格式可知，一行到底占用多少字节，是不一定的，每行的长度都可能不一样。所以行看起来并没有“对齐”。

> 内存对齐：CPU在访问内存的时候，并不是按字节访问的，而是按照word size字长为单位访问。数据结构在内存里放置的时候，应该尽可能在边界上对齐，这样根据字长的偏移，直接就定位到数据结构的位置了。

## 逻辑上按主键序排列
行在物理上按照数据插入顺序存放，**但在逻辑上，他们形成了一个链表，按照主键的顺序从小到大排列**。

之前在[Innodb - 行]({% post_url 2022-01-13-innodb-row %})里说过，每一行的metadata都有一个`nex_record`，记录着下一个按照主键顺序比它大的那条记录的地址。

这样，一个page内所有的行都相当于有序排列了。MySQL又定义了两条特殊的记录：
- infimum：类似于min，永远是最小的记录，所以这个链表它永远是第一个节点；
- supremum：类似于max，永远是最大的记录，所以这个链表它永远是最后一个节点；

# 有序链表的查询
所有的记录都有序之后，该怎么按照主键查找一条记录呢？遍历肯定是可以的，但是这种O(n)的复杂度MySQL肯定是不会接受的。

> 在有序的数组中查找一个元素，有二分查找，可以达到O(logN)的复杂度，有序链表呢？对，跳表！

在页内，MySQL采用了跳表的思想加速查找，类似一个只有一层目录的跳表：
- 把这拥有n个节点的链表分组，假设分为m个组；
- 每个组的最大的一条记录，作为这个组的“带头大哥”记录下来。m个组就有m个带头大哥；
- 把这些带头大哥的地址放到另一个地方，mysql称为 **slot数组**。**注意，这个是数组，而真正的跳表，它的第一层目录也是链表**；
- 当需要查找一个主键是否存在时，**先和slot数组做二分查找**，最后锁定一个slot，主键值小于这个slot的带头大哥，最后再到大哥所在的组内做遍历查询（小范围的遍历）；

这是一个logN的二分查找加上一个小范围的遍历查找，还是logN的复杂度。

> 它很像只有一层目录的跳表，区别在于跳表的目录依然是链表。

这些slot放到哪里呢？肯定不是page的数据部分。mysql专门给他们找了个地方，叫 **page directory页目录**。这个叫法很符合他们的身份：他们就像是行的目录一样，查字典时应该先查目录，缩小查找范围，再去目录指定的区间做小范围遍历查找。

# metadata
行有metadata，页自然也需要。比如记录下页的相关属性：
- slot的数量：相当于slot数组的个数，方便做二分查找；
- 当前可以插入数据的空闲地址；
- 总空闲空间；
- 最后一条插入记录的地址；

因为行的删除是标记删除，所以为了后期插入数据直接覆盖它们，或者在最终不需要的时候把他们彻底删除，page还维护了一个已删除链表，里面的节点是每一个被删除的行。

# tailer
page的尾部，放一些校验和之类的字段。

# header
header是所有类型的page的通用信息。MySQL有很多种page，像这种存放数据的page叫做 **索引页**。

> 为什么不叫数据页？因为innodb **索引即数据**，后面介绍索引的时候会说到。

header存放了一些很重要的信息：
- 页类型，比如索引页；
- 页号：当前页的offset；
- 上一个page的地址；
- 下一个page的地址；
- 页属于哪个表；

没错，**页与页之间也组成了一个链表，就像行记录与行记录之间一样**！当行记录多到一页放不下的时候，就会放到临近的页里，临近的页也组成了链表。相邻的行记录主键大小并不相邻，只是通过链表达到了逻辑上的主键有序排列。页与页构成的链表同样如此，逻辑上不同页之间的行记录也是按主键有序排列的，但实际上页与页之间的物理位置可能隔着很多其他页，所以他们的物理页号未必相邻。

> 等等！页内的行记录链表为了加快查询速度，使用了一个简单的跳表。**内层的行链表可以，外层的页链表岂不是也可以**？是的，它就是B+树索引！这就是下一个故事了~

