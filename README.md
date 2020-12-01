#redis cluster 迁移分享

- 为什么迁移 redis cluster ?
- redis cluster 是什么？
- redis sentinel 向 redis cluster 迁移做了什么？

#为什么迁移 redis cluster ?
- 业务角度


    - 省钱
- 技术角度


    - 无redis db
    - 单点
    - 主从
    - sentinel 
        - 数据集中内存占用大
        - 单CPU峰值压力大 
    - cluster  
        - 多CPU
        - 数据分散存储

#redis cluster 是什么？
![1.png](https://i.loli.net/2020/11/29/ryc8oSQd3RufBIa.png)

集群示意图

![集群示意图.png](https://i.loli.net/2020/11/29/TeVIr9Pv8D5cMwU.png)

#### 数据存储方式  

```shell script
- 虚拟槽分区(16384)
>slot = crc16("foo") mod NUMER_SLOTS
Redis虚拟槽分区的特点：
解耦了数据与节点之间的关系
·解耦数据和节点之间的关系，简化了节点扩容和收缩难度。
·节点自身维护槽的映射关系，不需要客户端或者代理服务维护槽分区
元数据。
·支持节点、槽、键之间的映射查询，用于数据路由、在线伸缩等场
景。
```

#### 节点通信
```shell script
Redis集群采用P2P的Gossip（流言）协议
节点间会频繁的通信，交换信息，一段时间后所有节点都会知道集群完整的信息，就像流言传播方式一样
集群信息变更，会更新节点纪元, 其他节点接收信息会采用新纪元信息，吃最新的瓜
```


#### 集群在线伸缩
```shell script
集群伸缩=槽和数据在节点之间的移动， 是以槽为基本单元移动。
MOVED 节点确认槽不属于自己管理，返回正确的节点给客户端 
ASK 数据迁移过程时，一个槽的数据同时存在源节点和目标节点中，源节点返回客户端数据可能在目标节点时的意思，
但是目标节点现在还不管理槽，需要客户端使用ASK(命令)强制,要求目标节点处理一次查询请求

redis-trib.rb工具支持
```

# adidas redis sentinel 向 redis cluster 迁移做了什么？


	
### 问题？

###参考
- 《Redis官方文档》Redis集群教程 :<https://ifeve.com/redis-cluster-tutorial/>
- 《Redis官方教程》Redis集群规范:<https://ifeve.com/redis-cluster-spec/>
-  redis 作者介绍redis cluster : <https://redis.io/presentation/Redis_Cluster.pdf>
- 《Redis深度历险：核心原理和应用实践》 集群3: 众志成城--Cluster
- 《Redis设计与实现 - 黄健宏》 第一部分 第二部分 第三部分
- 《Redis开发与运维(完整版)》 第10章


