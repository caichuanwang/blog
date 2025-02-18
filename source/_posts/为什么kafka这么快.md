---
title: 为什么kafka这么快
date: 2024-12-24 11:27:41
tags: 
    - kafka
description: 解答为什么kafka这么快的原因。
categories: 
    - kafka
---

### 为什么Kafka快？

kafka的快是多个原因共同促就得。但是关键因素有三个

- 顺序I/O

  总所周知，机械磁盘的读写分成顺序读写和随机读写，因为随机读写需要有寻址的操作，所以导致其速度比顺序读写慢了三个数量级。而Kafka采用就时顺序读写。

  Kafka的message是不断追加到本地磁盘文件末尾的，而不是随机的写入，这使得Kafka写入吞吐量得到了显著提升 。我们从Kafka的Partition就知道，每一条新的消息就是加在Partition的末尾的。如果所示

  ![image-20230615162833093](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/image-20230615162833093.png)

  因为这样，也就导致了Kafka没有删除的操作。消费者就使用客户端保存的offset对Kafka进行读取。


- 页缓存(page cache)

  Kafka 使用了操作系统本身的`Page Cache`,并没有利用JVM里的内存，这样好处是

  - 避免对JVM内存消耗过多
  - 避免GC问题，JVM中的数据会被GC回收，但是系统缓存不会
  - 更加可靠，即使JVM重启进程也不会丢失系统缓存

  写入页缓存的数据会再次`flush`到磁盘中，可以使用`log.flush`设置刷入磁盘时间。

  如果发生设备断电等，`没有刷入磁盘的数据仍会丢失`。

  

- 零拷贝

  ![image-20240902203108288](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/image-20240902203108288.png)

  linux操作系统找中有个`sendfile`方法，可以利用内核缓存区进行“零拷贝”发送文件， “零拷贝” 机制 允许操作系统将数据从Page Cache 直接发送到网络，只需要最后一步的copy操作将数据复制到 NIC 缓冲区， 这样避免将数据在内核空间和用户空间之间反复复制数据 。示意图如下：

  ![image-20230615164949098](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/image-20230615164949098.png)

> 零拷贝并不止真的零次拷贝，而是只使用了一次拷贝，相比较之前的4次拷贝，提高了效率
>
> 这里的零拷贝是指`零次 CPU 拷贝`，这里发生的copy都不用CPU参与，而是`DMA在操作`

- 批量处理

  Kafka 的客户端和 broker 还会在通过网络发送数据之前，在一个批处理中累积多条记录 (包括读和写)。记录的批处理分摊了网络往返的开销，使用了更大的数据包从而提高了带宽利用率

- 数据压缩

  Producer 可将数据压缩后发送给 broker，从而减少网络传输代价，目前支持的压缩算法有：Snappy、Gzip、LZ4。数据压缩一般都是和批处理配套使用来作为优化手段的。
