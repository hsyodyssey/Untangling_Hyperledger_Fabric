# Untangling_Hyperledger_Fabric

In this repo, we investigate some essential details that are easily overlooked in Hyperledger Fabric (v2.3).

## Core Component in Hyperledger Fabric

### Read-Write Set

在"fabric/vendor/github.com/hyperledger/fabric-protos-go/ledger/rwset/kvrwset/kv_rwset.pb.go" 文件中，我们可以找到关于Read/Write set 以及Read Set中关于Version的定义。

```Golang
// Version encapsulates the version of a Key
// A version of a committed key is maintained as the height of the transaction that committed the key.
// The height is represenetd as a tuple <blockNum, txNum> where the txNum is the position of the transaction
// (starting with 0) within block
type Version struct {
 BlockNum             uint64   `protobuf:"varint,1,opt,name=block_num,json=blockNum,proto3" json:"block_num,omitempty"`
 TxNum                uint64   `protobuf:"varint,2,opt,name=tx_num,json=txNum,proto3" json:"tx_num,omitempty"`
 XXX_NoUnkeyedLiteral struct{} `json:"-"`
 XXX_unrecognized     []byte   `json:"-"`
 XXX_sizecache        int32    `json:"-"`
}
```

Hyperledger Fabric中关于Key的Version由两部分组成，BlockNum以及txNum。BlockNum表示transaction被打包进的区块的高度，txNum用于表示交易在区块中的位置(从0号位置开始统计)。

与WorldState相关的代码中，version被定义在“fabric/core/ledger/internal/version/version.go”文件的Height结构体中。

```Golang
// Height represents the height of a transaction in blockchain
type Height struct {
 BlockNum uint64
 TxNum    uint64
}

// NewHeight constructs a new instance of Height
func NewHeight(blockNum, txNum uint64) *Height {
 return &Height{blockNum, txNum}
}
```

### Transaction Validation

在Hyperledger 中Endearment node会并发的模拟执行Transaction。这种方式可以增加系统的Throughput，但是潜在的会带来读写冲突的问题。Hyperledger Fabric 基于MVCC的思想来解决这个问题。

Hyperledger Fabric是一种EOV结构的Blockchain。在validation阶段，Validator通过判断Transaction的Read-Write Set来判断Transaction是否invalid。

具体的是，Validator会循环遍历Transaction的Read Set中所有的Key以及其的Version，并与该Key在当前World StateDB中的Version进行对比。如果两者不相同，说明出现了读写冲突，该Transaction会被标记为Invalid，其Write set的结果最终不会被写入到World StateDB中。

```Golang
// 循环遍历kvrwse数组中所有的kvRead
func (v *validator) validateReadSet(ns string, kvReads []*kvrwset.KVRead, updates *privacyenabledstate.PubUpdateBatch) (bool, error) {
 for _, kvRead := range kvReads {
  if valid, err := v.validateKVRead(ns, kvRead, updates); !valid || err != nil {
   return valid, err
  }
 }
 return true, nil
}
```

Validator的代码位于，"fabric/core/ledger/kvledger/txmgmt/validation/validator.go"中。

```Golang
// validateKVRead performs mvcc check for a key read during transaction simulation.
// i.e., it checks whether a key/version combination is already updated in the statedb (by an already committed block)
// or in the updates (by a preceding valid transaction in the current block)
func (v *validator) validateKVRead(ns string, kvRead *kvrwset.KVRead, updates *privacyenabledstate.PubUpdateBatch) (bool, error) {
 readVersion := rwsetutil.NewVersion(kvRead.Version)
 if updates.Exists(ns, kvRead.Key) {
  logger.Warnw("Transaction invalidation due to version mismatch, key in readset has been updated in a prior transaction in this block",
   "namespace", ns, "key", kvRead.Key, "readVersion", readVersion)
  return false, nil
 }
 committedVersion, err := v.db.GetVersion(ns, kvRead.Key)
 if err != nil {
  return false, err
 }

 logger.Debugw("Comparing readset version to committed version",
  "namespace", ns, "key", kvRead.Key, "readVersion", readVersion, "committedVersion", committedVersion)

 if !version.AreSame(committedVersion, readVersion) {
  logger.Warnw("Transaction invalidation due to version mismatch, readset version does not match committed version",
   "namespace", ns, "key", kvRead.Key, "readVersion", readVersion, "committedVersion", committedVersion)
  return false, nil
 }
 return true, nil
}
```

### Transaction Endorsement

在Hyperledger Fabric中，endorsement policy是在Chaincode的初始化的时候定义的。默认值设置为需要当前Channel中多数的Organization的同意。

在以太坊中，系统通过计算1.Gas Limit 2.EVM调用栈的深度来解决停机问题。

在Hyperledger Fabric中，Endorsement Peers是网络中唯一执行合约的节点。为了解决chaincode中的停机问题，Peer通过设置CORE_EXECUTECHAINCODE_TIMEOUT 来设置transaction可以执行的时间上限。一旦超过了这个上线，transaction 将会因为Execute timeout而失败。

```Golang
const (
 defaultExecutionTimeout = 30 * time.Second
 minimumStartupTimeout   = 5 * time.Second
)

type Config struct {
 TotalQueryLimit int
 TLSEnabled      bool
 Keepalive       time.Duration
 ExecuteTimeout  time.Duration
 InstallTimeout  time.Duration
 StartupTimeout  time.Duration
 LogFormat       string
 LogLevel        string
 ShimLogLevel    string
 SCCAllowlist    map[string]bool
}
```

### Smart Contract and Chaincode

按照Hyperledger 官方文档的解释，一个或多个多个Smart Contract构成了一个Chaincode。Hyperledger中的Smart Contract使用基于通用编程语言开发，包括Go，Java和JavaScript。使用通用编程器语言增加了合约开发的灵活度，也带来了更多的不确定性。
为了约束合约的开发，减少合约代码中的不确定性，开发人员在编写Smart Contract时需要继承官方Fabric SDK中的接口函数，合约函数需要调用Fabric SDK官方提供SDK才能与链上数据进行交互。具体的可以阅读官方文档中Transaction Context一章。

For Example, Fabric SDK中关于World State的API:

- <async> getState(key) [Retrieves the current value of the state variable key]
- <async> putState(key, value) [Writes the state variable key of value value to the state store. If the variable already exists, the value will be overwritten.
- <async> deleteState(key)
- <async> getStateByRange(startKey, endKey) [Returns a range iterator over a set of keys in the ledger. The iterator can be used to iterate over all keys between the startKey (inclusive) and endKey (exclusive). The query is re-executed during validation phase to ensure result set has not changed since transaction endorsement (phantom reads detected).]
- getStateByRangeWithPagination(startKey, endKey, pageSize, bookmark)
