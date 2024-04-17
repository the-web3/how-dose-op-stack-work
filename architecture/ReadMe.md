[![English](https://img.shields.io/badge/English-README-blue)](EnglishReadme.md)


# architecture

[![architecture](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/architecture/images/architecture.png)](https://github.com/guoshijiang/how-dose-op-stack-work)


- Op-stack 主要由 op-node, op-geth, op-batcher, op-proposer, CrossDomainMessenger, OptimismPortal, Bridge contracts 和 L2OutputOracle contract 等角色组成
  - op-node 加 op-geth(Seqeuencer & Verifier)：负责交易打包落块，交易的状态推导,  数据传输同步等
  - op-batcher:  提交交易数据到 L1 的 EOA 地址
  - op-proposer 提交区块状态到 L1 的 L2OutputOracle 合约
  - CrossDomainMessenger 跨链信使合约，主要功能是 sendMessage 和 relyMessage, 负责 L1->L2, L2->L1 的通信合约；
  - Bridge contracts：桥合约，主要功能是承载充值提现，L1->L2: L1 将资金锁定在桥中，L2 Mint 相应的资金；L2->L1:  L2 Burn 相应的资金； L1 解锁对应的资金。

