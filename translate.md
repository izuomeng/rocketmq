# 第4组 文档翻译(第四题)
# 1 
# NameServer最佳实践
在 Apache RocketMQ 中，NameServer(名称服务器)旨在协调分布式系统的各个组件，而协调的方式主要是通过管理主题路由信息来实现的。

管理由两部分组成：

- Brokers 定期更新保存在每个名称服务器中的元数据。
- 名称服务器是为客户端提供最新的路由信息服务的，包括生产者、消费者和命令行客户端。

### 编程的方式

对于 brokers，我们可以在 broker 的配置文件中指定`namesrvAddr=name-server-ip1:port;name-server-ip2:port`

对于生产者和消费者，我们可以给他们提供下列姓名服务器地址列表：

```
DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
producer.setNamesrvAddr("name-server1-ip:port;name-server2-ip:port");

DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
consumer.setNamesrvAddr("name-server1-ip:port;name-server2-ip:port");
```

如果你从 shell 中使用管理命令行，你也可以这样指定：

```
sh mqadmin command-name -n name-server-ip1:port;name-server-ip2:port -X OTHER-OPTION
```

一个简单的例子是 `sh mqadmin -n localhost:9876 clusterList` 指定在名称服务器节点上查询集群信息。

如果你将管理工具集成到你自己的仪表盘中，你可以：

```
DefaultMQAdminExt defaultMQAdminExt = new DefaultMQAdminExt("please_rename_unique_group_name");
defaultMQAdminExt.setNamesrvAddr("name-server1-ip:port;name-server2-ip:port");
```

### Java 参数

还可以通过指定后续的 java 参数 rocketmq.namesrvv.addr 来为你的应用程序提供名称服务器地址列表。

### 环境变量

你可以设置 NAMESRV_ADDR 环境变量。如果设置了，Broker 和 clients 将检查并使用这个值。

### HTTP 端点(HTTP Endpoint)

如果你没有使用前面提到的方法指定名称服务器地址列表，Apache RocketMQ 将每 2 分钟访问一次 HTTP 端点以获取和更新名称服务器地址列表，初始延迟 10 秒。

默认情况下，终点是：

```
http://jmenv.tbsite.net:8080/rocketmq/nsaddr
```

你可以使用这个 Java 选项：`rocketmq.namesrv.domain` 覆盖`jmenv.tbsite.net`，也可以使用 `rocketmq.namesrv.domain.subgroup` 覆盖 `nsaddr` 部分

如果在生产环境中运行 Apache RocketMQ，建议使用此方法，因为它提供了最大程度的灵活性——你可以动态地添加或删除名称服务器节点，而无需根据你的名称服务器的系统负载重新启动代理和客户端。

### 优先级

先介绍的方法优先于后一种方法：

编程方式 > Java 选项 > 环境变量 > HTTP 端点

# 2 
# 生产者的最佳实践

为用户提供的一些小技巧

## 发送状态 SendStatus

当发送一则消息时，你会得到一个发送结果，结果中包含了发送状态。首先，我们假定消息的 isWaitStoreMsgOK 字段值为 true(默认配置)，否则在没有异常抛出的情况下，这个字段的值总是 SEND_OK，下面是对其他可能值的描述:

### FLUSH_DISK_TIMEOUT

如果 Broker 将消息存储配置的 FlushDiskType 字段设置为 SYNC_FLUSH(默认是 ASYNC_FLUSH)，并且
Broker 没有在消息存储配置的 syncFlushTimeout 时间内(默认是 5 秒)完成磁盘刷新，会得到这个返回值

### FLUSH_SLAVE_TIMEOUT

如果 Broker 的角色是 SYNC_MASTER(默认是 ASYNC_MASTER)，并且 slave Broker 没有在消息存储配置的 syncFlushTimeout 时间内(默认是 5 秒)完成与主 Broker 的同步，会得到这个返回值

### SLAVE_NOT_AVAILABLE

如果 Broker 的角色是 SYNC_MASTER(默认是 ASYNC_MASTER)，但是没有配置 slave Broker，会得到这个返回值

### SEND_OK

得到这个返回值并不意味着它是可靠的。为了确保不会丢失消息，还应该启用 SYNC_MASTER 或者 SYNC_FLUSH

### Duplication 或者 Missing

如果你得到 FLUSH_DISK_TIMEOUT 和 FLUSH_SLAVE_TIMEOUT 这两个返回值，并且 Broker 在这个时候刚好宕机了，你会丢失所发送的消息。这种时候你有两个选择，一是放任不管，当然这个消息也会丢失；二是重新发送这个消息，这有可能会引发消息重复。通常情况下我们建议重发消息并且在消费时处理消息重复的问题，除非你觉得丢失几条消息也没什么大不了。但是要注意，如果你得到的返回值是 SLAVE_NOT_AVAILABLE，那么重发消息也是没用的，这时你应该记录下来，向集群管理员反映问题。

## 超时

在客户端(Client)向 Broker 发送请求并等待响应的过程中，如果经过最大等待时间仍然未得到响应，客户端会给出一个远程超时的警告。一般而言，客户端默认的等待时间是 3 秒，但也可以利用 send(msg, timeout)代替 send(msg)的方式，对超时参数进行修改。由于 Broker 将消息写入磁盘或进行主从同步的操作本身需要花费一定的时间，因此并不建议将超时时间设置得过小。而如果超时参数超过 syncFlushTimeout 也没什么大的影响，因为 Broker 可能在超时之前已经返回了 FLUSH_SLAVE_TIMEOUT 或 FLUSH_SLAVE_TIMEOUT。

## 消息大小

建议发送消息的大小不要超过 512k。

## 异步发送

默认的 send(msg)方式会使得操作在前一个响应信息返回之前一直处于中断状态。因此，如果你想要获得更好的性能，建议使用 send(msg, timeout)代替 send(msg)，这种方式可以异步返回响应信息。

## 生产者组

一般而言，信息的生产者对于整个过程信息传递的流程并没有什么影响。在默认条件下，同一个 JVM 中，相同的生产者组只能创建一个生产者，通常情况下这是足够的。但如果你处理的是一个复杂的事务，则需要进一步关注生产者对于整个流程的影响。

## 线程安全

生产者是线程安全的，可以放心使用。

## 性能

如果你需要处理大数据，并要在一个 JVM 中使用超过一个生产者，我们建议你：

- 用较少的生产者进行异步发送(3-5)个足够
- 为每个生产者设置实例名称

