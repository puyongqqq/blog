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

6. 其他问题

   
