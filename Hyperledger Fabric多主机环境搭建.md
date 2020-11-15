# Hyperledger Fabric多主机环境搭建

> 本文所有操作都在root用户下进行

## 地址规划和配置

|               域名/IP                | 端口 |
| :----------------------------------: | :--: |
|  orderer0.example.com/10.245.150.80  | 7050 |
|  orderer1.example.com/10.245.150.81  | 7050 |
|  orderer2.example.com/10.245.150.82  | 7050 |
| peer0.org1.example.com/10.245.150.83 | 7051 |
| peer1.org1.example.com/10.245.150.84 | 7051 |
| peer2.org1.example.com/10.245.150.85 | 7051 |
| peer0.org2.example.com/10.245.150.86 | 7051 |
| peer1.org2.example.com/10.245.150.87 | 7051 |
| peer2.org2.example.com/10.245.150.88 | 7051 |
|  ca.org1.example.com/10.245.150.89   | 7054 |
|  ca.org2.example.com/10.245.150.90   | 7054 |
| ca.orderer.example.com/10.245.150.91 | 7054 |
| cello-master(manager)/10.245.150.92  | 9000 |

据此修改各台虚拟机的网络配置

1. `vi /etc/sysconfig/network-scripts/ifcfg-ens192`

   ```shell
   # 添加下面几项
   DNS1=8.8.8.8     # DNS配置
   BOOTPROTO="static"   # 使用静态IP
   IPADDR=10.245.150.80  # 主机的IP地址,这五个配置只有这一项不同
   NETMASK=255.255.255.0   # 掩码
   GATEWAY=10.245.150.1    # 网关地址
   ```

2. `vi /etc/resolv.conf`

   ```shell
   nameserver 8.8.8.8
   ```

3. `vi /etc/sysconfig/network`

   ```shell
   HOSTNAME=orderer0.example.com   # 主机名配置
   NETWORKING=yes
   ```

4. 重启网络服务

   ```shell
   systemctl restart network
   ```

5. 后面操作涉及主机之间的文件复制，所以为每台主机配置域名/IP映射

   ```shell
   vim /etc/hosts
   ```

## 安装docker

所有的虚拟机都需要执行

```shell
# 删除之前安装的docker
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
rm -rf /opt/docker-ce*

# 新的docker安装到/opt目录下
cd /opt
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-17.06.2.ce-1.el7.centos.x86_64.rpm
yum -y install docker-ce-17.06.2.ce-1.el7.centos.x86_64.rpm
```

修改docker的镜像源

```shell
tee /etc/docker/daemon.json  << eof
{
		"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","http://hub-mirror.c.163.com"]
}
eof
```

启动docker并开启开机自启

```shell
systemctl enable docker
systemctl start docker
```

## 安装docker-compose

```shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

# 验证是否安装成功
docker-compose --version
```

## 搭建集群

在manager节点上使用`docker swarm`命令建立集群

```shell
# 复制该命令执行的结果
docker swarm init
```

在其他所有主机上执行以下命令将节点以manager角色加入集群

```shell
# 这一命令是通过在manager节点上执行 docker swarm join-token manager 查出来的
docker swarm join --token SWMTKN-1-3sn1cy4cura0l11at59ezjsxggjrctdoyanatbs9bk2gf923jq-98xstquvlt8msc1b8wi7o8h6k 10.245.150.92:2377
```

在manager节点上执行`docker node ls`查看集群中所有的节点以确定所有的节点都成功加入集群

![image-20201110154054631](https://gitee.com//tiansir-wg/blogimg/raw/master/imgs/20201110154100.png)

## 创建overlay网络

在manager节点上执行以下命令创建一个二层网络将各容器连接起来

```shell
docker network create --attachable --driver overlay fabric-network
```

可以在其他节点上执行`docker network ls`查看该网络

## 启动CA并生成加密材料

以org1为例:

1. 启动`ca.org1.example.com`容器

   ```shell
   docker-compose -f dockerfiles/ca.org1.example.com.yaml up -d
   ```

2. 



```shell
# 1.启动ca.org1.example.com
docker-compose -f dockerfiles/ca.org1.example.com.yaml up -d
# 2.将fabric-network-multihost文件夹拷贝到peer0.org1
scp -r organizations root@peer0.org1.example.com:`pwd`
# 3.在peer0.org1上进行注册
chmod +x registerOrg1.sh
./registerOrg1.sh
# 4.将生成的证书材料拷贝到manager、peer1.org1、peer2.org1上
scp -r organizations root@cello-master:`pwd`
scp -r organizations root@peer1.org1.example.com:`pwd`
scp -r organizations root@peer2.org1.example.com:`pwd`
```

org2

```shell
# 1.启动ca.org2.example.com
docker-compose -f dockerfiles/ca.org2.example.com.yaml up -d
# 2.将organizations文件夹拷贝到peer0.org2
scp -r organizations/ root@peer0.org2.example.com:`pwd`
# 3.在peer0.org2上进行注册
chmod +x registerOrg2.sh
./registerOrg2.sh
# 4.将生成的证书材料拷贝到manager、peer1.org2、peer2.org2上
scp -r organizations/ root@cello-master:`pwd`
scp -r organizations/ root@peer1.org2.example.com:`pwd`
scp -r organizations/ root@peer2.org2.example.com:`pwd`
```

orderer

```shell
# 1.启动ca.orderer.example.com
docker-compose -f dockerfiles/ca.orderer.example.com.yaml up -d
# 2.将organizations文件夹拷贝到orderer1.example.com
scp -r organizations/ root@orderer0.example.com:`pwd`
# 3.在peer0.org2上进行注册
chmod +x registerOrderer.sh
./registerOrderer.sh
# 4.将生成的证书材料拷贝到manager、orderer1.example.com、orderer2.example.com上
scp -r organizations/ root@cello-master:`pwd`
scp -r organizations/ root@orderer1.example.com:`pwd`
scp -r organizations/ root@orderer2.example.com:`pwd`
```

## 创世系统创世块

在manager上

```shell
export FABRIC_CFG_PATH=$PWD
configtxgen -profile TwoOrgsOrdererGenesis -channelID system-channel -outputBlock ./system-genesis-block/genesis.block
```

## 创建通道事务

manager上

```shell
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/mychannel.tx -channelID mychannel
```

## 创建锚节点事务

```shell
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP

configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP
```

## 启动容器

每次重新执行时先执行`docker system prune --volumes`

从manager上复制channel-artifacts和system-genesis-block文件夹到CA以外的所有其他主机

然后运行相应的脚本

```shell
docker-compose -f dockerfiles/orderer0.example.com.yaml up -d
docker-compose -f dockerfiles/orderer1.example.com.yaml up -d
docker-compose -f dockerfiles/orderer2.example.com.yaml up -d
docker-compose -f dockerfiles/peer0.org1.example.com.yaml up -d
docker-compose -f dockerfiles/peer1.org1.example.com.yaml up -d
docker-compose -f dockerfiles/peer2.org1.example.com.yaml up -d
docker-compose -f dockerfiles/peer0.org2.example.com.yaml up -d
docker-compose -f dockerfiles/peer1.org2.example.com.yaml up -d
docker-compose -f dockerfiles/peer2.org2.example.com.yaml up -d
```

## 创建通道

manager上

```shell
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

# 可以指定 --outputBlock,默认是./$CHANNEL_ID.block
peer channel create -o orderer0.example.com:7050 -c mychannel -f ./channel-artifacts/mychannel.tx  --tls --cafile $ORDERER_CA --outputBlock ./channel-artifacts/mychannel.block
```

## 加入节点

manager上

```shell
# peer0.org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer channel join -b ./channel-artifacts/mychannel.block

# peer1.org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer1.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer channel join -b ./channel-artifacts/mychannel.block

# peer2.org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer2.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer channel join -b ./channel-artifacts/mychannel.block


# peer0.org2
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer channel join -b ./channel-artifacts/mychannel.block


# peer1.org2
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer1.org2.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer channel join -b ./channel-artifacts/mychannel.block

# peer2.org2
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer2.org2.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer channel join -b ./channel-artifacts/mychannel.block
```

## 更新锚节点

manager上

```shell
# peer0.org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer channel update -o orderer0.example.com:7050 -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile $ORDERER_CA

# peer0.org2
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer channel update -o orderer0.example.com:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile $ORDERER_CA
```

> 以上执行完毕后将channel-artifacts/文件夹和mychannel.block复制到其他的虚拟机相应的目录下

## 打包链码

需要先在manager上安装go语言环境

```shell
cd /opt
wget https://studygolang.com/dl/golang/go1.15.4.linux-amd64.tar.gz
tar -zxvf go1.15.4.linux-amd64.tar.gz
cd go
echo "export GOROOT=$PWD" >> /etc/profile
echo "export PATH=$PATH:$PWD/bin" >> /etc/profile
. /etc/profile
```

在manager上，下载链码所需的依赖

```shell
cd /root/fabric-network-multihost
pushd ./chaincode/fabcar/go
GO111MODULE=on
go env -w GOPROXY="https://goproxy.io,direct"
go mod vendor
popd
```

打包链码

```shell
peer lifecycle chaincode package fabcar.tar.gz --path ./chaincode/fabcar/go --lang golang --label fabcar_1
```

## 安装链码

manager上

```shell
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz

# peer1.org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer1.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz

# peer2.org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer2.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz

# peer0.org2
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz


# peer1.org2
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer1.org2.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz

# peer2.org2
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer2.org2.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz
```

安装完后在对应的虚拟机上执行`docker images`可看到有一个fabcar镜像

注意记下package_id，后面会用到，如`fabcar_1:762e0fe3dbeee0f7b08fb6200adeb4a3a20f649a00f168c0b3c2257e53b6e506`

## 链码审核

在manager上执行

```shell
# org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export ORDERER_CA=${PWD}/crypto-config/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD
export PACKAGE_ID=fabcar_1:762e0fe3dbeee0f7b08fb6200adeb4a3a20f649a00f168c0b3c2257e53b6e506

peer lifecycle chaincode approveformyorg -o orderer0.example.com:7050 --tls --cafile $ORDERER_CA --channelID mychannel --name fabcar --version 1 --package-id ${PACKAGE_ID} --sequence 1
```

执行以下命令查看审批状态

```shell
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name fabcar --version 1 --sequence 1

# 结果
Chaincode definition for chaincode 'fabcar', version '1', sequence '1' on channel 'mychannel' approval status by org:
Org1MSP: true
Org2MSP: false
```

继续在manager上执行

```shell
# org2
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD
export PACKAGE_ID=fabcar_1:762e0fe3dbeee0f7b08fb6200adeb4a3a20f649a00f168c0b3c2257e53b6e506

peer lifecycle chaincode approveformyorg -o orderer0.example.com:7050 --tls --cafile $ORDERER_CA --channelID mychannel --name fabcar --version 1 --package-id ${PACKAGE_ID} --sequence 1
```

查看审批状态

```shell
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name fabcar --version 1 --sequence 1

# 结果
Chaincode definition for chaincode 'fabcar', version '1', sequence '1' on channel 'mychannel' approval status by org:
Org1MSP: true
Org2MSP: true
```

## 提交链码

在manager上

```shell
export PEER_CONN_PARMS="--peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt"
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode commit -o orderer0.example.com:7050 --tls --cafile $ORDERER_CA --channelID mychannel --name fabcar $PEER_CONN_PARMS --version 1 --sequence 1 
```

查看提交状态

```shell
peer lifecycle chaincode querycommitted --channelID mychannel --name fabcar

#结果
Committed chaincode definition for chaincode 'fabcar' on channel 'mychannel':
Version: 1, Sequence: 1, Endorsement Plugin: escc, Validation Plugin: vscc, Approvals: [Org1MSP: true, Org2MSP: true]
```

## 链码初始化

manager上

```shell
export PEER_CONN_PARMS="--peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt"
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer chaincode invoke -o orderer2.example.com:7050 --tls true --cafile $ORDERER_CA -C mychannel -n fabcar $PEER_CONN_PARMS -c '{"Args":["initLedger"]}'
```

## 链码查询测试

```shell
# peer0.org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer chaincode query -C mychannel -n fabcar -c '{"Args":["queryAllCars"]}'
```





