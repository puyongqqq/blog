# 问题
### 1. 日志为何要判断再打印？

```
1. 这完全是考虑到log4j的使用效率, 日志为禁用时日志的优化。

log4j可以通过配置来确定某个category的输出级别level, 共有四种, 级别从低到高分别是：debug -> info -> error -> fatel.日志输出的时候, 只会输出大于等于该级别的日志, 也就是设置了INFO之后, DEBUG是不会被输出, 只会输出INFO、ERROR和FATAL级别的日志.

但即使日志关闭了, 日志的语句还是会被执行的(只是不输出而已), 因此日志的参数还是会构造, 例如logger.debug(buildLongString()), 虽然它不会打印语句, 但是buildFullString还是被执行了, 白费功夫.

因此对于性能损耗比较大的日志, 最好先判断日志级别再执行.
if (logger.isDebugEnabled()) {
    logger.debug("消耗性能");
}

2. 日志打印，使用占位符。可以避免大量的字符串拼接，从而减少服务器频繁的出发GC，造成毫无意义的性能损耗。
```

### 2. Spring 事务

```
Spring事务：通过@Transactional注解标注类或者方法，在spring启动时，会扫描标注@Transactional注解的类，在BeanPostProcessor#postProcessAfterInitialization中生成代理类。标注类，会将类中所有public方法以事务执行；标注方法，会覆盖类的注解配置。

rollbackFor: 回滚异常
propagation：传播行为（7种）
isolation： 隔离级别（4种）

AOP：
1. cglib：基于继承（class）。同一个类中，嵌套方法调用，invoke不会触发AOP（Spring中使用这种），invokeSuper会触发AOP。基于asm字节码技术。
2. jdk：基于接口（interface）。同一个类中，嵌套方法调用，不会重复触发AOP。

事务失效：
1. 数据库本身不支持事务。
2. 事务方法中捕获了异常或抛出非受检异常。
3. bean未被Spring管理。
4. 方法非public。
5. 当前类的非事务方法调用事务方法（原因：生成代理类，执行invoke时，不会走代理类）。
```

### 3. kakfa - docker

```
* 注意： kafka作为消息队列，它只是一个服务，而数据的存储依赖于zookeeper。kafka默认有zookeeper，但生产环境建议将zookeeper单独部署。原因有待补充。
```

1. 拉取zookeeper镜像

   ```shell
   docker pull wurstmeister/zookeeper:latest
   ```

2. 编写docker-compose.yml

   ```yml
   version: '3'
   
   services:
     zoo1:
       image: wurstmeister/zookeeper
       restart: unless-stopped
       hostname: zoo1
       ports:
         - "2181:2181"
       container_name: zookeeper
   
     # kafka version: 1.1.0
   
     # scala version: 2.12
   
     kafka1:
       image: wurstmeister/kafka
       ports:
   
      - "9092:9092"
        vironment:
              KAFKA_ADVERTISED_HOST_NAME: localhost
              KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181"
              KAFKA_BROKER_ID: 1
              KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
              KAFKA_CREATE_TOPICS: "stream-in:1:1,stream-out:1:1"
            depends_on:
           - zoo1
             ntainer_name: kafka
   ```

3. 运行docker-compose

   ```
   docker-compose up -d
   ```

4. 验证

   ```shell
   # 查看容器是否启动
   docker ps 
   
   # 查看kafka能否正常工作
   docker exec -it kafka /bin/bash
   cd /opt/kafka_2.13-2.7.0
   ./kafka-console-producer.sh --broker-list localhost:9092 --topic sun                             # 窗口1：创建主题 sun
   ./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic sun --from-beginning       # 窗口2：监听主题 sun
   
   # 窗口1输入"hello"，窗口2接收到"hello"，表明kafka可用。 
   
   # zookeeper查看数据节点
   docker exec -it kafka /bin/bash
   cd bin/
   ./zkCli.sh                                                              # 连接到zookeeper服务
   ls /                                                                    # 查看zookeeper根节点下所有目录
   ls /brokers/topics                                                      # 查看zookeeper所有的topic
   ```

5. 清除数据

   `必须顺序执行！！！`

   ```shell
   # 进入kafka
   docker exec -it kafka /bin/bash 
   rm -rf /kafka/*                               # 删除kafka的日志，执行完成之后，kafka会断掉，容器会stop
   
   # 进入zookeeper
   docker exec -it kafka /bin/bash
   cd ./bin
   ./zkCli.sh
   rmr /brokers/topics                           # 删除zookeeper所有的主题(也可选择删除指定主题) 
   
   # 重启
   docker start kafka
   ```

### 4. RestFul

​	

### 5. I/O

1. 同步阻塞IO（Blocking IO）: 传统I/O模型，阻塞IO，指的是需要内核IO操作彻底完成后，才返回到用户空间，执行用户的操作。阻塞指的是用户空间程序的执行状态，用户空间程序需等到IO操作彻底完成。传统的IO模型都是同步阻塞IO。在java中，默认创建的socket都是阻塞的。。是jdk1.4版本之前默认的IO方式。

   ```
   1. 程序使用read()系统调用，系统由用户态转换为内核态，磁盘中的数据由DMA（Direct memory access）的方式读取到内核读缓冲区（kernel buffer）。DMA过程中CPU不需要参与数据的读写，而是DMA处理器直接将硬盘数据通过总线传输到内存中。
   2. 系统由内核态转为用户态，当程序要读的数据已经完全存入内核读缓冲区以后，程序会将数据由内核读缓冲区，写入到用户缓冲区，这个过程需要CPU参与数据的读写。
   3. 程序使用write()系统调用，系统由用户态切换到内核态，数据从用户缓冲区写入到网络缓冲区（Socket Buffer），这个过程需要CPU参与数据的读写。
   4. 系统由内核态切换到用户态，网络缓冲区的数据通过DMA的方式传输到网卡的驱动（存储缓冲区）中（protocol engine）
   
   总结：普通的拷贝过程经历了四次内核态和用户态的切换（上下文切换），两次CPU从内存中进行数据的读写过程，这种拷贝过程相对来说比较消耗系统资源。
   ```

   网络发送缓冲区：

   内核缓冲区：

   用户缓冲区：

   DMA：直接内存访问。

   **内存地址映射其实是操作系统将内存地址和磁盘文件做一个映射，读写这块内存，相当于直接对磁盘文件进行读写，但是实际上的读还是要经过OS读取到内存`PageCache`中，写过程也需要OS自动进行脏页置换到磁盘中。**

2. 同步非阻塞IO（Non-blocking IO）：非阻塞IO，指的是用户程序不需要等待内核IO操作完成后，内核立即返回给用户一个状态值，用户空间无需等到内核的IO操作彻底完成，可以立即返回用户空间，执行用户的操作，处于非阻塞的状态。

3. IO多路复用（IO Multiplexing）：即经典的Reactor设计模式，有时也称为异步阻塞IO，Java中的Selector和Linux中的epoll都是这种模型。

4. 异步IO（Asynchronous IO）：异步IO，指的是用户空间与内核空间的调用方式反过来。用户空间线程是变成被动接受的，内核空间是主动调用者。
   

### 6. 如何保证双写一致性？如何平衡系统响应时间。

1. 使用分布式锁。优点：能够保证数据一致性。缺点：增加原本程序的复杂性，增加了成本来保证分布式锁。
2. 将读写操作串行化。优点：能够保证数据一致性。缺点：无法充分发挥现代服务器多核CPU的处理能力。
3. 正常修改数据库。由另一个线程监听binlog来更新缓存。优点：平衡更新和查询的响应速度。缺点：会存在更新延迟。

### 7. CMS，G1，ZGC区别。

CMS(Concurrent Mark Sweep)收集器是一种以获取最短回收停顿时间为目标的收集器，停顿时间短，用户体验就好。

基于`标记-清除`算法，并发收集、低停顿，运作过程复杂，分4步：

- **初始标记**：仅仅标记GC Roots能直接关联到的对象，速度快，但是需要“Stop The World”
- **并发标记**：就是进行追踪引用链的过程，可以和用户线程并发执行。
- **重新标记**：修正并发标记阶段因用户线程继续运行而导致标记发生变化的那部分对象的标记记录，比初始标记时间长但远比并发标记时间短，需要“Stop The World”
- **并发清除**：清除标记为可以回收对象，可以和用户线程并发执行

由于整个过程耗时最长的并发标记和并发清除都可以和用户线程一起工作，所以总体上来看，CMS收集器的内存回收过程和用户线程是并发执行的。

CSM收集器有3个缺点：

- **对CPU资源非常敏感**：并发收集虽然不会暂停用户线程，但因为占用一部分CPU资源，还是会导致应用程序变慢，总吞吐量降低。CMS的默认收集线程数量是=(CPU数量+3)/4；当CPU数量多于4个，收集线程占用的CPU资源多于25%，对用户程序影响可能较大；不足4个时，影响更大，可能无法接受。
- **无法处理浮动垃圾**（在并发清除时，用户线程新产生的垃圾叫浮动垃圾）,可能出现"Concurrent Mode Failure"失败：并发清除时需要预留一定的内存空间，不能像其他收集器在老年代几乎填满再进行收集；如果CMS预留内存空间无法满足程序需要，就会出现一次"Concurrent Mode Failure"失败；这时JVM启用后备预案：临时启用Serail Old收集器，而导致另一次Full GC的产生；
- **产生大量内存碎片**：CMS基于"标记-清除"算法，清除后不进行压缩操作产生大量不连续的内存碎片，这样会导致分配大内存对象时，无法找到足够的连续内存，从而需要提前触发另一次Full GC动作。


G1（Garbage-First）是JDK7-u4才正式推出商用的收集器。G1是面向服务端应用的垃圾收集器。它的使命是未来可以替换掉CMS收集器。采用了`标记-复制`算法

G1收集器特性：

- **并行与并发**：能充分利用多CPU、多核环境的硬件优势，缩短停顿时间；能和用户线程并发执行。
- **分代收集**：G1可以不需要其他GC收集器的配合就能独立管理整个堆，采用不同的方式处理新生对象和已经存活一段时间的对象。
- **空间整合**：整体上看采用标记整理算法，局部看采用复制算法（两个Region之间），不会有内存碎片，不会因为大对象找不到足够的连续空间而提前触发GC，这点优于CMS收集器。
- **可预测的停顿**：除了追求低停顿还能建立可以预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间不超N毫秒，这点优于CMS收集器。

为什么能做到可预测的停顿？

```
是因为可以有计划的避免在整个Java堆中进行全区域的垃圾收集。G1收集器将内存分大小相等的独立区域（Region），新生代和老年代概念保留，但是已经不再物理隔离。G1跟踪各个Region获得其收集价值大小，在后台维护一个优先列表；每次根据允许的收集时间，优先回收价值最大的Region（名称Garbage-First的由来）；这就保证了在有限的时间内可以获取尽可能高的收集效率。
```

对象被其他Region的对象引用了怎么办？

```
判断对象存活时，是否需要扫描整个Java堆才能保证准确？在其他的分代收集器，也存在这样的问题（而G1更突出）：新生代回收的时候不得不扫描老年代？ 
无论G1还是其他分代收集器，JVM都是使用Remembered Set来避免全局扫描： 
每个Region都有一个对应的Remembered Set； 
每次Reference类型数据写操作时，都会产生一个Write Barrier 暂时中断操作； 
然后检查将要写入的引用指向的对象是否和该Reference类型数据在不同的 Region（其他收集器：检查老年代对象是否引用了新生代对象）； 
如果不同，通过CardTable把相关引用信息记录到引用指向对象的所在Region对应的Remembered Set中；
进行垃圾收集时，在GC根节点的枚举范围加入 Remembered Set ，就可以保证不进行全局扫描，也不会有遗漏。
```

![image-20210407104651177](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20210407104651177.png)

不计算维护Remembered Set的操作，回收过程可以分为4个步骤（与CMS较为相似）：

- **初始标记**：仅仅标记GC Roots能直接关联到的对象，并修改TAMS(Next Top at Mark Start)的值，让下一阶段用户程序并发运行时能在正确可用的Region中创建新对象，需要“Stop The World”
- **并发标记**：从GC Roots开始进行可达性分析，找出存活对象，耗时长，可与用户线程并发执行
- **最终标记**：修正并发标记阶段因用户线程继续运行而导致标记发生变化的那部分对象的标记记录。并发标记时虚拟机将对象变化记录在线程Remember Set Logs里面，最终标记阶段将Remember Set Logs整合到Remember Set中，比初始标记时间长但远比并发标记时间短，需要“Stop The World”
- **筛选回收**：首先对各个Region的回收价值和成本进行排序，然后根据用户期望的GC停顿时间来定制回收计划，最后按计划回收一些价值高的Region中垃圾对象。回收时采用复制算法，从一个或多个Region复制存活对象到堆上的另一个空的Region，并且在此过程中压缩和释放内存；可以并发进行，降低停顿时间，并增加吞吐量。



ZGC（Z Garbage Collector）是一款由Oracle公司研发的，以低延迟为首要目标的一款垃圾收集器。它是基于**动态Region**内存布局，（暂时）**不设年龄分代**，使用了**读屏障**、**染色指针**和**内存多重映射**等技术来实现**可并发的标记-整理算法**的收集器。在JDK 11新加入，还在实验阶段，主要特点是：**回收TB级内存（最大4T），停顿时间不超过10ms**。

动态Region

- **小型Region（Small Region）**：容量固定为2MB，用于放置小于256KB的小对象。
- **中型Region（Medium Region）**：容量固定为32MB，用于放置大于等于256KB但小于4MB的对象。·
- **大型Region（Large Region）**：容量不固定，可以动态变化，但必须为2MB的整数倍，用于放置4MB或以上的大对象。**每个大型Region中只会存放一个大对象**，最小容量可低至4MB，所有大型Region可能小于中型Region。大型Region在ZGC的实现中是不会被重分配的，因为复制一个大对象的代价非常高昂。

染色指针技术
HotSpot虚拟机的标记实现方案有如下几种：
把标记直接记录在对象头上（如Serial收集器）；
把标记记录在与对象相互独立的数据结构上（如G1、Shenandoah使用了一种相当于堆内存的1/64大小的，称为BitMap的结构来记录标记信息）；
直接把标记信息记在引用对象的指针上（如ZGC）
染色指针是一种直接将少量额外的信息存储在指针上的技术。目前在Linux下64位的操作系统中高18位是不能用来寻址的，但是剩余的46为却可以支持64T的空间，到目前为止我们几乎还用不到这么多内存。于是ZGC将46位中的高4位取出，用来存储4个标志位，剩余的42位可以支持4T的内存。
- Linux下64位指针的高18位不能用来寻址，所有不能使用；
- Finalizable：表示是否只能通过finalize()方法才能被访问到，其他途径不行；
- Remapped：表示是否进入了重分配集（即被移动过）；
- Marked1、Marked0：表示对象的三色标记状态；
- 最后42用来存对象地址，最大支持4T；

三色标记

在并发的可达性分析算法中我们使用三色标记（Tri-color Marking）来标记对象是否被收集器访问过：

- **白色**：表示对象尚未被垃圾收集器访问过。显然在可达性分析刚刚开始的阶段，所有的对象都是白色的，若在分析结束的阶段，仍然是白色的对象，即代表不可达。
- **黑色**：表示对象已经被垃圾收集器访问过，且这个对象的所有引用都已经扫描过。黑色的对象代表已经扫描过，它是安全存活的，如果有其他对象引用指向了黑色对象，无须重新扫描一遍。**黑色对象不可能直接（不经过灰色对象）指向某个白色对象**。
- **灰色**：表示对象已经被垃圾收集器访问过，但这个对象上至少存在一个引用还没有被扫描过。

ZGC的回收过程。

- **并发标记（Concurrent Mark）**：与G1、Shenandoah一样，并发标记是遍历对象图做可达性分析的阶段，它的初始标记和最终标记也会出现短暂的停顿，整个标记阶段只会更新染色指针中的Marked 0、Marked 1标志位。
- **并发预备重分配（Concurrent Prepare for Relocate）**：这个阶段需要根据特定的查询条件统计得出本次收集过程要清理哪些Region，将这些Region组成**重分配集（Relocation Set）**。ZGC每次回收都会扫描所有的Region，用范围更大的扫描成本换取省去G1中记忆集的维护成本。
- **并发重分配（Concurrent Relocate）**：重分配是ZGC执行过程中的核心阶段，这个过程要把重分配集中的存活对象复制到新的Region上，并为重分配集中的每个Region维护一个转发表（Forward Table），记录从旧对象到新对象的转向关系。
- **并发重映射（Concurrent Remap）**：重映射所做的就是修正整个堆中指向重分配集中旧对象的所有引用，但是ZGC中对象引用存在“自愈”功能，所以这个重映射操作并不是很迫切。

