---
layout: post
title: "Hyperledger-Fabric-Install"
subtitle: '区块链技术分享'
author: "Elvis"
header-style: text
tags:
  - 区块链
  - 技术分享
---

###  区块链基本概念

##### 定义

- **区块链**起源于中本聪的比特币，作为比特币的底层技术，本质上是一个**去中心化的数据库**。
-  **区块链技术**是一种不依赖第三方、通过自身分布式节点进行网络数据的存储、验证、传递和交流的一种**技术方案**。
-  区块链是一种由多方共同维护，使用密码学保证传输和访问安全，能够实现数据一致存储、无法篡改、无法抵赖的**技术体系**。
-  大多为**块链结构**实现数据存储；也存在非链式的DAG、Hashgraph等区块链系统
- 区块链技术只能确保链上数据的真实性与可信性，可防止不被轻易攻击且不易被随意篡改，却无法解决链下虚假数据或真实数据“转移”到链上数据过程中的真实性问题。

##### 公有链vs联盟链vs私有链

- 公有链：任意区块链服务客户都可以使用，任意节点均可接入，由所有节点共同参与共识和读写数据，具有较强的去中心化特征，如比特币和以太坊；

- 联盟链：只有利益相关的特定区块链服务客户才能使用，节点只有经过授权许可后方可接入网络，接入节点按照规则参与共识和读写数据，具有较弱的去中心化的特征，如Hyperledger Fabric；

- 私有链：仅由单个区块链服务客户使用，仅有授权的节点才能接入，并按照规则参与共识和读写数据。

![image-20200106210102820](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/image-20200106210102820.png)

###  区块链核心构成

##### 密码学算法

- 区块链中采用了现代密码学中的哈希算法、对称加密算法、非对称加密算法等来保证数据机密性、完整性、抗抵赖性等安全特性。

##### p2p网络通信

- 区块链网络通常采用P2P（Peer to Peer）协议，节点之间直接通过交换方式共享信息，又被称为对等计算。P2P网络中的每个节点地位平等，不需要中心服务器节点来分配任务，每个节点可同时作为服务提供者与服务请求者。这种分布式架构避免了集中式架构中心节点的性能瓶颈，可以有效利用网络节点的性能与网络带宽，从而提升系统的整体效率。

##### 共识机制

- 共识机制是一个经典的分布式计算领域问题。在区块链系统中，共识机制是指实现不同信任主体节点之间建立信任、获取权益的数学算法 ，提供给分布式网络参识节点以用于确认交易动作引起的账本中的状态数据变化，并且能够达成最终一致性。即使出现节点故障或不可信节点等情况，区块链系统上已经发生的交易也能够按照正确的预期方式执行，而不会出现全网节点数据与账本状态不一致的情况。目前，常见的共识机制包括PoW、PoS、DPoS、PBFT等，它们在合规监管、性能效率、资源消耗以及容错性等方面都有各自不同的特点。

##### 智能合约

- 在区块链技术领域，智能合约是指基于预定事件触发、不可篡改、自动执行的计算机程序 。

### hyperledger介绍

超级账本，即Hyperledger项目是区块链技术中第一个面向企业应用场景的开源分布式账本平台。2015年12月由Linux基金会主导并牵头，IBM、Intel、Cisco等制造和科技行业的巨头共同宣布了Hyperledger联合项目成立。Hyperledger将区块链技术引入联盟链的应用场景中，为未来基于区块链技术打造高效率的商业网络打下基础，为透明、公开、去中心化的企业级分布式账本技术提供开源参考实现，目前已加入的成员超过260家，国外的如IBM、Intel、Cisco、Oracle、RedHat、Samsung、Fujitsu、Airbus等；国内的如百度、小米、腾讯、联想、华为、浪潮、京东、迅雷等。

###  hyperledger fabric介绍(v1.4.x)

Hyperledger Fabric是一个区块链的实现，由Digital Asset和IBM提供，是Linux基金会托管的Hyperledger项目之一。其目标是面向企业级应用场景的许可区块链（Permissioned Chain），**用于解决多个弱信任企业主体间的信任问题，以降低企业间复杂繁琐业务流程带来的信任成本，实现在可控主体范围内共享敏感数据**，从而有效提升企业主体间大规模协作活动的效率。

Hyperledger Fabric具有如下鲜明的技术特点：

- 支持可插拔的架构；
- 基于PKI体系与X.509标准身份证书的安全管理体系；
- 支持多通道、隐私数据集合等多粒度的数据隐私保护特性；
- Peer、Orderer等节点可扩展性良好；
- 支持多种链码（智能合约）开发语言（Node.js、Go、Java等）；
- 基于Docker容器技术提供链码运行时环境等。（后续可能提供不依赖docker运行环境）



##### 单机学习环境firstnetwork搭建运行（centos7  + fabric v1.4.0 ）

1. 基础软件环境安装

   - 安装cURL（最新版本）

   - 安装Docker和Docker Compose（最新版本）

   - Go语言环境（最新版本），设置好GOPATH环境变量

   - git安装 yum -y install git

   - 配置镜像加速 http://f1361db2.m.daocloud.io

     vim  /etc/docker/daemon.json添加配置

     ```json
     {
         "registry-mirrors":["你个人的加速器地址"]
     }
     ```

     ```shell
     # 重启daemon
     systemctl daemon-reload
     # 重启docker服务
     systemctl restart docker
     ```

2. 单机学习环境fabric安装

   - 将fabric-samples下载到`$GOPATH/src/github.com/hyperledger`目录中

     ```shell
     mkdir -p $GOPATH/src/github.com/hyperledger
     cd $GOPATH/src/github.com/hyperledger
     # 克隆fabric-samples项目并切换到v1.4tag
     git clone https://github.com/hyperledger/fabric-samples.git
     cd fabric-samples
     git checkout -b sample v1.4.0
     ```

   - 安装Fabric Binaries和Fabric相关的Docker镜像

     ```shell
     cd $GOPATH/src/github.com/hyperledger/fabric-samples/scripts
     # 安装Fabric、Fabric-ca以及第三方Docker镜像
     # ./bootstrap.sh <fabric> <fabric-ca> <thirdparty>
     ./bootstrap.sh 1.4.0 1.4.0 0.4.14
     ```

   - 运行byfn

     ```shell
     cd $GOPATH/src/github.com/hyperledger/fabric-samples/first-network
     # 编译通过Golang开发的chaincode并启动相关的容器
     ./byfn.sh up
     ```

     ```shell
     # ./byfn.sh 运行参数
     Usage: 
       byfn.sh <mode> [-c <channel name>] [-t <timeout>] [-d <delay>] [-f <docker-compose-file>] [-s <dbtype>] [-l <language>] [-o <consensus-type>] [-i <imagetag>] [-v]
         <mode> - one of 'up', 'down', 'restart', 'generate' or 'upgrade'
           - 'up' - bring up the network with docker-compose up
           - 'down' - clear the network with docker-compose down
           - 'restart' - restart the network
           - 'generate' - generate required certificates and genesis block
           - 'upgrade'  - upgrade the network from version 1.3.x to 1.4.0
         -c <channel name> - channel name to use (defaults to "mychannel")
         -t <timeout> - CLI timeout duration in seconds (defaults to 10)
         -d <delay> - delay duration in seconds (defaults to 3)
         -f <docker-compose-file> - specify which docker-compose file use (defaults to docker-compose-cli.yaml)
         -s <dbtype> - the database backend to use: goleveldb (default) or couchdb
         -l <language> - the chaincode language: golang (default) or node
         -o <consensus-type> - the consensus-type of the ordering service: solo (default) or kafka
         -i <imagetag> - the tag to be used to launch the network (defaults to "latest")
         -v - verbose mode
       byfn.sh -h (print this message)
     
     Typically, one would first generate the required certificates and 
     genesis block, then bring up the network. e.g.:
     	byfn.sh generate -c mychannel
     	byfn.sh up -c mychannel -s couchdb
         byfn.sh up -c mychannel -s couchdb -i 1.4.0
     	byfn.sh up -l node
     	byfn.sh down -c mychannel
         byfn.sh upgrade -c mychannel
     
     Taking all defaults:
     	byfn.sh generate
     	byfn.sh up
     	byfn.sh down
     ```

   - 进入cli容器

     ```shell
     docker exec -it cli bash
     ```

##### bootstrap.sh与byfn.sh脚本讲解 （待完善）

bootstrap.sh脚本执行后将下载并提取设置网络所需的所有特定于平台的二进制文件，并保存在本地仓库中，然后将Docker Hub中的Hyperledger Fabric Docker镜像下载到本地Docker注册表中，并将其标记为“最新”

byfn.sh了解安装过程，通道创建、熟悉交易流程

·configtxgen：生成初始区块及通道交易配置文件的工具。

·cryptogen：生成组织结构及相应的身份文件的工具。

脚本执行后将下载并提取设置网络所需的所有特定于平台的二进制文件，并保存在本地仓库中，然后将Docker Hub中的Hyperledger Fabric Docker镜像下载到本地Docker注册表中，并将其标记为“最新”。

### hyperledger explorer安装与演示

##### 安装步骤

- git拉取代码

  ```shell
  cd $GOPATH/src/github.com/hyperledger
  #确保当前版本支持fabric 1.4.x
  git clone https://github.com/hyperledger/blockchain-explorer.git
  ```

- 配置文件介绍

  blockchain-explorer/docker-compose.yaml

  ```yaml
  # SPDX-License-Identifier: Apache-2.0
  version: '2.1'
  
  volumes:
    pgdata:
    walletstore:
    grafana-storage:
    prometheus-storage:
  
  networks:
    mynetwork.com:
      external:
        name: net_byfn    # fabric网络名称，docker newwork ls 可查看
  
  services:
  
    explorerdb.mynetwork.com:
      image: hyperledger/explorer-db:latest
      container_name: explorerdb.mynetwork.com
      hostname: explorerdb.mynetwork.com
      environment:
        - DATABASE_DATABASE=fabricexplorer
        - DATABASE_USERNAME=hppoc
        - DATABASE_PASSWORD=password
      volumes:
        - ./app/persistence/fabric/postgreSQL/db/createdb.sh:/docker-entrypoint-initdb.d/createdb.sh
        - pgdata:/var/lib/postgresql/data
      networks:
        - mynetwork.com
  
    explorer.mynetwork.com:
      image: hyperledger/explorer:latest
      container_name: explorer.mynetwork.com
      hostname: explorer.mynetwork.com
      environment:
        - DATABASE_HOST=explorerdb.mynetwork.com
        - DATABASE_USERNAME=hppoc
        - DATABASE_PASSWD=password
        - LOG_LEVEL_APP=debug
        - LOG_LEVEL_DB=debug
        - LOG_LEVEL_CONSOLE=info
        - LOG_CONSOLE_STDOUT=true
        - DISCOVERY_AS_LOCALHOST=false
      volumes:
        - ./examples/net1/config.json:/opt/explorer/app/platform/fabric/config.json
        - ./examples/net1/connection-profile:/opt/explorer/app/platform/fabric/connection-profile
        - ./examples/net1/crypto:/tmp/crypto
        - walletstore:/opt/wallet
      command: sh -c "sleep 16&& node /opt/explorer/main.js && tail -f /dev/null"
      ports:
        - 8090:8080
      networks:
        - mynetwork.com
  
    proms:
      container_name: proms
      image: prom/prometheus:latest
      volumes:
        - ./app/platform/fabric/artifacts/operations/balance-transfer/prometheus.yml:/etc/prometheus/prometheus.yml
        - prometheus-storage:/prometheus
      ports:
        - '9090:9090'
      networks:
        - mynetwork.com
  
    grafana:
      container_name: grafana
      image: grafana/grafana:latest
      volumes:
        - ./app/platform/fabric/artifacts/operations/balance-transfer/balance-transfer-grafana-dashboard.json:/var/lib/grafana/dashboards/mydashboard.json
        - ./app/platform/fabric/artifacts/operations/grafana_conf/provisioning:/etc/grafana/provisioning
        - grafana-storage:/var/lib/grafana
      ports:
        - '3000:3000'
      networks:
        - mynetwork.com
  ```

  blockchain-explorer/examples/net1/config.json

  ```json
  {
  	"network-configs": {
  		"first-network": {
  			"name": "firstnetwork",
  			"profile": "./connection-profile/first-network.json"
  		}
  	},
  	"license": "Apache-2.0"
  }
  ```

  blockchain-explorer/examples/net1/connection-profile/first-network.json

  ```json
  {
  	"name": "first-network",
  	"version": "1.0.0",
  	"client": {
  		"tlsEnable": true,
  		"adminUser": "admin",
  		"adminPassword": "adminpw",
  		"enableAuthentication": false,
  		"organization": "Org1MSP",
  		"connection": {
  			"timeout": {
  				"peer": {
  					"endorser": "300"
  				},
  				"orderer": "300"
  			}
  		}
  	},
  	"channels": {
  		"mychannel": {
  			"peers": {
  				"peer0.org1.example.com": {}
  			},
  			"connection": {
  				"timeout": {
  					"peer": {
  						"endorser": "6000",
  						"eventHub": "6000",
  						"eventReg": "6000"
  					}
  				}
  			}
  		}
  	},
  	"organizations": {
  		"Org1MSP": {
  			"mspid": "Org1MSP",
  			"fullpath": true,
  			"adminPrivateKey": {
  				"path": "/tmp/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/e51315e18c53df1d202d16f5db4a50ca9576ef6d6dd84d576546f067c913368f_sk"
  			},
  			"signedCert": {
  				"path": "/tmp/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem"
  			}
  		}
  	},
  	"peers": {
  		"peer0.org1.example.com": {
  			"tlsCACerts": {
  				"path": "/tmp/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt"
  			},
  			"url": "grpcs://peer0.org1.example.com:7051",
  			"eventUrl": "grpcs://peer0.org1.example.com:7053",
  			"grpcOptions": {
  				"ssl-target-name-override": "peer0.org1.example.com"
  			}
  		}
  	}
  }
  ```

  blockchain-explorer/examples/net1/crypto/

  ```shell
  cd $GOPATH/src/github.com/hyperledger/blockchain-explorer
  rm -rf ./examples/net1/crypto/*
  # 拷贝证书文件到crypto目录
  mkdir -p ./examples/net1/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/
  cp $GOPATH/src/github.com/hyperledger/fabric-samples/first-network/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/*_sk   ./examples/net1/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/
  mkdir -p ./examples/net1/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/
  cp $GOPATH/src/github.com/hyperledger/fabric-samples/first-network/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem  ./examples/net1/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/
  mkdir -p ./examples/net1/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/
  cp $GOPATH/src/github.com/hyperledger/fabric-samples/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt     ./examples/net1/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/
  # 修改first-network.json中adminPrivateKey.path
  # 替换*_sk为新拷贝的文件名
  ```

- 运行docker-compose启动（blockchain-explorer目录下执行）

  ```shell
  docker-compose up
  ```

- 访问登录页面

  ```shell
  http://xxx.xxx.xxx.xxx:8090/
  用户名/密码：admin/adminpwd
  ```

  ![image-20200107122552222](/img/in-post/image-20200107122552222.png)

- 执行一笔交易，在explorer查看数据变化

  ```shell
  # docker exec -it cli bash进入cli容器
  export CHANNEL_NAME=mychannel
  # 查询数据
  peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
  # a转10给b
  peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}' --waitForEvent
  peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
  # a转10给b,失败情况
  peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}' --waitForEvent
  peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
  # 在explorer查看数据
  ```

- 卸载清除安装数据

  ```shell
  docker-compose down -v
  ```

### hyperledger fabric开发流程介绍 （待完善）

##### 链码开发

##### 链码部署

### 案例学习

https://github.com/kevin-hf/education基于区块链技术的实现的学历信息征信系统

### 学习参考

https://hyperledger-fabric.readthedocs.io/en/release-1.4/key_concepts.html 官方文档（推荐）

https://hyperledger-fabric.readthedocs.io/en/release-1.4/build_network.html

[https://www.taohui.pub/2018/05/26/%e5%8c%ba%e5%9d%97%e9%93%be%e5%bc%80%e6%ba%90%e5%ae%9e%e7%8e%b0hyperledger-fabric%e6%9e%b6%e6%9e%84%e8%af%a6%e8%a7%a3/](https://www.taohui.pub/2018/05/26/区块链开源实现hyperledger-fabric架构详解/)

https://www.bilibili.com/video/av37065233 北京大学肖臻老师《区块链技术与应用》公开课

https://space.bilibili.com/102734951/channel/detail?cid=69148 IBM微讲堂 超级账本Fabric v1.4系列课程

https://t-tlearning.com/detail/v_5ddb5769bd505_g883KJ3x/3 腾讯产品研发总监：探索区块链技术架构

https://weread.qq.com/web/reader/4c132830717cc12a4c13643kc81322c012c81e728d9d180

https://weread.qq.com/web/reader/82132290718019958217a07kc81322c012c81e728d9d180

