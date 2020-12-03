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
   
   cat >> /etc/hosts << EOF
   10.245.150.80 orderer0.example.com
   10.245.150.81 orderer1.example.com
   10.245.150.82 orderer2.example.com
   10.245.150.83 peer0.org1.example.com
   10.245.150.84 peer1.org1.example.com
   10.245.150.85 peer2.org1.example.com
   10.245.150.86 peer0.org2.example.com
   10.245.150.87 peer1.org2.example.com
   10.245.150.88 peer2.org2.example.com
   10.245.150.89 ca.org1.example.com
   10.245.150.90 ca.org2.example.com
   10.245.150.91 ca.orderer.example.com
   10.245.150.92 zk.server
   EOF
   ```

## 安装docker

所有的虚拟机都需要执行

```shell
cd /opt
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
rm -rf /opt/docker-ce*
yum install -y wget
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
docker swarm init
# 复制该命令执行的结果
docker swarm join-token manager
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

## 克隆源文件夹

源文件地址为 `git@github.com:Tiansir-wg/fabric-network-multihost.git`

如果直接运行，那么只需要将源文件克隆到每个主机的`/root`目录下，然后启动对应的容器即可，其它操作都可以不执行。如果不直接执行，先将源文件克隆到manager 主机的`/root`目录下，删除以下目录:

* `channel-artifacts`文件夹
* `fabcar.tar.gz`文件
* `system-genesis-block`文件夹
* `organizations/peerOrganization`文件夹
* `organizations/ordererOrganizations`文件夹
* `organizations/fabric-ca/org1`、`organizations/fabric-ca/org2`、`organizations/fabric-ca/ordererOrg/`三个目录下除了`fabric-ca-server-config.yaml`文件外的所有文件和目录



> 下面命令执行时如果没有指定目录则默认是在fabric-network-multihost目录下

## 启动CA并生成加密材料

以org1为例:

1. `ca.org1.example.com  `节点上启动`ca.org1.example.com`容器

   ```shell
   docker-compose -f dockerfiles/ca.org1.example.com.yaml up -d
   ```

2. 将`organizations`文件夹复制到`peer0.org1.example.com`下的`/root/fabric-network-multihost`目录下

   ```shell
   scp -r organizations root@peer0.org1.example.com:`pwd`
   ```

3. 在`peer0.org1.example.com `上进行注册 

   ```sshell
   chmod +x registerOrg1.sh
   ./registerOrg1.sh
   ```

4. 将`organizations`文件夹复制到`peer1.org1.example.com`、`peer2.org1.example.com`、`manager`节点的`/root/fabric-network-multihost`目录下

   ```shell
   # 4.将生成的证书材料拷贝到manager、peer1.org1、peer2.org1上
   scp -r organizations root@cello-master:`pwd`
   scp -r organizations root@peer1.org1.example.com:`pwd`
   scp -r organizations root@peer2.org1.example.com:`pwd`
   
   scp -r fabric-network-multihost root@peer1.org1.example.com:`pwd`
   scp -r fabric-network-multihost root@peer2.org1.example.com:`pwd`
   ```

下面是org1相关的所有配置

```shell
# 1.启动ca.org1.example.com
docker-compose -f dockerfiles/ca.org1.example.com.yaml up -d
# 2.将fabric-network-multihost文件夹拷贝到peer0.org1
scp -r organizations root@peer0.org1.example.com:`pwd`
# 3.在peer0.org1上进行注册
chmod +x registerOrg1.sh
./registerOrg1.sh
# 4.将生成的证书材料拷贝到manager、peer1.org1、peer2.org1上
# scp -r organizations root@cello-master:`pwd`
scp -r organizations root@peer1.org1.example.com:`pwd`
scp -r organizations root@peer2.org1.example.com:`pwd`
```

下面是org2相关的所有配置

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

scp -r fabric-network-multihost/ root@peer1.org2.example.com:`pwd`
scp -r fabric-network-multihost/ root@peer2.org2.example.com:`pwd`
```

下面是 orderer相关的所有配置 

```shell
# 1.启动ca.orderer.example.com
docker-compose -f dockerfiles/ca.orderer.example.com.yaml up -d
# 2.将organizations文件夹拷贝到orderer1.example.com
scp -r organizations/ root@orderer0.example.com:`pwd`
# 3.在orderer0上进行注册
chmod +x registerOrderer.sh
./registerOrderer.sh
# 4.将生成的证书材料拷贝到manager、orderer1.example.com、orderer2.example.com上
scp -r organizations/ root@cello-master:`pwd`
scp -r fabric-network-multihost/ root@orderer1.example.com:`pwd`
scp -r fabric-network-multihost/ root@orderer2.example.com:`pwd`
```

> 以上执行完毕后将manager节点的`organizations`文件夹拷贝到其他所有节点对应位置上

## 创建system-channel创世块

在manager节点上执行

```shell
export FABRIC_CFG_PATH=$PWD
configtxgen -profile TwoOrgsOrdererGenesis -channelID system-channel -outputBlock ./system-genesis-block/genesis.block
```

## 创建通道事务

在manager节点上执行

```shell
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/mychannel.tx -channelID mychannel
```

## 创建锚节点事务

在manager节点上执行

```shell
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP

configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP
```

## 启动容器

> 每次重新执行时先执行`docker system prune --volumes`

从manager上复制`channel-artifacts`和`system-genesis-block`文件夹到CA以外的所有其他主机

然后在各主机运行相应的脚本

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

在manager节点上执行

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

在manager节点上执行

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

在manager节点上执行

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
# 安装到/opt目录下
cd /opt
wget https://studygolang.com/dl/golang/go1.15.4.linux-amd64.tar.gz
tar -zxvf go1.15.4.linux-amd64.tar.gz

# 配置go语言相关的环境变量
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

然后打包链码

```shell
peer lifecycle chaincode package fabcar.tar.gz --path ./chaincode/fabcar/go --lang golang --label fabcar_1
```

最后将生成`fabcar.tar.gz`文件拷贝到所有的peer节点上

## 安装链码

在`peer0.org1.example.com`节点 上

```shell
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz
```

在`peer1.org1.example.com`节点上

```shell
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer1.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz
```

在`peer2.org1.example.com`节点上

```shell
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer2.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz
```

在`peer0.org2.example.com`节点上

```shell
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz
```

在`peer1.org2.example.com`节点上

```shell
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer1.org2.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer lifecycle chaincode install fabcar.tar.gz
```

在`peer2.org2.example.com`节点上

```shell
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
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD
export PACKAGE_ID=fabcar_1:762e0fe3dbeee0f7b08fb6200adeb4a3a20f649a00f168c0b3c2257e53b6e506

peer lifecycle chaincode approveformyorg -o orderer0.example.com:7050 --tls --cafile $ORDERER_CA --channelID mychannel --name fabcar --version 1 --package-id ${PACKAGE_ID} --sequence 1
```

可以执行以下命令查看审批状态

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

在manager节点上执行

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

manager节点上执行

```shell
export PEER_CONN_PARMS="--peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt"
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export FABRIC_CFG_PATH=$PWD

peer chaincode invoke -o orderer2.example.com:7050 --tls true --cafile $ORDERER_CA -C mychannel -n fabcar $PEER_CONN_PARMS -c '{"Args":["initLedger"]}'
```

## 链码查询测试

在`peer0.org1.example.com`节点上执行

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

至此搭建成功，后面通过fabric-gateway-java提供的api与fabric区块链网络进行交互

## 生成ccp文件 

在manager节点上运行`ccp-generate.sh`脚本生成`connection-org1.yaml`文件，将该文件拷贝到Java项目目录下，后面就可以使用该文件连接区块链网络

