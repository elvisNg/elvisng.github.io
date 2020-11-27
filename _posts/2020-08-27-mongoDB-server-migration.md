---
layout: post
title: "MongoDB-Server-Migration"
subtitle: 'Mongo'
author: "Elvis"
header-style: text
tags:
  - MongoServer
  - 技术分享

---



## Mongo服务器迁移，单机To副本集，不停机

#### #  背景：

线上多个应用运用同一个数据库，数据库采用主从架构，因为历史原因，使用了普通硬盘，并且线上A应用最近用户急增，读写总量曾高达150万qps，所以急需做横向扩容，并且写入更新量较高，所以采用双主方案。导致IO持续100%，所以数据库不能停机迁移数据。

#### #  方案：

用mongoShake做线上服务器数据动态迁移，三台SSD搭建副本集方案。Mongo版本Mongo3.6.4

#### # 实践遇到的坑：

1. mongoShake源直接配置为从库，不适用主库迁移。报错Connect to mongodb://admin:***@127.0.0.1:18808 failed. no reachable servers，服务不可达。需把主从节点都配上，把mongo_connect_mode设置为secondaryPreferred，让mongoshake自适应选择主/从库作为数据源拉取数据
2. 虽然新数据库架构配置了SSD，但mongoShake的瓶颈就落在了网络上了，因为数据近百G导致同步速度比预期的慢。需先把新的副本集先不配置从节点，让mongoShake到主节点所有数据完成后，再配置上副本集，使从节点数据同步。



#### #方案实践：

#### 1.1 配置mongo1.conf

```
bind_ip=0.0.0.0
port=18808
directoryperdb=true
dbpath=/data/solution/mongodb3.6/data
logpath=/data/solution/mongodb3.6/log/mongod.log
pidfilepath=/data/solution/mongodb3.6/mongod.pid
fork=true
serviceExecutor=adaptive 
wiredTigerCacheSizeGB=96
logappend=true
#先屏蔽以下三个参数，设置主库账密
#auth=true
#replSet=rs0
#keyFile=/data/solution/mongodb3.6/mongodb-keyfile

```

#### 1.2 启动mongodb1

```
./bin/mongod -f etc/mongo1.conf
```

#### 1.3 配置主库账密（注意修改密码）

```
./mongo --port 18808
use admin
db.createUser({ user: "admin", pwd: "xxx", roles: [{role:"userAdminAnyDatabase", db: "admin"}]})
```

#### 1.4  开启副本集与账密验证，mongo1.conf（注意修改集群名称）

```
bind_ip=0.0.0.0
port=18808
directoryperdb=true
dbpath=/data/solution/mongodb3.6/data
logpath=/data/solution/mongodb3.6/log/mongod.log
pidfilepath=/data/solution/mongodb3.6/mongod.pid
fork=true
serviceExecutor=adaptive 
wiredTigerCacheSizeGB=96
logappend=true
auth=true
replSet=rs0
keyFile=/data/solution/mongodb3.6/mongodb-keyfile
```

#### 1.5 升级admin账号权限，配置初始化副本集

###### 因初始化的admin权限只为admin库的管理权限，需升级为全库集群权限

```

db.grantRolesToUser( "admin" , [ { role: "dbOwner", db: "admin" },{ "role": "clusterAdmin", "db": "admin" },{ "role": "userAdminAnyDatabase", "db": "admin" },{ "role": "dbAdminAnyDatabase", "db": "admin" },{ role: "root", db: "admin" } ])
```

###### 配置初始化副本集

```
rsconf = {_id: "rs0",members: [{_id: 0,host: "mongodb1.example.net:18808"}]}
rs.initiate(rsconf)
```

###### 添加副本集节点

```
rs.add("mongodb2.example.net")
rs.add("mongodb3.example.net")
```



以上副本集就搭建完成，下面是mongoShake迁移操作。

#### 2.1 修改mongoShake配置，collector_mongodb0.conf

###### 			配置参考collector.conf

```
# 同步模式，all表示全量+增量同步，full表示全量同步，incr表示增量同步。
sync_mode = all

# mongodb://username1:password1@host1,host2 注意密码不能用@ 用了需要用%40转码
mongo_urls = mongodb://username1:password1@primaryA,secondaryB

# tunnel pipeline type. now we support rpc,file,kafka,mock,direct
# 通道模式。
tunnel = direct
#目的端连接串信息，逗号分隔不同的mongod
incr_sync.tunnel.address = mongodb://username:password@10.5.5.5:5005, 10.6.6.6:6006, 10.7.7.7:7007 
# 如果希望以change stream拉取，该值需要配置change_stream，支持>=4.0.1版本。
incr_sync.mongo_fetch_method = oplog 
```



####  2.2 启动collector

```
# ./start.sh
# MONGOSHAKE_OPLOGNS=oplog.\$main ./collector.linux -conf=./collector_mongodb0.conf 1>> ./c.output 2>&1
export MONGOSHAKE_OPLOGNS=oplog.\$main
./hypervisor --daemon --exec="./collector.linux -conf=./collector_mongodb0.conf 1>> ./c.output 2>&1" 1>>hypervisor.output 2>&1
#也可以以nohup方式启动
#!/usr/bin/env bash
export MONGOSHAKE_OPLOGNS=oplog.\$main
nohup ./collector.linux -conf=./collector_mongodb0.conf >> /data/solution/mongoshake/collections_output.log 2>&1 &
```



#### 2.3 监控mongoShake同步情况

```
vinllen@~/code/MongoShake$ curl -s  http://127.0.0.1:9101/progress | python -m json.tool
{
    "progress": "23.40%", // 大概的进度，100%表示全量同步完成
    "total_collection_number": 47, // 一共有多少个表
    "finished_collection_number": 11, // 已经完成同步表的数目
    "processing_collection_number": 6, // 正在同步的表的数目
    "wait_collection_number": 30, // 等待同步的表的数目
    "collection_metric": { // 各个表同步的详细信息
        "a.b": "100.00% (4/4)", // db=a, collection=b, 同步完成100%，一共有4条数据，同步了4条数据
        "pock.hh": "9.20% (8064/87667)",  // db=pock, collection=hh, 同步完成9.20%，一共有87667条数据，同步了8064条数据
        "pock.hh2": "100.00% (28/28)",
        "test.mmm": "100.00% (1/1)",
        "test.test": "100.00% (7/7)",
        "ycsb.usertable2": "0.32% (3584/1113323)",
        "ycsb.usertable3": "0.26% (2176/822695)",
        "ycsb.usertable4": "0.51% (2176/428967)",
        "yy.x": "100.00% (4/4)",
        "zz-2018-12-11.cc1": "0.00% (0/289)",
        "zz-2018-12-11.cc2": "0.00% (0/308)",
        "zz-2018-12-11.cc3": "-", // -表示还未开始同步
        "zz-2018-12-12.cc1": "-",
        "zz.flush": "100.00% (1/1)",
        "zz.mmm": "100.00% (25/25)",
        "zz.x": "100.00% (42/42)",
        ... // 省略部分
    }
}
```



