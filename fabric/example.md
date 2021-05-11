# Fabric v1.4.x升级至v2.2.0  

以`fabric-samples v1.4.8`为例，将v1.4.8升级至v2.2.0。注意，所有节点以滚动的方式进行升级，这样可以保证即使单个节点数据备份过程出错也不会影响整个网络的运行。先升级orderer节点，再升级peer节点。

## 前期准备  

1. v1.4.xFabric网络
2. v2.2.0 docker镜像：
   1. hyperledger/fabric-ca
   2. hyperledger/fabric-tools
   3. hyperledger/fabric-peer
   4. hyperledger/fabric-orderer
   5. hyperledger/fabric-ccenv
   6. hyperledger/fabric-baseos  

## 升级orderer节点  

升级每个orderer时，都需要执行以下操作。

### 1、配置orderer容器中的环境变量  

方便起见，orderer容器运行时需要的环境变量可以记录在文件中，具体内容如下：  

```
FABRIC_LOGGING_SPEC=INFO
FABRIC_CFG_PATH=/etc/hyperledger/fabric
ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
ORDERER_GENERAL_GENESISMETHOD=file
ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
ORDERER_GENERAL_LOCALMSPID=OrdererMSP
ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
# enabled TLS
ORDERER_GENERAL_TLS_ENABLED=true
ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
ORDERER_KAFKA_TOPIC_REPLICATIONFACTOR=1
ORDERER_KAFKA_VERBOSE=true
ORDERER_GENERAL_CLUSTER_CLIENTCERTIFICATE=/var/hyperledger/orderer/tls/server.crt
ORDERER_GENERAL_CLUSTER_CLIENTPRIVATEKEY=/var/hyperledger/orderer/tls/server.key
ORDERER_GENERAL_CLUSTER_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
```  

### 2、设置升级时的环境变量  

在升级排序节点前导入以下环境变量：  

- <font color="red">ORDERER_CONTAINER</font>：排序节点的容器名称。注意，每个节点升级时你都样设置一遍。
- <font color="red">LEDGERS_BACKUP</font>：存放备份数据的路径。就如下面的示例中，每个节点都有它自己的子目录来存放它的账本。目录如果不存在的话，你需要手动创建。
- <font color="red">IMAGE_TAG</font>：你期望升级到的Fabric版本，例如v2.0。  

示例如下：  

```bash
export ORDERER_CONTAINER=orderer.example.com
export LEDGERS_BACKUP=/root/upgrade/backup
export IMAGE_TAG=2.2.0
```  

### 3、升级  

```bash
export FABRIC_SAMPLES=/root/go/src/github.com/hyperledger/fabric-samples/
# 停止容器
docker stop $ORDERER_CONTAINER
# 备份账本和MSPs
docker cp $ORDERER_CONTAINER:/var/hyperledger/production/orderer/ $LEDGERS_BACKUP/$ORDERER_CONTAINER
# 删除容器
docker rm -f $ORDERER_CONTAINER
# 升级容器
docker run -d -v $LEDGERS_BACKUP/$ORDERER_CONTAINER/:/var/hyperledger/production/orderer/ -v $FABRIC_SAMPLES/first-network/channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block -v $FABRIC_SAMPLES/first-network/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp:/var/hyperledger/orderer/msp -v $FABRIC_SAMPLES/first-network/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/:/var/hyperledger/orderer/tls  --env-file ./env_order.list  --net net_byfn --name $ORDERER_CONTAINER  -p 7050:7050 hyperledger/fabric-orderer:$IMAGE_TAG orderer  
```  

## 升级peer节点  

以peer0org1节点为例，以下操作每个peer节点都需要执行。  

### 1、配置peer运行时的环境变量  

方便起见，peer容器运行时需要的环境变量可以记录在文件中，具体内容如下：   

```
CORE_PEER_LOCALMSPID=Org1MSP
CORE_PEER_GOSSIP_USELEADERELECTION=true
CORE_PEER_ID=peer0.org1.example.com
CORE_PEER_ADDRESS=peer0.org1.example.com:7051
FABRIC_CFG_PATH=/etc/hyperledger/fabric
CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=net_byfn
CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051
CORE_PEER_PROFILE_ENABLED=true
CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
CORE_PEER_LISTENADDRESS=0.0.0.0:7051
CORE_PEER_GOSSIP_ORGLEADER=false
CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:7052
CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org1.example.com:8051
CORE_PEER_TLS_ENABLED=true
FABRIC_LOGGING_SPEC=INFO
CORE_PEER_CHAINCODEADDRESS=peer0.org1.example.com:7052
```

### 2、设置升级时的环境变量  

在升级peer节点前导入以下环境变量：  

- <font color="red">PEER_CONTAINER</font>：peer节点的容器名称。注意，每个节点升级时你都样设置一遍。
- <font color="red">LEDGERS_BACKUP</font>：存放备份数据的路径。就如下面的示例中，每个节点都有它自己的子目录来存放它的账本。目录如果不存在的话，你需要手动创建。
- <font color="red">IMAGE_TAG</font>：你期望升级到的Fabric版本，例如v2.0。  

示例如下：  

```bash
export PEER_CONTAINER=peer0.org1.example.com
export LEDGERS_BACKUP=/root/upgrade/backup
export IMAGE_TAG=2.2.0
```  

### 3、升级  

```bash
# 停止容器
docker stop $PEER_CONTAINER
# 备份账本和MSPs
docker cp $PEER_CONTAINER:/var/hyperledger/production $LEDGERS_BACKUP/$PEER_CONTAINER
# 删除CC容器
CC_CONTAINERS=$(docker ps | grep dev-$PEER_CONTAINER | awk '{print $1}')
if [ -n "$CC_CONTAINERS" ] ; then docker rm -f $CC_CONTAINERS ; fi
# 删除CC镜像
CC_IMAGES=$(docker images | grep dev-$PEER | awk '{print $1}')
if [ -n "$CC_IMAGES" ] ; then docker rmi -f $CC_IMAGES ; fi
# 删除peer容器
docker rm -f $PEER_CONTAINER
# 升级peer节点数据库
docker run --rm -v $LEDGERS_BACKUP/$PEER_CONTAINER:/var/hyperledger/production/ -v /var/run/:/host/var/run/ -v $FABRIC_SAMPLES/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp:/etc/hyperledger/fabric/msp -v $FABRIC_SAMPLES/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls:/etc/hyperledger/fabric/tls -p 7051:7051 --env-file env_peer.list --net net_byfn --name $PEER_CONTAINER  hyperledger/fabric-peer:$IMAGE_TAG peer node upgrade-dbs
# 升级peer节点
docker run -d -v $LEDGERS_BACKUP/$PEER_CONTAINER:/var/hyperledger/production/ -v /var/run/:/host/var/run/ -v $FABRIC_SAMPLES/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp:/etc/hyperledger/fabric/msp -v $FABRIC_SAMPLES/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls:/etc/hyperledger/fabric/tls -p 7051:7051 --env-file env_peer.list --net net_byfn --name $PEER_CONTAINER  hyperledger/fabric-peer:$IMAGE_TAG peer node start
```