[比赛](https://tianchi.aliyun.com/programming/introduction.htm?spm=5176.100066.333.1.3f82c0718qNL2J&raceId=231600)持续了将近两个月，终于结束了，虽有遗憾，但也“不枉此行”。
最终名次为初赛30名，复赛第30名，似乎是一种巧合。
本博客文章主要讲讲自己的思路，叙述有点乱，将就看看==。    

## 一. 初赛：《基于Open-Messaging实现进程内消息引擎》  
出题者的初衷应该是让大家去了解他们自己家的RocketMQ。  
该题目是实现进程内的消息引擎，类似于Kafka/RocketMQ，使用磁盘作持久化介质(我不太了解RokectMQ, 其大致思想和kafka类似，毕竟RocketMQ以kafka为原本实现的),kafka中每个Topic分为多个Partition, 每个Partition分成多个Segment, 消息虽然不能保证全局顺序性，但在同一Partition内能保证顺序性，其实现主要是append log，即同一个Partition的消息Partition文件尾部。所以这给出了本题的一个解决思路：要保证同一个Producer消息的顺序性，肯定得写入同一个文件，只是这个"同一个文件"有着不同的实现方式。  
### 实现方式  
####  1. 全局一个文件   
这种方式好处在于读取时可顺序读取文件，同时可以利用到Linux的read ahead特性，这样尽量能读到Page cache中的内容。  
但这种方式有一个明显的缺点： 写入需要全局锁，并且当Producer越多，锁竞争越大，可扩展性很差。  

####  2. 每个Producer中每个TOPIC/QUEUE一个文件  
这种方式中，每个Producer端只需为每个TOPIC/QUEUE创建一个唯一文件名即可，同时记录一个索引文件，然后根据不同的TOPIC/QUEUE写入不同的文件，Consumer端在订阅消息时根据索引文件查询得各自订阅TOPIC/QUEUE的文件名，再读取消息即可。  
这种方式有以下有点：  
- 可完全无锁；  
- Consumer端可不用读取无用的消息，只需根据订阅的TOPIC/QUEUE读取相应的文件即可。  

但这种方式最大的缺点是：  
- Producer端写入太多文件，例如有10个Producer，有90个TOPIC以及10个QUEUE，那么将会创建1000个文件，IO竞争太过剧烈。  

#### 3. 每个Producer一个文件  
每个Producer一个文件，就不怕TOPIC类型过多，因为不管有多少类型的TOPIC，都是写入一个文件，这样会大大减少IO竞争，同时同一个Producer中也不需要锁来保证一致性，在Consumer中还可利用多线程读取文件，然后每个读取线程分发消息到各个Consumer。  
这种方式有一个缺点是不管Consumers是否订阅某个TOPIC,都会读取，当然解决方式是在消息头部放入大小字段，当判断到该TOPIC没有被任何一个Consumer订阅，可直接跳过该TOPIC。  

### 实现中优化点  
#### 1. 消息的压缩   
**这个优化点才是初赛能上分的关键**。    
为什么压缩？  
如果消息能压缩到很小，那么占用的文件大小就会很小，初赛给的环境是4G内存，其中JVM堆大小为2560M，剩余1536M，因为在Producer端运行完后，
不会刷新磁盘，所以若所有消息占用的文件大小小于1536M，那么都会缓存到Page Cache中，所以Consumer消费时只需要读取内存即可，而不用触发真正的
文件读取，其实kafka中也有这种特性，具体可见江南白衣的博客[从Apache Kafka 重温文件高效读写](http://calvin1978.blogcn.com/articles/kafkaio.html)  

消息的压缩包括了字段的压缩和多个消息集合的压缩，字段压缩中可将MessageHeader中的消息头字符串压缩为1到2字节大小的标志，字段的压缩还包括表示消息大小的length压缩，比如因为消息头字符串大小很小，那么length可使用byte表示...；多个消息集合的压缩即是多个消息组合后使用JDK中的压缩算法压缩后再写入文件，这种方式成效较差，因为JDK的压缩算法效率不太高，除非自己实现压缩算法。  

#### 2. 使用MappedByteBuffer
map方式应该算是Java中最为高效的文件读写方式了。


### 总结  
总结起来，初赛有以下关键点：  
- 尽量不使用锁；  
- 尽量减少文件IO竞争；  
- 尽量压缩消息，从而在Page Cache中就能读到消息而不需要到文件中读取。  

## 二 复赛：《模拟阿里双十一分布式数据同步》  

# 未完待续 


