---
layout: post
title: "mongoshake-master-slave"
subtitle: 'Mongo线上高可用双主架构优化性能提高45%'
author: "Elvis"
header-style: text
mermaid: true
tags:
  - Mongo(Master-Slave)
  - MongoShake
  - Mongo3.6.4
---

### 现状：

##### 背景：

线上多个应用运用同一个数据库，所以数据库不能停机迁移数据。并且线上A应用最近用户急增，读写总量曾高达150万qps，所以急需做横向扩容，并且写入更新量较高，所以采用双主方案。

##### 方案：

采用mongoShake做线上数据动态迁移，仅剩的两台机做互为主从方案。Mongo版本Mongo3.6.4

![](/img/in-post/optimizemongo/mongoshake-maseter-slave.png)

##### 实践遇到的坑：

1. 互为主从架构属于3.4版本的Feature，所以系统会自动把version降到3.4版本，具体表现为账密无法验证。需把等级用命令升级为3.6版本。
2. 互为主从的覆盖，第一点的修改版本和新建账密，必须两个数据库已启动并建立互为主从关系。不然会被覆盖
3. mongShake迁移的速度过快，主从同步的速度有瓶颈，并且差距过大。默认主从同步会加上--autoresync 参数来防止容灾，但这里需要去除，不然通过主从去同步数据的时候mongodb2会因为与mongodb1数据差距过大而进行回滚。
4. mongShake迁移会往mongodb1写oplog，oplog默认大小只有20G，因为oplog的数据结构是链表，当写的速度过快，读的数据过慢会产生主从同步完成，但数据丢掉的问题。数据20G以上的此方案建议oplog扩容。


## 方案实践



#### 1.1 开启主从复制安全验证

随机生成keyFile或者手动写入,key的长度必须是6-1024的base64字符，unix必须相同组权限。--auth --keyFile参数指定

```bash
shell>openssl rand -base64 128 > /data/solution/mongo/key

shell>chmod 400 /data/solution/mongo/key
```



#### 1.2 启动mongodb1

> 注：以下必须关闭自动重新同步，不然mongoshake使得主从差异过大，自动重新同步会破坏已建立的主从库关系，导致所有验证失效。

启动mongodb1（不启动差异自动重新同步）

```bash
nohup ./mongod --port 18809 --bind_ip 0.0.0.0 --dbpath  /data/solution/mongo/mongodbdata/   --master --slave --source sourcehost1:18809 --wiredTigerCacheSizeGB 96 --auth --keyFile /data/solution/mongo/key 1>>output.log 2>&1 &
```

#### 1.3 启动mongodb2（参照mongogdb1）

启动mongodb2（不启动差异自动重新同步）

```bash
nohup ./mongod --port 18809 --bind_ip 0.0.0.0 --dbpath  /data/solution/mongo/mongodbdata/   --master --slave --source sourcehost2:18809 --wiredTigerCacheSizeGB 96 --auth --keyFile /data/solution/mongo/key 1>>output.log 2>&1 &
```

设置admin用户

```bash
./mongo --port 18809
shell> use admin
shell> db.createUser({user:"admin",pwd:"admin",roles:["root"]})

```

配置setFeatureCompatibilityVersion，并检查集合db.system.version.find()，确保集合存在，并版本确定为3.6

```bash
./mongo --port 18809
shell> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
shell> db.adminCommand( { setFeatureCompatibilityVersion: "3.6" } )
shell> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
```


### 2.1 启动mongoshake，同步原数据库mongodb0到mongodb1

#### mongoshake collector启动配置

配置参考collector.conf

```txt
# 同步模式，all表示全量+增量同步，full表示全量同步，incr表示增量同步。
sync_mode = all

# mongodb://username1:password1@host
mongo_urls = mongodb://mongodb0:18088

# tunnel pipeline type. now we support rpc,file,kafka,mock,direct
# 通道模式。
tunnel = direct

# direct模式用于直接写入MongoDB，其余模式用于一些分析，或者远距离传输场景，
# 注意，如果是非direct模式，需要通过receiver进行解析，具体参考FAQ文档。
# 此处配置通道的地址，格式与mongo_urls对齐。
tunnel.address = mongodb://mongodb1:18088
```

#### 2.2 启动collector

```bash
# ./start.sh
# MONGOSHAKE_OPLOGNS=oplog.\$main ./collector.linux -conf=./collector_mongodb0.conf 1>> ./c.output 2>&1
export MONGOSHAKE_OPLOGNS=oplog.\$main
./hypervisor --daemon --exec="./collector.linux -conf=./collector_mongodb0.conf 1>> ./c.output 2>&1" 1>>hypervisor.output 2>&1
#也可以以nohup方式启动
#!/usr/bin/env bash
export MONGOSHAKE_OPLOGNS=oplog.\$main
nohup ./collector.linux -conf=./collector_mongodb0.conf >> /data/solution/mongoshake/collections_output.log 2>&1 &
```



#### 2.3 mongoshake全量同步后关闭mongoshake

```bash
ps -ef | grep collector
kill -9 pid
```

> mongoshake全量同步后自动切换为增量同步，并生成一个mongoshake的库，存储最后一次增量同步的时间与节点，所以直接kill掉就好



#### 3.1 等待mongodb1、mongodb2互为主从同步完成所有数据后，以autoresync重新启动mongodb1、mongodb2

重启mongodb1（开启自动重新同步）

```bash
nohup ./mongod --port 18809 --bind_ip 0.0.0.0 --dbpath  /data/solution/mongo/mongodbdata/   --master --slave --source sourcehost1:18809 --autoresync --wiredTigerCacheSizeGB 96 --auth --keyFile /data/solution/mongo/key 1>>output.log 2>&1 &
```

重启mongodb2（开启差异自动重新同步）

```bash
nohup ./mongod --port 18809 --bind_ip 0.0.0.0 --dbpath  /data/solution/mongo/mongodbdata/   --master --slave --source sourcehost2:18809 --autoresync --wiredTigerCacheSizeGB 96 --auth --keyFile /data/solution/mongo/key 1>>output.log 2>&1 &
```



#### 3.2 重启mongoshake

```bash
export MONGOSHAKE_OPLOGNS=oplog.\$main
./hypervisor --daemon --exec="./collector.linux -conf=./collector_mongodb0.conf 1>> ./c.output 2>&1" 1>>hypervisor.output 2>&1
```




### 4.1 在原数据库mongodb0所在服务器启动mongodb0-1

启动mongodb0-1

```bash
./mongod --port 18809 --bind_ip 0.0.0.0 --dbpath ./db --master --wiredTigerCacheSizeGB 1
```

### mongoshake同步数据切换

#### 停止mongodb0到mongodb1的mongoshake同步

```bash
ps -ef | grep collector
kill -9 xxx
```

#### 4.2 mongodb1作为源库，mongodb0-1作为目标库，通过mongoshake全量加增量同步数据

```bash
export MONGOSHAKE_OPLOGNS=oplog.\$main
./hypervisor --daemon --exec="./collector.linux -conf=./collector_mongodb1.conf 1>> ./c.output 2>&1" 1>>hypervisor.output 2>&1
```

#### 4.3 mongodb2作为源库，mongodb0-1作为目标库，增量同步数据

注意：这一步在mongodb1同步完成后再操作

```bash
export MONGOSHAKE_OPLOGNS=oplog.\$main
./hypervisor --daemon --exec="./collector.linux -conf=./collector_mongodb2.conf 1>> ./c.output 2>&1" 1>>hypervisor.output 2>&1
```

### 4.4 切换应用服务访问的数据库为mongodb1和mongodb2