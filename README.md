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
   
