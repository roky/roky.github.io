---
layout: post
title:  "项目总结之memcached篇"
date:   2013-06-2 00:10:00
categories: nosql
excerpt: 
---
  
* content
{:toc}


最近公司要做一个产品，短信报警。公司的报警系统基于C#开发的，java在公司产品中通讯功能相对来说，用的比较少。所以当决策在web开发这套产品的时候，也面临的技术选型问题。主要从几个方面来考虑：

1、由于需要考虑web通讯，（看了web通讯这块，选择了pushlet技术）

2、后端通讯采用mina后服务器通讯，并将信息推送给web。

3、并发量，短信报警应用不是太广，报警量不是太多，但是考虑到web通讯主要基于轮训技术，还是考虑了集群。

由于集群，之前公司的框架主要基于ecached缓存的，基于单机版本的。引入了memcached分布式缓存技术。

---

## 什么是Memcached

许多Web应用程序都将数据保存到RDBMS中，应用服务器从中读取数据并在浏览器中显示。但随着数据量的增大，访问的集中，就会出现REBMS的负担加重，数据库响应恶化，网站显示延迟等重大影响。Memcached是高性能的分布式内存缓存服务器。一般的使用目的是通过缓存数据库查询结果，减少数据库的访问次数，以提高动态Web 应用的速度、提高扩展性。如图

![](/images/memcached/m1.jpg)

## Memcached特点

1、	协议简单：memcached的服务器客户端通信并不使用复杂的XMPP/MQTT等格式，而是使用简单的基于文本的协议。

![](/images/memcached/m2.png)

2、基于libevent的事件处理：libevent是个程序库，他将Linux的epoll、BSD类操作系统的kqueue等时间处理功能封装成统一的接口。memcached使用这个libevent库，因此能在Linux、BSD、Solaris等操作系统上发挥其高性能。

3、内置内存存储方式：为了提高性能，memcached中保存的数据都存储在memcached内置的内存存储空间中。由于数据仅存在于内存中，因此重启memcached，重启操作系统会导致全部数据消失。另外，内容容量达到指定的值之后memcached回自动删除不适用的缓存。

4、Memcached不互通信的分布式：memcached尽管是“分布式”缓存服务器，但服务器端并没有分布式功能。各个memcached不会互相通信以共享信息。他的分布式主要是通过客户端实现的。

客户端key的服务器分配和获取

服务器的分配是根据每个服务器的权重，建立一个服务器地址集合，根据key的hashcode进行分配，有普通hashcode分布和一致性哈希两种方式。
Add和get都是通过同一个key的hashcode查找服务器分配，所以能够保证存和取是一致

### 普通hashcode分布

只是普通分布，很简单，只是根据权重将服务器地址放入buckets这个List里
获取keyHashcode：
long hc = getHash( key, hashCode )
long bucket = hc % buckets.size(); //“%”是取余运算符 ，可以根据hashcode可以知道除以服务器的数量，可以知道分配到哪台服务器的编号
if ( bucket < 0 ) bucket *= -1; 
获取服务器字符：
String server= buckets.get( (int)bucket；

### 一致性hashcode分布

一致性hashcode分布，只是根据权重将服务器地址放入TreeMap<Long,String> consistentBuckets这个TreeMap里。<br>
获取keyHashCode：<br>
SortedMap<Long,String> tmap =<br>
			this.consistentBuckets.tailMap( hv );<br>
		return ( tmap.isEmpty() ) ? this.consistentBuckets.firstKey() : tmap.firstKey();<br>
获取服务器字符：<br>
String server=consistentBuckets.get( bucket );<br>

## Memcached的内存管理

memcached的内存分配是预先分配内存，常规的程序使用内存无非是两种，一种是预先分配，一种是动态分配。动态分配从效率的角度来讲相对来说要慢 点，因为它需要实时的去分配内存使用，但是这种方式的好处就是可以节约内存使用空间；memcached采用的是预先分配的原则，这种方式是拿空间换时间 的方式来提高它的速度，这种方式会造成不能很高效的利用内存空间，但是memcached采用了Slab Allocation机制来解决内存碎片的问题。

Slab Allocator的基本原理是按照预先规定的大小，将分配的内存分割成特定长度的块，已完全解决内存碎片问题。Slab Allocation的原理相当简单。将分配的内存分割成各种尺寸的块（chucnk），并把尺寸相同的块分成组（chucnk的集合）如图：

![](/images/memcached/m3.jpg)

memcached会针对客户端发送的数据选择slab并缓存到chunk中，这样就有一个弊端那就是比如要缓存的数据大小是50个字节，如果被分配到如上图88字节的chunk中的时候就造成了33个字节的浪费，虽然在内存中不会存在碎片，但是也造成了内存的浪费，这也是我上面说的拿空间换时间的原因，不过memcached对于分配到的内存不会释放，而是重复利用。默认情况下如下图chunk是1.25倍的增加的, memcached  -d start –u root -vv 



### Slab Allocation 的主要术语
•	Page :分配给Slab 的内存空间，默认是1MB。分配给Slab 之后根据slab 的大小切分成chunk.
•	Chunk : 用于缓存记录的内存空间。
•	Slab Class:特定大小的chunk 的组。

### 在Slab 中缓存记录的原理

Memcached根据收到的数据的大小，选择最合适数据大小的Slab 。
memcached中保存着slab内空闲chunk的列表，根据该列表选择chunk,然后将数据缓存于其中。

![](/images/memcached/m4.jpg)

### Memcached在数据删除方面有效里利用资源

Memcached删除数据时数据不会真正从memcached中消失。Memcached不会释放已分配的内存。记录超时后，客户端就无法再看见该记录（invisible 透明），其存储空间即可重复使用。

Lazy Expriationmemcached内部不会监视记录是否过期，而是在get时查看记录的时间戳，检查记录是否过期。这也是memcached的一种惰性过期机制。这种技术称为lazy expiration. 这样不需要开启一个线程不停的扫描，节省了CPU资源。

对于缓存存储容量满的情况下的删除需要考虑多种机制，一方面是按队列机制，一方面应该对应缓存对象本身的优先级，根据缓存对象的优先级进行对象的删除。

### LRU:从缓存中有效删除数据的原理

memcached内部也维护着一套LRU（Least Recently Used）置换算法，当设定的内存满的时候，会进行最近很少使用的数据置换出去从而分配空间，所以对于提升 memcached命中率的问题主要还是一是根据业务存放的value值来调整好chunk的大小以达到最大效率的利用内存；二是扩大内存保证所有缓存的 数据不被置换出去。

##客户端实现原理

![](/images/memcached/m6.png)
![](/images/memcached/m7.png)

## 大数据传输处理

如果需要压缩，且大于压缩阀值（30720 30M）长度时才成立

![](/images/memcached/m5.png)

## failover和failback机制

failover：服务器故障，则对key重新哈希选择下一个服务器（判断是否故障，只用判断客户端和服务是否已经连接，并且可以读写或者服务器与服务器）
failback，用一个hashmap存储连接失败的服务器和对应的失效持续时间，每次获取连接时，都探测是否到了重试时间

---


