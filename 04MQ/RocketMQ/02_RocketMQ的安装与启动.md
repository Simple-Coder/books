# 第2章 RocketMQ的安装与启动

## 一、基本概念

### 1、消息（Message）

消息是指，消息系统所传输信息的物理载体，生产和消费数据的最小单位，每条消息必须属于一个主题。

### 2、主题（Topic）

![image-20220716000115680](assets/image-20220716000115680.png)

Topic表示一类消息的集合，每个主题包含若干条消息，每条消息只能属于一个主题，是RocketMQ进行消息订阅的基本单位。 topic:message 1:n message:topic 1:1

一个生产者可以同时发送多种Topic的消息；而一个消费者只对某种特定的Topic感兴趣，即只可以订阅和消费一种Topic的消息。 producer:topic 1:n consumer:topic 1:1

### 3、标签（tag）

​		为消息设置的标签，用于同一主题下区分不同类型的消息。来自同一业务单元的消息，可以根据不同业务目的在同一主题下设置不同标签。标签能够有效地保持代码的清晰度和连贯性，并优化RocketMQ提供的查询系统。消费者可以根据Tag实现对不同子主题的不同消费逻辑，实现更好的扩展性。

Topic是消息的一级分类，Tag是消息的二级分类。

Topic：货物

tag=上海

tag=江苏

tag=浙江

------- 消费者 -----

topic=货物 tag = 上海

topic=货物 tag = 上海|浙江

topic=货物 tag = *

### 4、队列（Queue）

存储消息的物理实体。一个Topic中可以包含多个Queue，每个Queue中存放的就是该Topic的消息。一个Topic的Queue也被称为一个Topic中消息的分区（Partition）。

一个Topic的Queue中的消息只能被一个消费者组中的一个消费者消费。一个Queue中的消息不允许同一个消费者组中的多个消费者同时消费。

![image-20220716000340067](assets/image-20220716000340067.png)

在学习参考其它相关资料时，还会看到一个概念：分片（Sharding）。分片不同于分区。在RocketMQ中，分片指的是存放相应Topic的Broker。每个分片中会创建出相应数量的分区，即Queue，每个Queue的大小都是相同的。

![image-20220716000417637](assets/image-20220716000417637.png)

### 5、消息标识（MessageId/key）

RocketMQ中每个消息拥有唯一的MessageId，且可以携带具有业务标识的Key，以方便对消息的查询。不过需要注意的是，MessageId有两个：在生产者send()消息时会自动生成一个MessageId（msgId)，当消息到达Broker后，Broker也会自动生成一个MessageId(offsetMsgId)。msgId、offsetMsgId与key都称为消息标识。

* msgId：由producer端生成，其生成规则为：

  <font color="blue">producerIp + 进程pid + MessageClientIDSetter类的ClassLoader的hashCode +当前时间 + AutomicInteger自增计数器</font>

* offsetMsgId：由broker端生成，其生成规则为：brokerIp + 物理分区的offset（Queue中的偏移量）

* key：由用户指定的业务相关的唯一标识

## 二、系统架构

![image-20220716000640790](assets/image-20220716000640790.png)

RocketMQ架构上主要分为四部分构成：

### 1、Producer

​		消息生产者，负责生产消息。Producer通过MQ的负载均衡模块选择相应的Broker集群队列进行消息投递，投递的过程支持快速失败并且低延迟。

> 例如，业务系统产生的日志写入到*MQ*的过程，就是消息生产的过程 
>
> 再如，电商平台中用户提交的秒杀请求写入到*MQ*的过程，就是消息生产的过程

RocketMQ中的消息生产者都是以生产者组（Producer Group）的形式出现的。生产者组是同一类生产者的集合，这类Producer发送相同Topic类型的消息。一个生产者组可以同时发送多个主题的消息。

### 2、Consumer

​		消息消费者，负责消费消息。一个消息消费者会从Broker服务器中获取到消息，并对消息进行相关业务处理。

> 例如，*QoS*系统从*MQ*中读取日志，并对日志进行解析处理的过程就是消息消费的过程。 
>
> 再如，电商平台的业务系统从*MQ*中读取到秒杀请求，并对请求进行处理的过程就是消息消费的过程。 

​		RocketMQ中的消息消费者都是以消费者组（Consumer Group）的形式出现的。消费者组是同一类消费者的集合，这类Consumer消费的是同一个Topic类型的消息。消费者组使得在消息消费方面，实现<font color="blue">负载均衡</font>（将一个Topic中的不同的Queue平均分配给同一个Consumer Group的不同的Consumer，注意，并不是将消息负载均衡）和<font color="blue">容错</font>（一个Consmer挂了，该Consumer Group中的其它Consumer可以接着消费原Consumer消费的Queue）的目标变得非常容易。

![image-20220716093723482](assets/image-20220716093723482.png)

​		消费者组中Consumer的数量应该小于等于订阅Topic的Queue数量。如果超出Queue数量，则多出的Consumer将不能消费消息。

![image-20220716093747820](assets/image-20220716093747820.png)

​	不过，一个Topic类型的消息可以被多个消费者组同时消费。

>  注意， 
>
> 1）消费者组只能消费一个*Topic*的消息，不能同时消费多个*Topic*消息 
>
> 2）一个消费者组中的消费者必须订阅完全相同的*Topic*

### 3、NameServer

#### **功能介绍**

​		NameServer是一个Broker与Topic路由的注册中心，支持Broker的动态注册与发现。

​		RocketMQ的思想来自于Kafka，而Kafka是依赖了Zookeeper的。所以，在RocketMQ的早期版本，即在MetaQ v1.0与v2.0版本中，也是依赖于Zookeeper的。从MetaQ v3.0，即RocketMQ开始去掉了Zookeeper依赖，使用了自己的NameServer。

主要包括两个功能：

- <font color="red">**Broker管理**</font>：接受Broker集群的注册信息并且保存下来作为路由信息的基本数据；提供心跳检测机制，检查Broker是否还存活。

- <font color="red">**路由信息管理**</font>：每个NameServer中都保存着Broker集群的整个路由信息和用于客户端查询的队列信息。Producer和Conumser通过NameServer可以获取整个Broker集群的路由信息，从而进行消息的投递和消费。

#### **路由注册**

​		NameServer通常也是以集群的方式部署，不过，NameServer是无状态的，即NameServer集群中的各个节点间是无差异的，各节点间相互不进行信息通讯。那各节点中的数据是如何进行数据同步的呢？在Broker节点启动时，轮询NameServer列表，与每个NameServer节点建立长连接，发起注册请求。在NameServer内部维护着⼀个Broker列表，用来动态存储Broker的信息。

> 注意，这是与其它像*zk*、*Eureka*、*Nacos*等注册中心不同的地方。 
>
> 这种*NameServer*的无状态方式，有什么优缺点： 
>
> 优点：*NameServer*集群搭建简单，扩容简单。 
>
> 缺点：对于*Broker*，必须明确指出所有*NameServer*地址。否则未指出的将不会去注册。也正因为如此，*NameServer*并不能随便扩容。因为，若*Broker*不重新配置，新增的*NameServer*对于 *Broker*来说是不可见的，其不会向这个*NameServer*进行注册。 

​		Broker节点为了证明自己是活着的，为了维护与NameServer间的长连接，会将最新的信息以<font color="blue">心跳包</font>的方式上报给NameServer，每30秒发送一次心跳。心跳包中包含 BrokerId、Broker地址(IP+Port)、 Broker名称、Broker所属集群名称等等。NameServer在接收到心跳包后，会更新心跳时间戳，记录这个Broker的最新存活时间。

#### **路由剔除**

​		由于Broker关机、宕机或网络抖动等原因，NameServer没有收到Broker的心跳，NameServer可能会将其从Broker列表中剔除。

NameServer中有⼀个定时任务，每隔10秒就会扫描⼀次Broker表，查看每一个Broker的最新心跳时间戳距离当前时间是否超过120秒，如果超过，则会判定Broker失效，然后将其从Broker列表中剔除。

> 扩展：对于*RocketMQ*日常运维工作，例如*Broker*升级，需要停掉*Broker*的工作。*OP*需要怎么 
>
> 做？
>
> *OP*需要将*Broker*的读写权限禁掉。一旦*client(Consumer*或*Producer)*向*broker*发送请求，都会收 
>
> 到*broker*的*NO_PERMISSION*响应，然后*client*会进行对其它*Broker*的重试。 
>
> 当*OP*观察到这个*Broker*没有流量后，再关闭它，实现*Broker*从*NameServer*的移除。 
>
> *OP*：运维工程师 
>
> *SRE*：*Site Reliability Engineer*，现场可靠性工程师

#### **路由发现**

RocketMQ的路由发现采用的是Pull模型。当Topic路由信息出现变化时，NameServer不会主动推送给客户端，而是客户端定时拉取主题最新的路由。默认客户端每30秒会拉取一次最新的路由。

> 扩展： 
>
> *1*）*Push*模型：推送模型。其实时性较好，是一个*“*发布*-*订阅*”*模型，需要维护一个长连接。而
>
> 长连接的维护是需要资源成本的。该模型适合于的场景： 
>
> 实时性要求较高
>
> *Client*数量不多，*Server*数据变化较频繁 
>
> *2*）*Pull*模型：拉取模型。存在的问题是，实时性较差。 
>
> *3*）*Long Polling*模型：长轮询模型。其是对*Push*与*Pull*模型的整合，充分利用了这两种模型的优 
>
> 势，屏蔽了它们的劣势。

#### 客户端NameServer选择策略

> 这里的客户端指的是*Producer*与*Consumer*

​		客户端在配置时必须要写上NameServer集群的地址，那么客户端到底连接的是哪个NameServer节点呢？客户端首先会生产一个随机数，然后再与NameServer节点数量取模，此时得到的就是所要连接的节点索引，然后就会进行连接。如果连接失败，则会采用round-robbin策略，逐个尝试着去连接其它节点。

首先采用的是<font color="blue">随机策略</font>进行的选择，失败后采用的是<font color="blue">轮询策略</font>。 

> 扩展：*Zookeeper Client*是如何选择*Zookeeper Server*的？ 
>
> 简单来说就是，经过两次*Shuff le*，然后选择第一台*Zookeeper Server*。 详细说就是，将配置文件中的*zk server*地址进行第一次shufle，然后随机选择一个。这个选择出 的一般都是一个*hostname*。然后获取到该*hostname*对应的所有*ip*，再对这些*ip*进行第二次 shuffle，从shuffle过的结果中取第一个*server*地址进行连接。

### 4、Broker

#### **功能介绍**

​		Broker充当着消息中转角色，负责存储消息、转发消息。Broker在RocketMQ系统中负责接收并存储从生产者发送来的消息，同时为消费者的拉取请求作准备。Broker同时也存储着消息相关的元数据，包括消费者组消费进度偏移offset、主题、队列等。

> *Kafka 0.8*版本之后，*offset*是存放在*Broker*中的，之前版本是存放在*Zookeeper*中的。

#### **模块构成**

下图为Broker Server的功能模块示意图

![image-20220716094735611](assets/image-20220716094735611.png)

- <font color="red">**Remoting Module**</font>：整个Broker的实体，负责处理来自clients端的请求。而这个Broker实体则由以下模块构成。

- <font color="red">**Client Manager**</font>：客户端管理器。负责接收、解析客户端(Producer/Consumer)请求，管理客户端。例如，维护Consumer的Topic订阅信息

- <font color="red">**Store Service**</font>：存储服务。提供方便简单的API接口，处理消息存储到物理硬盘和消息查询功能。

- <font color="red">**HA Service**</font>：高可用服务，提供Master Broker 和 Slave Broker之间的数据同步功能。

- <font color="red">**Index Service**</font>：索引服务。根据特定的Message key，对投递到Broker的消息进行索引服务，同时也提供根据Message Key对消息进行快速查询的功能。

#### **集群部署**

![image-20220716095315429](assets/image-20220716095315429.png)

​		为了增强Broker性能与吞吐量，Broker一般都是以集群形式出现的。各集群节点中可能存放着相同Topic的不同Queue。不过，这里有个问题，如果某Broker节点宕机，如何保证数据不丢失呢？其解决方案是，将每个Broker集群节点进行横向扩展，即将Broker节点再建为一个HA集群，解决单点问题。

Broker节点集群是一个主从集群，即集群中具有Master与Slave两种角色。Master负责处理读写操作请求，Slave负责对Master中的数据进行备份。当Master挂掉了，Slave则会自动切换为Master去工作。所以这个Broker集群是主备集群。一个Master可以包含多个Slave，但一个Slave只能隶属于一个Master。 Master与Slave 的对应关系是通过指定相同的BrokerName、不同的BrokerId 来确定的。BrokerId为0表 示Master，非0表示Slave。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有NameServer。

### 5、工作流程

#### 具体流程

- （1）启动NameServer，NameServer启动后开始监听端口，等待Broker、Producer、Consumer连接。

- （2）启动Broker时，Broker会与所有的NameServer建立并保持长连接，然后每30秒向NameServer定时发送心跳包。

- （3）发送消息前，可以先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，当然，在创建Topic时也会将Topic与Broker的关系写入到NameServer中。不过，这步是可选的，也可以在发送消息时自动创建Topic。 

- （4）Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取路由信息，即当前发送的Topic消息的Queue与Broker的地址（IP+Port）的映射关系。然后根据算法策略从队选择一个Queue，与队列所在的Broker建立长连接从而向Broker发消息。当然，在获取到路由信息后，Producer会首先将路由信息缓存到本地，再每30秒从NameServer更新一次路由信息。

- （5）Consumer跟Producer类似，跟其中一台NameServer建立长连接，获取其所订阅Topic的路由信息，然后根据算法策略从路由信息中获取到其所要消费的Queue，然后直接跟Broker建立长连接，开始消费其中的消息。Consumer在获取到路由信息后，同样也会每30秒从NameServer更新一次路由信息。不过不同于Producer的是，Consumer还会向Broker发送心跳，以确保Broker的存活状态。

#### **Topic的创建模式**

手动创建Topic时，有两种模式：

- 集群模式：该模式下创建的Topic在该集群中，所有Broker中的Queue数量是相同的。

- Broker模式：该模式下创建的Topic在该集群中，每个Broker中的Queue数量可以不同。

自动创建Topic时，默认采用的是Broker模式，会为每个Broker默认创建4个Queue。 

#### **读写队列**

从物理上来讲，读/写队列是同一个队列。所以，不存在读/写队列数据同步问题。读/写队列是逻辑上进行区分的概念。一般情况下，读/写队列数量是相同的。

例如，创建Topic时设置的写队列数量为8，读队列数量为4，此时系统会创建8个Queue，分别是0 1 2 3 4 5 6 7。Producer会将消息写入到这8个队列，但Consumer只会消费0 1 2 3这4个队列中的消息，4 5 6 7中的消息是不会被消费到的。

再如，创建Topic时设置的写队列数量为4，读队列数量为8，此时系统会创建8个Queue，分别是0 1 2 3 4 5 6 7。Producer会将消息写入到0 1 2 3 这4个队列，但Consumer只会消费0 1 2 3 4 5 6 7这8个队列中的消息，但是4 5 6 7中是没有消息的。此时假设Consumer Group中包含两个Consuer，Consumer1消费0 1 2 3，而Consumer2消费4 5 6 7。但实际情况是，Consumer2是没有消息可消费的。

也就是说，当读/写队列数量设置不同时，总是有问题的。那么，为什么要这样设计呢？

其这样设计的目的是为了，方便Topic的Queue的缩容。

例如，原来创建的Topic中包含16个Queue，如何能够使其Queue缩容为8个，还不会丢失消息？可以动态修改写队列数量为8，读队列数量不变。此时新的消息只能写入到前8个队列，而消费都消费的却是16个队列中的数据。当发现后8个Queue中的消息消费完毕后，就可以再将读队列数量动态设置为8。整个缩容过程，没有丢失任何消息。

perm用于设置对当前创建Topic的操作权限：2表示只写，4表示只读，6表示读写。

## 三、单机安装与启动

### 1、准备工作

#### **软硬件需求**

系统要求是64位的，JDK要求是1.8及其以上版本的

![image-20220716100256440](assets/image-20220716100256440.png)

#### 下载RocketMQ安装包

![image-20220716100347413](assets/image-20220716100347413.png)

将下载的安装包上传到Linux并解压。

![image-20220716113606042](assets/image-20220716113606042.png)

### 2、修改初始内存

#### 修改runserver.sh

使用vim命令打开`bin/runserver.sh`文件。现将这些值修改为如下：

![image-20220716115548094](assets/image-20220716115548094.png)

#### 修改runbroker.sh

使用vim命令打开`bin/runbroker.sh`文件。现将这些值修改为如下：

![image-20220716115737582](assets/image-20220716115737582.png)

### 3、启动

#### 启动NameServer

```shell
nohup sh bin/mqnamesrv & 

tail -f ~/logs/rocketmqlogs/namesrv.log 
```

![image-20220716121118156](assets/image-20220716121118156.png)

#### 启动broker

```shell
nohup sh bin/mqbroker -n localhost:9876 & 

tail -f ~/logs/rocketmqlogs/broker.log
```

![image-20220716121237807](assets/image-20220716121237807.png)

### 4、发送/接收消息测试

#### 发送消息

```shell
export NAMESRV_ADDR=localhost:9876 
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
```

#### 接收消息

```
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

### 5、关闭server

```
sh bin/mqshutdown broker

sh bin/mqshutdown namesrv

```

## 四、集群搭建理论

![image-20220716122843571](assets/image-20220716122843571.png)

### 1、数据复制与刷盘策略

![image-20220716122912326](assets/image-20220716122912326.png)

#### 复制策略

复制策略是Broker的Master与Slave间的数据同步方式。分为同步复制与异步复制：

- 同步复制：消息写入master后，master会等待slave同步数据成功后才向producer返回成功ACK

- 异步复制：消息写入master后，master立即向producer返回成功ACK，无需等待slave同步数据成功

> 异步复制策略会降低系统的写入延迟，*RT*变小，提高了系统的吞吐量

#### 刷盘策略

刷盘策略指的是broker中`消息`的落盘方式，即消息发送到broker内存后消息持久化到磁盘的方式。分为同步刷盘与异步刷盘：

- 同步刷盘：当消息持久化到broker的磁盘后才算是消息写入成功。

- 异步刷盘：当消息写入到broker的内存后即表示消息写入成功，无需等待消息持久化到磁盘。

> *1*）异步刷盘策略会降低系统的写入延迟，*RT*变小，提高了系统的吞吐量 
>
> *2*）消息写入到*Broker*的内存，一般是写入到了*PageCache* 
>
> *3*）对于异步 刷盘策略，消息会写入到*PageCache*后立即返回成功*ACK*。但并不会立即做落盘操作，而是当*PageCache*到达一定量时会自动进行落盘。

### 2、Broker集群模式

根据Broker集群中各个节点间关系的不同，Broker集群可以分为以下几类：

#### 单Master

只有一个broker（其本质上就不能称为集群）。这种方式也只能是在测试时使用，生产环境下不能使用，因为存在单点问题。

#### 多Master

broker集群仅由多个master构成，不存在Slave。同一Topic的各个Queue会平均分布在各个master节点上。

>  优点：配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢），性能最高；
>
> 缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅（不可消费），消息实时性会受到影响。

> 以上优点的前提是，这些*Master*都配置了*RAID*磁盘阵列。如果没有配置，一旦出现某*Master*宕机，则会发生大量消息丢失的情况。 

#### 多Master多Slave模式-异步复制

​		broker集群由多个master构成，每个master又配置了多个slave（在配置了RAID磁盘阵列的情况下，一个master一般配置一个slave即可）。master与slave的关系是主备关系，即master负责处理消息的读写请求，而slave仅负责消息的备份与master宕机后的角色切换。

​		异步复制即前面所讲的复制策略中的异步复制策略，即消息写入master成功后，master立即向producer返回成功ACK，无需等待slave同步数据成功。

​		该模式的最大特点之一是，当master宕机后slave能够自动切换为master。不过由于slave从master的同步具有短暂的延迟（毫秒级），所以当master宕机后，这种异步复制方式可能会存在少量消息的丢失问题。

> *Slave*从*Master*同步的延迟越短，其可能丢失的消息就越少 
>
> 对于*Master*的*RAID*磁盘阵列，若使用的也是异步复制策略，同样也存在延迟问题，同样也可能会丢失消息。但*RAID*阵列的秘诀是微秒级的（因为是由硬盘支持的），所以其丢失的数据量会更少。 

#### 多Master多Slave模式-同步双写

该模式是多<font color="blue">`Master`</font>多<font color="blue">`Slave`</font>模式的<font color="blue">**`同步复制`**</font>实现。所谓同步双写，指的是消息写入master成功后，master会等待slave同步数据成功后才向producer返回成功ACK，即master与slave都要写入成功后才会返回成功ACK，也即双写。

该模式与异步复制模式相比，优点是消息的安全性更高，不存在消息丢失的情况。但单个消息的RT略高，从而导致性能要略低（大约低10%）。

该模式存在一个大的问题：对于目前的版本，Master宕机后，Slave不会自动切换到Master。

#### 最佳实践

一般会为Master配置RAID10磁盘阵列，然后再为其配置一个Slave。即利用了RAID10磁盘阵列的高效、安全性，又解决了可能会影响订阅的问题。

> *1*）*RAID*磁盘阵列的效率要高于*Master-Slave*集群。因为*RAID*是硬件支持的。也正因为如此， 
>
> 所以*RAID*阵列的搭建成本较高。 
>
> *2*）多*Master+RAID*阵列，与多*Master*多*Slave*集群的区别是什么？ 
>
> - 多*Master+RAID*阵列，其仅仅可以保证数据不丢失，即不影响消息写入，但其可能会影响到消息的订阅。但其执行效率要远高于多Master多Slave集群 
>
> - 多*Master*多*Slave*集群，其不仅可以保证数据不丢失，也不会影响消息写入。其运行效率要低于多Master+RAID阵列

## 五、集群搭建实践

### 1、集群架构

这里要搭建一个双主双从异步复制的Broker集群。为了方便，这里使用了两台主机来完成集群的搭建。这两台主机的功能与broker角色分配如下表。

| 序号 | **主机名**/IP |    **IP**     |      **功能**       |  **BROKER**角色  |
| :--: | :-----------: | :-----------: | :-----------------: | :--------------: |
|  1   |  rocketmq01   | 192.168.50.11 | NameServer + Broker | Master1 + Slave2 |
|  2   |  rocketmq02   | 192.168.50.12 | NameServer + Broker | Master2 + Slave1 |

### 2、修改RocketMQ01配置文件

#### 配置文件位置

要修改的配置文件在rocketMQ解压目录的conf/2m-2s-async目录中。

![image-20220716184612332](assets/image-20220716184612332.png)

#### 修改broker-a.properties

将该配置文件内容修改为如下：

```properties
# 指定整个broker集群的名称，或者说是RocketMQ集群的名称 
brokerClusterName=DefaultCluster 
# 指定master-slave集群的名称。一个RocketMQ集群可以包含多个master-slave集群 
brokerName=broker-a 
# master的brokerId为0 
brokerId=0
# 指定删除消息存储过期文件的时间为凌晨4点 
deleteWhen=04 
# 指定未发生更新的消息存储文件的保留时长为48小时，48小时后过期，将会被删除 
fileReservedTime=48 
# 指定当前broker为异步复制master 
brokerRole=ASYNC_MASTER 
# 指定刷盘策略为异步刷盘 
flushDiskType=ASYNC_FLUSH 
# 指定Name Server的地址 
namesrvAddr=192.168.50.11:9876;192.168.50.12:9876
```

#### 修改broker-b-s.properties

将该配置文件内容修改为如下：

```properties
brokerClusterName=DefaultCluster 
# 指定这是另外一个master-slave集群 
brokerName=broker-b 
# slave的brokerId为非0 
brokerId=1 
deleteWhen=04 
fileReservedTime=48 
# 指定当前broker为slave 
brokerRole=SLAVE 
flushDiskType=ASYNC_FLUSH 
namesrvAddr=192.168.50.11:9876;192.168.50.12:9876
# 指定Broker对外提供服务的端口，即Broker与producer与consumer通信的端口。默认 10911。由于当前主机同时充当着master1与slave2，而前面的master1使用的是默认端口。这 里需要将这两个端口加以区分，以区分出master1与slave2 
listenPort=11911 
# 指定消息存储相关的路径。默认路径为~/store目录。由于当前主机同时充当着master1与 slave2，master1使用的是默认路径，这里就需要再指定一个不同路径 
storePathRootDir=~/store-s 
storePathCommitLog=~/store-s/commitlog 
storePathConsumeQueue=~/store-s/consumequeue 
storePathIndex=~/store-s/index 
storeCheckpoint=~/store-s/checkpoint 
abortFile=~/store-s/abort
```

#### 其他配置

除了以上配置外，这些配置文件中还可以设置其它属性。

```properties
#指定整个broker集群的名称，或者说是RocketMQ集群的名称 
brokerClusterName=rocket-MS 
#指定master-slave集群的名称。一个RocketMQ集群可以包含多个master-slave集群 
brokerName=broker-a
#0 表示 Master，>0 表示 
Slave brokerId=0 
#nameServer地址，分号分割 
namesrvAddr=nameserver1:9876;nameserver2:9876 
#默认为新建Topic所创建的队列数 
defaultTopicQueueNums=4 
#是否允许 Broker 自动创建Topic，建议生产环境中关闭 
autoCreateTopicEnable=true 
#是否允许 Broker 自动创建订阅组，建议生产环境中关闭 
autoCreateSubscriptionGroup=true 
#Broker对外提供服务的端口，即Broker与producer与consumer通信的端口 
listenPort=10911 
#HA高可用监听端口，即Master与Slave间通信的端口，默认值为listenPort+1 
haListenPort=10912 
#指定删除消息存储过期文件的时间为凌晨4点 
deleteWhen=04 
#指定未发生更新的消息存储文件的保留时长为48小时，48小时后过期，将会被删除 
fileReservedTime=48 
#指定commitLog目录中每个文件的大小，默认1G 
mapedFileSizeCommitLog=1073741824 
#指定ConsumeQueue的每个Topic的每个Queue文件中可以存放的消息数量，默认30w条 
mapedFileSizeConsumeQueue=300000 
#在清除过期文件时，如果该文件被其他线程所占用（引用数大于0，比如读取消息），此时会阻止 此次删除任务，同时在第一次试图删除该文件时记录当前时间戳。该属性则表示从第一次拒绝删除 后开始计时，该文件最多可以保留的时长。在此时间内若引用数仍不为0，则删除仍会被拒绝。不过 时间到后，文件将被强制删除 
destroyMapedFileIntervalForcibly=120000 
#指定commitlog、consumequeue所在磁盘分区的最大使用率，超过该值，则需立即清除过期文件
diskMaxUsedSpaceRatio=88 
#指定store目录的路径，默认在当前用户主目录中 
storePathRootDir=/usr/local/rocketmq-all-4.5.0/store 
#commitLog目录路径 
storePathCommitLog=/usr/local/rocketmq-all-4.5.0/store/commitlog 
#consumeueue目录路径 
storePathConsumeQueue=/usr/local/rocketmq-all-4.5.0/store/consumequeue 
#index目录路径 
storePathIndex=/usr/local/rocketmq-all-4.5.0/store/index 
#checkpoint文件路径 
storeCheckpoint=/usr/local/rocketmq-all-4.5.0/store/checkpoint 
#abort文件路径 
abortFile=/usr/local/rocketmq-all-4.5.0/store/abort 
#指定消息的最大大小 
maxMessageSize=65536 
#Broker的角色
# - ASYNC_MASTER 异步复制Master 
# - SYNC_MASTER 同步双写Master 
# - SLAVE
brokerRole=SYNC_MASTER 
#刷盘策略 
# - ASYNC_FLUSH 异步刷盘 
# - SYNC_FLUSH 同步刷盘 
flushDiskType=SYNC_FLUSH 
#发消息线程池数量 
sendMessageThreadPoolNums=128 
#拉消息线程池数量 
pullMessageThreadPoolNums=128 
#强制指定本机IP，需要根据每台机器进行修改。官方介绍可为空，系统默认自动识别，但多网卡 时IP地址可能读取错误 
brokerIP1=192.168.3.105
```

### 3、修改RocketMQ02配置文件

#### 修改broker-b.properties

将该配置文件内容修改为如下：

```properties
brokerClusterName=DefaultCluster 
brokerName=broker-b 
brokerId=0 
deleteWhen=04 
fileReservedTime=48 
brokerRole=ASYNC_MASTER 
flushDiskType=ASYNC_FLUSH 
namesrvAddr=192.168.50.11:9876;192.168.50.12:9876
```

#### 修改broker-a-s.properties

将该配置文件内容修改为如下：

```properties
brokerClusterName=DefaultCluster 
brokerName=broker-a 
brokerId=1 
deleteWhen=04 
fileReservedTime=48 
brokerRole=SLAVE 
flushDiskType=ASYNC_FLUSH
namesrvAddr=192.168.50.11:9876;192.168.50.12:9876
listenPort=11911 
storePathRootDir=~/store-s 
storePathCommitLog=~/store-s/commitlog 
storePathConsumeQueue=~/store-s/consumequeue 
storePathIndex=~/store-s/index 
storeCheckpoint=~/store-s/checkpoint 
abortFile=~/store-s/abort
```

### 4、启动服务器

#### 启动NameServer集群

分别启动rocketmqOS1与rocketmqOS2两个主机中的NameServer。启动命令完全相同。

```shell
nohup sh bin/mqnamesrv & 
tail -f ~/logs/rocketmqlogs/namesrv.log
```

#### 启动两个Master

分别启动rocketmqOS1与rocketmqOS2两个主机中的broker master。注意，它们指定所要加载的配置文件是不同的。

```shell
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-a.properties & 
tail -f ~/logs/rocketmqlogs/broker.log
```

```shell
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-b.properties & 
tail -f ~/logs/rocketmqlogs/broker.log
```

#### 启动两个Slave

分别启动rocketmqOS1与rocketmqOS2两个主机中的broker slave。注意，它们指定所要加载的配置文件是不同的。

```
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-b-s.properties & 
tail -f ~/logs/rocketmqlogs/broker.log
```

```
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-a-s.properties & 
tail -f ~/logs/rocketmqlogs/broker.log
```

## 六、mqadmin命令

在mq解压目录的bin目录下有一个mqadmin命令，该命令是一个运维指令，用于对mq的主题，集群，broker 等信息进行管理。

### 1、修改bin/tools.sh

在运行mqadmin命令之前，先要修改mq解压目录下bin/tools.sh配置的JDK的ext目录位置。本机的ext目录在/usr/java/jdk1.8.0_161/jre/lib/ext 。使用vim命令打开tools.sh文件，并在JAVA_OPT配置的-Djava.ext.dirs这一行的后面添加ext的路径。

![image-20220716190357614](assets/image-20220716190357614.png)

### 2、运行mqadmin

直接运行该命令，可以看到其可以添加的commands。通过这些commands可以完成很多的功能。

### 3、官网命令详解

该命令在官网中有详细的用法解释。

https://github.com/apache/rocketmq/blob/master/docs/cn/operation.md 

![image-20220716190452116](assets/image-20220716190452116.png)