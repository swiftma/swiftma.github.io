# EOS 实现分析 (draft)

### 看待角度

1. 单节点独享智能合约平台 
2. 独享 -> SaaS模式
3. 单节点 -> 多节点容错共识 

### 对智能合约开发者的API

* 存储：multi-index table，不是以太坊的key-value db，背后利用boost的multi_index_container和内存映射文件实现     
* 计算：C++ 14，WAVM
* 其他API：authorization、transaction, action, log, assert等

### 对用户的API (cleos)

* 钱包管理
* 账户创建
* 合约管理 
* 转账、提交transaction/action
* 权限设置
* 信息查询（blockchain, block, transaction, account, table, code, currency等）
* 买卖内存，抵押/赎回 (token for CPU/带宽)
* block producer(BP)注册、投票
* peers管理

### 实现

* 状态
  * 当前状态：chainbase - multi-index-container + 内存映射文件 + undo session
  * 不可逆块：两个文件blocks.index和blocks.log 
* 架构  
  * c++ 14, boost, cmake （依赖库、编译器版本等，不像以太坊有规范、有黄皮书和多种语言实现）
  * 松耦合：plugin, publish/subscribe, plugin/libraries/contracts的分工
  * producer-plugin打包block

### 独享 -> SaaS模式

* 计费
  * 使用状态存储消耗Token，CPU和带宽按抵押的Token比例分配
* 安全
  * 白名单、黑名单
  * API Server和Block Producer的分离

### 单节点 -> 多节点容错共识

* BFT-DPoS：时刻在投票，126秒选举一次，选出21个BP，每个BP出12块，超过2/3 BP认可变为不可逆
* 实现：net-plugin，与peer互相同步块和交易信息

### 以太坊 VS EOS

- PoW/PoS VS BFT-DPoS
- action消耗gas VS 抵押token
- 细粒度按指令计算消耗 VS 粗略度按时间计算
- EVM VS WASM
- 有规范、多语言实现 VS 绑定C++ 14/Boost/编译器版本
- 合约不可修改 VS 可修改
- 合约数据模型：key-value db VS multi-index table

  
