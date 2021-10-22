# Untangling_Hyperledger_Fabric

The ordering service will order transactions in a arbitrarily way or just by the transaction arrive time.

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
