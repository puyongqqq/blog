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

   
### 4. RestFul



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

### 6. 如何保证双写一致性？如何平衡系统响应时间。

1. 使用分布式锁。优点：能够保证数据一致性。缺点：增加原本程序的复杂性，增加了成本来保证分布式锁。
2. 将读写操作串行化。优点：能够保证数据一致性。缺点：无法充分发挥现代服务器多核CPU的处理能力。
3. 正常修改数据库。由另一个线程监听binlog来更新缓存。优点：平衡更新和查询的响应速度。缺点：会存在更新延迟。

### 7. CMS，G1，ZGC区别。

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
> Lock：从JDK1.5开始，Java提供了与Synchronized具有相同功能API可调用的一个接口，还提供了尝试获取锁（tryLock），可中断锁（lockInterruptibly）等功能。而它一个常用的实现类ReentrantLock，通过结合AQS实现了相关锁功能，其中还包括了对公平锁和非公平锁的两种实现锁机制下的不同实现。其底层主要是基于CAS（底层依赖于硬件上的原子性）去竞争锁，失败的线程会将其封装成Node，将其顺序串联起来，组成一个双向队列（CLH），然后被LockSupport.park()。

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
      <td>可重入 可判断 可公平（两者皆可）</td>
    </tr>
    <tr>
      <td>性能</td>
      <td>少量同步</td>
      <td>大量同步</td>
    </tr>
  </tbody>
</table>



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
   protected final boolean tryAcquire(int acquires) {
       final Thread current = Thread.currentThread();
       // 获取锁的状态
       int c = getState();
       if (c == 0) {
           // 没有线程持有锁
           if (!hasQueuedPredecessors() &&   // 判断是否有正在排队的线程
               compareAndSetState(0, acquires)) { // 无线程排队。尝试设置state为1
               // 设置当前线程持为持有锁的线程
               setExclusiveOwnerThread(current);
               return true;
           }
       }
       else if (current == getExclusiveOwnerThread()) {
           // 判断是否为持有锁的线程重入，如果是，则state + 1
           int nextc = c + acquires;
           if (nextc < 0)
           	throw new Error("Maximum lock count exceeded");
           setState(nextc);
           return true;
       }
   
       // 1. state开始为0， 但是CAS没有成功
       // 2. 有线程正在持有锁
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

   

2. 非公平锁（ReentrantLock#NonfairSync）只有CLH（双向链表）中的第一个节点和正在尝试获取锁的线程去竞争，第一个节点拿到锁，则新线程会进入等待队列，其余流程和公平锁一致。



### 11. 线程池ThreadPool










## 参考链接

1. <a href="https://tech.meituan.com/2020/08/06/new-zgc-practice-in-meituan.html" target="_blank">新一代垃圾回收器ZGC的探索与实践</a>

2. <a href="https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html" target="_blank">从ReentrantLock的实现看AQS的原理及应用</a>

3. <a href="https://github.com/farmerjohngit/myblog/issues/12" target="_blank">死磕Synchronized底层实现"</a>

4. <a href="https://github.com/farmerjohngit/myblog/issues/7" target="_blank">Lock(ReentrantLock)底层实现分析</a>


