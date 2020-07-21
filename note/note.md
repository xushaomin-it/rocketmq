# RocketMQ学习笔记

## RocketMQ核心目录说明
- broker: broker模块(broker启动进程)
- client: 消息客户端, 包含消息生产者, 消息消费者相关类
- common: 公共包
- dev: 开发者信息
- distribution: 部署实例文件夹
- example: RocketMQ实例代码
- filter: 消息过滤相关基础类
- filtersrv: 消息过滤服务器实现相关类(Filter启动进程)
- logappender: 日志实现相关类
- namesrv: NameServer实现相关类(NameServer启动进程)
- openmessaging: 消息开放标准
- remoting: 远程通信模块, 基于netty
- srvutil: 服务器工具类
- store: 消息存储实现相关类
- style: checkstyle相关实现
- test: 测试相关类
- tools: 工具类, 监控命令相关实现类


## 源码阅读配置准备
在项目路径下新建 myconf目录 详情可看myconf目录配置  
配置`org.apache.rocketmq.namesrv.NamesrvStartup`类启动参数
```
Environment variables: ROCKETMQ_HOME=F:\learn\rocketmq\myconf
```
配置`org.apache.rocketmq.broker.BrokerStartup`类启动参数
```
Program arguments: -c F:\learn\rocketmq\myconf\conf\broker.conf
Environment variables: ROCKETMQ_HOME=F:\learn\rocketmq\myconf
```
配置完成后启动 这两个类

![环境配置1](picture/环境配置1.png)

![环境配置2](picture/环境配置2.png)



