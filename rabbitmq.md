# 消息队列

- 消息： 应用间传送的数据
- 消息队列： 是一种应用间通信的方式，消息发送后立即返回，由消息系统来确保消息的可靠传递

消息的生产者只管发送消息，不用管谁来取消息，用消息干嘛；消息的消费者只管消费消息，不用管谁发送的消息。异步协作

使用场景： 业务解耦，最终一致性，广播，错峰流控等

## Rabbit MQ

由Erlang开发，实现了AMQP(高级消息队列协议)

1. 可靠性：使用一些机制来保证可靠性，如持久化，传输确认，发布确认
2. 灵活路由：进入消息队列前，通过Exchange来路由消息。对于典型的路由功能，RabbitMQ已经提供了一些内置的Exchange。针对复杂的路由功能，可以将多个Exchange绑定在一起，也可以通过插件机制实现自己的Exchange
3. 消息集群：多个RabbitMQ组成一个集群，形成一个逻辑代理
4. 高可用
5. 多协议
6. 多语言
7. 管理界面
8. 跟踪机制
9. 插件机制

### 消息模型

所有MQ从模型抽象上来讲都是一样的过程：

消费者订阅某个队列，生产者创建消息，然后发布到队列中，最后消息发送到监听的消费者

### rabbit MQ 基本概念

<img src="https://upload-images.jianshu.io/upload_images/5015984-367dd717d89ae5db.png?imageMogr2/auto-orient/strip|imageView2/2/w/554/format/webp"/>

1. Message ： 消息，由消息头和消息体组成。消息头由一系列可选属性组成，包括routing-key(路由键),priority(相对于其他消息的优先权),delivery-mode(指出该消息是否需要持久化存储)等。
2. Publisher:消息生产者
3. Exchange：用于接收生产者发送的消息，并将这些消息路由给服务器中的队列
4. Binding： 用于消息队列和交换器之间的关联。一个绑定就是基于路由键将交换器和消息队列连接起来的路由规则，可以将路由器（Exchange)理解成由Binding构成的路由表
5. Queue：消息队列，用于保存消息直到发送给消费者。是消息的容器，也是消息的终点。消息可以投入多个队列。消息会一直在对垒中，直到被消费者取走。
6. Connection：一个TCP
7. Channel：为了复用TCP而产生都抽象，双向数据流通道。一线程对应一channel，一进程对应一Connection
8. Consumer：消费者，从Queue中取走消息的客户端应用程序
9. Virtual Host：虚拟主机，表示一批交换器，消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个 vhost 本质上就是一个 mini 版的 Rabbit MQ 服务器，拥有自己的队列、交换器、绑定和权限机制。
10. Broker：表示消息队列服务器实体

### AMQP中的消息路由

生产者把消息发送到Exchange上，消息最终到达队列被消费者接收，而Binding决定交换器的消息应该发送到哪个队列。

<img src="https://upload-images.jianshu.io/upload_images/5015984-7fd73af768f28704.png?imageMogr2/auto-orient/strip|imageView2/2/w/484/format/webp"/>



### Exchange类型

Exchange根据分发策略不同，分为4种:headers,direct,fanout,topic.

1. headers，匹配header而不是匹配路由键(routing-key)，和direct完全一致，而且性能差很多，不用了
2. direct：消息中的路由键(routing-key)如果和Binding中的binding key 一致，交换器就将消息发到对应的队列中。完全匹配，单播的模式
3. fanout：广播，每个发到fanout类型交换器的消息都会被分到所有绑定的队列上。fanout不处理路由键，只是简单地将队列绑定到交换器上。fanout类型转发消息是最快的。
4. topic：交换机通过模式匹配分配消息的路由键属性，将路由键和某个模式进行匹配，此时队列需要绑定到一个模式上。她将路由键和绑定的字符串切分成单词，这些单词之间用点号"."隔开。它同时也会识别两个通配符：”符号"#"和符号"*"。星号匹配一个单词，井号匹配零到多个单词。

### rabbit MQ安装

````shell
scoop install rabbitmq
# 打开可视化web管理插件
rabbitmq-plugins enable rabbitmq_management
# 关闭可视化web管理插件
rabbitmq-plugins disable rabbitmq_management
#以应用方式启动
rabbitmq-server # 关闭命令行即停止
#安装为windows服务
rabbitmq-service install
# 以服务方式启动
rabbitmq-service start
# 停止
rabbitmq-service stop
````

### 集群

#### 集群节点类型

- 磁盘节点
- 内存节点

**单节点系统必须是磁盘节点**：否则会丢失配置信息

**集群中至少有一个磁盘节点**：节点离开或者假如集群是必须告知磁盘节点

如果集群中唯一的磁盘节点崩溃，那么集群不能进行以下操作：

- 不能创建队列
- 不能创建交换器
- 不能创建绑定
- 不能添加用户
- 不能更改权限
- 不能增删节点

**解决方案**：设置多个磁盘节点

#### 集群搭建

```shell
# 安装多个RabbitMQ
# cookie值必须相等，
docker run -d --hostname rabbit1 --name myrabbit1 -p 15672:15672 -p 5672:5672 -e RABBITMQ_ERLANG_COOKIE='rabbitcookie' rabbitmq:3.6.15-management

docker run -d --hostname rabbit2 --name myrabbit2 -p 5673:5672 --link myrabbit1:rabbit1 -e RABBITMQ_ERLANG_COOKIE='rabbitcookie' rabbitmq:3.6.15-management

docker run -d --hostname rabbit3 --name myrabbit3 -p 5674:5672 --link myrabbit1:rabbit1 --link myrabbit2:rabbit2 -e RABBITMQ_ERLANG_COOKIE='rabbitcookie' rabbitmq:3.6.15-management
# 节点2加入集群
docker exec -it myrabbit2 bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster --ram rabbit@rabbit1
rabbitmqctl start_app
exit
# 节点3加入集群
docker exec -it myrabbit3 bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster --ram rabbit@rabbit1
rabbitmqctl start_app
exit
## 搭建完成
### 设置节点类型
rabbitmqctl stop_app
rabbitmqctl change_cluster_node_type dist
rabbitmqctl change_cluster_node_type ram
rabbitmqctl start_app
### 移除节点
rabbitmqctl stop_app
rabbitmqctl restart
rabbitmqctl start_app
```

​	集群重启顺序：

- 启动顺序：磁盘节点 => 内存节点
- 关闭顺序：内存节点 => 磁盘节点

磁盘节点的生命周期要比磁盘节点长一点

集群中节点之间只交换元数据，而不互相保存队列中的内容（每个节点负责若干个队列），也就是说如果有单个节点挂掉，并且这个节点是个内存节点，，那么消息就真的丢失了。

#### 镜像队列

作用：双活冗余。

镜像队列在集群中的其他节点拥有从队列拷贝，一旦主节点不可用，最老的从队列，将会选举为新的主队列。

**工作原理**：可以视为隐藏的fanout交换器，将消息发放到主队列与从队列

#### 设置镜像队列

```shell
rabbitmqctl set_policy name regexPattern imageDefinition
# 例如：匹配所有名称是amp开头的队列都存储在2个节点上的命令
rabbitmqctl set_policy mypolicy "^amp*" '{"ha-mode":"exactly","ha-params":2}'
```

- 参数1： 名称 ，随便填
- 参数2：正则匹配模式
- 参数3： 镜像定义：json，有三个属性：ha-mode,ha-params,ha-sync-mode
    - ha-mode 镜像模式，all/exactly/nodes ：all,存储在所有节点，exactly，存储在**指定数量**个节点上，nodes，存储在**指定名称**的节点上
    - ha-params: 参数,ha-mode的补充（指定xx)
    - ha-sync-mode:镜像同步消息的方式：automatic（自动）,manually（手动）



```shell
## 查看镜像队列
rabbitmqctl list_policies
## 删除镜像队列
rabbitmqctl clear_policy
```

