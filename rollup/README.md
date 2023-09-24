# Op-stack rollup 流程详解

## 1.概述

Op-stack 的 rollup 由两个服务来承担
- op-batcher 服务：主要职责是负责将交易数据提交到 Layer1 的 EOA 地址
- op-proposer 服务：主要职责是负责将交易状态提交到 Layer1 的 L2OutputOracle 合约

## 2.Rollup 架构

[![rollup_art](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/rollup/rollup_art.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

## 3.op-batcher

### 3.1 执行流程图

[![op-batcher](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/rollup/op-batcher/op-batcher.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

### 3.2 详细执行流程
- loadBlocksIntoState 会执行以下逻辑
  - calculateL2BlockRangeToStore 获取并判断需要提交的最新 L2 的 start 和 end 块号，起始的区块为 L2 当前安全的最高块；结束区块为 L2 当前最高的不安全的区块。
  - 拿到提交的开始块和结束区块之后，从起始区块开始获取区块信息，并调用 AddL2Block 函数将区块加到 channelManager 的 blocks []*types.Block 中。
  - 调用 loadBlockIntoState 函数返回的区块，校验区块是否需要重新提交，若需要，将 l.lastStoredBlock  置成 eth.BlockID{}；否则就将 l.lastStoredBlock 置成 eth.ToBlockID(block)；latestBlock  置成 block；
  - L2BlockToBlockRef 从 L2 块引用源中提取基本的 L2BlockRef 信息，根据区块号判断必要时回退到创世信息
- publishStateToL1 方法会执行一下逻辑
  - publishTxToL1 方法获取要提交的数据数据构建交易发送到 Layer1 网络，并将发送出去的交易扔到 receiptCh chan TxReceipt[T] channel 里面。
    - l1Tip：获取当前 L1 提示作为 L1BlockRef。 假定传递的上下文是生命周期上下文，因此它在内部使用网络超时进行包装。
    - recordL1Tip：将上一个 L1BlockRef 更换成 l1Tip 获取到的最新的 L1BlockRef
    - TxData：收集需要 rollup 的交易数据；TxData 返回应提交给 L1 的下一个 tx 数据。 目前，每个事务仅使用一帧。 如果待处理的通道已满，则仅返回该通道的剩余帧，直到成功完全发送到 L1。 如果没有挂起的帧，它将返回 io.EOF。
      - nextTxData: 如果有待处理帧或通道管理器关闭，从 channel 里面收集提交的数据并返回
        - NextTxData：从 channel 里面读取交易数据，将交易数据记录到 pendingTransactions 并返回
          - NextFrame：返回下一个可用帧。 在检查是否有下一帧可用之前，必须调用 HasFrame。 如果在没有下一帧时调用，则会发生恐慌。
      - processBlocks: 将块从块队列添加到待处理通道，直到队列耗尽或通道已满。
        - AddBlock: 将块加入到 channelBuilder 里面
          - AddBlock：向通道压缩管道添加一个块。 之后应该调用 IsFull 来测试通道是否已满。 如果已满，则必须启动新通道。如果调用 AddBlock，即使通道已满，也会返回 ChannelFullError。 AddBlock 还返回从区块的第一个交易中提取的 L1BlockInfo，以供调用者后续使用。 之后调用 OutputFrames() 创建帧。
            - BlockToBatch：将块转换为可以轻松进行 RLP 编码的批处理对象
            - AddBatch：AddBatch 将批次添加到通道。 如果添加批次出现问题，它会返回 RLP 编码的字节大小和错误。 它返回的唯一标记错误是 ErrTooManyRLPBytes。 如果返回此错误，则应关闭该通道并创建一个新通道。如果您需要在将块添加到通道之前访问 BatchData，则 AddBatch 应与 BlockToBatch 一起使用。 无法使用 AddBlock 访问批次数据。编码完成的数据压缩之后写到了 ChannelOut 里面
      - registerL1Block: 在挂起的通道上注册给定的块，仅在处理完所有待处理块后才注册当前 L1 头。 即使现在会触发超时，最好还是让所有待处理的区块都包含在这个通道中提交。
      - outputFrames:  从 currentChannel 读取 frames
        - OutputFrames: 从 channelBuilder 读取 frames
          - OutputFrames: 使用通道输出创建新帧。 它应该在 AddBlock 之后、使用 HasFrame 和 NextFrame 迭代可用帧之前调用。 如果通道尚未满，它将保守地仅从压缩输出中提取可用的帧。 如果已满，则通道将关闭，并且将创建所有剩余的帧，可能还有一个小的剩余帧。
            - outputReadyFrames: 只要通道输出压缩管道中有足够的数据，outputReadyFrames 就会创建新帧。 这是优化的一部分，已经生成帧并将它们作为交易发送出去，同时仍在通道构建器中收集块。
              - outputFrame:  创建一个新帧并将其添加到帧队列中。 请注意，压缩的输出数据必须在底层 ChannelOut 上可用，否则将生成空帧。这里其实就是将 frame 数据添加到 frames []frameData 结构中，以供 nextTxData 读取对应的 frame 数据。
    - sendTransaction 将交易发送到一层，并把交易发送状态更新到 receiptCh chan TxReceipt[T] channel 里面；sendTransaction 使用给定的“数据”创建交易并将其提交到批处理收件箱地址。 它目前使用底层的“txmgr”来处理交易发送和价格管理。 这是一种阻塞方法。 不应同时调用它。
      - Send： 发送将等待，直到挂起的交易数量低于最大挂起数量，然后发送下一个交易。 实际的 tx 发送是非阻塞的，收据在提供的收据通道上返回。 如果通道未缓冲，则 goroutine 将被阻止完成，直到从通道读取数据为止。
      - sendTx：调用 txMgr send 方法发送交易，将交易的状态写到  receiptCh chan TxReceipt[T] channel 里面
  - handleReceipt 获取从 channel 处理交易的状态，并将成功处理的交易从 channel 里面移除
- handleReceipt 获取从 channel 处理交易的状态，并将成功处理的交易从 channel 里面移除


## 4.op-proposer
### 4.1 执行流程图

[![op-batcher](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/rollup/op-proposer/op-proposer.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

### 4.2.详细执行流程
- FetchNextOutputInfo: 获取 L2 上的区块的 output，方便后续组装提交，output 结构如下
```
type OutputResponse struct {
    Version               Bytes32     `json:"version"`
    OutputRoot            Bytes32     `json:"outputRoot"`
    BlockRef              L2BlockRef  `json:"blockRef"`
    WithdrawalStorageRoot common.Hash `json:"withdrawalStorageRoot"`
    StateRoot             common.Hash `json:"stateRoot"`
    Status                *SyncStatus `json:"syncStatus"`
}
```

  - NextBlockNumber：获取下一批次需要提交的区块区间，区间计算为 latestBlockNumber() + SUBMISSION_INTERVAL SUBMISSION_INTERVAL 的值可以在部署L2OutputOracle 合约的时候指定。
  - SyncStatus：获取 L2 块的 SafeL2 和 FinalizedL2 的状态和块信息，
  - fetchOutput：上面检查完 nextCheckpointBlock 符合规则之后，去 L2 上获取需要提交的 stateRoot
    - OutputAtBlock: 根据块高获取 output, 里面包含 stateRoot，这里最终是调用 eth_getProof 去计算并获取 stateRoot，代码调用流程可以参考上图。
  提示: 这里并不是一个块提交一次 stateRoot, 而是根据 SUBMISSION_INTERVAL 配置的值来计算一批块的 stateRoot，最终将 stateRoot 提交到 L2OutputOracle 合约
- sendTransaction：使用 output 构建 stateRoot 提交交易，将交易提交到一层链， 下面是交易打包的数据细节
  
```
return abi.Pack(
    "proposeL2Output",
    output.OutputRoot,
    new(big.Int).SetUint64(output.BlockRef.Number),
    output.Status.CurrentL1.Hash,
    new(big.Int).SetUint64(output.Status.CurrentL1.Number))
```

## 5.小结

本节我们主要是讲解了 rollup 交易数据和 stateRoot 的详细流程，通过本节的学习，相信大家对 stateRoot 的提交流程会更了解，下一讲我们将讲解 op 新增 optimism_xxx 这组接口的作用与代码。
