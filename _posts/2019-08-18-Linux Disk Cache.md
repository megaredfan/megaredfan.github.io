---
layout: post
title: "Linux磁盘缓存机制"
---

## 前言

最近遇到了一起跟磁盘IO相关的线上故障，借此总结一下之前不太了解的Linux磁盘缓存相关的知识。

总的来说磁盘缓存出现的原因大概有两个：第一是访问磁盘的速度远慢于访问内存的速度，通过在内存中缓存磁盘内容可以提高访问速度；第二是根据程序的局部性原理，数据一旦被访问过，就很有可能在短时间内再次被访问，所以在内存中缓存磁盘内容可以提高程序运行速度。

### 局部性原理

程序局部性原理：程序在执行时呈现出局部性规律，即在一段时间内，整个程序的执行仅限于程序中的某一部分。相应地，执行所访问的存储空间也局限于某个内存区域，具体来说，局部性通常有两种形式：时间局部性和空间局部性。

时间局部性：被引用过一次的存储器位置在未来会被多次引用（通常在循环中）。

空间局部性：如果一个存储器的位置被引用，那么将来他附近的位置也会被引用。

## 页缓存

Linux系统中为了减少对磁盘的IO操作，会将打开的磁盘内容进行缓存，而缓存的地方则是物理内存，进而将对磁盘的访问转换成对内存的访问，有效提高程序的速度。Linux的缓存方式是利用物理内存缓存磁盘上的内容，称为页缓存（page cache）。

页缓存是由内存中的物理页面组成的，其内容对应磁盘上的物理块。页缓存的大小会根据系统的内存空闲大小进行动态调整，它可以通过占用内存以扩张大小，也可以自我收缩以缓解内存使用压力。

在虚拟内存机制出现以前，操作系统使用块缓存系列，但是在虚拟内存出现以后，操作系统管理IO的粒度更大，因此采用了页缓存机制，页缓存是基于页的、面向文件的缓存机制。

### 页缓存的读取

Linux系统在读取文件时，会优先从页缓存中读取文件内容，如果页缓存不存在，系统会先从磁盘中读取文件内容更新到页缓存中，然后再从页缓存中读取文件内容并返回。大致过程如下：

1. 进程调用库函数read发起读取文件请求
2. 内核检查已打开的文件列表，调用文件系统提供的read接口
3. 找到文件对应的inode，然后计算出要读取的具体的页
4. 通过inode查找对应的页缓存，1）如果页缓存节点命中，则直接返回文件内容；2）如果没有对应的页缓存，则会产生一个缺页异常（page fault）。这时系统会创建新的空的页缓存并从磁盘中读取文件内容，更新页缓存，然后重复第4步
5. 读取文件返回

所以说，所有的文件内容的读取，无论最初有没有命中页缓存，最终都是直接来源于页缓存。

### 页缓存的写入

因为页缓存的存在，当一个进程调用write时，对文件的更新仅仅是被写到了文件的页缓存中，让后将对应的页标记为dirty，整个过程就结束了。Linux内核会在周期性地将脏页写回到磁盘，然后清理掉dirty标识。

由于写操作只会把变更写入页缓存，因此进程并不会因此为阻塞直到磁盘IO发生，如果此时计算机崩溃，写操作的变更可能并没有发生在磁盘上。所以对于一些要求比较严格的写操作，比如数据系统，就需要主动调用fsync等操作及时将变更同步到磁盘上。读操作则不同，read通常会阻塞直到进程读取到数据，而为了减少读操作的这种延迟，Linux系统还是用了“预读”的技术，即从磁盘中读取数据时，内核将会多读取一些页到页缓存中。

#### 回写线程

页缓存的回写是由内核中的单独的线程来完成的，回写线程会在以下3种情况下进行回写：

1. 空闲内存低于阈值时。当空闲内存不足时，需要释放掉一部分缓存，由于只有不脏的页才能被释放，所以需要把脏页都回写到磁盘，使其变为可回收的干净的页。
2. 脏页在内存中处理时间超过阈值时。这是为了确保脏页不会无限期的留在内存中，减少数据丢失的风险。
3. 当用户进程调用sync和fsync系统调用时。这是为了给用户进程提供强制回写的方法，满足回写要求严格的使用场景。

回写线程的实现
|    名称    | 版本 |  说明  |
| ---------- | --- | ----- |
| bdflush |  2.6版本以前 | bdflush 内核线程在后台运行，系统中只有一个 bdflush 线程，当内存消耗到特定阀值以下时，bdflush 线程被唤醒。kupdated 周期性的运行，写回脏页。 但是整个系统仅仅只有一个 bdflush 线程，当系统回写任务较重时，bdflush 线程可能会阻塞在某个磁盘的I/O上，导致其他磁盘的I/O回写操作不能及时执行。|
| pdflush       |  2.6版本引入 | pdflush 线程数目是动态的，取决于系统的I/O负载。它是面向系统中所有磁盘的全局任务的。 但是由于 pdflush 是面向所有磁盘的，所以有可能出现多个 pdflush 线程全部阻塞在某个拥塞的磁盘上，同样导致其他磁盘的I/O回写不能及时执行。|
| flusher线程       |  2.6.32版本以后引入 | flusher 线程的数目不是唯一的，同时flusher线程不是面向所有磁盘的，而是每个flusher线程对应一个磁盘|

### 页缓存的回收

Linux中页缓存的替换逻辑是一个修改过的LRU实现，也称为双链策略。和以前不同，Linux维护的不再是一个LRU链表，而是维护两个链表：活跃链表和非活跃链表。处于活跃链表上的页面被认为是“热”的且不会被换出，而在非活跃链表上的页面则是可以被换出的。在活跃链表中的页面必须在其被访问时就处于非活跃链表中。两个链表都被伪LRU规则维护：页面从尾部加入，从头部移除，如同队列。两个链表需要维持平衡–如果活跃链表变得过多而超过了非活跃链表，那么活跃链表的头页面将被重新移回到非活跃链表中，一遍能再被回收。双链表策略解决了传统LRU算法中对仅一次访问的窘境。而且也更加简单的实现了伪LRU语义。这种双链表方式也称作LRU/2。更普遍的是n个链表，故称LRU/n。

## 总结

在这次遇到的线上故障中，根本原因在于在业务逻辑中使用了临时文件做缓存，一个临时文件创建后如果在短时间内删除，这时候对这个文件的操作都是在页缓存内进行，不会实际回写到磁盘。当程序出现问题响应变慢时，临时文件存活时间变长，就可能会使其被回写到磁盘上，导致磁盘压力过大，进而影响整个系统。
