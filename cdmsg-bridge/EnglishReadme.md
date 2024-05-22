[![Chinese](https://img.shields.io/badge/Chinese-README-blue)](readme.md)


# Deeply understand the Op-stack cross-chain calling process and the deposit and withdrawal of ETH and ERC20

# 1. Messenger Contract
The main function of the messenger contract is cross-chain communication, and the core methods are sendMessage and relayMessage;
- sendMessage: Send packaged messages from the source chain to the target chain, maintain the incremented msgNonce, and ensure the uniqueness and security of cross-chain messages.
- relayMessage: Construct a MsgHash comparison of the message sent from the source chain on the target chain to ensure the consistency of the message and then schedule and execute the message on the current chain.
In op-stack, there are respectively in L1 and L2 layers
- L1CrossDomainMessenger
- L2CrossDomainMessenger
Inherited from the CrossDomainMessenger contract to carry cross-chain communication. How the cross-chain message contract is called will be explained in detail in the recharge and withdrawal chapters.

# 2. OptimismPortal
The OptimismPortal contract is the deposit and withdrawal contract of op-stack.

## 2.1 Recharge
The recharge transaction will call depositTransaction; depositTransaction will throw the TransactionDeposited event, which contains the following information:
emit TransactionDeposited(from, _to, DEPOSIT_VERSION, opaqueData);
After the op-node monitors the contract event, it will execute the recharge transaction on the second layer. We will talk about the detailed introduction of the op-node later.

## 2.2 Withdrawal

For cash withdrawals, OptimismPortal will do two things:
- The first is the transaction proof proveWithdrawalTransaction, which updates the transaction proven to pass to
mapping(bytes32 => ProvenWithdrawal) public provenWithdrawals
As long as you enter it, the transaction means that it has been proven.

- The other is the operation of relay transaction. The finalizeWithdrawalTransaction function will put the relay transaction into
mapping(bytes32 => bool) public finalizedWithdrawals
As long as the transaction enters the finalizedWithdrawals Map, it is proven that it has been relayed.

# 3. Bridge Contract
In this section, we will draw a detailed flowchart of the recharge and withdrawal on the entire bridge.

# 4. L1->L2 recharge

## 4.1 ETH Deposit

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/ethdeposit.png)](https://github.com/guoshijiang/how-dose-op-stack-work)


-User
   - The user calls depositETH to recharge his own address, or calls depositETHTo to recharge the specified to address;
- L1StandardBridge
   - Whether you call depositETH or depositETHTo, you will eventually enter a function _initiateETHDeposit; no operation is done in _initiateETHDeposit, and _initiateBridgeETH is called directly
   - In the _initiateBridgeETH function, there will be the following operations
     - Determine whether msg.sender has enough funds for recharge
     - Throws an _emitETHBridgeInitiated event
     - Call the sendMessage method of the CrossDomainMessenger contract
- CrossDomainMessenger
   - In CrossDomainMessenger, the _sendMessage function of L1CrossDomainMessenger will be called. After successful execution, msgNonce will be added.
- L1CrossDomainMessenger
   - The _sendMessage of the L1CrossDomainMessenger contract will call the depositTransaction function of OptimismPortal
   - depositTransaction will throw TransactionDeposited event
- Op-node and op-geth
   - Op-node listens to TransactionDeposited, constructs the Attrs of the recharge transaction, and points op-geth to execute the recharge transaction in L2
   - Op-geth will call finalizeDeposit of the L2StandardBridge contract
- L2StandardBridge
   - The finalizeBridgeETH function will execute SafeCall.call, causing the position and the recharge process to be completed.

Note: For ETH recharge, since there is a receive method in OptimismPortal and L1StandardBridge

-OptimismPortal
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

Therefore, directly transferring ETH to OptimismPortal and L1StandardBridge is also an ETH recharge transaction.


## 4.2 ERC20 Deposit

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/erc20deposit.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

-User
   - The user calls depositERC20 to recharge his own address, or calls depositERC20To to recharge the specified to address;
- L1StandardBridge
   - Whether you call depositERC20 or depositERC20To, you will eventually enter a function _initiateBridgeERC20;
   - Determine whether it is OptimismMintableERC20. If so, call the burn method to destroy the token; if not, transfer the token to the bridge and record the funds into the ledger; the ledger code is as follows:
   ```
   mapping(address => mapping(address => uint256)) public deposits;
   deposits[_localToken][_remoteToken] = deposits[_localToken][_remoteToken] + _amount;
   ```
   - Next, call sendMessage of CrossDomainMessenger to send a cross-chain message. After successfully calling the _sendMessage method of L1CrossDomainMessenger, the nonce will be automatically incremented by 1.
- L1CrossDomainMessenger
   - _sendMessage will call depositTransaction of OptimismPortal,
   - depositTransaction will throw TransactionDeposited event
- Op-node and op-geth
   - Op-node listens to TransactionDeposited, constructs the Attrs of the recharge transaction, and points op-geth to execute the recharge transaction in L2
   - Op-geth will call finalizeDeposit of the L2StandardBridge contract
- L2StandardBridge
   - The finalizeBridgeERC20 function will perform mint Token or token Transfer operations.
   - If it is OptimismMintableERC20, perform mint operation
   - If it is not OptimismMintableERC20, update the ledger with the following code:

   ```
   mapping(address => mapping(address => uint256)) public deposits;
   deposits[_localToken][_remoteToken] = deposits[_localToken][_remoteToken] - _amount;
   ```

## 4.3 ERC721 Deposit

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/erc721deposit.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

Regarding the recharge of ERC721, the current logic is fragmented and not completely connected, so I won’t go into details here.

# 5. L2->L1 withdrawal

## 5.1 ETH Withdrawal

### 5.1.1 ETH withdrawal transaction to L1

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/ethwithdraw.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

-User
   - The user calls withdraw to withdraw coins to his own address, or calls withdrawTo to withdraw money to the specified to address;
- L2StandardBridge
   - Both withdraw and withdrawTo will call _initiateWithdrawal; this method will determine whether it is an ETH withdrawal or an ERC20 withdrawal.
- CrossDomainMessenger & L2CrossDomainMessenger
   - For ETH withdrawal, enter the _initiateBridgeETH function, which will call the sendMessage method of the CrossDomainMessenger contract. After the logic in the sendMessage method is successful, msgNonce will be incremented by 1, and then the _sendMessage method of the L2CrossDomainMessenger will be called.
-L2ToL1MessagePasser
   - After calling the _sendMessage of the L2CrossDomainMessenger contract to the initiateWithdrawal of the L2ToL1MessagePasser, the withdrawal transaction data will be constructed to calculate the withdrawalHash, and the Hash will be set to true in the map data structure.
   - msgNonce of maintained L2 is incremented by 1
- Op-node & Op-geth
   - The second layer generates withdrawal transactions
- Op-batcher & Op-proposer, the detailed process of Op-batcher and Op-proposer will be introduced later, here is a simple overview
   - Op-batch rolls up the withdrawal transaction data to L1
   - Op-proposer submits the stateroot of a batch of data related to withdrawal transactions to L1

### 5.1.2 ETH Cliam transaction process

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/ethcliam.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

- The user calls the SDK to obtain the status of the withdrawal transaction. If the status is READY_TO_PROVE, it means that the transaction can be proved; the user can call the proveMessage of the SDK to prove the transaction, or directly call the proveWithdrawalTransaction method of the OptimismPortal contract to prove the transaction. When the proof transaction is generated , when the transaction passes the challenge period, the transaction status will change to READY_FOR_RELAY; at this time, the user can call the finalizeMessage method of the SDK to claim funds, or directly call the finalizeWithdrawalTransaction method of the OptimismPortal contract to claim funds.
- Directly calling the contract requires various combination calls, which is more troublesome. It is recommended to use SDK for transaction proof and cliam.
- The picture above is the complete call flow chart using SDK cliam

## 5.2 ERC20 Withdrawal

### 5.2.1 ERC20 withdrawal transaction to L1

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/erc20withdraw.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

- The difference between the above process and ETH withdrawal is that
   - If it is OptimismMintableERC20 contract Token burn
   - Otherwise, add the corresponding amount in the deposits ledger and perform token safeTransfer

### 5.2.2 ERC20 Cliam transaction process

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/cdmsg-bridge/images/erc20cliam.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

- The difference between the above process and the ETH client is that finalize is changed from finalizeERC20Withdrawal to finalize and the logic in finalizeBridgeERC20 is also inconsistent.
   - finalizeBridgeERC20
     - If token of type OptimismMintableERC20, mint
     - Otherwise, subtract the corresponding amount from the deposits ledger and perform token safeTransfer.

# 6. Summary

This section mainly explains the cross-chain calling process of op-stack and the recharge process of ETH and ERC20 Token. In the next lecture, we will explain the rollup process of op-stack in detail.
