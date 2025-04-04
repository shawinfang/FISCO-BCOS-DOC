# 搭建Max版区块链网络

标签：``Max版区块链网络`` ``部署``

------------

为了能够支撑海量交易上链场景，v3.x推出了Max版本FISCO BCOS，Max版本FISCO BCOS旨在提供**海量存储服务、高性能可扩展的执行模块**、**高可用的故障恢复机制**。
Max版FISCO BCOS节点采用分布式存储TiKV，执行模块独立成服务，存储和执行均可横向扩展，且支持自动化主备恢复。

本章通过单机搭建Max版本单节点FISCO BCOS联盟链，帮助用户掌握Max版本FISCO BCOS区块链的部署流程。请参考[这里](../../quick_start/hardware_requirements.md)使用支持的**硬件和平台**进行操作。


```eval_rst
.. note::
   - Max版本FISCO BCOS使用 ``BcosBuilder/max`` 工具进行建链和扩容等相关操作，该工具的介绍请参考 `BcosBuilder <./max_builder.html>`_ 
   - FISCO BCOS 3.x基于tars进行微服务构建和管理，搭建Max版本FISCO BCOS之前，需先安装tars服务，本章介绍了docker版本tars服务的搭建流程，若需要了解更多tars部署、构建相关的信息，请参考 `这里 <https://doc.tarsyun.com/#/markdown/TarsCloud/TarsDocs/installation/README.md>`_
   - 本章基于Docker搭建tars服务，请确保拥有 ``root`` 权限
   - 搭建Max版本FISCO BCOS需先部署TiKV集群，TiKV集群的部署请参考 `这里 <https://tikv.org/docs/5.1/deploy/install/install/>`_
```

## 1. 安装依赖

部署工具`BcosBuilder`依赖`python3, curl, docker, docker-compose`，根据您使用的操作系统，使用以下命令安装依赖。

**安装Ubuntu依赖(版本不小于Ubuntu18.04)**

```shell
sudo apt-get update
sudo apt-get install -y curl docker.io docker-compose python3 wget
```

**安装CentOS依赖(版本不小于CentOS 7)**

```shell
sudo yum install -y curl docker docker-compose python3 python3-devel wget
```

**安装macOS依赖**

```shell
brew install curl docker docker-compose python3 wget
```

## 2. 下载区块链构建工具BcosBuilder

```eval_rst
.. note::
   - 部署工具 ``BcosBuilder`` 配置和使用请参考 `这里 <./max_builder.html>`_
   - 若从github下载部署工具 ``BcosBuilder`` 网速太慢，请尝试: curl -#LO https://osp-1257653870.cos.ap-guangzhou.myqcloud.com/FISCO-BCOS/FISCO-BCOS/releases/v3.0.1/BcosBuilder.tgz && tar -xvf BcosBuilder.tgz
```

```shell
# 创建操作目录
mkdir -p ~/fisco && cd ~/fisco

# 下载区块链构建工具BcosBuilder
curl -#LO https://github.com/FISCO-BCOS/FISCO-BCOS/releases/download/v3.0.1/BcosBuilder.tgz && tar -xvf BcosBuilder.tgz

# Note: 若网速太慢，可尝试如下命令下载部署脚本:
curl -#LO https://osp-1257653870.cos.ap-guangzhou.myqcloud.com/FISCO-BCOS/FISCO-BCOS/releases/v3.0.1/BcosBuilder.tgz && tar -xvf BcosBuilder.tgz

# 安装构建工具依赖包
cd BcosBuilder && pip3 install -r requirements.txt
```
## 3. 安装、启动并配置tars服务

**tars服务的安装、启动和配置请参考[这里](../pro/installation.html#id2).**


## 4. 部署TiKV集群

**下载和安装tiup**

```bash
$ curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
```

**启动tikv v5.4.0**

```bash
# 部署并启动tikv(这里设机器的物理ip为172.25.0.3)
$ nohup tiup playground v5.4.0 --mode tikv-slim --host=172.25.0.3 -T tikv_demo --without-monitor > ~/tikv.log 2>&1 &
# 获取tikv监听端口(tikv的默认监听端口是2379)
$ cat ~/tikv.log
tiup is checking updates for component playground ...timeout!
Starting component `playground`: /home/fisco/.tiup/components/playground/v1.9.4/tiup-playground v5.4.0 --mode tikv-slim --host=172.25.0.3 -T tikv_demo --without-monitor
Playground Bootstrapping...
Start pd instance:v5.4.0
Start tikv instance:v5.4.0
PD client endpoints: [172.25.0.3:2379]
```


## 5. 部署Max版本区块链系统

Max版本FISCO BCOS包括RPC服务、Gateway服务、区块链节点服务`BcosMaxNodeService`以及区块链执行服务`BcosExecutorService`:

- RPC服务：负责接收客户端请求，并将请求转发到节点进行处理， RPC服务可横向扩展，一个RPC服务可接入多个区块链节点服务;
- Gateway服务：负责跨机构区块链节点之间的网络通信，Gateway服务横向可扩展，一个Gateway服务可接入多个区块链节点服务;
- 区块链节点服务`BcosMaxNodeService`：负责提供区块链调度相关的服务，包括区块打包、共识、执行调度、提交调度等，节点服务通过接入到RPC服务和Gateway服务获取网络通信功能;
- 区块链执行服务`BcosExecutorService`：负责区块执行，可横向扩展、动态扩容。

本章以在单机上部署单节点区块链为例，介绍Max版本FISCO BCOS搭建部署流程。

```eval_rst
.. note::
   - 部署Max版本区块链系统前，请先参考 `这里 <../pro/installation.html#id2>`_ 搭建tars服务，并申请token
   - 如果没有申请token，请参考【3.2 配置tars服务】申请token
   - 如果忘记了访问tars服务的token，可通过tars网页管理平台的【admin】->【用户中心】->【token管理】获取token列表
   - 部署Max版本区块链之前，请先确保您的tars服务是启动的状态
   - 须将申请的token配置到配置文件 ``config.toml`` 的 ``tars.tars_token`` 配置选项才可进行后续所有的部署步骤
   - 部署Max版本区块链之前，请先确保参考部署了tikv，且须保证每个Max节点对应一个tikv服务，多个Max节点不可共用tikv服务
```

### 5.1 下载二进制

构建Max版本FISCO BCOS前，需要先下载二进制包，`BcosBuilder`的提供了基于linux的静态二进制包下载功能，可部署到`tarsnode`中，下载最新二进制的命令如下：

```eval_rst
.. note::
   - 可通过 ``python3 build_chain.py -h`` 查看部署脚本使用方法
   - 二进制默认下载到 ``binary`` 目录
   - 若下载二进制比较慢，请尝试: ``python3 build_chain.py download_binary -t cdn``
```

```shell
# 进入操作目录
cd ~/fisco/BcosBuilder/max

# 运行build_chain.py脚本下载二进制，二进制包默认下载到binary目录
python3 build_chain.py download_binary
```

### 5.2 部署RPC服务

类似于Pro版本FISCO BCOS，Max版本区块链系统也包括RPC服务，可通过建链脚本`BcosBuilder`部署和构建RPC服务，`BcosBuilder/max/conf`目录下提供了示例配置文件`config-deploy-example.toml`，可在机构`agencyA`的`172.25.0.3`机器上部署RPC服务，其占用的监听端口为`20200`。

```eval_rst
.. note::
   请确保默认端口20200没有被占用，若被占用，可考虑手动修改配置 ``config.toml`` 配置没有被占用的端口
```

```shell
# 进入操作目录
cd ~/fisco/BcosBuilder/max

# 从conf目录拷贝配置
cp conf/config-deploy-example.toml config.toml
```

此时拷贝的`config.toml`为整个`BcosBuilder/max`使用的配置文件，配置详情请参考链接：[配置介绍](./max_builder.md)。

```shell
# 将生成的token配置到config.toml的tars_token字段
# 这里生成的token是eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOiJhZG1pbiIsImlhdCI6MTYzODQzMTY1NSwiZXhwIjoxNjY3MjAyODU1fQ.430ni50xWPJXgJdckpOTktJB3kAMNwFdl8w_GIP_3Ls，实际使用时，请替换为实际申请的token
# linux系统(macOS系统跳过本步骤):
sed -i 's/tars_token = ""/tars_token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOiJhZG1pbiIsImlhdCI6MTYzODQzMTY1NSwiZXhwIjoxNjY3MjAyODU1fQ.430ni50xWPJXgJdckpOTktJB3kAMNwFdl8w_GIP_3Ls"/g' config.toml
# macos系统(linux系统跳过本步骤):
sed -i .bkp 's/tars_token = ""/tars_token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOiJhZG1pbiIsImlhdCI6MTYzODQzMTY1NSwiZXhwIjoxNjY3MjAyODU1fQ.430ni50xWPJXgJdckpOTktJB3kAMNwFdl8w_GIP_3Ls"/g' config.toml

# 配置RPC服务用于动态watch Max节点主备信息的etcd集群，这里复用了tikv pd的etcd集群，设tikv pd的访问地址为172.25.0.3:2379，若pd的访问端口为其他端口，则需要手动配置`config.toml`的`failover_cluster_url`配置选项，具体的配置选项如下:
[[agency]]
name = "agencyA"
failover_cluster_url = "172.25.0.3:2379"

#部署并启动RPC服务
python3 build_chain.py chain -o deploy -t rpc
```

执行上述命令后，当脚本输出`deploy service success, type: rpc`时，则说明RPC服务部署成功，详细日志输出如下：

```shell
=========================================================
----------- deploy service, type: rpc -----------
=========================================================
----------- generate service config -----------
* generate service config for 172.25.0.3 : agencyABcosRpcService
* generate config for the rpc service
* generate generated/rpc/chain0/172.25.0.3/agencyABcosRpcService/config.ini.tmp
* generate cert, output path: generated/rpc/chain0/172.25.0.3/agencyABcosRpcService
* generate sdk cert, output path: generated/rpc/chain0/172.25.0.3/agencyABcosRpcService
* generate config for the rpc service success
gen configuration for service agencyABcosRpcService success
----------- generate service config success -----------
=========================================================
deploy_service to 172.25.0.3, app: chain0, name: agencyABcosRpcService
deploy service agencyABcosRpcService
* add config for service agencyABcosRpcService, node: 172.25.0.3, config: ca.crt
* add config for service agencyABcosRpcService, node: 172.25.0.3, config: ssl.key
* add config for service agencyABcosRpcService, node: 172.25.0.3, config: ssl.crt
* add config for service agencyABcosRpcService, node: 172.25.0.3, config: config.ini
upload tar package generated/./agencyABcosRpcService.tgz success, config id: 14
----------- deploy service success, type: rpc -----------
=========================================================
```

部署过程中生成的RPC服务相关的配置位于`generated/rpc/${chainID}`目录，具体如下：

```shell
$ tree generated/rpc/chain0
generated/rpc/chain0
├── 172.25.0.3 
│   ├── agencyABcosRpcService # 机构A的RPC服务目录
│   │   ├── config.ini.tmp    # 机构A的RPC服务的配置文件
│   │   ├── sdk               # SDK证书目录，SDK客户端可从本目录拷贝证书连接RPC服务
│   │   │   ├── ca.crt
│   │   │   ├── cert.cnf
│   │   │   ├── sdk.crt
│   │   │   └── sdk.key
│   │   └── ssl               # RPC服务证书目录
│   │       ├── ca.crt
│   │       ├── cert.cnf
│   │       ├── ssl.crt
│   │       └── ssl.key
└── ca                       # CA证书目录，主要包括CA证书、CA私钥，请妥善保存CA证书和CA私钥，RPC服务的扩容等操作均需要提供CA证书和CA私钥
    ├── ca.crt
    ├── ca.key
    ├── ca.srl
    └── cert.cnf
```

RPC服务启动成功后，可在tars网页管理平台看到服务列表`agencyABcosRpcService`，且每个服务均是`active`的状态.


```eval_rst
.. note::
   - 如果忘记了访问tars服务的token，可通过tars网页管理平台的【admin】->【用户中心】->【token管理】获取token列表
   - **请妥善保存部署服务过程中生成的RPC服务CA证书和CA私钥，用于SDK证书申请、RPC服务扩容等操作**
```

### 5.3 部署Gateway服务

RPC服务部署完成后，需要再部署Gateway服务，用于建立机构之间的网络连接。在建链工具`BcosBuilder/max`目录下，执行如下命令，可部署并启动2机构Gateway服务，对应的Gateway服务名分别为`agencyABcosGatewayService`，ip为`172.25.0.3`，占用的端口分别为`30300`(进行本操作前，请确保机器的`30300`端口没被占用)。

```shell
# 进入操作目录
cd ~/fisco/BcosBuilder/max

# 部署并启动Gateway服务
python3 build_chain.py chain -o deploy -t gateway
```

执行上述命令后，当脚本输出`deploy service success, type: gateway`时，则说明RPC服务部署成功，详细日志输出如下：

```shell
=========================================================
----------- deploy service, type: gateway -----------
=========================================================
----------- generate service config -----------
* generate service config for 172.25.0.3 : agencyABcosGatewayService
* generate config for the gateway service
* generate generated/gateway/chain0/172.25.0.3/agencyABcosGatewayService/config.ini
* generate cert, output path: generated/gateway/chain0/172.25.0.3/agencyABcosGatewayService
* generate gateway connection file: generated/gateway/chain0/172.25.0.3/agencyABcosGatewayService/nodes.json
* generate config for the gateway service success
gen configuration for service agencyABcosGatewayService success
----------- generate service config success -----------
=========================================================
deploy_service to 172.25.0.3, app: chain0, name: agencyABcosGatewayService
deploy service agencyABcosGatewayService
* add config for service agencyABcosGatewayService, node: 172.25.0.3, config: nodes.json
* add config for service agencyABcosGatewayService, node: 172.25.0.3, config: ca.crt
* add config for service agencyABcosGatewayService, node: 172.25.0.3, config: ssl.key
* add config for service agencyABcosGatewayService, node: 172.25.0.3, config: ssl.crt
* add config for service agencyABcosGatewayService, node: 172.25.0.3, config: config.ini
upload tar package generated/./agencyABcosGatewayService.tgz success, config id: 14
----------- deploy service success, type: gateway -----------
=========================================================
```

部署过程中生成的RPC服务相关的配置位于`generated/gateway/${chainID}`目录，具体如下：

```shell
$ tree generated/gateway/chain0
generated/gateway/chain0
├── 172.25.0.3
│   ├── agencyABcosGatewayService # 机构A的Gateway服务配置路径
│   │   ├── config.ini       # 机构A的Gateway配置文件
│   │   ├── nodes.json        # 机构A的Gateway服务连接配置
│   │   └── ssl                   # 机构A的Gateway服务证书配置
│   │       ├── ca.crt
│   │       ├── cert.cnf
│   │       ├── ssl.crt
│   │       └── ssl.key
└── ca                          # Gateway服务的根证书配置，请妥善保存根证书和根证书私钥
    ├── ca.crt
    ├── ca.key
    ├── ca.srl
    └── cert.cnf
```

```eval_rst
.. note::
   - **请妥善保存部署服务过程中生成的RPC服务CA证书和CA私钥，用于Gateway服务扩容等操作**
```

Gateway服务启动成功后，可在tars网页管理平台看到服务列表`agencyABcosGatewayService`，且每个服务均是`active`的状态.

### 5.4 部署区块链节点服务

RPC服务和Gateway服务均部署完成后，可部署区块链节点服务。在建链工具`BcosBuilder/max`目录下，执行如下命令，可部署并启动单机构单节点区块链服务，对应的服务名分别为`agencyAgroup0node0BcosMaxNodeService`和`agencyAgroup0node0BcosExecutorService`，链ID均为`chain0`，群组ID均为`group0`。

```shell
# 进入操作目录
cd ~/fisco/BcosBuilder/max

# 配置Max节点的部署IP，以及对应的tikv存储pd访问地址，如下:
    [[agency.group]]
        group_id = "group0"

        [[agency.group.node]]
        node_name = "node0"
        # the tikv storage pd-addresses
        pd_addrs="172.25.0.3:2379"
        key_page_size=10240
        deploy_ip = ["172.25.0.3"]
        executor_deploy_ip=["172.25.0.3"]
        monitor_listen_port = "3901"
        # the tikv storage pd-addresses
        # monitor log path example:"/home/fisco/tars/framework/app_log/"
        monitor_log_path = ""

# 部署并启动区块链节点服务
python3 build_chain.py chain -o deploy -t node
```
执行上述命令后，当脚本输出`deploy all nodes of the given group success`时，则说明区块链节点服务部署成功，详细日志输出如下：

```shell
=========================================================
----------- generate config for all nodes -----------
----------- generate genesis config for group group0 -----------
* generate pem file for agencyAgroup0node0BcosMaxNodeService
	- pem_path: ./generated/chain0/group0/agencyAgroup0node0BcosMaxNodeService/node.pem
	- node_id_path: ./generated/chain0/group0/agencyAgroup0node0BcosMaxNodeService/node.nodeid
	- node_id: 9b30cccfd752b9715e6c2fc79b20d5deccf03225015afe6add3aaf0324264b60fff684f4ddee31b18170cac896341b26a24b65c9746dc8563e46e2d3c321f673

	- sm_crypto: 0
* generate_genesis_config_nodeid
* consensus_type: pbft
* block_tx_count_limit: 1000
* leader_period: 1
* gas_limit: 3000000000
* compatibility_version: 3.0.0
* generate_genesis_config_nodeid success
* store genesis config for chain0.group0
	 path: generated/chain0/group0/config.genesis
* store genesis config for chain0.group0 success
* store genesis config for agencyAgroup0node0BcosMaxNodeService
	 path: ./generated/chain0/group0/agencyAgroup0node0BcosMaxNodeService/config.genesis
* store genesis config for agencyAgroup0node0BcosMaxNodeService success
----------- generate genesis config for group0 success -----------
----------- generate ini config for BcosMaxNodeService of group group0 -----------
* store ini config for agencyAgroup0node0BcosMaxNodeService
	 path: ./generated/chain0/group0/agencyAgroup0node0BcosMaxNodeService/config.ini
* store ini config for agencyAgroup0node0BcosMaxNodeService success
----------- generate ini config for BcosMaxNodeService of group group0 success -----------
----------- generate ini config for BcosExecutorService of group group0 -----------
* generate pem file for agencyAgroup0node0BcosMaxNodeService
	- pem_path: ./generated/chain0/group0/agencyAgroup0node0BcosMaxNodeService/node.pem
	- node_id_path: ./generated/chain0/group0/agencyAgroup0node0BcosMaxNodeService/node.nodeid
	- node_id: 9b30cccfd752b9715e6c2fc79b20d5deccf03225015afe6add3aaf0324264b60fff684f4ddee31b18170cac896341b26a24b65c9746dc8563e46e2d3c321f673

	- sm_crypto: 0
* store executor ini config for agencyAgroup0node0BcosExecutorService
	 path: ./generated/chain0/group0/agencyAgroup0node0BcosExecutorService/config.ini
* store executor ini config for agencyAgroup0node0BcosExecutorService success
* store executor genesis config for agencyAgroup0node0BcosExecutorService
	 path: ./generated/chain0/group0/agencyAgroup0node0BcosExecutorService/config.genesis
* store executor genesis config for agencyAgroup0node0BcosExecutorService success
----------- generate ini config for BcosExecutorService of group group0 success -----------
----------- generate config for all nodes success -----------
deploy services for all the group nodes
deploy service agencyAgroup0node0BcosNodeService
upload tar package generated/./agencyAgroup0node0BcosNodeService.tgz success, config id: 16
deploy service agencyBgroup0node0BcosNodeService
upload tar package generated/./agencyBgroup0node0BcosNodeService.tgz success, config id: 17
=========================================================
```

部署过程中生成的服务相关的配置位于`generated/${chainID}`(`chainID`默认为`chain`)目录，具体如下：

```shell
$ tree generated/chain0
generated/chain0
└── group0
    ├── agencyAgroup0node0BcosExecutorService
    │   ├── config.genesis 
    │   └── config.ini
    ├── agencyAgroup0node0BcosMaxNodeService
    │   ├── config.genesis  # 创世块配置
    │   ├── config.ini      # 区块链节点配置
    │   ├── node.nodeid
    │   └── node.pem        # 区块链节点服务签名私钥
    └── config.genesis
```

```eval_rst
.. note::
   - 建议部署RPC和Gateway服务之后再部署区块链节点服务
   - 部署Max版本区块链节点之前，请确保部署并启动了tikv
```

区块链节点服务启动成功后，可在tars网页管理平台看到服务列表`agencyAgroup0node0BcosMaxNodeService`和`agencyAgroup0node0BcosExecutorService`，且每个服务均是`active`的状态。


## 6. 配置及使用控制台

控制台同时适用于Air/Pro/Max版本的FISCO BCOS区块链，且在体验上完全一致。Max版本区块链体验环境搭建完毕后，可配置并使用控制台向Max版本区块链发送交易。

### 6.1 安装依赖

```eval_rst
.. note::
   - 控制台的配置方法和命令请参考 `这里 <../../develop/console/console_config.html>`_
```

使用控制台之前，需先安装java环境：

```shell
# ubuntu系统安装java
sudo apt install -y default-jdk

#centos系统安装java
sudo yum install -y java java-devel
```

### 6.2 下载、配置并使用控制台

**步骤1：下载控制台**

```shell
cd ~/fisco && curl -LO https://github.com/FISCO-BCOS/console/releases/download/v3.0.1/download_console.sh && bash download_console.sh
```
```eval_rst
.. note::
    - 如果因为网络问题导致长时间无法下载，请尝试 `cd ~/fisco && curl -#LO https://gitee.com/FISCO-BCOS/console/raw/master/tools/download_console.sh`
```

**步骤2：配置控制台**

- 拷贝控制台配置文件

若RPC服务未采用默认端口，请将文件中的20200替换成RPC服务监听端口。

```shell
# 最新版本控制台使用如下命令拷贝配置文件
cp -n console/conf/config-example.toml console/conf/config.toml
```

- 配置控制台证书

```shell
# 可通过命令 find . -name sdk找到所有SDK证书路径
cp -r ~/fisco/BcosBuilder/max/generated/rpc/chain0/agencyBBcosRpcService/172.25.0.3/sdk/* console/conf
```

**步骤3：启动并使用控制台**

```shell
cd ~/fisco/console && bash start.sh
```

输出下述信息表明启动成功 否则请检查conf/config.toml中节点端口配置是否正确以及是否配置了SDK证书：

```shell
=============================================================================================
Welcome to FISCO BCOS console(3.0.1)!
Type 'help' or 'h' for help. Type 'quit' or 'q' to quit console.
 ________ ______  ______   ______   ______       _______   ______   ______   ______
|        |      \/      \ /      \ /      \     |       \ /      \ /      \ /      \
| $$$$$$$$\$$$$$|  $$$$$$|  $$$$$$|  $$$$$$\    | $$$$$$$|  $$$$$$|  $$$$$$|  $$$$$$\
| $$__     | $$ | $$___\$| $$   \$| $$  | $$    | $$__/ $| $$   \$| $$  | $| $$___\$$
| $$  \    | $$  \$$    \| $$     | $$  | $$    | $$    $| $$     | $$  | $$\$$    \
| $$$$$    | $$  _\$$$$$$| $$   __| $$  | $$    | $$$$$$$| $$   __| $$  | $$_\$$$$$$\
| $$      _| $$_|  \__| $| $$__/  | $$__/ $$    | $$__/ $| $$__/  | $$__/ $|  \__| $$
| $$     |   $$ \\$$    $$\$$    $$\$$    $$    | $$    $$\$$    $$\$$    $$\$$    $$
 \$$      \$$$$$$ \$$$$$$  \$$$$$$  \$$$$$$      \$$$$$$$  \$$$$$$  \$$$$$$  \$$$$$$

=============================================================================================
[group0]: />
```

- 用控制台获取信息

```shell
# 获取网络连接信息：
[group0]: /> getPeers
PeersInfo{
    p2pNodeID='3082010a0282010100c1d64abf0af11ceaa69b237090a5078ccbc122aedbf93486100ae65cb38cbf2a6969b80f2beca1abba7f0c1674876b332380a4b76387d62445ba8da7190b54850ed8c3fb4d6f6bafbd4744249a55805c0d804db9aa0f105c44c3381de20c763469892fc11a2bc8467c523592c9b2738069d6beb4cfb413f90e0be53205eca1cf3618100c625667f0592fd682aabe9cfbca7f7c53d79eeb5961ed9f144681b32c9fa55fc4d80b5ffbf32a9f71e900bc1c9a92ce0a485bb1003a915f9215bd7c42461cd52d1b2add644e8c1c273aa3668d4a707771b1a99d6bfcbfdf28be5b9c619eefb0c182ea7e666c5753c79499b1959df17ad5bd0996b9d7f0d62aa53d2b450203010001',
    endPoint='0.0.0.0:30300',
    groupNodeIDInfo=[
        NodeIDInfo{
            group='group',
            nodeIDList=[
                4af0433ac2d2afe305b88e7faae8ea4e94b14c63e78ca93c5c836ece6d0fbcb3d2a476a74ae8fb0a11e9662c0ce9861427c314aea7386cb3b619a4cb21ab227a
            ]
        }
    ],
    peers=[]
}

# 获取节点列表信息
[group0]: /> getGroupPeers
peer0: 4af0433ac2d2afe305b88e7faae8ea4e94b14c63e78ca93c5c836ece6d0fbcb3d2a476a74ae8fb0a11e9662c0ce9861427c314aea7386cb3b619a4cb21ab227a

[group0]: /> getSealerList
[
    Sealer{
        nodeID='4af0433ac2d2afe305b88e7faae8ea4e94b14c63e78ca93c5c836ece6d0fbcb3d2a476a74ae8fb0a11e9662c0ce9861427c314aea7386cb3b619a4cb21ab227a',
        weight=1
    }
]
```

### 6.3. 部署和调用合约

**步骤1：编写HelloWorld合约**

HelloWorld合约提供了两个接口`get()`和`set()`，用于获取/设置合约变量`name`，合约内容如下：

```c++
pragma solidity >=0.6.10 <0.8.20;
contract HelloWorld {
    string name;

    constructor() public {
        name = "Hello, World!";
    }

    function get() public view returns (string memory) {
        return name;
    }

    function set(string memory n) public {
        name = n;
    }
}
```

**步骤2: 部署HelloWorld合约**

为了方便用户快速体验，HelloWorld合约已经内置于控制台中，位于控制台目录`contracts/solidity/HelloWorld。sol`，参考下面命令部署：

```shell
# 在控制台输入以下指令 部署成功则返回合约地址
[group0]: /> deploy HelloWorld
transaction hash: 0x0fe66c42f2678b8d041624358837de34ac7db195abb6f5a57201952062190590
contract address: 0x6849F21D1E455e9f0712b1e99Fa4FCD23758E8F1
currentAccount: 0x537149148696c7e6c3449331d77ddfaabc3c7a75

# 查看当前块高
[group0]: /> getBlockNumber
1
```

**步骤3. 调用HelloWorld合约**

```shell
# 调用get接口获取name变量，此处的合约地址是deploy指令返回的地址
[group0]: /> call HelloWorld 0x6849F21D1E455e9f0712b1e99Fa4FCD23758E8F1 get
---------------------------------------------------------------------------------------------
Return code: 0
description: transaction executed successfully
Return message: Success
---------------------------------------------------------------------------------------------
Return value size:1
Return types: (string)
Return values:(Hello, World!)
---------------------------------------------------------------------------------------------

# 查看当前块高，块高不变，因为get接口不更改账本状态
[group0]: /> getBlockNumber
1

# 调用set方法设置name
[group0]: /> call HelloWorld 0x6849F21D1E455e9f0712b1e99Fa4FCD23758E8F1 set "Hello, FISCO BCOS"
transaction hash: 0x2f7c85c2c59a76ccaad85d95b09497ad05ca7983c5ec79c8f9d102d1c8dddc30
---------------------------------------------------------------------------------------------
transaction status: 0
description: transaction executed successfully
---------------------------------------------------------------------------------------------
Receipt message: Success
Return message: Success
Return value size:0
Return types: ()
Return values:()
---------------------------------------------------------------------------------------------
Event logs
Event: {}

# 查看当前块高，因为set接口修改了账本状态，块高增加到2
[group0]: /> getBlockNumber
2

# 退出控制台
[group0]: /> exit
```
