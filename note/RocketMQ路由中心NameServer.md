## RocketMQ路由中心NameServer
### NameServer架构设计, 启动流程
#### 1. NameServer架构设计
![RocketMQ物理部署图](picture/RocketMQ物理部署图.png)

Broker 消息服务器在启动时向所有 Name Server 注册，消息生产者（Producer）在发送消
息之前先从 Name Server 获取 Broker 服务器地址列表，然后根据负载算法从列表中选择一
台消息服务器进行消息发送. NameSErver与每台Broker服务器保持长链接,
并间隔30s检测Broker是否存活, 如果检测到Broker宕机, 则从路由注册表中将其移除,
但是路由变化不会马上通知消息生产者, 这是为了降低NameServer实现的复杂性,
在消息发送段提供容错机制来保证消息发送的高可用性.

#### 2. NameServer启动流程
- 第一步: 解析配置文件
- 第二步: 根据启动属性创建NamesrvController实例, 并初始化该实例,
  NameServerController实例为NameServer核心控制器
- 第三步: 注册JVM钩子函数并启动服务器, 以便监听Broker, 消息生产者的网络请求

```java
 public static NamesrvController main0(String[] args) {

        try {
            // NameServer启动三个步骤

            // 1. 创建NamesrvController, 在创建之前, 会先读取系统配置, NamesrvController为NameServer核心控制器
            NamesrvController controller = createNamesrvController(args);
            // 2. 启动NameServer, 在启动的同时注册jvm钩子, 启动服务器, 监听Broker, 消息生产者的网络请求
            start(controller);
            String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
            log.info(tip);
            System.out.printf("%s%n", tip);
            return controller;
        } catch (Throwable e) {
            e.printStackTrace();
            System.exit(-1);
        }

        return null;
    }
```
### NameServer路由注册,故障剔除
NameServer主要作用是为消息生产者和消息消费者提供关于主题Topic的路由信息,
所以NameServer需要存储路由的基础信息, 还要能够管理Broker节点, 包括路由注册,
路由删除等功能.

- `org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager`
  NameServer路由实现类

NameServer主要存储信息
```java
    // topic消息队列路由信息, 消息发送时根据路由表进行负载均衡
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    // Broker基础信息, 包含brokerName, 所属集群名称, 主备Broker地址
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    // Broker集群信息
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    // Broker状态
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    // Broker的FilterServer列表, 用于类模式消息过滤
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

##### NameServer路由注册

RocketMQ路由注册是通过Broker与NameServer的心跳功能实现的,
Broker启动时向集群猴子那个所有的NameServer发送心跳语句,
每个30s向集群中所有NameServer发送心跳包,
NameServer收到Broker心跳包时会更新brokerLiveTable缓存中BrokerLiveInfo的lastUpdateTimestamp,
然后Name Server每隔10s扫描brokerLiveTable, 如果连续120s没有收到心跳包,
NameServer将移除该Broker的路由信息同时关闭Socket连接.

1. Broker发送心跳包

- `org.apache.rocketmq.broker.BrokerController.start`

```java
// Broker发送心跳包定时任务, 每隔30s向NameServer发送心跳包
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
                } catch (Throwable e) {
                    log.error("registerBrokerAll Exception", e);
                }
            }
        }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
```

- `org.apache.rocketmq.broker.out.BrokerOuterAPI#registerBrokerAll`

该方法主要是遍历NameServer列表, Broker消息服务器一次向NameServer发送心跳包

```java
for (final String namesrvAddr : nameServerAddressList) { // 遍历所有NameServer列表
                brokerOuterExecutor.execute(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            // 分别向NameServer注册
                            RegisterBrokerResult result = registerBroker(namesrvAddr,oneway, timeoutMills,requestHeader,body);
                            if (result != null) {
                                registerBrokerResultList.add(result);
                            }

                            log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
                        } catch (Exception e) {
                            log.warn("registerBroker Exception, {}", namesrvAddr, e);
                        } finally {
                            countDownLatch.countDown();
                        }
                    }
                });
            }
```

2. NameServer处理心跳包

`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor`网络处理器解析请求类型,
如果请求类型为`RequestCode REGISTER_BROKER`,
则请求最终转发到`RouteInfoManager#registerBroker`

`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager.registerBroker`
中clusterAddrTable的维护
```java
                // 加写锁, 防止并发修改RouteInfoManager中的路由表
                this.lock.writeLock().lockInterruptibly();

                Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
                // 1. 判断Broker所属集群是否存在, 如果不存在, 则创建
                if (null == brokerNames) {
                    brokerNames = new HashSet<String>();
                    this.clusterAddrTable.put(clusterName, brokerNames);
                }
                brokerNames.add(brokerName);

```

brokerAddrTable维护  
维护BrokerData信息, 先从brokerAddrTable根据BrokerName尝试获取Broker信息,
如果不存在, 则新建BrokerData, 并放入到brokerAddrTable, registerFirst设置为true; 如果存在, 直接替换原先的, registerFirst = false, 表示非第一次注册

```java
 BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                // brokerAddrTable的维护
                if (null == brokerData) {
                    registerFirst = true;
                    brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                    this.brokerAddrTable.put(brokerName, brokerData);
                }
```

topicQueueTable维护

```java
 if (null != topicConfigWrapper
                    && MixAll.MASTER_ID == brokerId) {
                    if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                        || registerFirst) {
                        ConcurrentMap<String, TopicConfig> tcTable =
                            topicConfigWrapper.getTopicConfigTable();
                        if (tcTable != null) {
                            for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                                // 根据topicConfig创建QueueData数据结构, 然后更新topicQueueTable
                                this.createAndUpdateQueueData(brokerName, entry.getValue());
                            }
                        }
                    }
                }
```

更新BrokerLiveInfo, 存活Broker信息表, BrokeLiveInfo是执行路由删除的重要依据

最后, 注册Broker的过滤器Server地址列表,
一个Broker上会关联多个FilterServer消息过滤服务器

小结:  
在NameServer的路由注册中, NameServe与Broker保持长链接,
Broker状态存储在brokerLiveTable中, NameServer每收到一个心跳包,
将更新brokerLiveTable中关于Broker的状态信息以及路由表(topicQueueTable,
brokerAddrTable, brokerLiveTable, filterServerTable).
使用了读写锁对并发写的控制, 但能够保证并发读, 保证消息发送的高并发.