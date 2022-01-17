# Untangling_Hyperledger_Fabric

In this repo, we investigate some essential details that are easily overlooked in Hyperledger Fabric (v2.3).

## Core Component in Hyperledger Fabric

### Read-Write Set

在"fabric/vendor/github.com/hyperledger/fabric-protos-go/ledger/rwset/kvrwset/kv_rwset.pb.go" 文件中，我们可以找到关于Read/Write set 以及Read Set中关于Version的定义。

```go
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

与World State相关的代码中，version被定义在“fabric/core/ledger/internal/version/version.go”文件的Height结构体中。

```go
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

在Hyperledger Fabric中Endearment node会并发的模拟执行Transaction。这种并发的方式可以增加系统的Throughput，但是会带来潜在的读写冲突的问题。在Hyperledger Fabric 通过使用基于MVCC的技术来解决Transaction之间的读写冲突问题。

Hyperledger Fabric是一种EOV结构的Blockchain。在validation阶段，Validator通过判断Transaction的Read-Write Set来判断Transaction是否invalid。

具体的是，Validator会循环遍历Transaction的Read Set中所有的Key以及其的Version，并与该Key在当前World StateDB中的Version进行对比。如果两者不相同，说明出现了读写冲突，该Transaction会被标记为Invalid，其Write set的结果最终不会被写入到World StateDB中。

```go
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

在以太坊中，系统通过限制合约调用所需要的Gas Limit，或者通过限制EVM调用栈的深度来解决停机问题。

在Hyperledger Fabric中，Endorsement Peers是网络中唯一会执行合约的成员节点。与以太坊不同的是，Hyperledger Fabric没有Gas的概念，以及没有过渡层虚拟机来构建调用栈。为了解决chaincode中的停机问题，Peer通过设置CORE_EXECUTECHAINCODE_TIMEOUT来设置transaction可以执行的时间上限。一旦Transaction的运行时间超过了这个上线，则transaction将会因为Execute timeout而失败。

```go
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

按照Hyperledger 官方文档的解释，一个或多个Smart Contract构成了一个Chaincode。Hyperledger中的Smart Contract使用基于通用编程语言开发，包括Go，Java和JavaScript。使用通用编程器语言增加了合约开发的灵活度，也带来了更多的不确定性。
为了约束合约的开发，减少合约代码中的不确定性，开发人员在编写Smart Contract时需要继承官方Fabric SDK中的接口函数，合约函数需要调用Fabric SDK官方提供SDK才能与链上数据进行交互。具体的可以阅读官方文档中Transaction Context一章。

For Example, Fabric SDK中关于World State的API:

- <async> getState(key) [Retrieves the current value of the state variable key]
- <async> putState(key, value) [Writes the state variable key of value value to the state store. If the variable already exists, the value will be overwritten.
- <async> deleteState(key)
- <async> getStateByRange(startKey, endKey) [Returns a range iterator over a set of keys in the ledger. The iterator can be used to iterate over all keys between the startKey (inclusive) and endKey (exclusive). The query is re-executed during validation phase to ensure result set has not changed since transaction endorsement (phantom reads detected).]
- getStateByRangeWithPagination(startKey, endKey, pageSize, bookmark)

### Compared with Ethereum

首先关于链上数据管理上，Hyperledger Fabric与Ethereum两者的账本都是以K/V-based State Object作为基本数据元素，通过World State来维护管理State Object的最新状态。具体的来说，World State本质上是一个K/V-based State Database(StateDB)。

在State database的具体实现上，Hyperledger Fabric与Ethereum采用的模型有较大的不同。

首先Ethereum使用MPT对State Objects进行管理。并在LevelDB的基础上封装了一层StateDB层提供向上的接口。上层应用逻辑都需要通过StateDB提供的接口来间接访问底层的LevelDB结构。通常有两种需要查询修改World State的情况，1. Ethereum链上的合约逻辑需要 2.外部开发人员/用户需要。这两种情况调用函数是相同的，本质上都是StateDB提供的接口函数。StateDB不仅提供了State Object管理需要的逻辑代码，还包括了一些应用层面的逻辑，比如CreateAccount。

此外，在Ethereum中，World State包含的所有的State Object的Key值都是通过加密算法随机生成的。即使是State Object的拥有者也无法在其Key值生成前(生成Key值的函数调用之前)，获取到其Key值的信息。因为Key值的生成依赖于安全的哈希算法，所以在点查询(Point Query)/随机查询State Object时速度较快。同样因为函数的随机性，而在Range Query上表现相对较差。

另一方面，在Hyperledger Fabric提供了对CouchDB/LevelDB提供的K/V相关的API进行封装，向上层逻辑提供Put/Get/Delete State Database的操作。从变量的声明中，我们可以看出Hyperledger Fabric设计思想上的不同。Hyperledger Fabric将Blockchain的组件都数据库化。

```Golang
// kvLedger provides an implementation of `ledger.PeerLedger`.
// This implementation provides a key-value based data model
type kvLedger struct {
 ledgerID               string
 bootSnapshotMetadata   *SnapshotMetadata
 blockStore             *blkstorage.BlockStore
 pvtdataStore           *pvtdatastorage.Store
 txmgr                  *txmgr.LockBasedTxMgr
 historyDB              *history.DB
 configHistoryRetriever *collectionConfigHistoryRetriever
 snapshotMgr            *snapshotMgr
 blockAPIsRWLock        *sync.RWMutex
 stats                  *ledgerStats
 commitHash             []byte
 hashProvider           ledger.HashProvider
 config                 *ledger.Config

 // isPvtDataStoreAheadOfBlockStore is read during missing pvtData
 // reconciliation and may be updated during a regular block commit.
 // Hence, we use atomic value to ensure consistent read.
 isPvtstoreAheadOfBlkstore atomic.Value

 commitNotifierLock sync.Mutex
 commitNotifier     *commitNotifier
}
```

另一方面，Hyperledger Fabric底层数据库上并没有很多业务层面的逻辑。Hyperledger Fabric中所有的World State中的基础数据元素(State Object)的key是可以由开发人员/用户自行提供的。用户或者开发人员可以提供任意的String类型的Key作为World State中State Object的Key值，用于检索。这个特征可以使得Hyperledger Fabric很轻易的提供基于Key值的Range Query，例如上面Section提到的getStateByRange(startKey, endKey)。

## Hyperledger Application Development

### From the Coarse-grained view

Fabric SDK用于提供给开发人员与链上合约交互的API。目前Fabric 官方提供了基于NodeJS以及Java的API文档。其中NodeJS文档相对比较全面。相比于Ethereum的Web3对外提供的API，Fabric SDK中提供的API相对较少。都是一些直接与链上数据进行交互的API。

在SDK中，最重要的Class是Gateway Class，甚至Java版本的SDK文档就叫做Hyperledger Fabric Gateway SDK for Java。下面是官方文档的例子：

```Java
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.concurrent.TimeoutException;

import org.hyperledger.fabric.gateway.Contract;
import org.hyperledger.fabric.gateway.ContractException;
import org.hyperledger.fabric.gateway.Gateway;
import org.hyperledger.fabric.gateway.Network;
import org.hyperledger.fabric.gateway.Wallet;
import org.hyperledger.fabric.gateway.Wallets;

class Sample {
    public static void main(String[] args) throws IOException {
        // Load an existing wallet holding identities used to access the network.
        Path walletDirectory = Paths.get("wallet");
        Wallet wallet = Wallets.newFileSystemWallet(walletDirectory);

        // Path to a common connection profile describing the network.
        Path networkConfigFile = Paths.get("connection.json");

        // Configure the gateway connection used to access the network.
        Gateway.Builder builder = Gateway.createBuilder()
                .identity(wallet, "user1")
                .networkConfig(networkConfigFile);

        // Create a gateway connection
        try (Gateway gateway = builder.connect()) {

            // Obtain a smart contract deployed on the network.
            // 获取Channel
            Network network = gateway.getNetwork("mychannel");
            // 获取Channel上安装的合约
            Contract contract = network.getContract("fabcar");

            // Submit transactions that store state to the ledger.
            byte[] createCarResult = contract.createTransaction("createCar")
                    .submit("CAR10", "VW", "Polo", "Grey", "Mary");
            System.out.println(new String(createCarResult, StandardCharsets.UTF_8));

            // Evaluate transactions that query state from the ledger.
            byte[] queryAllCarsResult = contract.evaluateTransaction("queryAllCars");
            System.out.println(new String(queryAllCarsResult, StandardCharsets.UTF_8));

        } catch (ContractException | TimeoutException | InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
