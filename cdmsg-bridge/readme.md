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

# 
