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

2. 日志打印，使用占位符。可以避免大量的字符串拼接，从而减少服务器频繁的触发GC，造成毫无意义的性能损耗。
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

> 注意： kafka作为消息队列，它只是一个服务，而数据的存储依赖于zookeeper。kafka默认有zookeeper，但生产环境建议将zookeeper单独部署。原因有待补充。 

1. 拉取zookeeper镜像

   ```shell
   docker pull wurstmeister/zookeeper:latest
   ```

2.  编写docker-compose.yml

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

3.  运行docker-compose

   ```shell
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

6. 其他问题

   
### 4. HTTP发展史与RestFul

> HTTP协议发展至今，大致经历了0.9, 1.0, 1.1(最常用), 2.0(逐渐普及), 3.0。它是应用层协议。对于Web发展功不可没。

#### 0.9

1. 只支持GET请求。
2. 仅支持html格式。

#### 1.0

1. 规范请求格式，包含请求行，请求头，状态码等信息。
2. 新增POST、HEAD请求方式。
3. 支持更多的内容类型，包括图片、视频、二进制文件等。

#### 1.1

1. 新增PUT、DELETE、OPTIONS、PATCH、TRACE等请求方式，RestFul基于此提出。
2. 支持长连接（WebSocket）。
3. Keep-Alive支持，即多个请求可以重复使用一个TCP连接，服务器端需要按顺序返回结果（队头阻塞）。并发多个请求需要多个TCP连接，浏览器为了控制资源会有6-8个TCP连接都限制。

#### 2.0 

1. Body支持二进制传输。
2. HTTP/2支持header压缩，并且支持header信息索引（客户端和服务端有一张相同的索引表，不同的header对应不同的索引号，发送请求时不会再发header，而是发索引号）
3. HTTP/2支持服务端主动推送功能，如果一个网页中含有大量的静态资源（js、css、图片等），之前版本是当该网页传输完成后解析所有html代码，然后再去传输网页中包含的资源，HTTP/2版本可以在网页没有传输完之前就主动把该网页中包含的静态资源推送到客户端，这样省去了客户端再次发请求的过程。
4. 新增多路复用代替原来的序列和阻塞机制。在多路复用中是基于二进制数据帧的传输、消息、流，所以可以做到乱序的传输。多路复用对同一域名下所有请求都是基于流，所以不存在同域并行的阻塞。

### 5. I/O

1. 同步阻塞IO（Blocking IO）: 传统I/O模型，阻塞IO，指的是需要内核IO操作彻底完成后，才返回到用户空间，执行用户的操作。阻塞指的是用户空间程序的执行状态，用户空间程序需等到IO操作彻底完成。传统的IO模型都是同步阻塞IO。在java中，默认创建的socket都是阻塞的。。是jdk1.4版本之前默认的IO方式。

   ```shell
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

### 6. 如何保证双写一致性？如何平衡系统响应时间

1. 使用分布式锁。优点：能够保证数据一致性。缺点：增加原本程序的复杂性，增加了成本来保证分布式锁。
2. 将读写操作串行化。优点：能够保证数据一致性。缺点：无法充分发挥现代服务器多核CPU的处理能力。
3. 正常修改数据库。由另一个线程监听binlog来更新缓存。优点：平衡更新和查询的响应速度。缺点：会存在更新延迟。

### 7. CMS，G1，ZGC区别

#### CMS

在Hotstop系列虚拟机中，没有一款JVM将CMS设置为默认的垃圾收集器。

>  查看jdk默认的垃圾收集器
>  java -XX:+PrintCommandLineFlags -version


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

#### G1

G1（Garbage-First）是JDK7-u4才正式推出商用的收集器。G1是面向服务端应用的垃圾收集器。它的使命是未来可以替换掉CMS收集器。整体采用了`标记-整理`算法，每个`Region`之间采用`复制`算法。

G1收集器特性：

- **并行与并发**：能充分利用多CPU、多核环境的硬件优势，缩短停顿时间；能和用户线程并发执行。
- **分代收集**：G1可以不需要其他GC收集器的配合就能独立管理整个堆，采用不同的方式处理新生对象和已经存活一段时间的对象。
- **空间整合**：整体上看采用标记整理算法，局部看采用复制算法（两个Region之间），不会有内存碎片，不会因为大对象找不到足够的连续空间而提前触发GC，这点优于CMS收集器。
- **可预测的停顿**：除了追求低停顿还能建立可以预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间不超N毫秒，这点优于CMS收集器。

为什么能做到可预测的停顿？

> 是因为可以有计划的避免在整个Java堆中进行全区域的垃圾收集。G1收集器将内存分大小相等的独立区域（Region），新生代和老年代概念保留，但是已经不再物理隔离。G1跟踪各个Region获得其收集价值大小，在后台维护一个优先列表；每次根据允许的收集时间，优先回收价值最大的Region（名称Garbage-First的由来）；这就保证了在有限的时间内可以获取尽可能高的收集效率。

对象被其他Region的对象引用了怎么办？

> 判断对象存活时，是否需要扫描整个Java堆才能保证准确？在其他的分代收集器，也存在这样的问题（而G1更突出）：新生代回收的时候不得不扫描老年代？ 
> 无论G1还是其他分代收集器，JVM都是使用Remembered Set来避免全局扫描： 
> 每个Region都有一个对应的Remembered Set； 
> 每次Reference类型数据写操作时，都会产生一个Write Barrier 暂时中断操作； 
> 然后检查将要写入的引用指向的对象是否和该Reference类型数据在不同的 Region（其他收集器：检查老年代对象是否引用了新生代对象）； 
> 如果不同，通过CardTable把相关引用信息记录到引用指向对象的所在Region对应的Remembered Set中；
> 进行垃圾收集时，在GC根节点的枚举范围加入 Remembered Set ，就可以保证不进行全局扫描，也不会有遗漏。

不计算维护Remembered Set的操作，回收过程可以分为4个步骤（与CMS较为相似）：

- **初始标记**：仅仅标记GC Roots能直接关联到的对象，并修改TAMS(Next Top at Mark Start)的值，让下一阶段用户程序并发运行时能在正确可用的Region中创建新对象，需要“Stop The World”
- **并发标记**：从GC Roots开始进行可达性分析，找出存活对象，耗时长，可与用户线程并发执行
- **最终标记**：修正并发标记阶段因用户线程继续运行而导致标记发生变化的那部分对象的标记记录。并发标记时虚拟机将对象变化记录在线程Remember Set Logs里面，最终标记阶段将Remember Set Logs整合到Remember Set中，比初始标记时间长但远比并发标记时间短，需要“Stop The World”
- **筛选回收**：首先对各个Region的回收价值和成本进行排序，然后根据用户期望的GC停顿时间来定制回收计划，最后按计划回收一些价值高的Region中垃圾对象。回收时采用复制算法，从一个或多个Region复制存活对象到堆上的另一个空的Region，并且在此过程中压缩和释放内存；可以并发进行，降低停顿时间，并增加吞吐量。

#### ZGC

ZGC（Z Garbage Collector）是一款由Oracle公司研发的，以低延迟为首要目标的一款垃圾收集器。它是基于**动态Region**内存布局，（暂时）**不设年龄分代**，使用了**读屏障**、**染色指针**和**内存多重映射**等技术来实现**可并发的标记-整理算法**的收集器。在JDK 11新加入，还在实验阶段，主要特点是：**回收TB级内存（最大4T），停顿时间不超过10ms**。

![](https://p0.meituan.net/travelcube/40838f01e4c29cfe5423171f08771ef8156393.png@1812w_940h_80q)

> ZGC只有三个STW阶段：**初始标记**，**再标记**，**初始转移**。其中，初始标记和初始转移分别都只需要扫描所有GC Roots，其处理时间和GC Roots的数量成正比，一般情况耗时非常短；再标记阶段STW时间很短，最多1ms，超过1ms则再次进入并发标记阶段。即，ZGC几乎所有暂停都只依赖于GC Roots集合大小，停顿时间不会随着堆的大小或者活跃对象的大小而增加。与ZGC对比，G1的转移阶段完全STW的，且停顿时间随存活对象的大小增加而增加。

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

ZGC的回收过程。

- **并发标记（Concurrent Mark）**：与G1、Shenandoah一样，并发标记是遍历对象图做可达性分析的阶段，它的初始标记和最终标记也会出现短暂的停顿，整个标记阶段只会更新染色指针中的Marked 0、Marked 1标志位。
- **并发预备重分配（Concurrent Prepare for Relocate）**：这个阶段需要根据特定的查询条件统计得出本次收集过程要清理哪些Region，将这些Region组成**重分配集（Relocation Set）**。ZGC每次回收都会扫描所有的Region，用范围更大的扫描成本换取省去G1中记忆集的维护成本。
- **并发重分配（Concurrent Relocate）**：重分配是ZGC执行过程中的核心阶段，这个过程要把重分配集中的存活对象复制到新的Region上，并为重分配集中的每个Region维护一个转发表（Forward Table），记录从旧对象到新对象的转向关系。
- **并发重映射（Concurrent Remap）**：重映射所做的就是修正整个堆中指向重分配集中旧对象的所有引用，但是ZGC中对象引用存在“自愈”功能，所以这个重映射操作并不是很迫切。



三色标记 （CMS、G1、ZGC 都采用了三色标记算法）

在并发的可达性分析算法中我们使用三色标记（Tri-color Marking）来标记对象是否被收集器访问过：

- **白色**：表示对象尚未被垃圾收集器访问过。显然在可达性分析刚刚开始的阶段，所有的对象都是白色的，若在分析结束的阶段，仍然是白色的对象，即代表不可达。
- **黑色**：表示对象已经被垃圾收集器访问过，且这个对象的所有引用都已经扫描过。黑色的对象代表已经扫描过，它是安全存活的，如果有其他对象引用指向了黑色对象，无须重新扫描一遍。**黑色对象不可能直接（不经过灰色对象）指向某个白色对象**。
- **灰色**：表示对象已经被垃圾收集器访问过，但这个对象上至少存在一个引用还没有被扫描过。

三色标记存在的问题。

- **浮动垃圾**。并发标记的过程中，若一个已经被标记成黑色或者灰色的对象，突然变成了垃圾，由于不会再对黑色标记过的对象重新扫描,所以不会被发现，那么这个对象不是白色的但是不会被清除，重新标记也不能从GC Root中去找到，所以成为了浮动垃圾，**浮动垃圾对系统的影响不大，留给下一次GC进行处理即可**。
- **对象漏标问题**。并发标记的过程中，一个业务线程将一个未被扫描过的白色对象断开引用成为垃圾（删除引用），同时黑色对象引用了该对象（增加引用）（这两部可以不分先后顺序）；因为黑色对象的含义为其属性都已经被标记过了，重新标记也不会从黑色对象中去找，导致该对象被程序所需要，却又要被GC回收，此问题会导致系统出现问题。

无论是对引用关系记录的插入还是删除，虚拟机的记录操作都是通过写屏障实现的。CMS是基于增量更新来做并发标记的，G1、Shenandoah则是用原始快照来实现。

### 8. 对象分配

> 在虚拟机的规范中，只是要求对象优先在Eden区创建，否则在老年代创建。由于Hotspot的JIT技术成熟，使得对象在堆上分配内存并不是一定的了。
>
> JIT：通过 javac 将可以将Java程序源代码编译，转换成 java 字节码，JVM 通过解释字节码将其翻译成对应的机器指令，逐条读入，逐条解释翻译。Java编译器经过解释执行，其执行速度必然会比直接执行可执行的二进制字节码慢很多。为了解决这种效率问题，引入了 JIT（Just In Time ，即时编译） 技术。
>
> 有了JIT技术之后，Java程序还是通过解释器进行解释执行，当JVM发现某个方法或代码块运行特别频繁的时候，就会认为这是“热点代码”（Hot Spot Code)。然后JIT会把部分“热点代码”翻译成本地机器相关的机器码，并进行优化，然后再把翻译后的机器码缓存起来，以备下次使用。
>
> 如何确定热点代码？
>
> 1. 方法计数器（Counter Based Hot Spot Detection)。默认阈值在 C1 模式下是 1500 次，在 C2 模式在是 10000 次。采用这种方法的虚拟机会为每个方法，甚至是代码块建立计数器，统计方法的执行次数，某个方法超过阀值就认为是热点方法，触发JIT编译。
> 2. 回边计数器。回边计数器用于统计一个方法中循环体代码执行的次数，在字节码中遇到控制流向后跳转的指令称为“回边”（Back Edge），该值用于计算是否触发 C1 编译的阈值，在不开启分层编译的情况下，C1 默认为 13995，C2 默认为 10700。建立回边计数器的主要目的是为了触发 OSR（On StackReplacement）编译，即栈上编译。在一些循环周期比较长的代码段中，当循环达到回边计数器阈值时，JVM 会认为这段是热点代码，JIT 编译器就会将这段代码编译成机器语言并缓存，在该循环时间段内，会直接将执行代码替换，执行缓存的机器语言。
>
> 编译优化
>
> JIT在做了热点检测识别出热点代码后，除了会对其字节码进行缓存，还会对代码做各种优化。这些优化中，比较重要的几个有：逃逸分析、 锁消除、 锁膨胀、 方法内联、 空值检查消除、 类型检测消除、 公共子表达式消除等。

当对象刚创建时，**优先考虑在栈上分配内存**。因为栈上分配内存效率很高，**当栈帧从虚拟机栈 pop 出去时，对象就被回收了**。但在栈上分配内存时，必须保证此对象不会被其他栈帧所引用，否则此栈帧被 pop 出去时，就会出现对象逃逸，产生 bug。

如果此对象不能在栈上分配内存，则判断此对象是否是大对象，如果对象过大，则直接分配到老年代（具体多大这个阈值可以通过-XX:PretenureSizeThreshold参数设置）。

否则考虑在 TLAB （Thread Local Allocation Buffer，线程本地分配缓冲区）上分配内存，这块内存是伊甸区为每个线程分配的一块区域，它的大小是伊甸区的 1% （可以通过-XX:TLABWasteTargetPercent设置），作用是减少线程间互相争抢伊甸区空间，以减少同步操作。

伊甸区的对象经过 GC，存活的对象在 Survivor 1 区和 Survivor 2 区不断拷贝，到达一定年龄后到达老年代。

老年代的垃圾在 FGC 时被回收。

这就是 Java 中的整个 GC 过程。

![thisisimage](https://mmbiz.qpic.cn/mmbiz_png/icHoerKO3NjJdBsBDiaw3cMer4icUZ8E1ItgUNOgWicHpRzrweH3ZL3geW4A64TwNiau79pHzstcnzR1uNW5NBlCXJw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 对象分配方式？

   1. 指针碰撞（Serial、ParNew等带Compact过程的收集器）

      > 假设Java堆中内存是绝对规整的，所有用过的内存都放在一边，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把那个指针向空闲空间那边挪动一段与对象大小相等的距离，这种分配方式称为“指针碰撞”（Bump the Pointer）。

   2. 空闲列表（CMS这种基于Mark-Sweep算法的收集器）

      > 如果Java堆中的内存并不是规整的，已使用的内存和空闲的内存相互交错，那就没有办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记录上哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录，这种分配方式称为“空闲列表”（Free List）。

2. 什么是安全点、安全区？

   安全点：程序执行期间的所有GC Root已知并且所有堆对象的内容一致的点。
   选择：1.方法调用 2.循环跳转 3.异常抛出。 
   安全区：线程在执行到某一个区域时，此时内存中对象的引用不会发生变化。

3. 标量替换和逃逸分析？（对象在栈上分配的技术基础）

   1. 标量替换

      > 允许将对象打散分配在栈上，比如若一个对象拥有两个字段，会将这两个字段视作局部变量进行分配。

   2. 逃逸分析

      > 逃逸分析的目的是判断对象的作用域是否有可能逃逸出函数体。

   3. 锁消除

      > 如果是在单线程环境下，其实完全没有必要使用线程安全的容器，但就算使用了，因为不会有线程竞争。

4. TLAB是什么？默认占用的空间是多少？

   > TLAB（Thread local Allocation Buffer，本地线程分配缓冲区），一块线程私有的专用内存分配区域，默认情况下占整个Eden区的1%。在同一时刻，可能存在多个线程同时申请内存分配，因此对象分配都需要进行同步（内存分配是使用**CAS+失败重试**保证更新操作的原子性）。在极端情况下，分配对象的效率就会降低。基于此，JVM采用TLAB来避免多线程冲突，给对象分配内存时，通过给线程私有的TLAB上分配对象，提高的对象分配的效率。如果TLAB的剩余空间不足以创建某个对象时，在虚拟机内部维护了一个refill_waste的值，当请求对象大于refill_waste时，会选择在堆中分配，若小于该值，则会废弃当前TLAB，新建TLAB来分配对象。

### 9. Synchronized锁升级过程

> 32位JVM，MarkWord结构

<table cellpadding="0" cellspacing="0">
	<tr>
		<th> 锁状态 </th>
		<th> 23 bits </th>
        <th> 2 bits </th>
        <th> 4 bits </th>
        <th> 1 bits </th>
        <th> 2 bits </th>
 	</tr>
	<tr>
		<td> 无锁 </td>
		<td colspan="2" style="color:#f33b45;"> identity hash code（首次调用） </td>
		<td> 分代年龄 </td>
        <td> 0 </td>
        <td> 01 </td>
 	</tr>
    <tr>
		<td> 偏向锁 </td>
		<td > Thread ID </td>
        <td > epoch </td>
		<td> 分代年龄 </td>
        <td> 1 </td>
        <td> 01 </td>
 	</tr>
    <tr>
		<td> 轻量级锁 </td>
		<td colspan="4"> 指向线程栈中Lock Record的指针 </td>
        <td> 00 </td>
 	</tr>
    <tr>
		<td> 重量级锁 </td>
		<td colspan="4"> 指向监视器（monitor）的指针 </td>
        <td> 10 </td>
 	</tr>
    <tr>
		<td> GC标记 </td>
		<td colspan="4"> 0 </td>
        <td> 11 </td>
 	</tr>
 </table>


> 64位JVM，MarkWord结构

<table cellpadding="0" cellspacing="0">
  <thead>
    <tr>
      <th style="width:112px;">锁状态</th>
      <th style="width:204px;">25 bits</th>
      <th colspan="2" style="width:255px;">31 bits</th>
      <th style="width:80px;">1 bit</th>
      <th style="width:89px;">4 bits</th>
      <th style="width:68px;">1 bit</th>
      <th style="width:117px;">2 bits</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="width:112px;">无锁状态</td>
      <td style="width:204px;">unused</td>
      <td colspan="2" style="width:255px;">
        <span style="color:#f33b45;">identity</span>&nbsp;<span style="color:#f33b45;">hash code</span>（首次调用）
      </td>
      <td style="width:80px;">unused</td>
      <td style="width:89px;">分代年龄</td>
      <td style="width:68px;">0</td>
      <td style="width:117px;">01</td>
    </tr>
    <tr>
      <th style="width:112px;">锁状态</th>
      <th colspan="2" rowspan="1" style="width:400px;">54 bits</th>
      <th style="width:89px;">2 bits</th>
      <th style="width:80px;">1 bit</th>
      <th style="width:89px;">4 bits</th>
      <th style="width:68px;">1 bit</th>
      <th style="width:117px;">2 bits</th>
    </tr>
    <tr>
      <td style="width:112px;">偏向锁</td>
      <td colspan="2" rowspan="1" style="width:204px;">Thread ID</td>
      <td style="width:89px;">epoch</td>
      <td style="width:80px;">unused</td>
      <td style="width:89px;">分代年龄</td>
      <td style="width:68px;">1</td>
      <td style="width:117px;">01</td>
    </tr>
    <tr>
      <th style="width:112px;">锁状态</th>
      <th colspan="6" rowspan="1" style="width:204px;">62 bits</th>
      <th style="width:117px;">2 bits</th>
    </tr>
    <tr>
      <td style="width:112px;">轻量级锁</td>
      <td colspan="6" rowspan="1" style="width:685px;">指向线程栈中Lock Record的指针</td>
      <td style="width:117px;">00</td>
    </tr>
    <tr>
      <td style="width:112px;">重量级锁</td>
      <td colspan="6" style="width:685px;">指向监视器（monitor）的指针</td>
      <td style="width:117px;">10</td>
    </tr>
    <tr>
      <td style="width:112px;">GC标记</td>
      <td colspan="6" style="width:685px;">0</td>
      <td style="width:117px;">11</td>
    </tr>
  </tbody>
</table>



1. 新创建对象。处于可偏向状态（101，匿名偏向锁），也可以说是无锁。因为此时对象的偏向锁标志位为1，但是偏向线程Id为0，age = 0, epoch = 0。如果直接或者间接的生成了hashCode，那么这个对象将变为无锁状态（001）。
2. 一个线程访问且无竞争。首先会去判断是否为可偏向状态，如果是则通过CAS（V：进入同步块之前的Markword，A：无锁状态下的Markword，B：当前线程id）尝试获取锁，成功则获取到偏向锁，执行同步代码块，当同步代码块执行完之后，并不会释放这个偏向锁，对象的Markword仍然是第一个线程获取偏向锁的值。当这个线程再次进入时，会直接拿到偏向锁，执行同步代码块。当存在第二个线程去访问这个对象，那么通过CAS替换失败，然后锁升级为轻量级锁。（存在极端情况，第一个线程持有锁时，第二个线程去获取锁，然后锁升级，线程一执行到一个全局安全点时，线程一挂起，会通过CAS去替换Markword，成功（当前锁状态由偏向锁升级为轻量级锁）后唤醒线程一继续执行，此时线程二继续通过~~自旋，超过一定的次数或时间~~，未能获取到锁，此时锁会继续升级为重量级锁）

3. 轻量级锁和重量级锁在退出同步代码块时，会将对象恢复为无锁状态（001）。

### 10. Lock及AQS

> AQS：即AbstractQueuedSynchronizer，队列同步器，他提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列模型的简单框架。
>
> Lock：从JDK1.5开始，Java提供了与Synchronized具有相同功能API可调用的一个接口，还提供了尝试获取锁（tryLock），可中断锁（lockInterruptibly）等功能。而它一个常用的实现类ReentrantLock，通过结合AQS实现了相关锁功能，其中还包括了对公平锁和非公平锁的两种实现锁机制下的不同实现。其底层主要是基于CAS（底层依赖于硬件上的原子性）去竞争锁，失败的线程会将其封装成Node，将其顺序串联起来，组成一个双向队列（CLH变体，CLH本身是单向链表），然后被LockSupport.park()。

> Synchronized与Lock的比较

<table cellpadding="0" cellspacing="0">
  <thead>
    <tr>
      <th width="100px"> 类别 </th>
      <th>synchronized</th>
      <th>Lock</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>存在层次</td>
      <td>Java的关键字，在jvm层面上</td>
      <td>是一个类</td>
    </tr>
    <tr>
      <td>锁的释放</td>
      <td>1、以获取锁的线程执行完同步代码，释放锁 <br/>2、线程执行发生异常，jvm会让线程释放锁</td>
      <td>在finally中必须释放锁，不然容易造成线程死锁</td>
    </tr>
    <tr>
      <td>锁的获取</td>
      <td>假设A线程获得锁，B线程等待。如果A线程阻塞，B线程会一直等待</td>
      <td>分情况而定，Lock有多个锁获取的方式，具体下面会说道，大致就是可以尝试获得锁，线程可以不用一直等待</td>
    </tr>
    <tr>
      <td>锁状态</td>
      <td>无法判断</td>
      <td>可以判断</td>
    </tr>
    <tr>
      <td>锁类型</td>
      <td>可重入 不可中断 非公平</td>
      <td>可重入 可中断 可公平（两者皆可）</td>
    </tr>
  </tbody>
</table>

AQS（AbstractQueuedSynchronizer）分析 `oracle JDK 11`。

```java
abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {
    
    /**
     * 属性
     */
    ////////////////////////////////////////////////////////////////////////////////////////////////////////////
    // 头结点 虚节点 当前持有锁的线程
    private transient volatile Node head;

    // 阻塞的尾节点，每个新的节点进来，都插入到最后，也就形成了一个链表
    private transient volatile Node tail;

    // 这个是最重要的，代表当前锁的状态，0代表没有被占用，大于 0 代表有线程持有当前锁
    // 这个值可以大于 1，是因为锁可以重入，每次重入都加上 1
    private volatile int state;

    // 代表当前持有独占锁的线程，举个最重要的使用例子，因为锁可以重入
    // reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有了锁
    // if (currentThread == getExclusiveOwnerThread()) {state++}
    private transient Thread exclusiveOwnerThread; // 继承自AbstractOwnableSynchronizer
    ////////////////////////////////////////////////////////////////////////////////////////////////////////////
    
    
    /**
     * 内部类Node（非常重要）定义了双向链表节点的结构
     */
    ////////////////////////////////////////////////////////////////////////////////////////////////////////////
    static final class Node {
        // 标识节点当前在共享模式下
        static final Node SHARED = new Node();
        // 标识节点当前在独占模式下
        static final Node EXCLUSIVE = null;

        // ======== 下面的几个int常量是给waitStatus用的 ===========
        /** waitStatus value to indicate thread has cancelled */
        // 代码此线程取消了争抢这个锁
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        // 官方的描述是，其表示当前node的后继节点对应的线程需要被唤醒
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        // 本文不分析condition，所以略过吧，下一篇文章会介绍这个
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        // 同样的不分析，略过吧
        static final int PROPAGATE = -3;
        // =====================================================


        // 取值为上面的1、-1、-2、-3，或者0(以后会讲到)
        // 这么理解，暂时只需要知道如果这个值 大于0 代表此线程取消了等待，
        //    ps: 半天抢不到锁，不抢了，ReentrantLock是可以指定timeouot的。。。
        volatile int waitStatus;
        // 前驱节点
        volatile Node prev;
        // 后继节点
        volatile Node next;
        // 当前线程
        volatile Thread thread;
    }
    ////////////////////////////////////////////////////////////////////////////////////////////////////////////
    
    
    /**
     * 重要方法
     */
    ////////////////////////////////////////////////////////////////////////////////////////////////////////////

    // 如果tryAcquire(arg) 返回true, 也就结束了。 
    // 否则，acquireQueued方法会将线程压到队列中
    public final void acquire(int arg) {
        // 首先调用tryAcquire(1)一下，名字上就知道，这个只是试一试
        // 因为有可能直接就成功了呢，也就不需要进队列排队了，
        // 对于公平锁的语义就是：本来就没人持有锁，根本没必要进队列等待(又是挂起，又是等待被唤醒的)
        if (!tryAcquire(arg) && // 返回true。 非公平/公平锁能够调用lock能够直接拿到锁。抽象方法，具体实现参见下文。
            // tryAcquire(arg)没有成功，这个时候需要把当前线程挂起，放到阻塞队列中。
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            // 支持线程中断，真正处理中断响应的地方。如果线程在等待队列中排队的过程中（线程此时是出于park状态），
            // 是无法感知到响应中断请求的，所以需要等到线程拿到锁，被唤醒之后，返回Thread.interrupted()。
            // java线程中断。PS：线程中断有什么用？为什么要中断。
            // thread.interrupt(); // 仅仅设置线程中断状态位，无返回值，线程还是继续往下执行。
			// thread.isInterrupted(); // 返回当前线程的中断状态。
		    // Thread.interrupted(); // 返回当前线程的中断状态。执行后具有清除状态功能。
            selfInterrupt();
    }
    
    // 此方法的作用是把线程包装成Node，同时进入到队列中
    // 参数mode此时是Node.EXCLUSIVE，代表独占模式
    private Node addWaiter(Node mode) {
        // 将线程封装成Node
        // Node(Node nextWaiter) {
        //    this.nextWaiter = nextWaiter; // Node.EXCLUSIVE
        //    THREAD.set(this, Thread.currentThread()); // CAS设置当前thread为当前线程
        // }
        Node node = new Node(mode);
		// 轮询 + CAS 保证添加Node一定会成功。
        for (;;) {
            Node oldTail = tail;
            if (oldTail != null) {
                // 将当前的队尾节点，设置为自己的前驱 
                node.setPrevRelaxed(oldTail);
                // 用CAS把自己设置为队尾, 如果成功后，tail == node 了，这个节点成为阻塞队列新的尾巴
                if (compareAndSetTail(oldTail, node)) {
                    // 注意！！！非原子操作。在极端情况下可能存在数据不一致。
                    oldTail.next = node;
                    return node;
                }
            } else { 
                // 如果尾节点为null，则进行初始化队列。PS: 一个线程持有锁，当前线程获取锁失败。
                initializeSyncQueue();
            }
        }
    }
    
    // 下面这个方法，参数node，经过addWaiter(Node.EXCLUSIVE)，此时已经进入阻塞队列
    // 注意一下：如果acquireQueued(addWaiter(Node.EXCLUSIVE), arg))返回true的话，
    // 意味着上面这段代码将进入selfInterrupt()，所以正常情况下，下面应该返回false
    // 这个方法非常重要，应该说真正的线程挂起，然后被唤醒后获取锁，都在这个方法里了
    final boolean acquireQueued(final Node node, int arg) {
        // 线程是否被中断过
        boolean interrupted = false;
        try {
            for (;;) {
                // 获取当前节点的前驱节点
                final Node p = node.predecessor();
                // 如果当前线程是头结点，那么再给它一次获取锁的机会
                if (p == head && tryAcquire(arg)) {
                    // 成功则设置当前节点为头结点
                    setHead(node);
                    p.next = null; // help GC 头结点设置为null，便于GC回收
                    return interrupted;
                }
                // 到这里，说明上面的if分支没有成功，要么当前node本来就不是队头，
                // 要么就是tryAcquire(arg)没有抢赢别人，继续往下看 
                if (shouldParkAfterFailedAcquire(p, node)) // 维护Node中的waitStatus
                    // 要排队的线程在此处被park(),唤醒也是从此处开始
                    interrupted |= parkAndCheckInterrupt(); // 一真为真，返回当前线程是否被中断过。
            }
        } catch (Throwable t) {
            // 取消获取锁。
            cancelAcquire(node);
            if (interrupted)
                selfInterrupt();
            throw t;
        }
    }

    // 判断当前线程是否需要被park
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        // 前驱节点的状态
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL) 
            // 前驱节点的waitStatus == -1，说明前驱节点是拿到锁的线程。当前线程需要被park。
            return true;
        if (ws > 0) { // 前驱节点waitStatus大于0，说明前驱节点取消了排队。
            do {
                // 循环向前查找取消节点，把取消节点从队列中剔除
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            // 前驱节点的下一个节点，指向这个排队的节点node
            pred.next = node;
        } else {
            // 将waitStatus = 0（前驱节点是新建节点）
            // -2（Condition），-3（？） 
            pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
        }
        // 这个方法返回 false，那么会再走一次 for 循序，
        // 然后再次进来此方法，此时会从第一个分支返回 true
        return false;
    }
    
    // 这个方法很简单，因为前面返回true，所以需要挂起线程，这个方法就是负责挂起线程的
    // 这里用了LockSupport.park(this)来挂起线程，然后就停在这里了，等待被唤醒=======
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
    
    // 取消排队。被中断或者发生其他异常
    private void cancelAcquire(Node node) {
        // 忽略掉不存在的节点
        if (node == null)
            return;

        node.thread = null;

        // 跳过被取消的前置节点，从后往前找到最近的一个非cancel的节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;
		
        // 获取最近非cancel节点的后置节点，单纯为了CAS设置pred的后继节点
        Node predNext = pred.next;

		// 设置当前节点为cancel状态
        node.waitStatus = Node.CANCELLED;

        // 判断当前节点是否为尾结点，是则移除
        if (node == tail && compareAndSetTail(node, pred)) {
            // CAS替换pred的后置节点为null
            pred.compareAndSetNext(predNext, null);
        } else {
            // 当前node是非tail的节点
            int ws;
            // 如果当前节点不是head的后继节点，1:判断当前节点前驱节点的是否为SIGNAL，2:如果不是，则把前驱节点设置为SINGAL看是否成功
            // 如果1和2中有一个为true，再判断当前节点的线程是否为null
            // 如果上述条件都满足，把当前节点的前驱节点的后继指针指向当前节点的后继节点
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && pred.compareAndSetWaitStatus(ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    // 存在后继节点且后继节点的状态为非cancel。
                    pred.compareAndSetNext(predNext, next);
            } else {
                // 如果当前节点是head的后继节点，或者上述条件不满足，那就唤醒当前节点的后继节点
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
    
    // 解锁操作，如果线程完成了全部的锁释放（AQS中的state == 0），则返回true，否则返回false
    public final boolean release(int arg) {
        // 当前线程是否释放了全部的锁
        if (tryRelease(arg)) {
            // 获取头结点，如果存在头结点，且头结点的状态不等于0，则表示需要唤醒后继节点
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            // 锁被完全释放，返回true
            return true;
        }
        // 锁未被完全释放
        return false;
    }
	
    // 如果存在后继节点，则需要唤醒
    private void unparkSuccessor(Node node) {
        // 获取头结点的状态
        int ws = node.waitStatus;
       	
        if (ws < 0)
            // 设置为0，不存在竞争？无论成功与否都执行？
            // 由上一个方法可知，就算是有竞争，上一步的CAS也能保证
            // 同时只有一个线程能够拿到锁，所以，这个CAS一定是成功的
            node.compareAndSetWaitStatus(ws, 0);
        // 获取头结点的下一个节点
        Node s = node.next;
        // 这应该是一个优化，通常我们很少（基本不会）去取消排队的线程，
        // 所以，如果下一个是非cancel的节点的话，那就直接去unpark
        if (s == null || s.waitStatus > 0) {
            s = null;
            // 头结点的下一个节点不存在或者被cancel，那么从尾部去取这个链表中的
            // 第一个非cancel的节点
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            // 拿到要解锁的线程，执行unpark操作
            LockSupport.unpark(s.thread);
    }
    
    
    // 释放共享锁，如果当state == 0时，需要去unpark等待队列头部的线程。
    public final boolean releaseShared(int arg) {
        // 尝试以共享模式释放锁，如果state == 0时，返回true，否则false。
        if (tryReleaseShared(arg)) {
            // 需要唤醒等待队列中的线程
            doReleaseShared();
            return true;
        }
        return false;
    }
    
    // 共享模式释放锁
    private void doReleaseShared() {
        for (;;) {
            // TODO 设置头结点的逻辑放在了加锁的部分
            Node h = head;
            // 判断等待队列存在（队列中至少存在一个等待获取锁的节点Node）
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                // 如果头结点的状态为-1
                if (ws == Node.SIGNAL) {
                    // CAS修改头结点的waitStatus（保证并发情况下，只有一个线程能修改成功）
                    if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                        continue;
                    // 修改成功之后，唤醒头结点之后的第一个非cancel的节点
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         // Node.PROPAGATE 为了解决线程无法会唤醒的窘境 （Semaphore中会体现）
                         !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                    continue;
            }
            // 等待队列不存在
            if (h == head) 
                break;
        }
    }
    
    // 获取可中断的共享锁
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        // 以共享模式去尝试获取锁
        // tryAcquireShared(arg) 返回 > 0，表示后续线程还能够以共享形式获取到锁
        //                       返回 = 0，表示当前线程拿到锁，后续线程无法拿到锁。
        //                       返回 < 0，表示当前线程未拿到锁，后续线程也无法拿到锁。
        if (tryAcquireShared(arg) < 0)
            // -1的情况，线程需要排队
            doAcquireSharedInterruptibly(arg);
    }
    
    // 获取可中断的共享锁
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        // 将当前节点以共享模式添加链表中，并返回当前节点
        final Node node = addWaiter(Node.SHARED);
        try {
            for (;;) {
                // 获取当前的节点的前一个节点
                final Node p = node.predecessor();
                if (p == head) {
                    // 如果p是头结点，再给其一次获取锁的机会
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        // 设置头部和传播，node的后置节点如果是共享模式，则需要唤醒
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        return;
                    }
                }
                // 那么这个线程是否需要被挂起
                if (shouldParkAfterFailedAcquire(p, node) &&
                    // 挂起线程且检查中断
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
    
    // 设置头部和传播
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
		
        // 有上一个方法可知，propagate一定是大于等于0的，当其大于0，表示后续线程任然可以获得共享锁
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                // 释放锁，见上文
                doReleaseShared();
        }
    }
    ////////////////////////////////////////////////////////////////////////////////////////////////////////////
	
}


```




1. 公平锁（ReentrantLock#FairSync）一朝排队，永远排队。

   ```java
   public final void acquire(int arg) {
       if (!tryAcquire(arg) &&   // 尝试获取锁，成功则退出
       	acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 获取锁失败，去排队
           // 挂起当前线程
       	selfInterrupt();
   }
   ```

   ```java
   // 尝试直接获取锁，返回值是boolean，代表是否获取到锁
   // 返回true：1.没有线程在等待锁；2.重入锁，线程本来就持有锁，也就可以理所当然可以直接获取
   protected final boolean tryAcquire(int acquires) {
       final Thread current = Thread.currentThread();
       int c = getState();
       // state == 0 此时此刻没有线程持有锁
       if (c == 0) {
           // 虽然此时此刻锁是可以用的，但是这是公平锁，既然是公平，就得讲究先来后到，
           // 看看有没有别人在队列中等了半天了
           if (!hasQueuedPredecessors() &&
               // 如果没有线程在等待，那就用CAS尝试一下，成功了就获取到锁了，
               // 不成功的话，只能说明一个问题，就在刚刚几乎同一时刻有个线程抢先了 =_=
               // 因为刚刚还没人的，我判断过了
               compareAndSetState(0, acquires)) {
               // 到这里就是获取到锁了，标记一下，告诉大家，现在是我占用了锁
               setExclusiveOwnerThread(current);
               return true;
           }
       }
       // 会进入这个else if分支，说明是重入了，需要操作：state=state+1
       // 这里不存在并发问题
       else if (current == getExclusiveOwnerThread()) {
           int nextc = c + acquires;
           if (nextc < 0)
               throw new Error("Maximum lock count exceeded");
           setState(nextc);
           return true;
       }
       // 如果到这里，说明前面的if和else if都没有返回true，说明没有获取到锁
       // 回到上面一个外层调用方法继续看:
       // if (!tryAcquire(arg) 
       //        && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
       //     selfInterrupt();
       return false;
   }
   ```

   ```java
   final boolean acquireQueued(final Node node, int arg) {
       boolean interrupted = false;
       try {
         for (;;) {
           // 获取当前Node的前一个节点
           final Node p = node.predecessor();
           // 如果当前节点p是头，则进行尝试获取锁
           if (p == head && tryAcquire(arg)) {
             setHead(node);
             p.next = null; // help GC
             return interrupted;
           }
           // 如果获取锁失败，那么应该挂起
           if (shouldParkAfterFailedAcquire(p, node)) // 判断是否需要park
             interrupted |= parkAndCheckInterrupt();
         }
       } catch (Throwable t) {
         // 取消抢占锁
         cancelAcquire(node);
         // 响应中断
         if (interrupted)
           selfInterrupt(); 
         throw t;
       }
   }
   ```

   

2. 非公平锁（ReentrantLock#NonfairSync）只有CLH变体（双向链表）中的第一个节点和正在尝试获取锁的线程去竞争，第一个节点拿到锁，则新线程会进入等待队列，其余流程和公平锁一致。

   ```java
   // Sync#nonfairTryAcquire
   // 尝试获取非公平锁，如果获取锁失败，则进行排队，和公平锁一致
   final boolean nonfairTryAcquire(int acquires) {
       final Thread current = Thread.currentThread();
       int c = getState();
       if (c == 0) {
           // 非公平所在！！！
           // 只要有线程来竞争锁，就会先尝试通过CAS获取锁
           if (compareAndSetState(0, acquires)) {
               setExclusiveOwnerThread(current);
               return true;
           }
       }
       // 判断重入
       else if (current == getExclusiveOwnerThread()) {
           int nextc = c + acquires;
           if (nextc < 0) // overflow
               throw new Error("Maximum lock count exceeded");
           setState(nextc);
           return true;
       }
       return false;
   }
   ```
   
   
   
3. 常用并发工具类

   
   1. CountDownLatch
   2. CyclicBarrier
   3. Semaphore
   
   

### 11. 线程池ThreadPool

1. 核心参数

   1. corePoolSize：核心线程数，线程池中常驻线程的最大数量。

   2. maximumPoolSize：线程池中运行最大线程数(包括核心线程和非核心线程)。

      ```
      在很多文章中，都介绍了关于线程池参数如何如何计算，给出类似 coreSize = 2 * CPU核心数，maxSize = 25 * CPU核心数。并发任务的执行情况和任务类型相关，IO密集型和CPU密集型的任务运行起来的情况差异非常大，但这种占比是较难合理预估的，这导致很难有一个简单有效的通用公式帮我们直接计算出结果。
      ```

   3. keepAliveTime：非核心线程最大存活时间。

   4. unit：存活时间单位，通常和keepAliveTime搭配使用。

   5. workQueue：阻塞队列，核心线程处理来不及处理的任务会被添加到这里。

      ```
      ArrayBlockingQueue：由数组结构组成的有界阻塞队列，按照先进先出原则，要求设定初始大小。
      LinkedBlockingQueue：由链表结构组成的有界，按照先进先出原则，可以不设定初始大小，但是默认值大小为Integer,MAX_VALUE的阻塞队列。
      PriorityBlockingQueue：支持优先级排序的无界阻塞队列。默认情况下，按照自然顺序，要么实现compareTo（）方法，指定构造参数Comparator。
      DelayQueue：一个使用优先级队列实现的无界阻塞队列，支持延时获取元素的阻塞队列，元素必须要实现Delayed接口。适用场景：实现自己的缓存系统，订单到期，限时支付等等。
      SynchronousQueue：一个不存储元素的阻塞队列，没有容量，每一个put操作都要等待一个take操作，所以会产生生产者和消费者能力不匹配的问题。
      LinkedTransferQueue：由链表结构组成的无界阻塞队列。transfer()，必须要消费者消费了以后方法才会返回，tryTransfer()
      无论消费者是否接收，方法都立即返回。
      LinkedBlockingDeque：由链表结构组成的双向阻塞队列。可以从队列的头和尾插入和移除元素，实现工作密取，方法名带了First对头部操作，带了last从尾部操作，另外：add=addLast;remove=removeFirst; take=takeFirst。
      
      ```

   6. threadFactory：线程工厂，定制化线程的一些基础信息，比如线程的name。

   7. handler：拒绝策略，提交的任务已经超过了最大线程数，才会执行拒绝策略。

      ```
      AbortPolicy:直接抛出一个异常，默认策略
      DiscardPolicy: 直接丢弃任务
      DiscardOldestPolicy:抛弃下一个将要被执行的任务(最旧任务)
      CallerRunsPolicy:主线程中执行任务
      ```

2. 工作流程

   任务调度是线程池的主要入口，当用户提交了一个任务，接下来这个任务将如何执行都是由这个阶段决定的。了解这部分就相当于了解了线程池的核心运行机制。

   首先，所有任务的调度都是由execute方法完成的，这部分完成的工作是：检查现在线程池的运行状态、运行线程数、运行策略，决定接下来执行的流程，是直接申请线程执行，或是缓冲到队列中执行，亦或是直接拒绝该任务。其执行过程如下：

   1. 首先检测线程池运行状态，如果不是RUNNING，则直接拒绝，线程池要保证在RUNNING的状态下执行任务。
   2. 如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务。
   3. 如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中。
   4. 如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务。
   5. 如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

   ![](https://p0.meituan.net/travelcube/31bad766983e212431077ca8da92762050214.png)

3. 提交任务

   ```java
   public void execute(Runnable command) {
       if (command == null)
           throw new NullPointerException();
       /*
        * clt记录着runState和workerCount
        */
       int c = ctl.get();
       /*
        * workerCountOf方法取出低29位的值，表示当前活动的线程数；
        * 如果当前活动线程数小于corePoolSize，则新建一个线程放入线程池中；
        * 并把任务添加到该线程中。
        */
       if (workerCountOf(c) < corePoolSize) {
           /*
            * addWorker中的第二个参数表示限制添加线程的数量是根据corePoolSize来判断还是maximumPoolSize来判断；
            * 如果为true，根据corePoolSize来判断；
            * 如果为false，则根据maximumPoolSize来判断
            */
           if (addWorker(command, true))
               return;
           /*
            * 如果添加失败，则重新获取ctl值
            */
           c = ctl.get();
       }
       /*
        * 如果当前线程池是运行状态并且任务添加到队列成功
        */
       if (isRunning(c) && workQueue.offer(command)) {
           // 重新获取ctl值
           int recheck = ctl.get();
           // 再次判断线程池的运行状态，如果不是运行状态，由于之前已经把command添加到workQueue中了，
           // 这时需要移除该command
           // 执行过后通过handler使用拒绝策略对该任务进行处理，整个方法返回
           if (! isRunning(recheck) && remove(command))
               reject(command);
           /*
            * 获取线程池中的有效线程数，如果数量是0，则执行addWorker方法
            * 这里传入的参数表示：
            * 1. 第一个参数为null，表示在线程池中创建一个线程，但不去启动；
            * 2. 第二个参数为false，将线程池的有限线程数量的上限设置为maximumPoolSize，添加线程时根据maximumPoolSize来判断；
            * 如果判断workerCount大于0，则直接返回，在workQueue中新增的command会在将来的某个时刻被执行。
            */
           else if (workerCountOf(recheck) == 0)
               addWorker(null, false);
       }
       /*
        * 如果执行到这里，有两种情况：
        * 1. 线程池已经不是RUNNING状态；
        * 2. 线程池是RUNNING状态，但workerCount >= corePoolSize并且workQueue已满。
        * 这时，再次调用addWorker方法，但第二个参数传入为false，将线程池的有限线程数量的上限设置为maximumPoolSize；
        * 如果失败则拒绝该任务
        */
       else if (!addWorker(command, false))
           reject(command);
   }
   
   
   private boolean addWorker(Runnable firstTask, boolean core) {
       retry:
       for (;;) {
           int c = ctl.get();
           // 获取运行状态
           int rs = runStateOf(c);
           
           /*
            * 这个if判断
            * 如果rs >= SHUTDOWN，则表示此时不再接收新任务；
            * 接着判断以下3个条件，只要有1个不满足，则返回false：
            * 1. rs == SHUTDOWN，这时表示关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务
            * 2. firsTask为空
            * 3. 阻塞队列不为空
            * 
            * 首先考虑rs == SHUTDOWN的情况
            * 这种情况下不会接受新提交的任务，所以在firstTask不为空的时候会返回false；
            * 然后，如果firstTask为空，并且workQueue也为空，则返回false，
            * 因为队列中已经没有任务了，不需要再添加线程了
            */
           // Check if queue empty only if necessary.
           if (rs >= SHUTDOWN &&
               ! (rs == SHUTDOWN &&
                  firstTask == null &&
                  ! workQueue.isEmpty()))
               return false;
           for (;;) {
               // 获取线程数
               int wc = workerCountOf(c);
               // 如果wc超过CAPACITY，也就是ctl的低29位的最大值（二进制是29个1），返回false；
               // 这里的core是addWorker方法的第二个参数，如果为true表示根据corePoolSize来比较，
               // 如果为false则根据maximumPoolSize来比较。
               // 
               if (wc >= CAPACITY ||
                   wc >= (core ? corePoolSize : maximumPoolSize))
                   return false;
               // 尝试增加workerCount，如果成功，则跳出第一个for循环
               if (compareAndIncrementWorkerCount(c))
                   break retry;
               // 如果增加workerCount失败，则重新获取ctl的值
               c = ctl.get();  // Re-read ctl
               // 如果当前的运行状态不等于rs，说明状态已被改变，返回第一个for循环继续执行
               if (runStateOf(c) != rs)
                   continue retry;
               // else CAS failed due to workerCount change; retry inner loop
           }
       }
       
       boolean workerStarted = false;
       boolean workerAdded = false;
       Worker w = null;
       try {
           // 根据firstTask来创建Worker对象
           w = new Worker(firstTask);
           // 每一个Worker对象都会创建一个线程
           final Thread t = w.thread;
           if (t != null) {
               final ReentrantLock mainLock = this.mainLock;
               mainLock.lock();
               try {
                   // Recheck while holding lock.
                   // Back out on ThreadFactory failure or if
                   // shut down before lock acquired.
                   int rs = runStateOf(ctl.get());
                   // rs < SHUTDOWN表示是RUNNING状态；
                   // 如果rs是RUNNING状态或者rs是SHUTDOWN状态并且firstTask为null，向线程池中添加线程。
                   // 因为在SHUTDOWN时不会在添加新的任务，但还是会执行workQueue中的任务
                   if (rs < SHUTDOWN ||
                       (rs == SHUTDOWN && firstTask == null)) {
                       if (t.isAlive()) // precheck that t is startable
                           throw new IllegalThreadStateException();
                       // workers是一个HashSet
                       workers.add(w);
                       int s = workers.size();
                       // largestPoolSize记录着线程池中出现过的最大线程数量
                       if (s > largestPoolSize)
                           largestPoolSize = s;
                       workerAdded = true;
                   }
               } finally {
                   mainLock.unlock();
               }
               if (workerAdded) {
                   // 启动线程
                   t.start();
                   workerStarted = true;
               }
           }
       } finally {
           if (! workerStarted)
               addWorkerFailed(w);
       }
       return workerStarted;
   }
   ```

   

4. 总结

   ThreadPoolExecutor实现的顶层接口是Executor，**顶层接口Executor提供了一种思想：将任务提交和任务执行进行解耦。用户无需关注如何创建线程，如何调度线程来执行任务，用户只需提供Runnable对象，将任务的运行逻辑提交到执行器(Executor)中，由Executor框架完成线程的调配和任务的执行部分。**

   ExecutorService接口增加了一些能力：（1）**扩充执行任务的能力**，补充可以为一个或一批异步任务生成Future的方法；（2）提供了**管控线程池**的方法，比如停止线程池的运行。

   AbstractExecutorService则是上层的抽象类，**将执行任务的流程串联了起来，保证下层的实现只需关注一个执行任务的方法即可**。最下层的实现类ThreadPoolExecutor实现最复杂的运行部分，**ThreadPoolExecutor将会一方面维护自身的生命周期，另一方面同时管理线程和任务**，使两者良好的结合从而执行并行任务。

   线程池在内部实际上构建了一个**生产者消费者模型**，将线程和任务两者解耦，并不直接关联，从而良好的缓冲任务，复用线程。线程池的运行主要分成两部分：**任务管理、线程管理**。**任务管理部分充当生产者的角色**，当任务提交后，线程池会判断该任务后续的流转：（1）直接申请线程执行该任务；（2）缓冲到队列中等待线程执行；（3）拒绝该任务。线程管理部分是消费者，它们被统一维护在线程池内，根据任务请求进行线程的分配，当线程执行完任务后则会继续获取新的任务去执行，最终当线程获取不到任务的时候，线程就会被回收。

   

5. 线程池对自身状态的管理

   线程池内部使用一个变量(ctl)维护两个值：运行状态(runState)和线程数量 (workerCount)。

   ```java
   private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
   ```

   它同时包含两部分的信息：线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)，高3位保存runState，低29位保存workerCount，两个变量之间互不干扰。用一个变量去存储两个值，可避免在做相关决策时，出现不一致的情况，不必为了维护两者的一致，而占用锁资源，同理还有`ReentrantReadWriteLock#state`，使用高16位表示读锁，低16位表示写锁。

   ThreadPoolExecutor的运行状态有5种。

   ![](https://p0.meituan.net/travelcube/62853fa44bfa47d63143babe3b5a4c6e82532.png)

   其生命周期转换如下入所示。

   ![](https://p0.meituan.net/travelcube/582d1606d57ff99aa0e5f8fc59c7819329028.png)

6. 任务缓冲模块。

   任务缓冲模块是线程池能够管理任务的核心部分。线程池的本质是对任务和线程的管理，而做到这一点最关键的思想就是将任务和线程两者解耦，不让两者直接关联，才可以做后续的分配工作。线程池中是以生产者消费者模式，通过一个阻塞队列来实现的。阻塞队列缓存任务，工作线程从阻塞队列中获取任务。

   阻塞队列(BlockingQueue)是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

   线程池需要管理线程的生命周期，需要在线程长时间不运行的时候进行回收（默认核心线程是不会被回收的，如果需要回收，需要手动设置）。线程池使用一张**HashSet表**（workers）去持有线程的引用，这样可以通过添加引用、移除引用这样的操作来控制线程的生命周期。这个时候重要的就是如何判断线程是否在运行。线程池中线程的销毁依赖JVM自动的回收，线程池做的工作是根据当前线程池的状态维护一定数量的线程引用，防止这部分线程被JVM回收，当线程池决定哪些线程需要回收时，只需要将其引用消除即可。Worker被创建出来后，就会不断地进行轮询，然后获取任务去执行，核心线程可以无限等待获取任务，非核心线程要限时获取任务。当Worker无法获取到任务，也就是获取的任务为空时，循环会结束，Worker会主动消除自身在线程池内的引用。

   Worker是通过继承AQS，使用AQS来实现独占锁这个功能。没有使用可重入锁ReentrantLock，而是使用AQS，为的就是实现**不可重入的特性去反应线程现在的执行状态**。

   1. lock方法一旦获取了独占锁，表示当前线程正在执行任务中。 

   2. 如果正在执行任务，则不应该中断线程。 

   3. 如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可以对该线程进行中断。 

   4. 线程池在执行shutdown方法或tryTerminate方法时会调用interruptIdleWorkers方法来中断空闲的线程，interruptIdleWorkers方法会使用tryLock方法来判断线程池中的线程是否是空闲状态；如果线程是空闲状态则可以安全回收。

7. 真正的任务执行流程。

   1. while循环不断地通过getTask()方法获取任务。 
   2. getTask()方法从阻塞队列中取任务。 
   3. 如果线程池正在停止，那么要保证当前线程是中断状态，否则要保证当前线程不是中断状态。 
   4. 执行任务。 
   5. 如果getTask结果为null则跳出循环，执行processWorkerExit()方法，销毁线程。

### 12. 线程与进程及Java线程模型

1. 进程与线程

   在传统的操作系统（Windows系统）介绍中，会将进程和线程的关系进行如下介绍。

   > 进程：代表一个正在运行的一个应用程序。它有自己独立的存储空间，而多个进程之间数据是相互隔离的。它是操作系统进行资源分配的最小单位。
   >
   > 线程：表示一个能够运行的一个工作者。线程必须依附于进程，不能单独存在。它是CPU调度和分派的基本单位，它可以和同一进程下的其他线程共享进程全部资源。

   线程的状态：新建、就绪、运行、阻塞、死亡。

   Java中线程的状态（参考**Thread.State**类）有：新建（NEW）、可运行（RUNNABLE）、阻塞（BLOCKED）、等待（WAITING）、定时等待（TIMED_WAITING）、死亡（TERMINATED）。

   对Linux系统而言，并没有这么明确的关系划分。在Linux系统中，它所谓的线程其实是**轻量级进程**（LWP）。

   **内核线程**：真正与CPU交互的线程，由操作系统线程调度器去分配CPU执行。

   **轻量级进程**：轻量级进程是由**系统提供给用户操作内核线程的的接口的实现**。它是一种由内核支持的**用户线程**。它是基于内核线程的高级抽象，因此只有先支持内核线程，才能有LWP。

   **用户态线程（协程）**：指的是完全建立在用户空间的线程库，用户线程的建立，同步，销毁，调度完全在用户空间完成，不需要内核的帮助。因此这种线程的操作是极其快速的且低消耗的。

   无论在何种操作系统中，真正的**线程**创建都是在**系统内核态**完成的，它的整个生命周期都是由**操作系统**去维护，但是它同时也提供了对用户的可操作接口的封装和映射。也就是说，真正的线程其实是存在于系统内核态中，而在用户态，可以拿到这样的一个内核线程的映射对象来查看线程的状态。

   在Java中，每一个Thread.start()，都对应Linux一个**fork()**，此时的线程调用将全权交由操作系统的处理，它也会涉及到**用户态与内核态**的来回切换。

   通常单核CPU在某一时刻只能处理一个线程，多核CPU可以处理对应核心数的线程，**处理器能够真正去执行的只有KTL**（内核线程）。**真正执行的超线程技术就是利用特殊的硬件指令，把一个物理芯片模拟成两个逻辑处理核心**，让单个处理器都能使用线程级并行计算，进而兼容多线程操作系统和软件，**减少了CPU的闲置时间**，**提高的CPU的运行效率**。这种超线程技术(如双核四线程)由处理器硬件的决定，同时也需要操作系统的支持才能在计算机中表现出来。

   

   > 附: 关于Linux系统线程创建函数。
   >
   > fork()：是最简单的调用，不需要任何参数，仅仅是在创建一个子进程并为其创建一个独立于父进程的空间。fork使用COW（写时拷贝）机制，并且COW了父进程的栈空间。
   >
   > vfork()：是一个过时的应用，vfork也是创建一个子进程，但是子进程共享父进程的空间。在vfork创建子进程之后，父进程阻塞，直到子进程执行了exec()或者exit()。
   >
   > clone()：是fork的升级版本，不仅可以创建进程或者线程，还可以指定创建新的命名空间（namespace）、有选择的继承父进程的内存、甚至可以将创建出来的进程变成父进程的兄弟进程等等。clone和fork的调用方式也很不相同，clone调用需要传入一个函数，该函数在子进程中执行。此外，clone和fork最大不同在于clone不再复制父进程的栈空间，而是自己创建一个新的。
   >
   > exec()：用来用指定的程序替换当前进程的所有内容。通常用来创建一个全新的程序运行环境。

2. Java线程模型

   Java采用的传统系统模型是1:1的，即**一个LWP线程对应一个KLT线程**。

   > 一对一模型
   >
   > 优点：每个LWP都是独立的调度单元，一个线程阻塞不影响其他线程。
   > 缺点：因为与KLT一对一。而KLT创建，调度需要一定内核资源，因此创建数量有限。
   >
   > 
   >
   > 多对一模型
   >
   > 优点：线程的管理在用户空间进行。比较高效。
   >
   > 缺点：需要用户自己考虑线程的调度相关问题，因为系统对UT无感知，所以一个线程阻塞会导致进程阻塞。
   >
   > 
   >
   > 多对多模型
   >
   > 优点：集合了前两种模型的优点，去掉了他们的缺点。
   >
   > 缺点：实现难度大。

   在HostSpot系列虚拟机的后续版本中将要引入**Loom**框架，它解决的就是Java线程与操作系统线程的对应关系，采用的多对多的方式去支持更高的并发。而Go之所以能够是天生支持高并发，究其原因也是通过**GMP**这种多对多的线程模型来支持的。

### 13. 类加载机制及其相关应用

1. 类加载机制。

   类加载主要包括了：加载、验证、准备、解析、初始化。

   **加载**：将class字节码文件加载到内存中，并将这些数据转换成方法区中的运行时数据（静态变量、静态代码块、常量池等），在堆中生成一个Class类对象代表这个类（反射原理），作为方法区类数据的访问入口。这个文件的来源可以是.class文件、jar包、网络等。

   **验证**：确保加载的类信息符合JVM规范，它的使用不会对JVM造成危害。

   **准备**：正式为类变量(static变量)分配内存并设置类变量初始值的阶段，这些内存都将在方法区中进行分配。注意此时的设置初始值为默认值，具体赋值在初始化阶段完成。

   **解析**：虚拟机常量池内的符号引用替换为直接引用（地址引用）的过程。

   **初始化**：初始化阶段是执行类构造器<clinit>()方法的过程。类构造器<clinit>()方法是由编译器自动收集类中的所有类变量的**赋值**动作和**静态语句块(static块)**中的语句合并产生的。

   - 当初始化一个类的时候，如果发现其父类还没有进行过初始化、则需要先初始化其父类。
   - 虚拟机会保证一个类的<clinit>()方法在多线程环境中被正确加锁和同步。

2. 双亲委派模型。

   简单来说，一个类加载时，会优先让其父类进行加载，从ApplicationClassLoader、到ExtensionClassLoader、再到Bootstrap ClassLoader，层层网上进行加载。如果**父加载器能够加载或者已经加载那么直接返回，否则再交由子加载器去进行加载**。每一个类加载器，都有其对应的加载空间（路径）。

   > 设计初衷：
   >
   > 1. 避免重复加载；一个Claas在方法区中的定位是通过Claas全限定名和其类加载器共同决定。
   > 2. 保证核心`.class`不能被篡改。通过委托方式，不会去篡改核心`.clas`，即使篡改也不会去加载，即使加载也不会是同一个`.class`对象了。不同的加载器加载同一个`.class`也不是同一个`Class`对象。这样保证了`Class`执行安全。

3. 类加载及其破坏。

   1. 第一次。在JDK1.2之前未提出双亲委派模型但是已经存在ClassLoader，有开发者重写loadClass方法来实现用户类加载器。JDK为了兼容，提供了findClaas方法实现。
   2. 第二次。JNDI、JDBC的出现。默认的类加载器只能去加载对应空间（路径）下的Class，而数据库驱动不可能全部有JDK提供，所以提出了SPI这种规范。而 JDBC 的实现类在我们用户定义的 classpath 中，只能由应用类加载器去加载，所以启动类加载器只能委托子类来加载数据库厂商们提供的具体实现，这就违反了自下而上的委托机制。具体解决办法是搞了个线程上下文类加载器，通过setContextClassLoader()默认情况就是应用程序类加载器，然后利用Thread.current.currentThread().getContextClassLoader()获得类加载器来加载。
   3. 第三次。热部署。OSGI 就是利用自定义的类加载器机制来完成模块化热部署，而它实现的类加载机制就没有完全遵循自下而上的委托，有很多平级之间的类加载器查找，具体就不展开了，有兴趣可以自行研究一下。
   4. 第四次。JDK9进行了更新，一些基础jar被移除。当收到类加载请求，会先判断该类在具名模块中是否有定义，如果有定义就自己加载了，没的话再委派给父类。

### 14. 字节码增强技术及相关应用
> 字节码：由16进制数组成的文件称之为字节码。而JVM以两个十六进制值为一组，即以字节为单位进行读取。它与Java跨平台和类Java语言至关重要。Java之所以支持跨平台，在上层它做到了无论在何种操作系统下编译的文件，都能生成统一的字节码文件（.class）；在下层，它屏蔽了底层操作系统的差异性，不会因为操作系统的指令不同而出现不一致情况，只要符合JVM规范的Class文件，都能够被JVM锁加载。这也是JAVA能够经久不衰的原因。

1. 字节码增强技术
   1. ASM（Cglib）

      基于**访问者模式对字节码指令的修改**，**在方法执行之前和返回之前插入对应的字节码**，它是基于Class的代理，对目标类进行方法**重写**，以此实现AOP。如果目标方法上使用了`final`或者`static`修饰，那么这个方法是不能被重写，代理会失效。优点，可以自由控制字节码插入的位置，灵活，代理方法执行速度比JDK代理快。缺点：难度大，对开发者基础知识要求高。

   2. Javassist

      基于ASM的晦涩难懂，出现了**强调源代码层次操作字节码的Javassist**。利用Javassist实现字节码增强时，可以无须关注字节码刻板的结构，其优点就在于编程简单。直接使用java编码的形式，而不需要了解虚拟机指令，就能动态改变类的结构或者动态生成类。

   3. Instrument

      基于编译时插入字节码指令来说实现AOP，就要保证在生成并加载代理类之前没有加载过Target类。否则会抛出类重载错误，**JVM是不允许在运行时动态重载一个类**的。

      instrument是JVM提供的一个可以修改已加载类的类库，专门为Java语言编写的插桩服务提供支持。它需要依赖JVMTI的Attach API机制实现。**它通过接口注册各种事件勾子，在JVM事件触发时，同时触发预定义的勾子，以实现对各个JVM事件的响应**，事件包括**类文件加载**、**异常产生与捕获**、线程启动和结束、进入和退出临界区、成员变量修改、GC开始和结束、方法调用进入和退出、临界区竞争与等待、VM启动与退出等等。

      它的运行借助Agent的能力将Instrument注入到JVM中。通常有两种形式。

      1. 一是随Java进程启动而启动，经常见到的java -agentlib就是这种方式；
      2. 是运行时载入，通过attach API，将模块（jar包）动态地Attach到指定进程id的Java进程内。

### 15. 常用设计模式总结

 	1. 单例模式
 	2. 工厂模式
 	3. 代理模式
 	4. 策略模式（SPI）
 	5. 责任链模式（登录拦截、日志输出）
 	6. 建造者模式（Builder）
 	7. 观察者模式（Spring Event）
 	8. 访问者模式（ASM实现AOP代理）


## 参考链接

1. <a target="_blank" href="https://tech.meituan.com/2020/08/06/new-zgc-practice-in-meituan.html">新一代垃圾回收器ZGC的探索与实践</a>
2. <a target="_blank" href="https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html">从ReentrantLock的实现看AQS的原理及应用</a>
3. <a target="_blank" href="https://github.com/farmerjohngit/myblog/issues/1">死磕Synchronized底层实现</a>
4. <a target="_blank" href="https://github.com/farmerjohngit/myblog/issues/7">Lock(ReentrantLock)底层实现分析</a>
5. <a target="_blank" href="https://javadoop.com/post/AbstractQueuedSynchronizer">一行一行源码分析清楚AbstractQueuedSynchronizer</a>
6. <a target="_blank" href="https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html">Java线程池实现原理及其在美团业务中的实践</a>
7. <a target="_blank" href="http://www.ideabuffer.cn/2017/04/04/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%9AThreadPoolExecutor/">深入理解Java线程池：ThreadPoolExecutor</a>
8. <a target="_blank" href="https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html">字节码增强技术探索</a>

