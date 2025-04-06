# Kafka
## 1. 背景

**为什么我们需要Kafka这样一个桥梁**，连接起我们的应用系统、大数据批处理系统，以及大数据流式处理系统

1. 性能瓶颈

   没有kafka之前，一般数据是存放在HDFS上，但是**HDFS（GFS）这样的分布式文件系统，对于单个文件，只适合一个客户端顺序大批量的写入**，所以facebook开源的日志收集器Scribe做了改良，它并不是实时不断地向HDFS写入数据，而是定时地向HDFS上转存（Dump）文件

   ![](../images/Scribe.png)

   还是存在一些问题： 
   - 如果需要收集1min的日志，需要每分钟执行一个MapReduce的任务，不是一个高效的解决问题的办法 
   - 隐式依赖：可能会出现网络中断、硬件故障等等的问题，所以很有可能，在运行MapReduce任务去分析数据的时候，Scribe还没有把文件放到
      HDFS上。那么，MapReduce分析程序，就需要对这种情况进行容错，比如，下一分钟的时候，它需要去读取最近5分钟的数据，看看
      Scribe上传的新文件里，是否会有本该在上一分钟里读到的数据。而这些机制，意味着下游的 MapReduce任务，需要去了解上游的日志收集器的实现机制。并且两边根据像文件名规则之类的隐式协议产生了依赖

2. 和传统的“消息队列”系统有什么不同

   传统的消息队列，则关注的是小数据量下，是否每一条消息都被业务系统处理完成了。因为这些消息队列里的消息，可能就是一笔实际的业务交易，需要等待consumer处理完成，确认结果才行。但是整个系统的吞吐量却没有多大

   Scribe这样的日志收集系统，考虑的是能否高吞吐量地去传输日志，至于下游如何处理日志，它是不关心的。

   Kafka的整体设计，则考虑的是不仅要**实时传输数据**，而且需要下**游有大量不同的业务应用，去消费实时产生的日志文件**。并且，这些数据处理也不是非常重要的金融交易，而是对于大量数据的实时分析，然后反馈到线上

   > kafka后面幂等性，事务，exactly once能否支撑金融交易？
   
## 2.架构

### 2.1 Lambda架构

既然已经可以获得分钟级别的统计数据，那还需要 MapReduce 这样的批处理程序吗？

流式数据处理的问题：
1. 只能保障“至少一次（At Least Once）”的数据处理模式
   
   这个后续的kafka已经实现了exactly once

2. 批处理程序很容易修改，而流式处理程序则没有那么容易

   重放日志需要花费很多时间(没有kafka的情况下)、或者短时间内会消耗很多计算资源

Lambda架构的基本思想

![lambda架构.png](../images/lambda架构.png)

整个数据处理流程抽象成$View = Query(Data)$这样一个函数。我们所有的报表、看板、数据分析结果，都是来自对于原始日志的分析

原始日志就是主数据（Master Data），不管是批处理还是流式处理，都是对于原始数据的一次查询（Query）。而这些计算结果，其实就是一个基于特定查询的视图（View）

Lambda 结构，是由这样几部分组成的：
- 第一部分是输入数据，也就是**Master Data**，这部分就是原始日志
- 然后是一个**批处理层**（Batch Layer）和一个**实时处理层**（Speed Layer），它们分别是一系列MapReduce的任务，和一组Storm的Topology，获取相同的输入数据，然后各自计算出自己的计算结果
- 最后是一个**服务层**（Serving Layer），通常体现为一个数据库。批处理层的计算结果和实时处理层的结果，会写入到这个数据库里。后生成的批处理层的结果，会不断替代掉实时处理层的计算结果，也就是对于最终计算的数据进行修正
- 对于**外部用户**来说，他不需要和批处理层以及实时处理层打交道，而只需要通过像SQL这样的查询语言，直接去查询服务层就好了

### 2.2 Kappa架构

Lambda架构有一个显著的缺点，也就是什么事情都需要做两遍

1. 所有的视图，既需要在实时处理层计算一次，又要在批处理层计算一次。**即使没有修改任何程序，也需要双倍的计算资源**
2. 我们所有的数据处理的程序，也要撰写两遍。MapReduce 的程序和 Storm 的程序虽然要计算的是同样的视图，但是因为底层的框架完全不同，**代码我们就需要写两套。这样意味着，需要双倍的开发资源**

Kafka还没有成熟的时候，把数据分成批处理层和实时处理层是很难避免的。主要问题在于，我们重放实时处理层的日志是个开销很大的动作

![kappa架构.png](kappa架构.png)

相比于Lambda架构，Kappa架构去掉了Lambda 架构的批处理层，而是在**实时处理层，支持了多个视图版本**

如果要对Query进行修改，原来的实时处理层的代码可以先不用动，而是可以先部署一个新版本的代码，比如一个新的Topology。然后，对这个Topology对应日志的重放，在服务层生成一份新的数据结果表，也就是视图的一个新的版本

在日志重放完成之前，外部的用户仍然会查询旧的实时处理层产生的结果。而一旦日志重放完成，新的Topology能够赶上进度，处理到最新产生的日志，那么就可以让查询，切换到新的视图版本上来，旧的实时处理层的代码也就可以停掉了

> 流批一体

## 3. 应用与源码

### 3.1 生产者

生产者客户端的整体架构，如下所示：

![procuder.png](../images/producer.png)

从以下几个方面来看它的实现： 
- 数据分区分配策略 
- meta更新策略 
- RecordAccumulator的实现，即内存管理和分配 
- 网络层

**数据分区分配策略**

有两种分配策略：RoundRobinPartitioner和UniformStickyPartitioner

RoundRobinPartitioner就是每次均匀分配 

UniformStickyPartitioner是默认的Partitioner

KIP-480: Sticky Partitioner引入了UniformStickyPartitioner作为默认的分区器。这个是在Round-robin策略上的优化

会从本地缓存中拿对应topic的分区，所以具有粘性(Sticky),只有当newBatch或者indexCache为空的情况下才会重新计算分区
```Java
public int partition(String topic, Cluster cluster) {
   Integer part = indexCache.get(topic);
   if (part == null) {
   return nextPartition(topic, cluster, -1);
   }
   return part;
}
```

newBatch指的是该batch已经满或者到达了发送的时间。UniformStickyPartitioner计算分区也很简单，即随机数

```Text
if (availablePartitions.size() == 1) {
       newPart = availablePartitions.get(0).partition();
   } else {
       while (newPart == null || newPart.equals(oldPart)) {
           int random = Utils.toPositive(ThreadLocalRandom.current().nextInt());
           newPart = availablePartitions.get(random % availablePartitions.size()).partition();
       }
}
```

但是UniformStickyPartitioner有在某些场景下会有问题，在3.3.0废弃，[KIP-794](https://cwiki.apache.org/confluence/display/KAFKA/KIP-794%3A+Strictly+Uniform+Sticky+Partitioner)做了优化，解决**分配倾斜**

UniformStickyPartitioner会将更多消息分配给速度较慢的broker，并且可能导致“失控”的问题。因为“粘性”时间是由新的批量创建消息驱动的，这与broker的延迟成反比——较慢的broker消耗批量消息的速度较慢，因此它们会比速度更快的分区获得更多的“粘性”时间，从而是消息分配倾斜。

假设一个生产者写入的消息到3个分区（生产者配置为linger.ms=0），并且一个partition由于某种原因（leader broker选举更换或网络抖动问题等情况）导致稍微变慢了一点。生产者必须一直保留发送到这个partition的批次消息，直到partition变得可用。在保留这些批次消息的同时，因为生产者还没有准备好发送到这个分区，其他批次的消息又大批量发送的并开始堆积，从而可能导致每个批次都可能会被填满。

KIP-794对UniformStickyPartitioner做了优化，可以采用自适应分区切换

**切换策略**：分配分区的概率与队列长度成反比

每次partitionReady之后，更新partition的统计信息
```Text
topicInfo.builtInPartitioner.updatePartitionLoadStats(queueSizes, partitionIds, queueSizesIndex + 1);
 .....
partitionLoadStats = new PartitionLoadStats(queueSizes, partitionIds, length);
```

PartitionLoadStats的构造函数
```Java
  private final static class PartitionLoadStats {
    public final int[] cumulativeFrequencyTable;
    public final int[] partitionIds;
    public final int length;
}
```

主要的逻辑是cumulativeFrequencyTable的构造，注释中举了个例子
```Text
Example: 
  假设有3个partitions的队列长度分别是:
  0 3 1
  最大的queue的长度+1则等于3+1=4，再减去每个queue的长度则是
  4 1 3
  再做前缀和，则cumulativeFrequencyTable数组可以赋值为
  4 5 8
```
那构造了CFT数组如何去用呢，取一个随机数[0..8)，然后看它比CFT数组哪个值比较大则去对应下标。如随机到4的话，选择第二个Partition

**RecordAccumulator的实现，即内存管理和分配**

RecordAccumulator主要用来缓存消息以便Sender线程可以批量发送，进而减少网络传输的资源消耗以提升性能

![RecodeAccumulator.png](../images/RecodeAccumulator.png)

什么条件可以发送数据？

```Java
// Sende.java
private long sendProducerData(long now) {
    // 第二次进来的话已经有元数据
    Cluster cluster = metadata.fetch(); // 获取元数据
    // get the list of partitions with data ready to send
    // 判断哪些partition哪些队列可以发送，获取partition的leader partition对应的broker主机
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);
}
```

注释的很清楚，当然，元数据没有的情况下也无法发送
A destination node is ready to send data if:
1. There is at least one partition that is not backing off its send
2. and those partitions are not muted (to prevent reordering if "max.in.flight.requests.per.connection" is set to one)
3. and any of the following are true
   The record set is full
   The record set has sat in the accumulator for at least lingerMs milliseconds
   The accumulator is out of memory and threads are blocking waiting for data (in this case all partitions are immediately considered ready).
   The accumulator has been closed

```Java
public ReadyCheckResult ready(Cluster cluster, long nowMs) {
        Set<Node> readyNodes = new HashSet<>();
        long nextReadyCheckDelayMs = Long.MAX_VALUE;
        Set<String> unknownLeaderTopics = new HashSet<>();

        // free.queued()判断的是BufferPool的this.waiters.size()
        // waiters>0说明内存不够, BufferPool分配内存不够的时候，会执行this.waiters.addLast(moreMemory);
        boolean exhausted = this.free.queued() > 0;
        
        // 遍历所有的分区
        for (Map.Entry<TopicPartition, Deque<ProducerBatch>> entry : this.batches.entrySet()) {
            Deque<ProducerBatch> deque = entry.getValue();
            synchronized (deque) {
                // When producing to a large number of partitions, this path is hot and deques are often empty.
                // We check whether a batch exists first to avoid the more expensive checks whenever possible.
                ProducerBatch batch = deque.peekFirst();
                if (batch != null) {
                    TopicPartition part = entry.getKey();
                    // 根据分区可以获取分区在哪一台leader partition上
                    Node leader = cluster.leaderFor(part);
                    if (leader == null) {
                        // This is a partition for which leader is not known, but messages are available to send.
                        // Note that entries are currently not removed from batches when deque is empty.
                        unknownLeaderTopics.add(part.topic());
                    } else if (!readyNodes.contains(leader) && !isMuted(part)) {
                        long waitedTimeMs = batch.waitedTimeMs(nowMs);
                        
                        // backingOff: 重新发送的数据的时间到了
                        boolean backingOff = batch.attempts() > 0 && waitedTimeMs < retryBackoffMs;
                        
                        // 等待的发送时间在重试的状态下取得是重试等待时间，否则取的是lingerMs，默认是0，来一条消息发送一条消息
                        long timeToWaitMs = backingOff ? retryBackoffMs : lingerMs;
                        
                        // deque.size() > 1说明至少有一个批次已经写满了，因为最少有2个，那第一个肯定是写满的
                       // batch.isFull()说明批次写满了
                        boolean full = deque.size() > 1 || batch.isFull();
                        boolean expired = waitedTimeMs >= timeToWaitMs;
                        boolean transactionCompleting = transactionManager != null && transactionManager.isCompleting();
                        /**
                         * 1. full: 批次写满了发送无论时间有没有到
                         * 2. expired ：时间到了，批次没写满也得发送
                         * 3. exhausted: 内存不够，消息发送之后，会自动释放内存
                         * 4. closed: 关闭现场需要将缓存的数据发生出去
                         */
                        boolean sendable = full
                            || expired
                            || exhausted
                            || closed
                            || flushInProgress()
                            || transactionCompleting;
                        if (sendable && !backingOff) {
                            readyNodes.add(leader);
                        } else {
                            long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                            // Note that this results in a conservative estimate since an un-sendable partition may have
                            // a leader that will later be found to have sendable data. However, this is good enough
                            // since we'll just wake up and then sleep again for the remaining time.
                            nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
                        }
                    }
                }
            }
        }
        return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeaderTopics);
    }
```

**meta更新策略**


#### 网络层

![client元数据更新.png](../images/client元数据更新.png)

wakeup()方法用于唤醒在select()或select(long)方法调用中被阻塞的线程。当selector上的channel无就绪事件时，如果想要唤醒阻塞在select()操作上的线程去处理一些别的工作，可以调用wakeup()方法

![wakeup.png](../images/wakeup.png)

多路复用器获取的是**事件**而不是读取数据

写事件不需要注册，数据准备好之后，再注册写事件，wakeup马上发送数据

#### 应用

1. 无消息丢失配置怎么实现？

   一句话概括，Kafka 只对“已提交”的消息（committed message）做有限度的持久化保证
   
   “消息丢失”案例
   1. 生产者端
      - 设置acks=all。代表了对“已提交”消息的定义
      - Producer要使用带有回调通知producer.send(msg,callback)的发送API，不要使用producer.send(msg)
        。一旦出现消息提交失败的情况，可以有针对性地进行处理
      - 设置retries为一个较大的值。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了retries > 0的Producer
        能够自动重试消息发送，避免消息丢失
   2. 消费者端
      - 维持先消费消息，再更新位移
      - enable.auto.commit=false，手动提交位移
   3. broker端
      - 设置replication.factor>= 3，目前防止消息丢失的主要机制就是冗余
      - unclean.leader.election.enable=false。控制哪些Broker有资格竞选分区的Leader。不允许一个Broker落后原先的Leader太多当Leader，


### Controller

Controller在ZooKeeper的帮助下管理和协调整个Kafka

1. 选举控制器的规则

   第一个成功创建/controller节点的Broker会被指定为控制器

2. 控制器是做什么？
    - 主题管理（创建、删除、增加分区）
    - 分区重分配
    - Preferred 领导者选举
    - 集群成员管理（新增Broker、Broker主动关闭、Broker宕机）
    - 数据服务：控制器上保存了最全的集群元数据信息，其他所有Broker会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据

   **数据控制：** ZooKeeper是整个Kafka集群元数据的“真理之源（Source of Truth）”，那么，Controller承载了ZooKeeper上的所有元数据。
   事实上，集群Broker是不会与ZooKeeper直接交互去获取元数据的。相反地，它们总是与Controller进行通信，获取和更新最新的集群数据

3. 控制器故障转移（Failover）
   当运行中的控制器突然宕机或意外终止时，Kafka能够快速地感知到，并立即启用备用控制器来代替之前失败的控制器

   ![](ControllerFailover.png)
   
   Broker 0是控制器。当Broker 0宕机后，ZooKeeper通过Watch机制感知到并删除了/controller临时节点，然后进行重新选举

#### 实现

新的kafka源码把多线程的方案改成了单线程加事件队列的方案

Controller是在KafkaServer.scala#startup中初始化并且启动的

```Scala
/* start kafka controller */
kafkaController = new KafkaController(config, zkClient, time, metrics, brokerInfo, brokerEpoch, tokenManager, brokerFeatures, featureCache, threadNamePrefix)
kafkaController.startup()
```

```Scala
// 第1步：注册ZooKeeper状态变更监听器，它是用于监听Zookeeper会话过期的
 zkClient.registerStateChangeHandler(new StateChangeHandler {
   override val name: String = StateChangeHandlers.ControllerHandler
   override def afterInitializingSession(): Unit = {
     eventManager.put(RegisterBrokerAndReelect)
   }
   override def beforeInitializingSession(): Unit = {
     val queuedEvent = eventManager.clearAndPut(Expire)

     // Block initialization of the new session until the expiration event is being handled,
     // which ensures that all pending events have been processed before creating the new session
     queuedEvent.awaitProcessing()
   }
 })
 
 // 第2步：写入Startup事件到事件队列
 eventManager.put(Startup)
 
 // 第3步：启动ControllerEventThread线程，开始处理事件队列中的ControllerEvent
 eventManager.start()
```

这里主要看下ControllerEventManager eventManager是Controller事件管理器，负责管理事件处理线程

## 延时操作模块

因未满足条件而暂时无法被处理的Kafka请求

举个例子，配置了acks=all的生产者发送的请求可能一时无法完成，因为Kafka必须确保ISR中的所有副本都要成功响应这次写入。因此，通常情况下，这些请求没法被立即处理。只有满足了条件或发生了超时，Kafka才会把该请求标记为完成状态

### 怎么实现延时请求呢？

#### 1. 为什么不用Java的DelayQueue

DelayQueue有一个弊端：它插入和删除队列元素的时间复杂度是O(logN)。对于Kafka这种非常容易积攒几十万个延时请求的场景来说，该数据结构的性能是瓶颈

```Java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {

    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();

    /**
     * Thread designated to wait for the element at the head of
     * the queue.  This variant of the Leader-Follower pattern
     * (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to
     * minimize unnecessary timed waiting.  When a thread becomes
     * the leader, it waits only for the next delay to elapse, but
     * other threads await indefinitely.  The leader thread must
     * signal some other thread before returning from take() or
     * poll(...), unless some other thread becomes leader in the
     * interim.  Whenever the head of the queue is replaced with
     * an element with an earlier expiration time, the leader
     * field is invalidated by being reset to null, and some
     * waiting thread, but not necessarily the current leader, is
     * signalled.  So waiting threads must be prepared to acquire
     * and lose leadership while waiting.
     */
    private Thread leader;

    /**
     * Condition signalled when a newer element becomes available
     * at the head of the queue or a new thread may need to
     * become leader.
     */
    private final Condition available = lock.newCondition();

}
```

这边说用了Leader-Follower
pattern，刚开始没看懂，[stackoverflow](https://stackoverflow.com/questions/48493830/what-exactly-is-the-leader-used-for-in-delayqueue)
说的比较清楚。
Leader的用处主要是minimize unnecessary timed
waiting，其实是多个线程不必要的唤醒和睡眠。如果让所有线程都available.awaitNanos(delay)
进入take方法，它们将同时被调用，但只有一个可以真正从队列中获取元素，其他线程将再次陷入休眠，这是不必要的且浪费资源。

采用Leader-Follower模式，Leader available.awaitNanos(delay)、Follower available.await()
。因此Leader将首先醒来并检索过期的元素，然后在必要时向另一个等待线程发出信号。这样效率更高

主要的实现是take和offer方法

```Java
public boolean offer(E e) {
     final ReentrantLock lock = this.lock;
     lock.lock();
     try {
         q.offer(e); // q保存的是Delayed类型的数据，根据Comparable比较的小根堆
         if (q.peek() == e) { // 如果插入的元素在堆顶，唤醒其他可能在等待的线程
             leader = null;
             available.signal();
         }
         return true;
     } finally {
         lock.unlock();
     }
 }
```

take方法的实现
```Java
 public E take() throws InterruptedException {
     final ReentrantLock lock = this.lock;
     lock.lockInterruptibly();
     try {
         for (;;) {
             E first = q.peek();
             if (first == null)
                 available.await();
             else {
                 long delay = first.getDelay(NANOSECONDS);
                 if (delay <= 0L)
                     return q.poll();
                 first = null; // don't retain ref while waiting
                 if (leader != null) // leader在则等leader线程先获取，等待唤醒
                     available.await();
                 else {
                     Thread thisThread = Thread.currentThread();
                     leader = thisThread;
                     try {
                         available.awaitNanos(delay); // leader拿到的堆顶的元素，等待delay时间
                     } finally {
                         if (leader == thisThread)
                             leader = null;
                     }
                 }
             }
         }
     } finally {
         if (leader == null && q.peek() != null)
             available.signal();
         lock.unlock();
     }
 }
```


![zk临时节点.png](zk临时节点.png)

基于这种临时节点的机制，Controller 定义了 BrokerChangeHandler 监听器，专门负责监听 /brokers/ids 下的子节点数量变化。
一旦发现新增或删除 Broker，/brokers/ids 下的子节点数目一定会发生变化。这会被 Controller 侦测到，进而触发 BrokerChangeHandler 的处理方法，即 handleChildChange 方法。


config.brokerId：静态的配置文件


/**
* Returns true if this broker is the current controller.
  */
def isActive: Boolean = activeControllerId == config.brokerId

broker咋确定自己是不是Controller，成为Controller的成本


```Scala
  def registerZNodeChangeHandlerAndCheckExistence(zNodeChangeHandler: ZNodeChangeHandler): Boolean = {
    zooKeeperClient.registerZNodeChangeHandler(zNodeChangeHandler)
    val existsResponse = retryRequestUntilConnected(ExistsRequest(zNodeChangeHandler.path))
    existsResponse.resultCode match {
      case Code.OK => true
      case Code.NONODE => false
      case _ => throw existsResponse.resultException.get
    }
  }
```

```Scala
  case Startup => processStartup()

  private def processStartup(): Unit = {
    zkClient.registerZNodeChangeHandlerAndCheckExistence(controllerChangeHandler)
    elect()
  }  
  
  private def elect(): Unit = {
  // 1. 看数据能不能取回
    activeControllerId = zkClient.getControllerId.getOrElse(-1)
    /*
     * We can get here during the initial startup and the handleDeleted ZK callback. Because of the potential race condition,
     * it's possible that the controller has already been elected when we get here. This check will prevent the following
     * createEphemeralPath method from getting into an infinite loop if this broker is already the controller.
     */
    if (activeControllerId != -1) {
      debug(s"Broker $activeControllerId has been elected as the controller, so stopping the election process.")
      return
    }

    try {
    // 这边抛异常
      val (epoch, epochZkVersion) = zkClient.registerControllerAndIncrementControllerEpoch(config.brokerId)
      
      // 只有controller才会走下面的逻辑
      controllerContext.epoch = epoch
      controllerContext.epochZkVersion = epochZkVersion
      // 2. 临时设置自己为Controller
      activeControllerId = config.brokerId

      info(s"${config.brokerId} successfully elected as the controller. Epoch incremented to ${controllerContext.epoch} " +
        s"and epoch zk version is now ${controllerContext.epochZkVersion}")

      onControllerFailover()
    } catch {
      case e: ControllerMovedException =>
        maybeResign()

        if (activeControllerId != -1)
          debug(s"Broker $activeControllerId was elected as controller instead of broker ${config.brokerId}", e)
        else
          warn("A controller has been elected but just resigned, this will result in another round of election", e)
      case t: Throwable =>
        error(s"Error while electing or becoming controller on broker ${config.brokerId}. " +
          s"Trigger controller movement immediately", t)
        triggerControllerMove()
    }
  }
  
```

MetadataCache