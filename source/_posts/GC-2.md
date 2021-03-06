---
layout: post
title: C#.NET托管堆和垃圾回收(续)
category: GC
date: 2016-03-20 00:00:00
---


##大对象
 CLR将对象分成大对象和小对象。目前认为85000字节或者更大的对象是大对象。CLR以不同方式对待大小对象。
 
1. 大对象不是在小对象的地址空间分配的，而是在进程地址空间的其他地方分配。

2. 目前版本的GC不“压缩”大对象，因为在内存中移动它们代价过高。但这可能在进程中的大对象之间造成地址空间碎片化，以至于抛出OutMemoryException。CLR将来的版本可能会压缩大对象。
3. 大对象总是第2代，绝不可能是第0代或者第1代。所以只能为需要长时间存活的资源创建大对象。分配短时间存活的大对象会导致第2代被更频繁地回收，损失性能。大对象一般是大字符串（XML/JSON）或者用于I/O操作的字节数组（从文件/网络将字节读入缓冲区以便处理）。



##垃圾回收模式 
 CLR启动时会选择一个GC模式，进程中之前该模式都不会改变。
 
 有两个基本GC模式。

###工作站 
该模式针对客户端应用程序优化GC。GC造成的延时很低，应用程序线程挂起时间很短，避免用户感到焦虑。在该模式中,GC假定机器上运行的其他应用程序都不会消耗太多的CPU资源。

###服务器 
 该模式针对服务器端应用程序优化GC。被优化的主要是吞吐量和资源利用。GC假定机器上没有运行其他应用程序（无论客户端还是服务器应用程序），并假定机器的所有CPU都可以用来辅助完成GC。该模式造成托管堆被拆分成几个区域，每个CPU一个。开始垃圾回收时，垃圾回收器在每个CPU上运行一个特殊线程；每个线程都和其他线程并发回收它自己的区域。对于工作者线程行为一致的服务器应用程序，并发回收能很好进行。这个功能要求应用程序在多CPU计算机上运行，是线程能真正同时工作，从而得到性能上的提升。



应用程序默认以“工作站”GC模式运行。寄宿了CLR的服务器应用程序（如ASP.NET ）可请求CLR加载服务器 GC.但如果应用程序在单处理器计算机上运行，CLR总是使用“工作站”GC模式。

独立应用程序可以创建一个配置文件告诉CLR 使用CLR使用服务器回收器。配置文件要为应用程序添加gcServer元素。下面是一个示例配置文件：
<configuration>
       <runtime>
             <gcServer enabled="true">
      </runtime>
</configuration> 
可以使用GCSettings类的只读Boolean属性IsServerGC得到CLR是否处于“服务器”GC模式。



除了这两种模式，GC还支持两种子模式：并发(默认)或者非并发。
在并发方式中，垃圾回收器有一个额外的后台线程，它能在应用程序运行时并发标记对象。 程序运行时，垃圾回收器运行一个普通优先级的后台线程来查找不可达对象。找到之后，垃圾回收器再次挂起所以线程，判断是否要“压缩”内存。如决定压缩，内存会被压缩，根引用会被修正，应用程序线程恢复运行。这一次垃圾回收花费的时间比平常少，因为不可达对象集合已构造好了。但垃圾回收器也可能决定不压缩内存；事实上，垃圾回收器更倾向不压缩。可用内存多，垃圾回收器便不会压缩堆；这有利于增强性能，但会增大程序的工作集。使用并发垃圾回收器，应用程序消耗的内存通常比使用非并发垃圾回收器多。