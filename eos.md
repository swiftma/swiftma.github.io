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



------

### 以太坊 VS EOS

- PoW/PoS VS BFT-DPoS

- action消耗gas VS 抵押token

- 细粒度按指令计算消耗 VS 粗略度按时间计算

- EVM VS WASM

- 有规范、多语言实现 VS 绑定C++ 14/Boost/编译器版本

- 合约不可修改 VS 可修改

- 合约数据模型：key-value db VS multi-index table

  

### 同步的一些细节

一个block的生命周期：

      1. 收到notice，检查本地有没有

      1. 如果有，只是更新peer state，设置peer is_known=true,更新request time
      2. 如果没有，发送request_message，更新peer state, is_known=true，peer state中的block_num=0

      2. 收到block

      1. 如果本地已经有了，返回并进行同步操作(sync_master.recv_block)，否则加入received_block,更新 peer state

      2. 应用到本地。

      3. 广播block（存在优化空间，发送给所有peer block时检查是否需要发送）

         1. 从received_block中删掉
         2. 如果太大，发送notice_message，否则发送完整信息
         3. 更新Peer的block信息

      4. 更新block中包含的local_trx中的block_num，

      5. 更新peer state中的trx_state中的block_num



一个trx的生命周期：

1. 收到A的notice_msg，只有Id,比如123，更新本地peer记录，记录A有123，过期时间是两分钟。如果local_trx有该trx，说明本地有且已广播过了，不处理。

2. 请求trx 123

3. 收到trx 123，如果local_trx有该trx，说明已经处理过了，不处理，否则 放入received_trx.

4. 应用trx 123到本地。

5. 广播trx 123，不用广播给发送给自己的（通过检查received_trx判断来源），或者peer已经有，也不用再广播。如果peer有，只更新peer的expire_time，之前记录的expire_time是两分钟。

   1. 从received_trx中去掉，放入local_txns，如果local_trx有，说明已经广播过了，不处理
   2. 如果尺寸不大，就直接发送trx，否则发送notice_msg
   3. 发送完，更新关于peer的状态，更新peer的trx_state

6. 如果应用到本地失败，从received_trx中去掉该trx

    

   维护的Local_trx和peer的trx和block state会定期清理过期的trx，以及不可撤销的trx。

