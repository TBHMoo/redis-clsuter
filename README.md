#redis cluster 迁移分享

- 为什么迁移 redis cluster ?
- redis cluster 是什么？
- redis sentinel 向 redis cluster 迁移做了什么？

##为什么迁移 redis cluster ?
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

##redis cluster 是什么？
- redis cluster 是 redis实现的 高可用分布式数据库

![Pandao editor.md](https://pandao.github.io/editor.md/images/logos/editormd-logo-180x180.png "Pandao editor.md")


####数据存储方式  
- 虚拟槽分区(16384)
>slot = crc16("foo") mod NUMER_SLOTS
Redis虚拟槽分区的特点：
解耦了数据与节点之间的关系
·解耦数据和节点之间的关系，简化了节点扩容和收缩难度。
·节点自身维护槽的映射关系，不需要客户端或者代理服务维护槽分区
元数据。
·支持节点、槽、键之间的映射查询，用于数据路由、在线伸缩等场
景。

2、集群功能限制  
Redis集群相对单机在功能上存在一些限制

 (分布式数据库的代价 redis cluster 合并操作，mget， shareding-sphere 的union操作)

1、key作为数据分区的最小粒度，因此不能将一个大的键值对象如
2、key事务操作支持有限。同理只支持多key在同一节点上的事务操
作，当多个key分布在不同的节点上时无法使用事务功能。
3）key作为数据分区的最小粒度，因此不能将一个大的键值对象如
hash、list等映射到不同的节点。
4）不支持多数据库空间。单机下的Redis可以支持16个数据库，集群模
式下只能使用一个数据库空间，即db0。

#搭建集群
- 1）准备节点。 以集群模式启动节点 cluster-enabled yes
- 2）节点握手。 节点间通信，cluster meet host port 
节点建立握手之后集群还不能正常工作，这时集群处于下线状态，所有
的数据读写都被禁止。
- 3）分配槽。 {0..5000}   {0...5000} 写法不同
 cluster addslots {0..5000}
 cluster addslots {5001..10000}
 cluster addslots {10001..16383}
- 4) 设置从节点


172.17.0.2:6379> cluster nodes
c842afc66bf0693c4d4bace463b1efe36e8e677b 172.17.0.7:6384@16384 slave 27bd6fd3b08b1d27b6105ac404791e6a943563d7 0 1606644861007 2 connected
6ae787029fe418439a34e0fcf89c53f80b28aa4c 172.17.0.3:6380@16380 master - 0 1606644858000 1 connected 5001-10000
27bd6fd3b08b1d27b6105ac404791e6a943563d7 172.17.0.4:6381@16381 master - 0 1606644860000 2 connected 10001-16383
dd9a8d8593aa9ed10526e5e5fe2080b9f78894c9 172.17.0.2:6379@16379 myself,master - 0 1606644859000 0 connected 0-5000
5fbdc46f395f67846d8e0032554e55b1bc1a1dd1 172.17.0.5:6382@16382 slave dd9a8d8593aa9ed10526e5e5fe2080b9f78894c9 0 1606644859000 0 connected
619ef65eae65daf02639a6e15a1987fc513832a0 172.17.0.6:6383@16383 slave 6ae787029fe418439a34e0fcf89c53f80b28aa4c 0 1606644860002 1 connected

dummy client
172.17.0.2:6379> set foo bar
(error) MOVED 12182 172.17.0.4:6381
MOVED 错误表示这个 key foo 对应的槽 12182 是节点172.17.0.4:6381 负责，你应该连 172.17.0.4:6381 去执行命令 set foo bar

smart client
172.17.0.2:6379> cluster slots  / 有的版本是 cluster hints
1) 1) (integer) 5001
   2) (integer) 10000
   3) 1) "172.17.0.3"
      2) (integer) 6380
      3) "6ae787029fe418439a34e0fcf89c53f80b28aa4c"
   4) 1) "172.17.0.6"
      2) (integer) 6383
      3) "619ef65eae65daf02639a6e15a1987fc513832a0"
2) 1) (integer) 10001
   2) (integer) 16383
   3) 1) "172.17.0.4"
      2) (integer) 6381
      3) "27bd6fd3b08b1d27b6105ac404791e6a943563d7"
   4) 1) "172.17.0.7"
      2) (integer) 6384
      3) "c842afc66bf0693c4d4bace463b1efe36e8e677b"
3) 1) (integer) 0
   2) (integer) 5000
   3) 1) "172.17.0.2"
      2) (integer) 6379
      3) "dd9a8d8593aa9ed10526e5e5fe2080b9f78894c9"
   4) 1) "172.17.0.5"
      2) (integer) 6382
      3) "5fbdc46f395f67846d8e0032554e55b1bc1a1dd1"

172.17.0.2:6379> cluster keyslot foo
(integer) 12182

一个客户端通常会维护和集群节点的常驻链接

我想找邹大师要几台测试机器，去搭建集群的时候
ps.  docker 已经有成熟的redis cluster镜像工具，一键生成 redis cluster 集群。

理解集群建立的流程和细节，不过现在有成熟的搭建工具，比如 redis-trib.rb 和 官网的 dockr 镜像


#搭建集群

##为什么是redis cluster
1、业务层面
2、技术层面

迁移redis cluster
0、理论准备
《redis 深入历险》 
《Redis 设计与实现》     X2 
《Redis 开发与运维》   redis 官网推荐

1、搭建一个测试环境的 redis cluster 集群用于测试


2、替换 代码中所有使用 redis 的地方  
	a、弄清项目中目前redis 的使用情况  
	    4套 redis sentinel 集群 负责不同的业务场景

	    发现问题，存在两种序列化方式 部分key

	b、确认改动点
	
### 问题？

###参考
- 《Redis官方文档》Redis集群教程 :<https://ifeve.com/redis-cluster-tutorial/>
- 《Redis官方教程》Redis集群规范:<https://ifeve.com/redis-cluster-spec/>
- 《Redis深度历险：核心原理和应用实践》 集群3: 众志成城--Cluster
- 《Redis设计与实现 - 黄健宏》 第一部分 第二部分 第三部分
- 《Redis开发与运维(完整版)》 第10章

