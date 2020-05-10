---
title: BCOS 动态群组架构
subtitle: 概述
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - blockchain
  - bcos
categories:
  - blockchain
date: 2019-11-20 11:21:02
---

# BCOS 动态群组架构
分析 BCOS 动态群组架构的目的在于研究动态群组架构和共识机制中节点动态性的关系，最终基于 BCOS 的共识模块接口实现新的共识算法。

# 1. 群组架构的设计

> 来源：[FISCO BCOS开源社区：群组架构的设计](https://mp.weixin.qq.com/s?__biz=MzU5NTg0MjA4MA==&mid=2247483919&idx=1&sn=035d336c6888fa87eec1e579d7b503d1&scene=19#wechat_redirect)

## 1.1 架构设计全景

![架构设计全景](https://gitee.com/JP6907/Pic/raw/master/bcos/group-architecture.png)

如上图所示，群组架构自底向下主要划分为网络层、群组层，网络层主要负责区块链节点间通信，群组层主要负责处理群组内交易，每个群组均运行着一个独立的账本。

## 1.2 网络层

群组架构中，所有群组共享P2P网络，群组层传递给网络层的数据包中含有群组ID信息，接收节点根据数据包中的群组ID，将收到的数据包传递给目标节点的相应群组

为了做到群组间通信数据隔离，群组架构引入了账本白名单机制，下图展示了群组架构下群组间收发消息的流程：

![网络层](https://gitee.com/JP6907/Pic/raw/master/bcos/arcitecture-network.png)

**账本白名单**
每个群组均持有一个账本白名单，用于维护该群组的节点列表。为了保证账本白名单群组内一致性，仅可通过发交易共识的方式修改账本白名单。

**发包流程**

以node0的第一组向node1的第一组发送消息packetA为例：
(1) group1将消息packetA传递到网络层；
(2) 网络层模块对packetA进行编码，在packetA包头加上本群组ID，记为{groupID(1) + packetA}；
(3) 网络层访问账本白名单，判断node0是否是group1的节点，若非group1节点，则拒绝数据包；若是group1节点，则将编码后的包发送给目标节点node1。

**收包流程**
node1接收到node0 group1的数据包{groupID(1) + packetA}后：
(1) 网络层访问账本白名单，判断源节点node0是否是group1节点，若非group1节点，则拒绝数据包，否则将数据包传递给解码模块；
(2) 解码模块从数据包中解码出group ID=1和数据包packetA，将数据包packetA发送到group1。

通过账本白名单，可以有效地防止群组节点获取其他群组通信消息，保障了群组网络通信的隐私性。

## 1.3 群组层

群组层是群组架构的核心。为了实现群组间账本数据的隔离，每个群组持有单独的账本模块。
群组层自下向上一次分为核心层、接口层和调度层：核心层提供底层存储和交易执行接口；接口层是访问核心层的接口；调度层包括同步和共识模块，负责处理交易、同步交易和区块。

# 2. 动态群组架构的实现
动态群组架构涉及的各个模块之间的关系如图：

![模块关系图](https://gitee.com/JP6907/Pic/raw/master/bcos/model.png)

## 2.1 节点列表的读取
获取节点列表信息有两种方式
ConsensusEngineBase::sealerList()  直接返回 m_sealerList，这种方式是sealer需要获取当前节点所在组的节点列表时使用
BlockChainImp::sealerList()   从账本白名单中查询节点列表，保存在 P2PService 的 m_groupID2NodeList 中，并更新 BlockChainImp.m_sealerList，这种方式会在需要发送消息时调用 P2PService 时使用；

m_groupID2NodeList 由 P2PService 维护，保存了所有的群组节点列表信息。
m_sealerList 由 sealer 保存，会被 P2PService 更新，保存了当前节点所在 group 的节点列表信息。

## 2.2 群组信息的初始化
在账本初始化的时候，LedgerInitializer 会为每个group创建一个独立的账本，并从配置文件中读取群组信息，保存到 P2PService  的 m_groupID2NodeList 和 ConsensusEngineBase 的 m_sealerList 中。

## 2.3 群组信息保持更新
节点需要广播信息时会通过P2PServer模块获取群组信息，P2PServer会从账本节点白名单中读取最新的节点列表，并更新到 m_sealerList 和 m_groupID2NodeList 。

## 2.4 动态群组实现
账本白名单维护了群组信息，为了保证账本白名单群组内一致性，只能通过交易共识来修改账本白名单。系统已经内置了修改白名单的预编译合约(ConsensusPrecompiled)，console 已经实现了调用这些合约的命令，如将某节点添加到某group中：


```shell
# 2. 将node2加入到共识节点
# addSealer后面的参数是上步获取的node ID
$ [group:2]> addSealer 6dc585319e4cf7d73ede73819c6966ea4bed74aadbbcba1bbb777132f63d355965c3502bed7a04425d99cdcfb7694a1c133079e6d9b0ab080e3b874882b95ff4
{
    "code":0,
    "msg":"success"
}
```

addSealer实际上是通过 web3sdk 调用了 ConsensusPrecompiled 预编译合约的 addSealer 接口，来实现对账本白名单的修改。

## 2.5 账本白名单

```shell
storage::Table::Ptr table = openTable(context, SYS_CONSENSUS);
```

# 3. 源码分析
见[《BCOS动态群组源码分析》](http://zhoujiapeng.top/blockchain/bcos-group-architecture-code/)。

# 4. 总结
BCOS在网络层P2PService中已经实现了对节点动态性的控制，共识算法中只需要考虑单个群组内部的共识问题，不需要考虑动态性的问题。如果在共识算法中接入节点动态控制，反而会增加算法的复杂性，并且不会带来好处。