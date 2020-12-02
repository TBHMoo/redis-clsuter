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
集群伸缩=槽和数据在节点之间的移动,是以槽为基本单元移动。

ASK 数据迁移过程时，一个槽的数据同时存在源节点和目标节点中，源节点返回客户端数据可能在目标节点时的意思，
但是目标节点现在还不管理槽，需要客户端使用ASK(命令)强制,要求目标节点处理一次查询请求

MOVED 节点确认槽不属于自己管理，返回正确的节点给客户端 

redis-trib.rb工具支持
```

#### 集群高可用
```shell script
主节点挂了,响应最快的从节点,被提升为主节点
```

# adidas redis sentinel 向 redis cluster 迁移做了什么？
```shell script
理论准备 

现状分析和解决方案
    sentinel 集群现状
        4套sentinel 集群
    客户端使用 jedis
        问题: 缺点需要get值和set值时,需要显示编码序列化和反序列化,影响代码可读性
        方案: RedisTemplate 启动时初始化value的序列化方式,代码简洁,和redis命令契合度高,学习成本小
    存在两种序列化方式
        集成cluster 之后,一个客户端,需要区分使用不同序列化方式的key,
        维护两种序列化方式,代码改动量很大,容易出错,咨询林哥后决定统一成kryo,性能最好 
    上线cluster 时是否需要将 sentinel 中的数据迁移到 cluster? 不需要,memberToken 和refreshToken 影响APP的问题
    spring mvc 项目集成cluster
    
    集群功能限制  
    Redis集群相对单机在功能上存在一些限制
    当key集合属于不同节点时的合并操作,"不支持" (mget,pipline,trasation) 
    "不支持" 需要指定 solt_tag 才能使用
        保持合并操作现状的话，需要保证区分key位于同样的节点，几乎不可能（时间和代码阅读量上）,于是拆分了合并的操作
     (分布式数据库的代价 redis cluster 合并操作，mget， shareding-sphere 的union操作)

开发过程和上线方案
1)环境准备
    本来联系邹大师想要测试环境机器,用于自己搭建cluster 集群,直接得到了一个可用的测试集群! 节约了大量时间
2)重新实现客户端
   
3) 单元测试客户端get,set方法
   
4) 替换原有访问 sentinel 的客户端

5) 这个时候要压测了,beta环境被占用没有环境测试 注册到下单主流程 ,所以在压测环境观察redis相关报错日志加fix...

6) 因为决定不迁移数据,所以没有一次性把 sentinel 配置移除,以防修改不彻底

7）beta环境空出,测试介入主流程，测试了重度使用redis 的场景
    注册到下单主流程 ，登陆加速，商品缓存
8) 上线,谨慎起见让大家陪到了 11点发布（虽然最后看redis 内存和QPS 和5-6点差不多）
9） 林哥发现sentinel 链接数没有按预期下降 = sentinel 没有替换完全... hub,protostar-backend ,wormhole...
10) 移除所有项目对 redis-sentinel 配置的引用, spring boot 项目对cluster的集成

``` 
	
### 问题？

###参考
- 《Redis官方文档》Redis集群教程 :<https://ifeve.com/redis-cluster-tutorial/>
- 《Redis官方教程》Redis集群规范:<https://ifeve.com/redis-cluster-spec/>
-  redis 作者介绍redis cluster : <https://redis.io/presentation/Redis_Cluster.pdf>
- 《Redis深度历险：核心原理和应用实践》 集群3: 众志成城--Cluster
- 《Redis设计与实现 - 黄健宏》 第一部分 第二部分 第三部分
- 《Redis开发与运维(完整版)》 第10章


