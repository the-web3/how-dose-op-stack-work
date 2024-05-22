[![English](https://img.shields.io/badge/English-README-blue)](EnglishReadme.md)


# 深入理解 Op-stack 跨链调用过程及 ETH 和 ERC20 的充值提现

# 1. 信使合约
信使合约的主要功能是跨链通信，核心方法位 sendMessage 与 relayMessage；
- sendMessage: 将包裹的消息从源链发送到目标链上，维护自增的 msgNonce, 确报跨链消息的唯一性和安全性
- relayMessage: 将源链发过来的消息在目标链上构建 MsgHash 对比，确保消息的一致性之后在目前链上调度执行该消息
在 op-stack 中在 L1 和 L2 层分别有
- L1CrossDomainMessenger
- L2CrossDomainMessenger
继承自 CrossDomainMessenger 合约，来承载跨链通信，关于跨链消息合约如何被调用的，在充值，提现章节会有详细的解释。

# 2. OptimismPortal
OptimismPortal 合约是 op-stack 的充值提现纽带合约

## 2.1 充值
充值交易会调用到 depositTransaction; depositTransaction 会抛出 TransactionDeposited 事件，事件里面携带信息如下
emit TransactionDeposited(from, _to, DEPOSIT_VERSION, opaqueData);
op-node 监听到该合约事件之后，会在二层去执行充值交易，关于 op-node 的详细介绍，我们后面会讲到

## 2.2 提现

对于提现来说，OptimismPortal 会做两件事儿
- 一是交易证明proveWithdrawalTransaction，将证明通过的交易更新到 
mapping(bytes32 => ProvenWithdrawal) public provenWithdrawals
只要进入到里面都交易表示都是已经经过证明的

- 另一个是交易 relay transaction 的操作，finalizeWithdrawalTransaction 函数会将 relay 的交易放到 
mapping(bytes32 => bool) public finalizedWithdrawals
只要进入到 finalizedWithdrawals Map里面的交易都证明已经被 relay。

# 3. Bridge 合约
在本节中，我们会把整个桥的上的充值，提现详细的流程图画出来。

# 4. L1->L2 充值

## 4.1 ETH 充值

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/ethdeposit.png)](https://github.com/guoshijiang/how-dose-op-stack-work)


- User
  - 用户调用 depositETH 给自己的地址充值，或者调用 depositETHTo 给指定的 to 地址充值；
- L1StandardBridge
  - 不管是调用 depositETH 还是 depositETHTo，最终都会进入到一个函数 _initiateETHDeposit；_initiateETHDeposit 里面没有做任何操作，直接调用 _initiateBridgeETH
  - 在 _initiateBridgeETH 函数中，会有如下操作
    - 判断 msg.sender 是否有足够的资金以供充值
    - 抛出一个 _emitETHBridgeInitiated 事件
    - 调用 CrossDomainMessenger合约的 sendMessage 方法
- CrossDomainMessenger
  - 在  CrossDomainMessenger 里面会去调用 L1CrossDomainMessenger 的 _sendMessage 函数，执行成功之后，msgNonce 会加一下
- L1CrossDomainMessenger
  - L1CrossDomainMessenger 合约的 _sendMessage 会去调用 OptimismPortal 的 depositTransaction 函数
  - depositTransaction 会抛出 TransactionDeposited 事件
- Op-node 和 op-geth
  - Op-node 监听到 TransactionDeposited，会去构建充值交易的 Attrs, 指到 op-geth 在 L2 执行该充值交易
  - Op-geth 会去调用 L2StandardBridge 合约的  finalizeDeposit
- L2StandardBridge
  - finalizeBridgeETH 函数会去执行 SafeCall.call，导致位置，充值流程就完事儿
 
注意：对于 ETH 充值来说，由于 OptimismPortal 和 L1StandardBridge 里面存在 receive 方法

- OptimismPortal
```
receive() external payable {
   depositTransaction(msg.sender, msg.value, RECEIVE_DEFAULT_GAS_LIMIT, false, bytes(""));
}
```

- L1StandardBridge
```
receive() external payable override onlyEOA {
   _initiateETHDeposit(msg.sender, msg.sender, RECEIVE_DEFAULT_GAS_LIMIT, bytes(""));
}
```

因此，直接把 ETH 转入到 OptimismPortal 和 L1StandardBridge 里面，也属于 ETH 的充值交易

 
## 4.2 ERC20 充值

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/erc20deposit.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

- User
  - 用户调用 depositERC20 给自己的地址充值，或者调用 depositERC20To 给指定的 to 地址充值；
- L1StandardBridge
  - 不管是调用 depositERC20 还是 depositERC20To，最终都会进入到一个函数 _initiateBridgeERC20；
  - 判断是否为 OptimismMintableERC20 ，如果是， 调用 burn 方法销毁 token;  如果不是，将 token 转到桥里面，并将资金记录到账本中；账本代码如下：
  ```
  mapping(address => mapping(address => uint256)) public deposits;
  deposits[_localToken][_remoteToken] = deposits[_localToken][_remoteToken] + _amount;
  ```
  - 接下来调用 CrossDomainMessenger 的 sendMessage 发送跨链消息，调用 L1CrossDomainMessenger 的 _sendMessage 方法成功之后，会对 nonce 进行自增加 1
- L1CrossDomainMessenger
  - _sendMessage 会去调用 OptimismPortal 的 depositTransaction，
  - depositTransaction 会抛出 TransactionDeposited 事件
- Op-node 和 op-geth
  - Op-node 监听到 TransactionDeposited，会去构建充值交易的 Attrs, 指到 op-geth 在 L2 执行该充值交易
  - Op-geth 会去调用 L2StandardBridge 合约的  finalizeDeposit
- L2StandardBridge
  - finalizeBridgeERC20 函数会去执行 mint Token 或者 token Transfer 的操作
  - 若是 OptimismMintableERC20，执行 mint 操作
  - 若不是 OptimismMintableERC20，更新账本，代码如下：

  ```   
  mapping(address => mapping(address => uint256)) public deposits;
  deposits[_localToken][_remoteToken] = deposits[_localToken][_remoteToken] - _amount;
  ```

## 4.3 ERC721 充值

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/erc721deposit.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

关于 ERC721 的充值，目前逻辑是断层的，并没有完全关联起来，这里不再做过多的赘述。

# 5. L2->L1 提现

## 5.1 ETH 提现

### 5.1.1 ETH 提现交易到 L1 

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/ethwithdraw.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

- User
  - 用户调用 withdraw 给自己的地址提币，或者调用 withdrawTo 给指定的 to 地址提现；
- L2StandardBridge
  - withdraw 和 withdrawTo 都会去 call  _initiateWithdrawal; 该方法里面会去判断是 ETH 的提现还是 ERC20 的提现
- CrossDomainMessenger & L2CrossDomainMessenger
  - 若是 ETH 提现， 进入 _initiateBridgeETH 函数，该函数会去调用 CrossDomainMessenger 合约的sendMessage 方法，sendMessage 方法里面的逻辑成功后会对 msgNonce 自增 1，然后调用进入到了 L2CrossDomainMessenger 的 _sendMessage 方法。
- L2ToL1MessagePasser
  - 从 L2CrossDomainMessenger 合约的 _sendMessage 调入到  L2ToL1MessagePasser 的 initiateWithdrawal 之后，会进行提现交易的 data 的构造计算出 withdrawalHash，并将该 Hash 在 map 数据结构设置为 true
  - 维护的 L2 的  msgNonce 自增 1
- Op-node & Op-geth
  - 二层生成提现交易
- Op-batcher & Op-proposer,  关于 Op-batcher 和 Op-proposer 的详细流程，后面会有介绍，这里就是简单的概述
  - Op-batch 把提现交易的数据 rollup 到 L1
  - Op-proposer 把提现交易相关的一批次数据的 stateroot 提交到 L1
 
### 5.1.2 ETH Cliam 交易流程

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/ethcliam.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

- 用户调用 SDK 获取提现交易的状态，若状态为 READY_TO_PROVE, 说明交易可以进行证明； 用户可以调用 SDK 的 proveMessage 去进行交易的证明，或者直接 call OptimismPortal 合约 proveWithdrawalTransaction 方法进行交易的证明，当证明交易产生之后，等到交易过了挑战期，交易状态会变成 READY_FOR_RELAY；这个时候用户可以调用 SDK 的 finalizeMessage 方法进行资金的 cliam, 也可以直接调用 OptimismPortal 合约的 finalizeWithdrawalTransaction 方法进行资金的 cliam。
- 直接调用合约要进行各种组合调用，比较麻烦，建议用 sdk 进行交易的证明和 cliam
- 上图是使用 SDK cliam 的完整调用流程图

## 5.2 ERC20 提现

### 5.2.1 ERC20 提现交易到 L1 

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/erc20withdraw.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

-  上面流程和 ETH 提现不一样的点是
  - 如果是 OptimismMintableERC20 合约 Token burn 
  - 否则 deposits 账本里面的加上对应的金额，并做 token safeTransfer

### 5.2.2 ERC20 Cliam 交易流程

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/erc20cliam.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

- 上述流程和 ETH cliam 不一致的就是 finalize 的时将 finalizeERC20Withdrawal 变成了并且 finalizeBridgeERC20 里面的逻辑也不一致
  - finalizeBridgeERC20
    - 如果 OptimismMintableERC20 类型的 token, mint
    - 否则 deposits 账本里面的减去对应的金额，并做 token safeTransfer
   
# 6. 小结

本节主要讲解了 op-stack 的跨链调用过程以及 ETH 和 ERC20 Token 的充值过程，下一讲我们将详细讲 op-stack 的 rollup 流程。




