---
title: BCOS 动态群组架构源码分析
subtitle: 源码分析
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - blockchain
  - bcos
categories:
  - blockchain
date: 2019-11-20 11:17:05
---


# BCOS 动态群组架构源码分析
本文基于[《BCOS 动态群组架构》](http://zhoujiapeng.top/blockchain/bcos-group-architecture-code/)，从源码角度对动态群组架构的实现做进一步分析。

## 3.1 群组信息的初始化
这得从Ledger的构造说起。。。

### 3.1.1 groupId 和 protocol_id
Ledger 构造的时候需要传进一个 service，以及 groupId
```c++
Ledger(std::shared_ptr<dev::p2p::P2PInterface> service, dev::GROUP_ID const& _groupId,
        dev::KeyPair const& _keyPair, std::string const& _baseDir)
      : LedgerInterface(_keyPair), m_service(service), m_groupId(_groupId)
```
Ledger在初始化交易池的时候会计算一个protocol_id，它绑定了m_groupId
```c++
/// init txpool
bool Ledger::initTxPool()
{
    dev::PROTOCOL_ID protocol_id = getGroupProtoclID(m_groupId, ProtocolID::TxPool);
    ... 
    return true;
}
```
protocol_id 和 sealer 的构造有关。

### 3.1.2 pbftEngine 和 pbftSealer
看一下 Ledger 中 pbftSealer 的构造过程：
Ledger模块涉及到了PBFT，新的共识算法需要在这里做修改
```c++
std::shared_ptr<Sealer> Ledger::createPBFTSealer()
{
    ...
    //获取protocol_id
    dev::PROTOCOL_ID protocol_id = getGroupProtoclID(m_groupId, ProtocolID::PBFT);
    /// create consensus engine according to "consensusType"
    Ledger_LOG(DEBUG) << LOG_BADGE("initLedger") << LOG_BADGE("createPBFTSealer")
                      << LOG_KV("baseDir", m_param->baseDir()) << LOG_KV("Protocol", protocol_id);
   　
    //pbftSealer和protocol_id是绑定的
    std::shared_ptr<PBFTSealer> pbftSealer = std::make_shared<PBFTSealer>(m_service, m_txPool,
        m_blockChain, m_sync, m_blockVerifier, protocol_id, m_param->baseDir(), m_keyPair,
        m_param->mutableConsensusParam().sealerList); //这里读取了节点列表

    pbftSealer->setEnableDynamicBlockSize(m_param->mutableConsensusParam().enableDynamicBlockSize);
    pbftSealer->setBlockSizeIncreaseRatio(m_param->mutableConsensusParam().blockSizeIncreaseRatio);

    /// set params for PBFTEngine
    //使用pbftSealer创建pbftEngine
    std::shared_ptr<PBFTEngine> pbftEngine =
        std::dynamic_pointer_cast<PBFTEngine>(pbftSealer->consensusEngine());
    ...
    return pbftSealer;
}
```

createPBFTSealer 方法中 protocol_id 作为创建 pbftSealer 的一个参数，然后根据 pbftSealer 创建 pbftEngine
因此 Ledger、groupId、protocol_id、pbftSealer、pbftEngine 是一一对应的关系
传递关系为：
groupId -> protocol_id -> Ledger -> pbftSealer -> pbftEngine

### 3.1.3 service

Ledger 创建的时候还有另外一个构造参数 service，类型是 dev::p2p::P2PInterface，它用来提供 P2P网络服务，群组信息就是保存在 p2pservice 中的
这个 service 会在系统初始化的时候 Initializer 中被设置：
```c++
//Initializer.h
LedgerInitializer::Ptr ledgerInitializer() { return m_ledgerInitializer; }

//Initializer.cpp
void Initializer::init(std::string const& _path){
    ...
        m_ledgerInitializer = std::make_shared<LedgerInitializer>();
        //设置p2pservice
        m_ledgerInitializer->setP2PService(m_p2pInitializer->p2pService());
        m_ledgerInitializer->setKeyPair(m_secureInitializer->keyPair());
        m_ledgerInitializer->setChannelRPCServer(m_rpcInitializer->channelRPCServer());
        m_ledgerInitializer->initConfig(pt);
        
        m_rpcInitializer->setLedgerManager(m_ledgerInitializer->ledgerManager());
        m_rpcInitializer->initConfig(pt);
        m_ledgerInitializer->startAll();
    ...
}
```

### 3.1.4 Ledger 的初始化
Ledger的初始化由LedgerInitializer统一管理，它会为每个group创建独立的账本
```c++
LedgerInitializer::initLedgers() //为每个group创建独立的账本
{
    vector<dev::GROUP_ID> newGroupIDList;
    try
    {
        newGroupIDList = foreachLedgerConfigure(
            m_groupConfigPath, [&](dev::GROUP_ID const& _groupID, const string& _configFileName) {
                // skip existing group
                if (m_ledgerManager->isLedgerExist(_groupID))
                {
                    return false;
                }
                //为每个group创建Ledger
                bool succ = initLedger(_groupID, m_groupDataDir, _configFileName);
                if (!succ)
                {
                    INITIALIZER_LOG(ERROR)
                        << LOG_BADGE("LedgerInitializer") << LOG_DESC("initSingleGroup failed")
                        << LOG_KV("configFile", _configFileName);
                    ERROR_OUTPUT << LOG_BADGE("LedgerInitializer")
                                 << LOG_DESC("initSingleGroup failed")
                                 << LOG_KV("configFile", _configFileName) << endl;
                    BOOST_THROW_EXCEPTION(InitLedgerConfigFailed());
                    return false;
                }
                //从配置文件中读取节点列表
                h512s sealerList = m_ledgerManager->getParamByGroupId(_groupID)
                                       ->mutableConsensusParam()
                                       .sealerList;
                //将节点列表保存到p2pservice中
                m_p2pService->setNodeListByGroupID(_groupID, sealerList);
                LOG(INFO) << LOG_BADGE("LedgerInitializer init group succ")
                          << LOG_KV("groupID", _groupID);
                return true;
            });
    }
    catch (exception& e)
    {
        INITIALIZER_LOG(ERROR) << LOG_BADGE("LedgerInitializer")
                               << LOG_DESC("parse group config faield")
                               << LOG_KV("EINFO", boost::diagnostic_information(e));
        ERROR_OUTPUT << LOG_BADGE("LedgerInitializer") << LOG_DESC("parse group config faield")
                     << LOG_KV("EINFO", boost::diagnostic_information(e)) << endl;
        BOOST_THROW_EXCEPTION(e);
    }
    return newGroupIDList;
}
```


initLedgers 方法的流程为：
- 遍历所有group
- 根据groupId创建Ledger
- 从配置文件中读取sealerList
- 保存sealerList到p2pservice中

根据groupId从配置文件中读取节点列表：
```c++
h512s sealerList = m_ledgerManager->getParamByGroupId(_groupID)
                                       ->mutableConsensusParam()
                                       .sealerList;
m_p2pService->setNodeListByGroupID(_groupID, sealerList);
```
将sealer节点列表保存到p2pSerivce的m_groupID2NodeList中
```c++
void setNodeListByGroupID(GROUP_ID _groupID, dev::h512s _nodeList) override
    {
        RecursiveGuard l(x_nodeList);
        m_groupID2NodeList[_groupID] = _nodeList;
    }
```

群组关系会被储存到m_groupID2NodeList中

前面介绍过，有两个地方会保存群组信息：
- ConsensusEngineBase 中的 m_sealerList
- P2PService 中的 m_groupID2NodeList 中

initLedgers方法 或读取所有的群组信息保存到 m_groupID2NodeList 中，而 m_sealerList 是在 createPBFTSealer 方法中被初始化的：
```c++
std::shared_ptr<Sealer> Ledger::createPBFTSealer(){
    ...
    std::shared_ptr<PBFTSealer> pbftSealer = std::make_shared<PBFTSealer>(m_service, m_txPool,
        m_blockChain, m_sync, m_blockVerifier, protocol_id, m_param->baseDir(), m_keyPair,
        m_param->mutableConsensusParam().sealerList); //这里读取了节点列表
    ...
}
```
这里通过m_param读取了节点列表。
```c++
Ledger(std::shared_ptr<dev::p2p::P2PInterface> service, dev::GROUP_ID const& _groupId,
        dev::KeyPair const& _keyPair, std::string const& _baseDir)
      : LedgerInterface(_keyPair), m_service(service), m_groupId(_groupId)
    {
        m_param = std::make_shared<LedgerParam>();
        std::string prefix = _baseDir + "/group" + std::to_string(_groupId);
        if (_baseDir == "")
            prefix = "./group" + std::to_string(_groupId);
        m_param->setBaseDir(prefix);
        // m_keyPair = _keyPair;
        assert(m_service);
    }
```
可以看到，这里已经指定了groupid，所以m_sealerList 保存的只是当前节点所在group的节点列表。


### 3.1.5 总结
账本初始化 LedgerInitializer::initLedgers() 会为每个group初始化一个Ledger，并且从配置文件中读取群组信息，保存到 m_p2pService 的 m_groupID2NodeList 中，它保存了所有的群组信息。ConsensusEngineBase的m_sealerList 是在账本创建sealer的时候被初始化的，会根据groupId从配置文件中读取，它只保存当前节点所在group的节点列表信息。后面群组关系发生变化时，这两个变量会被更新。

## 3.2 群组信息的读取和动态更新
获取节点列表信息有两种方式
- ConsensusEngineBase::sealerList()  直接返回 m_sealerList，这种方式是sealer需要获取当前节点所在组的节点列表时使用
- BlockChainImp::sealerList()   从账本白名单中查询节点列表，保存在 P2PService 的 m_groupID2NodeList 中，并更新 ConsensusEngineBase.m_sealerList，这种方式会在需要发送消息时调用 P2PService 时使用；

### 3.2.1 获取群组信息的时机
对于第一种方法，在sealer可以直接调用，第二种方法是在消息广播时触发的，广播消息的时候需要获取节点列表，才知道消息要发往何处：
```c++
bool PBFTEngine::broadcastMsg(unsigned const& packetType, std::string const& key,
    bytesConstRef data, std::unordered_set<h512> const& filter, unsigned const& ttl)
{
    //从p2pservice获取session
    auto sessions = m_service->sessionInfosByProtocolID(m_protocolId);
    m_connectedNode = sessions.size();
    NodeIDs nodeIdList;
    for (auto session : sessions)
    {
        ...
        nodeIdList.push_back(session.nodeID());
        broadcastMark(session.nodeID(), packetType, key);
    }
    /// send messages according to node id
    m_service->asyncMulticastMessageByNodeIDList(
        nodeIdList, transDataToMessage(data, packetType, ttl));
    return true;
}

std::shared_ptr<dev::p2p::P2PInterface> m_service;
```
这里根据 m_protocolId 从 p2pservice 中获取 session，session 里面就保存了nodeID。前面已经说过，m_protocolId 是根据 groupId 生成的，因此 p2pservice 就可以根据 m_protocolId 从Ledge中读取所在group的节点列表。

### 3.2.2 ConsensusEngineBase::sealerList()
```c++
//ConsensusEngineBase.h
dev::h512s sealerList() const override
    {
        ReadGuard l(m_sealerListMutex);
        return m_sealerList;
    }
 /// append sealer
    void appendSealer(h512 const& _sealer) override
    {
        {
            WriteGuard l(m_sealerListMutex);
            m_sealerList.push_back(_sealer);
        }
        resetConfig();
    }
```
在sealer中需要读取节点列表时会调用这个方法，直接返回 m_sealerList，但它并不负责对m_sealerList的更新维护。这里的 appendSealer() 方法没有被任何地方调用，貌似是用不着的？

### 3.2.3 BlockChainImp::sealerList()
```c++
dev::h512s BlockChainImp::sealerList()
{
    int64_t blockNumber = number();
    UpgradableGuard l(m_nodeListMutex);
    if (m_cacheNumBySealer == blockNumber)
    {
        BLOCKCHAIN_LOG(TRACE) << LOG_DESC("[#sealerList]Get sealer list by cache")
                              << LOG_KV("size", m_sealerList.size());
        return m_sealerList;
    }
    //获取最新的节点列表
    dev::h512s list = getNodeListByType(blockNumber, NODE_TYPE_SEALER);
    UpgradeGuard ul(l);
    m_cacheNumBySealer = blockNumber;
    m_sealerList = list;　　　//更新 m_sealerList

    return list;
}
```


每次调用 BlockChainImp::sealerList()  都会调用 getNodeListByType 方法读取最新的sealer列表，并更新m_sealerList，使得在ConsensusEngineBase中调用sealerList返回的是最新的m_sealerList。

### 3.2.4 getNodeListByType()
getNodeListByType() 是从账本中读取节点列表:
```c++
dev::h512s BlockChainImp::getNodeListByType(int64_t blockNumber, std::string const& type)
{
    dev::h512s list;
    try
    {
        //获取共识模块的缓存
        Table::Ptr tb = getMemoryTableFactory()->openTable(storage::SYS_CONSENSUS);
        if (!tb)
        {
            BLOCKCHAIN_LOG(ERROR) << LOG_DESC("[#getNodeListByType]Open table error");
            return list;
        }
        //获取节点列表
        auto nodes = tb->select(PRI_KEY, tb->newCondition());
        if (!nodes)
            return list;

        for (size_t i = 0; i < nodes->size(); i++)
        {
            auto node = nodes->get(i);
            if (!node)
                return list;

            if ((node->getField(NODE_TYPE) == type) &&
                (boost::lexical_cast<int>(node->getField(NODE_KEY_ENABLENUM)) <= blockNumber))
            {
                h512 nodeID = h512(node->getField(NODE_KEY_NODEID));
                list.push_back(nodeID);
            }
        }
    }
    catch (std::exception& e)
    {
        BLOCKCHAIN_LOG(ERROR) << LOG_DESC("[#getNodeListByType]Failed")
                              << LOG_KV("EINFO", boost::diagnostic_information(e));
    }

    std::stringstream s;
    s << "[#getNodeListByType] " << type << ":";
    for (dev::h512 node : list)
        s << toJS(node) << ",";
    BLOCKCHAIN_LOG(TRACE) << LOG_DESC(s.str());

    return list;
}
```
从 SYS_CONSENSUS 的 MemoryTable (账本白名单)中读取的

### 3.2.5 updateConsensusNodeList()
除了在获取节点列表的时候会触发更新的操作，在ConsensusEngineBase中有直接更新节点列表的方法：
```c++
//ConsensusEngineBase.cpp
void ConsensusEngineBase::updateConsensusNodeList()
{
    try
    {
        std::stringstream s2;
        s2 << "[updateConsensusNodeList] Sealers:";
        {
            WriteGuard l(m_sealerListMutex);
            //获取sealer节点列表
            m_sealerList = m_blockChain->sealerList();
            ...
        }
        s2 << "Observers:";
        //获取observer节点列表
        dev::h512s observerList = m_blockChain->observerList();
        ...
    }
    ...
}
```


## 3.3 动态群组实现
通过控制台可以实现群组关系的变更

### 3.3.1 控制台
通过控制台可以让节点加入某group
新节点加入群组前，需要确保：
- 新加入NodeID存在
- 群组内节点正常共识：正常共识的节点会输出+++日志
```shell
# 2. 将node2加入到共识节点
# addSealer后面的参数是上步获取的node ID
$ [group:2]> addSealer 6dc585319e4cf7d73ede73819c6966ea4bed74aadbbcba1bbb777132f63d355965c3502bed7a04425d99cdcfb7694a1c133079e6d9b0ab080e3b874882b95ff4
{
    "code":0,
    "msg":"success"
}
```

查看控制台的源码：
https://github.com/FISCO-BCOS/console/blob/master/src/main/java/console/ConsoleClient.java
```java
 case "addSealer":
     precompiledFace.addSealer(params);
     break;
```
```java
precompiledFace.addSealer：
@Override
    public void addSealer(String[] params) throws Exception {
        ...
        String nodeId = params[1];
        if ("-h".equals(nodeId) || "--help".equals(nodeId)) {
            HelpInfo.addSealerHelp();
            return;
        }
        if (nodeId.length() != 128) {
            ConsoleUtils.printJson(
                    PrecompiledCommon.transferToJson(PrecompiledCommon.InvalidNodeId));
        } else {
            ConsensusService consensusService = new ConsensusService(web3j, credentials);
            String result;
            result = consensusService.addSealer(nodeId);
            ConsoleUtils.printJson(result);
        }
        System.out.println();
    }
```
调用了consensusService.addSealer方法，ConsensusService位于 web3sdk 中，控制台通过web3sdk 向链上发送了一个增加节点的请求。
```java
import org.fisco.bcos.web3j.precompile.consensus.ConsensusService;
```

### 3.3.2 Web3SDK
```java
//ConsensusService.java
public String addSealer(String nodeID) throws Exception {
        if (!isValidNodeID(nodeID)) {
            return PrecompiledCommon.transferToJson(PrecompiledCommon.P2pNetwork);
        }
        //获取原有节点列表
        List<String> sealerList = web3j.getSealerList().send().getResult();
        //如果已经存在该节点
        if (sealerList.contains(nodeID)) {
            return PrecompiledCommon.transferToJson(PrecompiledCommon.SealerList);
        }
        //不存在则添加
        TransactionReceipt receipt = consensus.addSealer(nodeID).send();
        return PrecompiledCommon.handleTransactionReceipt(receipt, web3j);
    }
```
这里通过调用consensus的addSealer方法。
consensus是一个合约，它的函数体是空的，说明它是预编译合约的接口合约，它的具体逻辑在源码的Precompiled模块中。
接口合约：
```
pragma solidity ^0.4.2;

contract Consensus {
    function addSealer(string nodeID) public returns(int);
    function addObserver(string nodeID) public returns(int);
    function remove(string nodeID) public returns(int);
}
```

### 3.3.3 源码预编译模块
```c++
libprecompiled/ConsensusPrecompiled.cpp
const char* const CSS_METHOD_ADD_SEALER = "addSealer(string)";
const char* const CSS_METHOD_ADD_SER = "addObserver(string)";
const char* const CSS_METHOD_REMOVE = "remove(string)";
ConsensusPrecompiled::ConsensusPrecompiled()
{
    name2Selector[CSS_METHOD_ADD_SEALER] = getFuncSelector(CSS_METHOD_ADD_SEALER);
    name2Selector[CSS_METHOD_ADD_SER] = getFuncSelector(CSS_METHOD_ADD_SER);
    name2Selector[CSS_METHOD_REMOVE] = getFuncSelector(CSS_METHOD_REMOVE);
}
```

call方法实现了合约方法的逻辑:
```c++
bytes ConsensusPrecompiled::call(
    ExecutiveContext::Ptr context, bytesConstRef param, Address const& origin)
{
    PRECOMPILED_LOG(TRACE) << LOG_BADGE("ConsensusPrecompiled") << LOG_DESC("call")
                           << LOG_KV("param", toHex(param));

    // parse function name
    uint32_t func = getParamFunc(param);
    bytesConstRef data = getParamData(param);

    dev::eth::ContractABI abi;
    bytes out;
    int count = 0;

    showConsensusTable(context);

    int result = 0;
    if (func == name2Selector[CSS_METHOD_ADD_SEALER])  //addSealer方法
    {
        // addSealer(string)
        std::string nodeID;
        abi.abiOut(data, nodeID);
        // Uniform lowercase nodeID
        boost::to_lower(nodeID);

        PRECOMPILED_LOG(DEBUG) << LOG_BADGE("ConsensusPrecompiled") << LOG_DESC("addSealer func")
                               << LOG_KV("nodeID", nodeID);
        if (nodeID.size() != 128u)
        {
            PRECOMPILED_LOG(ERROR) << LOG_BADGE("ConsensusPrecompiled")
                                   << LOG_DESC("nodeID length error") << LOG_KV("nodeID", nodeID);
            result = CODE_INVALID_NODEID;
        }
        else
        {
            storage::Table::Ptr table = openTable(context, SYS_CONSENSUS);
            
            //创建一个新condition对象，是一个查询条件，条件为nodeID
            auto condition = table->newCondition();
            condition->EQ(NODE_KEY_NODEID, nodeID);
            //根据condition从table查找是否已经存在该node
            auto entries = table->select(PRI_KEY, condition);
            auto entry = table->newEntry();
            entry->setField(NODE_TYPE, NODE_TYPE_SEALER);
            entry->setField(PRI_COLUMN, PRI_KEY);
            entry->setField(NODE_KEY_ENABLENUM,
                boost::lexical_cast<std::string>(context->blockInfo().number + 1));

            if (entries.get())
            {
                if (entries->size() == 0u)//为0,说明table里面不存在该nodeID
                {
                    entry->setField(NODE_KEY_NODEID, nodeID);
                    //插入该nodeID
                    count = table->insert(PRI_KEY, entry, std::make_shared<AccessOptions>(origin));
                    if (count == storage::CODE_NO_AUTHORIZED)
                    {
                        PRECOMPILED_LOG(DEBUG)
                            << LOG_BADGE("ConsensusPrecompiled") << LOG_DESC("permission denied");
                        result = storage::CODE_NO_AUTHORIZED;
                    }
                    else
                    {
                        PRECOMPILED_LOG(DEBUG) << LOG_BADGE("ConsensusPrecompiled")
                                               << LOG_DESC("addSealer successfully");
                        result = count;
                    }
                }
                else　　//entries不为０，说明table已经存在该node
                {
                    count = table->update(
                        PRI_KEY, entry, condition, std::make_shared<AccessOptions>(origin));
                    if (count == storage::CODE_NO_AUTHORIZED)
                    {
                        PRECOMPILED_LOG(DEBUG)
                            << LOG_BADGE("ConsensusPrecompiled") << LOG_DESC("permission denied");
                        result = storage::CODE_NO_AUTHORIZED;
                    }
                    else
                    {
                        PRECOMPILED_LOG(DEBUG) << LOG_BADGE("ConsensusPrecompiled")
                                               << LOG_DESC("addSealer successfully");
                        result = count;
                    }
                }
            }
        }
    }else if
    .....
}
```
addSealer 的基本流程为：
1. 打开SYS_CONSENSUS的table(账本节点白名单)
2. 使用nodeId创建查询条件 condition
3. 使用 condition 从 table 里面查找
3.1 查找得到，说明已经存在该nodeId
3.2 找不到，则插入 MemotyTable.insert()
```c++
storage::Table::Ptr Precompiled::openTable(
    ExecutiveContext::Ptr context, const std::string& tableName)
{
    TableFactoryPrecompiled::Ptr tableFactoryPrecompiled =
        std::dynamic_pointer_cast<TableFactoryPrecompiled>(
            context->getPrecompiled(Address(0x1001)));
    return tableFactoryPrecompiled->getMemoryTableFactory()->openTable(tableName);
}
```

### 3.3.4 总结
新增节点到群组的流程：
1. 通过 console 调用 addSealer 方法
2. addSealer 方法调用预编译合约 ConsensusPrecompiled 的 addSealer 方法
3. addSealer 方法将 nodeId 插入到名字为 SYS_CONSENSUS 的 MemotyTable(账本节点白名单) 中

使用发送交易共识的方式修改账本白名单，保证了账本白名单群组内的一致性。


## 3.4 动态群组实现总结
每个群组均持有一个账本白名单，用于维护该群组的节点列表。为了保证账本白名单群组内一致性，仅可通过发交易共识的方式修改账本白名单，修改账本白名单的合约是预编译合约。在运行过程中，需要获取群组信息的时候是从账本白名单中读取的，保证能够实时得到最新的节点列表。