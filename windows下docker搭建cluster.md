#搭建集群
>搭建集群分为四步  
- 准备节点，启动6个集群模式孤儿节点
- 节点间握手通信
- 分配数据槽
- 设置从节点

1）准备节点  本地使用docker启动6个集群模式节点。
```shell script
windows 安装docker
拉取redis 镜像
docker pull redis




```

1.a) 本地创建redis 集群配置文件  (注意路径不能有中文)
- vi create.sh
```shell script
#!/bin/sh
for port in `seq 6379 6384`
do
  mkdir -p ${port}/conf
  mkdir -p ${port}/data
  mkdir -p ${port}/log
done

chmod 777 create.sh
```

1.b) 依次在 ${port}/conf下新增配置文件
redis_${port}.conf


vi redis_port_conf.sh

```shell script
#!/bin/sh
for i in `seq 6379 6384`
do
  cd ./${i}/conf
  echo "#节点端口
        port ${i}
        # 开启集群模式
        cluster-enabled yes
        # 节点超时时间，单位毫秒
        cluster-node-timeout 15000
        # 集群内部配置文件
        cluster-config-file nodes-${i}.conf"  > redis-${i}.conf
  cd ../../  #回到下一个目录下
done
```


```shell script
# docker查看容器分配给redis 集群节点的ip
> docker inspect redisnode1 redisnode2 redisnode3 redisnode4 redisnode5 redisnode6 | grep -E "IPA|redisnode|HostPort"
        "Name": "/redisnode1",
                        "HostPort": "6379"
                        "HostPort": "6379"
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.2",
                    "IPAMConfig": null,
                    "IPAddress": "172.17.0.2",
        "Name": "/redisnode2",
                        "HostPort": "6380"
                        "HostPort": "6380"
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.3",
                    "IPAMConfig": null,
                    "IPAddress": "172.17.0.3",
        "Name": "/redisnode3",
                        "HostPort": "6381"
                        "HostPort": "6381"
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.4",
                    "IPAMConfig": null,
                    "IPAddress": "172.17.0.4",
        "Name": "/redisnode4",
                        "HostPort": "6382"
                        "HostPort": "6382"
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.5",
                    "IPAMConfig": null,
                    "IPAddress": "172.17.0.5",
        "Name": "/redisnode5",
                        "HostPort": "6383"
                        "HostPort": "6383"
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.6",
                    "IPAMConfig": null,
                    "IPAddress": "172.17.0.6",
        "Name": "/redisnode6",
                        "HostPort": "6384"
                        "HostPort": "6384"
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.7",
                    "IPAMConfig": null,
                    "IPAddress": "172.17.0.7",
```

```shell script
# 上面的启动的孤儿节点IP和端口情况
redisnode1 172.17.0.2 6379
redisnode2 172.17.0.3 6380
redisnode3 172.17.0.4 6381
redisnode4 172.17.0.5 6382
redisnode5 172.17.0.6 6383
redisnode6 172.17.0.7 6384
```

设置数据卷 用于挂载到启动的容器上
```shell script
>docker run --name redisnode1  -p 6379:6379 -v /E/work/docker/volume/redis/6379/conf/redis_6379.conf:/etc/redis/redis_6379.conf -v /E/work/docker/volume/redis/6379/data:/data -v /E/work/docker/volume/redis/6379/log:/log -d redis:latest redis-server /etc/redis/redis_6379.conf --cluster-enabled yes --port  6379 --cluster-node-timeout 15000 --cluster-config-file nodes-6379.conf
>docker run --name redisnode2  -p 6380:6379 -v /E/work/docker/volume/redis/6380/conf/redis_6380.conf:/etc/redis/redis_6380.conf -v /E/work/docker/volume/redis/6380/data:/data -v /E/work/docker/volume/redis/6380/log:/log -d redis:latest redis-server /etc/redis/redis_6380.conf --cluster-enabled yes --port  6380 --cluster-node-timeout 15000 --cluster-config-file nodes-6380.conf
>docker run --name redisnode3  -p 6381:6379 -v /E/work/docker/volume/redis/6381/conf/redis_6381.conf:/etc/redis/redis_6381.conf -v /E/work/docker/volume/redis/6381/data:/data -v /E/work/docker/volume/redis/6381/log:/log -d redis:latest redis-server /etc/redis/redis_6381.conf --cluster-enabled yes --port  6381 --cluster-node-timeout 15000 --cluster-config-file nodes-6381.conf
>docker run --name redisnode4  -p 6382:6379 -v /E/work/docker/volume/redis/6382/conf/redis_6382.conf:/etc/redis/redis_6382.conf -v /E/work/docker/volume/redis/6382/data:/data -v /E/work/docker/volume/redis/6382/log:/log -d redis:latest redis-server /etc/redis/redis_6382.conf --cluster-enabled yes --port  6382 --cluster-node-timeout 15000 --cluster-config-file nodes-6382.conf
>docker run --name redisnode5  -p 6383:6379 -v /E/work/docker/volume/redis/6383/conf/redis_6383.conf:/etc/redis/redis_6383.conf -v /E/work/docker/volume/redis/6383/data:/data -v /E/work/docker/volume/redis/6383/log:/log -d redis:latest redis-server /etc/redis/redis_6383.conf --cluster-enabled yes --port  6383 --cluster-node-timeout 15000 --cluster-config-file nodes-6383.conf
>docker run --name redisnode6  -p 6384:6379 -v /E/work/docker/volume/redis/6384/conf/redis_6384.conf:/etc/redis/redis_6384.conf -v /E/work/docker/volume/redis/6384/data:/data -v /E/work/docker/volume/redis/6384/log:/log -d redis:latest redis-server /etc/redis/redis_6384.conf --cluster-enabled yes --port  6384 --cluster-node-timeout 15000 --cluster-config-file nodes-6384.conf

```

2）节点握手。
```shell script
>cluster meet host1 port1
>cluster meet host2 port2
>cluster meet host3 port3
>cluster meet host4 port4
>cluster meet host5 port5
>cluster meet host6 port6

# 节点建立握手之后集群还不能正常工作，这时集群处于下线状态，所有
  的数据读写都被禁止
```


3）分配槽。 {0..5000}   {0...5000} 写法不同
```shell script
root@97f6629e7f6a:/data# redis-cli -h 172.17.0.2 -p 6379 cluster addslots {0..5000}
OK
root@97f6629e7f6a:/data# redis-cli -h 172.17.0.3 -p 6380 cluster addslots {5001..10000}
OK
root@97f6629e7f6a:/data# redis-cli -h 172.17.0.4 -p 6381 cluster addslots {10001..16383}
OK
```


4)设置从节点

节点ID在集群初始化时只创建一次，会被持久化到配置文件中
```shell script 
# 查找nodeId
172.17.0.2:6379> cluster nodes 
c842afc66bf0693c4d4bace463b1efe36e8e677b 172.17.0.7:6384@16384 master - 0 1606644479855 5 connected
6ae787029fe418439a34e0fcf89c53f80b28aa4c 172.17.0.3:6380@16380 master - 0 1606644479000 1 connected 5001-10000
27bd6fd3b08b1d27b6105ac404791e6a943563d7 172.17.0.4:6381@16381 master - 0 1606644478000 2 connected 10001-16383
dd9a8d8593aa9ed10526e5e5fe2080b9f78894c9 172.17.0.2:6379@16379 myself,master - 0 1606644477000 0 connected 0-5000
5fbdc46f395f67846d8e0032554e55b1bc1a1dd1 172.17.0.5:6382@16382 master - 0 1606644478000 3 connected
619ef65eae65daf02639a6e15a1987fc513832a0 172.17.0.6:6383@16383 master - 0 1606644477847 4 connected

# 设置主从
# 在从节点上执行  cluster replicate masternodeId 
root@97f6629e7f6a:/data# redis-cli -h 172.17.0.5 -p 6382 cluster replicate dd9a8d8593aa9ed10526e5e5fe2080b9f78894c9
OK
root@97f6629e7f6a:/data# redis-cli -h 172.17.0.6 -p 6383 cluster replicate 6ae787029fe418439a34e0fcf89c53f80b28aa4c
OK
root@97f6629e7f6a:/data# redis-cli -h 172.17.0.7 -p 6384 cluster replicate 27bd6fd3b08b1d27b6105ac404791e6a943563d7
OK
172.17.0.2:6379> cluster nodes
c842afc66bf0693c4d4bace463b1efe36e8e677b 172.17.0.7:6384@16384 slave 27bd6fd3b08b1d27b6105ac404791e6a943563d7 0 1606644861007 2 connected
6ae787029fe418439a34e0fcf89c53f80b28aa4c 172.17.0.3:6380@16380 master - 0 1606644858000 1 connected 5001-10000
27bd6fd3b08b1d27b6105ac404791e6a943563d7 172.17.0.4:6381@16381 master - 0 1606644860000 2 connected 10001-16383
dd9a8d8593aa9ed10526e5e5fe2080b9f78894c9 172.17.0.2:6379@16379 myself,master - 0 1606644859000 0 connected 0-5000
5fbdc46f395f67846d8e0032554e55b1bc1a1dd1 172.17.0.5:6382@16382 slave dd9a8d8593aa9ed10526e5e5fe2080b9f78894c9 0 1606644859000 0 connected
619ef65eae65daf02639a6e15a1987fc513832a0 172.17.0.6:6383@16383 slave 6ae787029fe418439a34e0fcf89c53f80b28aa4c 0 1606644860002 1 connected
```


dummy client
```shell script

172.17.0.2:6379> set foo bar
(error) MOVED 12182 172.17.0.4:6381
MOVED 错误表示这个 key foo 对应的槽 12182 是节点172.17.0.4:6381 负责，你应该连 172.17.0.4:6381 去执行命令 set foo bar
```

smart client
```shell script
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

# 一个客户端通常会维护和集群节点的常驻链接 
# dummy client 会多消耗一次IO，所以现在的客户端实现，都使用的 smart client
```

手动
redis-trib.rb

```shell script

```
理解集群建立的流程和细节，不过现在有成熟的搭建工具，比如 redis-trib.rb 和 官网的 dockr 镜像