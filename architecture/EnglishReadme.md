#architecture

[![architecture](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/architecture/images/architecture.png)](https://github.com/guoshijiang/how-dose-op-stack-work)


- Op-stack is mainly composed of op-node, op-geth, op-batcher, op-proposer, CrossDomainMessenger, OptimismPortal, Bridge contracts and L2OutputOracle contract.
   - op-node plus op-geth (Seqeuencer & Verifier): responsible for transaction packaging and block dropping, transaction status derivation, data transmission synchronization, etc.
   - op-batcher: Submit transaction data to the EOA address of L1
   - op-proposer submits the block status to the L2OutputOracle contract of L1
   - CrossDomainMessenger cross-chain messenger contract, its main functions are sendMessage and relyMessage, responsible for the communication contracts of L1->L2, L2->L1;
   - Bridge contracts: Bridge contracts, the main function is to carry deposits and withdrawals, L1->L2: L1 locks the funds in the bridge, L2 Mint the corresponding funds; L2->L1: L2 Burn the corresponding funds; L1 unlocks the corresponding funds.
