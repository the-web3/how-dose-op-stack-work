[![Chinese](https://img.shields.io/badge/Chinese-README-blue)](README.md)


# Optimistim op-node json rpc

## 1. Overview
- The default interface after starting the service baseUrl: http://127.0.0.1:7545
  
## 2. Optimism
### 2.1 Get op-stack version information
#### 2.1.1 optimism_version interface call
- Interface name: optimization_version
- Request method: post
- Request a demonstration:
```
{
     "jsonrpc":"2.0",
     "method":"optimism_syncStatus",
     "id":1
}
```
- Response demonstration:
```
{
     "jsonrpc": "2.0",
     "id": 1,
     "result": "v0.0.0-"
}
```
### 2.1.2 Detailed explanation of optimism_version code process

Return version directly after calling

```
func (n *nodeAPI) Version(ctx context.Context) (string, error) {
     recordDur := n.m.RecordRPCServerRequest("optimism_version")
     defer recordDur()
     return version.Version + "-" + version.Meta, nil
}
```
```
var (
     Version = "v0.10.14"
     Meta = "dev"
)
```

### 2.2 Obtain op-stack rollup configuration information
#### 2.2.1 optimism_rollupConfig interface call
- Interface name: optimization_rollupConfig
- Request method: post
- Request a demonstration:
```
{
     "jsonrpc":"2.0",
     "method":"optimism_rollupConfig",
     "id":1
}
```
- Response demonstration
```
{
     "jsonrpc": "2.0",
     "id": 1,
     "result": {
         "genesis": {
             "l1": {
                 "hash": "0xc73a8389a5aaa20922df51c2847cd9385b1b5e0ba679b775f8a2630cd015cd51",
                 "number": 0
             },
             "l2": {
                 "hash": "0xce549f8a96ab878b17b7f19f770844044d188e5bba6d2ea3a63bae3360745b9c",
                 "number": 0
             },
             "l2_time": 1695096865,
             "system_config": {
                 "batcherAddr": "0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc",
                 "overhead": "0x000000000000000000000000000000000000000000000000000000000000834",
                 "scalar": "0x0000000000000000000000000000000000000000000000000000000000f4240",
                 "gasLimit": 30000000
             }
         },
         "block_time": 2,
         "max_sequencer_drift": 300,
         "seq_window_size": 20000,
         "channel_timeout": 120,
         "l1_chain_id": 900,
         "l2_chain_id": 901,
         "regolith_time": 0,
         "batch_inbox_address": "0xff00000000000000000000000000000000000000",
         "deposit_contract_address": "0x6900000000000000000000000000000000000001",
         "l1_system_config_address": "0x6900000000000000000000000000000000000009"
     }
}
```
#### 2.2.2 Detailed explanation of optimism_rollupConfig code process

`RollupConfig(op-node/node/api.go)`->`n.config(op-node/node/api.go)`

-The return structure is as follows
```
type nodeAPI struct {
     config *rollup.Config
     client l2EthClient
     dr driverClient
     log log.Logger
     mrpcMetrics
}
```
```
type Config struct {
     // Genesis anchor point of the rollup
     Genesis Genesis `json:"genesis"`
     // Seconds per L2 block
     BlockTime uint64 `json:"block_time"`
     // Sequencer batches may not be more than MaxSequencerDrift seconds after
     // the L1 timestamp of the sequencing window end.
     //
     // Note: When L1 has many 1 second consecutive blocks, and L2 grows at fixed 2 seconds,
     // the L2 time may still grow beyond this difference.
     MaxSequencerDrift uint64 `json:"max_sequencer_drift"`
     // Number of epochs (L1 blocks) per sequencing window, including the epoch L1 origin block itself
     SeqWindowSize uint64 `json:"seq_window_size"`
     // Number of L1 blocks between when a channel can be opened and when it must be closed by.
     ChannelTimeout uint64 `json:"channel_timeout"`
     // Required to verify L1 signatures
     L1ChainID *big.Int `json:"l1_chain_id"`
     // Required to identify the L2 network and create p2p signatures unique for this chain.
     L2ChainID *big.Int `json:"l2_chain_id"`

     // RegolithTime sets the activation time of the Regolith network-upgrade:
     // a pre-mainnet Bedrock change that addresses findings of the Sherlock contest related to deposit attributes.
     // "Regolith" is the loose deposited rock that sits on top of Bedrock.
     // Active if RegolithTime != nil && L2 block timestamp >= *RegolithTime, inactive otherwise.
     RegolithTime *uint64 `json:"regolith_time,omitempty"`

     // Note: below addresses are part of the block-derivation process,
     // and required to be the same network-wide to stay in consensus.

     // L1 address that batches are sent to.
     BatchInboxAddress common.Address `json:"batch_inbox_address"`
     // L1 Deposit Contract Address
     DepositContractAddress common.Address `json:"deposit_contract_address"`
     // L1 System Config Address
     L1SystemConfigAddress common.Address `json:"l1_system_config_address"`
}
```

Note: op-batcher and op-preposer use the information here when they start up. Once the configuration changes, you need to restart op-node, and then restart op-batcher and op-preposer to take effect.

### 2.3 Synchronize op-stack Status information
#### 2.3.1 optimism_syncStatus interface call
- Interface name: optimization_syncStatus
- Request method: post
- Request a demonstration:

```
{
     "jsonrpc":"2.0",
     "method":"optimism_syncStatus",
     "id":1
}
```
- Response demonstration:

```
{
     "jsonrpc": "2.0",
     "id": 1,
     "result": {
         "current_l1": {
             "hash": "0x47db9d8d2a9d752dff26e645a8c42108cccc6b778df51253a5d3bad7cbe171b9",
             "number": 1622,
             "parentHash": "0x589f1d15637dc0f99266e1d3b2d4f21b4672c805c71fc1cae6320dd88c80fcf9",
             "timestamp": 1695098493
        },
        "current_l1_finalized": {
            "hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
            "number": 0,
            "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
            "timestamp": 0
        },
        "head_l1": {
            "hash": "0x5f02518f5fcf132eab01f55fee17353b2466eccf26dea5976b56219df9186203",
            "number": 7424,
            "parentHash": "0x966526a64be988d15a6df49acbfb2b879202616a7c102500bf081598eda9bc53",
            "timestamp": 1695109106
        },
        "safe_l1": {
            "hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
            "number": 0,
            "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
            "timestamp": 0
        },
        "finalized_l1": {
            "hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
            "number": 0,
            "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
            "timestamp": 0
        },
        "unsafe_l2": {
            "hash": "0x7c6be4624192eee80ede3f4b51206477e9a93b15cb289561744818e39037470a",
            "number": 6087,
            "parentHash": "0xce518c72d0d02d3e160ec26e698342ce7bf8e5bbf8de84f2e61d01a63b04080c",
            "timestamp": 1695109039,
            "l1origin": {
                "hash": "0xe11512ebdc88d836efd750a51ace6b404966af90c34ef19759aaa1bee700568a",
                "number": 5268
            },
            "sequenceNumber": 0
        },
        "safe_l2": {
            "hash": "0xaa8fb7e1b2d871d56e935eeaa78eb8039646fde50b500a3f315b14f900f8280b",
            "number": 807,
            "parentHash": "0xaa8b6ba2420fdb8568ce9eccf76e46dd45ab186e050be294051cbb25e553cc9c",
            "timestamp": 1695098479,
            "l1origin": {
                "hash": "0x9b1ca8527f379c863639eb524f048e6fdae259c932fee5021853af05c2288256",
                "number": 804
            },
            "sequenceNumber": 0
        },
        "finalized_l2": {
            "hash": "0xce549f8a96ab878b17b7f19f770844044d188e5bba6d2ea3a63bae3360745b9c",
            "number": 0,
            "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
            "timestamp": 1695096865,
            "l1origin": {
                "hash": "0xc73a8389a5aaa20922df51c2847cd9385b1b5e0ba679b775f8a2630cd015cd51",
                "number": 0
            },
            "sequenceNumber": 0
        },
        "queued_unsafe_l2": {
            "hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
            "number": 0,
            "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
            "timestamp": 0,
            "l1origin": {
                "hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                "number": 0
            },
            "sequenceNumber": 0
        }
    }
}
```

#### 2.3.2 Detailed explanation of optimism_syncStatus code flow
`SyncStatus(op-node/node/api.go)`-> `SyncStatus(op-node/rollup/driver/state.go)`->`syncStatus(op-node/rollup/driver/state.go)`

The code return structure is as follows:

```
return &eth.SyncStatus{
    CurrentL1:          s.derivation.Origin(),
    CurrentL1Finalized: s.derivation.FinalizedL1(),
    HeadL1:             s.l1State.L1Head(),
    SafeL1:             s.l1State.L1Safe(),
    FinalizedL1:        s.l1State.L1Finalized(),
    UnsafeL2:           s.derivation.UnsafeL2Head(),
    SafeL2:             s.derivation.SafeL2Head(),
    FinalizedL2:        s.derivation.Finalized(),
    UnsafeL2SyncTarget: s.derivation.UnsafeL2SyncTarget(),
}
```
We can see that the return value contains the current unsafe, safe and finalized block information of L1 and L2. This interface is also used in the rollup service to obtain the currently submitted blocks (safe status) and L2 Block information that has been generated but not submitted (unSafe status).

### 2.4 Get op-stack state root output information
#### 2.4.1 optimism_outputAtBlock interface call
- Interface name: optimization_outputAtBlock
- Request method: post
- Request a demonstration:
```
{
    "jsonrpc":"2.0",
    "method":"optimism_outputAtBlock",
    "params":["0xa"],
    "id":1
}
```
- Response demonstration:
  
```
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "version": "0x0000000000000000000000000000000000000000000000000000000000000000",
        "outputRoot": "0xa320665d67f71b761f7e65ef065a71e7b2015772f24ac959107b31b540b5c79e",
        "blockRef": {
            "hash": "0x364cd42fe675acc76f5acee58ec3d05422036078025ae7edf387736fa2fc4c77",
            "number": 10,
            "parentHash": "0x5dad8149226b0468e6a9f4e676e448e9f0f0cf7ed904f18456635d8dda5ba126",
            "timestamp": 1695096885,
            "l1origin": {
                "hash": "0xff0ea2db97724942684bcac3323a7f816a46b36ef81d912308a11362ef1c65d5",
                "number": 7
            },
            "sequenceNumber": 0
        },
        "withdrawalStorageRoot": "0x8ed4baae3a927be3dea54996b4d5899f8c01e7594bf50b17dc1e741388ce3d12",
        "stateRoot": "0x5c0b4a5eca962019c26d35de246af89e6f4c5ae84f1e1ea7322b378d07356c52",
        "syncStatus": {
            "current_l1": {
                "hash": "0xbd057f40bc82e5a36c488034921aefe69cf99ee2277b57cdc4bb0c7fe719e010",
                "number": 18194,
                "parentHash": "0x80cb8a14f8616f9300a3ef3e6a9b330693106d29e0289928f47d69b8a29de583",
                "timestamp": 1695120804
            },
            "current_l1_finalized": {
                "hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                "number": 0,
                "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                "timestamp": 0
            },
            "head_l1": {
                "hash": "0xe2dd477086d7abac9c3a0a4cb89c14935111ae6fd244a6fabe1a343bd8fb0b4a",
                "number": 18206,
                "parentHash": "0x1d54677e3ce73e4fdba1197479bcd6ac325b91bbbc1dec16a868f3d86564048e",
                "timestamp": 1695120816
            },
            "safe_l1": {
                "hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                "number": 0,
                "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                "timestamp": 0
            },
            "finalized_l1": {
                "hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                "number": 0,
                "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                "timestamp": 0
            },
            "unsafe_l2": {
                "hash": "0x168541a09e0c3cd04b5fef3e5857461fe5d4b861c7266a9b8a560f2ff6380d7c",
                "number": 11975,
                "parentHash": "0xf5cb4a16f0dc3c60b4f79fe03e2c6be71db708fc3a1053cbf66c981389a8e12c",
                "timestamp": 1695120815,
                "l1origin": {
                    "hash": "0x6c6d3ea564dc2ac69b506da8f11349347a897431ae52d43f4dd65c0912dc6455",
                    "number": 11156
                },
                "sequenceNumber": 0
            },
            "safe_l2": {
                "hash": "0xb7091f5916ada4302b3c3bbdc2df423ef26a134c54d0e96a22e7e79fe15dab20",
                "number": 10414,
                "parentHash": "0x08b2c1921606afba1e7d1b35f75638018947e9f5caa5f513c076c78be02fb7e1",
                "timestamp": 1695117693,
                "l1origin": {
                    "hash": "0xd27bee73396cdf7dd5d107b9833bd4986943b1604e978e866022fd7ebb285431",
                    "number": 9595
                },
                "sequenceNumber": 0
            },
            "finalized_l2": {
                "hash": "0xce549f8a96ab878b17b7f19f770844044d188e5bba6d2ea3a63bae3360745b9c",
                "number": 0,
                "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                "timestamp": 1695096865,
                "l1origin": {
                    "hash": "0xc73a8389a5aaa20922df51c2847cd9385b1b5e0ba679b775f8a2630cd015cd51",
                    "number": 0
                },
                "sequenceNumber": 0
            },
            "queued_unsafe_l2": {
                "hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                "number": 0,
                "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                "timestamp": 0,
                "l1origin": {
                    "hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
                    "number": 0
                },
                "sequenceNumber": 0
            }
        }
    }
}
```

#### 2.4.2 optimism_outputAtBlock 代码流程详解

[![ethdeposit](https://github.com/guoshijiang/how-dose-op-stack-work/blob/main/optimistim-api/op-node-api1.png)](https://github.com/guoshijiang/how-dose-op-stack-work)

Op-stack wrote the following code in the GetProof method of op-geth

```
if s.b.ChainConfig().IsOptimismPreBedrock(header.Number) {
    if s.b.HistoricalRPCService() != nil {
       var res AccountResult
       err := s.b.HistoricalRPCService().CallContext(ctx, &res, "eth_getProof", address, storageKeys, blockNrOrHash)
       if err != nil {
          return nil, fmt.Errorf("historical backend error: %w", err)
       }
       return &res, nil
    } else {
       return nil, rpc.ErrNoHistoricalFallback
    }
}
```

Here, the passed address will eventually be returned to the Proof of the account of the corresponding block. The code is as follows:

```
// Result structs for GetProof
type AccountResult struct {
    Address      common.Address  `json:"address"`
    AccountProof []string        `json:"accountProof"`
    Balance      *hexutil.Big    `json:"balance"`
    CodeHash     common.Hash     `json:"codeHash"`
    Nonce        hexutil.Uint64  `json:"nonce"`
    StorageHash  common.Hash     `json:"storageHash"`
    StorageProof []StorageResult `json:"storageProof"`
}
```

When we trace the code process, we can find that the address passed here is the address of the L2ToL1MessagePasserAddr pre-deployment contract, which means that the final generated stateroot is the state root based on an Account proof of L2ToL1MessagePasserAddr.

op-proposer Fetching the submitted stateroot is implemented through this interface.

## 3.Admin
### 3.1 Start seqeuencer
#### 3.1.1 admin_startSequencer interface call
- Interface name: admin_startSequencer
- Request method: post
- Request a demonstration:
```
```
- Response demonstration:
```
```

#### 3.1.2 Detailed explanation of admin_startSequencer code process

### 3.2 Stop seqeuencer
#### 3.2.1 admin_stopSequencer interface call
- Interface name: admin_stopSequencer
- Request method: post
- Request a demonstration:
```
```

- Response demonstration:
```
```

#### 3.2.2 Detailed explanation of admin_stopSequencer code flow

### 3.3 Reset the derivation process
#### 3.3.1 admin_resetDerivationPipeline interface call
- Interface name: admin_resetDerivationPipeline
- Request method: post
- Request a demonstration:
```
```

- Response demonstration:
```
```

#### 3.3.2 Detailed explanation of admin_resetDerivationPipeline code process