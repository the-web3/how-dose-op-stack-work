[![Chinese](https://img.shields.io/badge/Chinese-README-blue)](README.md)


# Detailed explanation of Op-stack rollup process

## 1. Overview

The rollup of Op-stack is undertaken by two services
- op-batcher service: its main responsibility is to submit transaction data to the EOA address of Layer1
- op-proposer service: its main responsibility is to submit the transaction status to the L2OutputOracle contract of Layer1

## 2.Rollup architecture

[![rollup_art](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/rollup/rollup_art.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

## 3.op-batcher

### 3.1 Execution flow chart

[![op-batcher](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/rollup/op-batcher/op-batcher.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

### 3.2 Detailed execution process
- loadBlocksIntoState will execute the following logic
   - calculateL2BlockRangeToStore obtains and determines the start and end block numbers of the latest L2 that need to be submitted. The starting block is the highest current safe block of L2; the end block is the highest current unsafe block of L2.
   - After getting the submitted start block and end block, get the block information starting from the start block, and call the AddL2Block function to add the block to blocks []*types.Block of channelManager.
   - Call the block returned by the loadBlockIntoState function to verify whether the block needs to be resubmitted. If necessary, set l.lastStoredBlock to eth.BlockID{}; otherwise, set l.lastStoredBlock to eth.ToBlockID(block); set latestBlock into block;
   - L2BlockToBlockRef extracts basic L2BlockRef information from the L2 block reference source, and falls back to the creation information if necessary based on the block number.
- The publishStateToL1 method will perform some logic
   - The publishTxToL1 method obtains the data to be submitted, constructs a transaction and sends it to the Layer1 network, and throws the sent transaction into the receiptCh chan TxReceipt[T] channel.
     - l1Tip: Get the current L1 tip as L1BlockRef. The passed context is assumed to be a lifecycle context, so it is wrapped internally with a network timeout.
     - recordL1Tip: Replace the previous L1BlockRef with the latest L1BlockRef obtained by l1Tip
     - TxData: Collect transaction data that requires rollup; TxData returns the next tx data that should be submitted to L1. Currently, only one frame is used per transaction. If a pending channel is full, only the remaining frames for that channel are returned until successfully fully transmitted to L1. If there are no pending frames, it returns io.EOF.
       - nextTxData: If there are pending frames or the channel manager is closed, collect the submitted data from the channel and return it
         - NextTxData: Read transaction data from channel, record transaction data to pendingTransactions and return
           - NextFrame: Returns the next available frame. HasFrame must be called before checking if the next frame is available. If called when there is no next frame, a panic will occur.
       - processBlocks: Add blocks from the block queue to the pending channel until the queue is exhausted or the channel is full.
         - AddBlock: Add the block to channelBuilder
           - AddBlock: Adds a block to the channel compression pipeline. IsFull should then be called to test whether the channel is full. If it is full, a new channel must be started. If AddBlock is called, a ChannelFullError is returned even if the channel is full. AddBlock also returns the L1BlockInfo extracted from the first transaction of the block for subsequent use by the caller. Then call OutputFrames() to create the frame.
             - BlockToBatch: Convert blocks into batch objects that can be easily RLP encoded
             - AddBatch: AddBatch adds a batch to a channel. If there is a problem adding the batch, it returns the RLP encoded byte size and an error. The only tagged error it returns is ErrTooManyRLPBytes. If this error is returned, the channel should be closed and a new one created. AddBatch should be used with BlockToBatch if you need to access the BatchData before adding the block to the channel. Unable to access batch data using AddBlock. The encoded data is compressed and written to ChannelOut.
       - registerL1Block: Register the given block on the pending channel, registering the current L1 header only after all pending blocks have been processed. Even though a timeout will now be triggered, it is better to have all pending blocks submitted in this channel.
       - outputFrames: read frames from currentChannel
         - OutputFrames: read frames from channelBuilder
           - OutputFrames: Create new frames using channel outputs. It should be called after AddBlock and before iterating through the available frames using HasFrame and NextFrame. If the channel is not already full, it will conservatively only extract available frames from the compressed output. If it is full, the channel will be closed and all remaining frames will be created, possibly with a small remaining frame.
             - outputReadyFrames: outputReadyFrames creates new frames as long as there is enough data in the channel output compression pipeline. This is part of the optimization, already generating frames and sending them out as transactions while still collecting blocks in the channel builder.
               - outputFrame: Creates a new frame and adds it to the frame queue. Note that compressed output data must be available on the underlying ChannelOut, otherwise empty frames will be generated. This is actually adding the frame data to the frames []frameData structure so that nextTxData can read the corresponding frame data.
     - sendTransaction sends the transaction to one layer and updates the transaction sending status into the receiptCh chan TxReceipt[T] channel; sendTransaction creates a transaction using the given "data" and submits it to the batch inbox address. It currently uses the underlying "txmgr" to handle transaction routing and price management. This is a blocking method. It should not be called simultaneously.
       - Send: Send will wait until the number of pending transactions is below the maximum number of pending transactions, then send the next transaction. The actual tx send is non-blocking and the receipt is returned on the provided receipt channel. If the channel is unbuffered, the goroutine will be blocked from completing until data is read from the channel.
       - sendTx: Call the txMgr send method to send the transaction and write the transaction status to the receiptCh chan TxReceipt[T] channel
   - handleReceipt gets the status of transactions processed from the channel and removes successfully processed transactions from the channel
- handleReceipt gets the status of transactions processed from the channel and removes successfully processed transactions from the channel


## 4.op-proposer
### 4.1 Execution flow chart

[![op-batcher](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/rollup/op-proposer/op-proposer.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

### 4.2. Detailed execution process
- FetchNextOutputInfo: Get the output of the block on L2 to facilitate subsequent assembly and submission. The output structure is as follows
```
typeOutputResponse struct {
     Version Bytes32 `json:"version"`
     OutputRoot Bytes32 `json:"outputRoot"`
     BlockRef L2BlockRef `json:"blockRef"`
     WithdrawalStorageRoot common.Hash `json:"withdrawalStorageRoot"`
     StateRoot common.Hash `json:"stateRoot"`
     Status *SyncStatus `json:"syncStatus"`
}
```

   - NextBlockNumber: Get the block interval that needs to be submitted in the next batch. The interval is calculated as latestBlockNumber() + SUBMISSION_INTERVAL. The value of SUBMISSION_INTERVAL can be specified when deploying the L2OutputOracle contract.
   - SyncStatus: Get the status and block information of SafeL2 and FinalizedL2 of L2 block,
   - fetchOutput: After checking that nextCheckpointBlock complies with the rules, go to L2 to get the stateRoot that needs to be submitted.
     - OutputAtBlock: Get the output based on the block height, which contains stateRoot. Here, eth_getProof is finally called to calculate and obtain stateRoot. The code calling process can refer to the figure above.
   Tip: This is not to submit stateRoot once for each block, but to calculate the stateRoot of a batch of blocks based on the value configured in SUBMISSION_INTERVAL, and finally submit the stateRoot to the L2OutputOracle contract
- sendTransaction: Use output to construct stateRoot to submit the transaction and submit the transaction to a layer of chain. The following are the data details of the transaction packaging.
  
```
return abi.Pack(
     "proposeL2Output",
     output.OutputRoot,
     new(big.Int).SetUint64(output.BlockRef.Number),
     output.Status.CurrentL1.Hash,
     new(big.Int).SetUint64(output.Status.CurrentL1.Number))
```

## 5. Summary

In this section, we mainly explain the detailed process of rollup transaction data and stateRoot. After studying this section, I believe everyone will have a better understanding of the stateRoot submission process. In the next lecture, we will explain the function and code of the new optimization_xxx interface in the op. .