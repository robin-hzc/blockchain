api目录

为Lotus命令提供后台服务，多数命令都需要与后台服务进行通信，这些命令通常会涉及到链上数据。本目录抽象了节点定义，定义了若干go interface，如 Common（定义节点通用功能）、FullNode （定义一个全节点的行为，继承自Common）、StorageMiner（存储矿工，也从Common继承）和相关的函数。

对应于上面几种主要接口，也提供了几个struct，分别为CommonStruct，FullNodeStruct，StorageMinerStruct作为实现，这些实现使用了代理模式，只是简单地将请求转发给各自的Internal成员，具体的功能函数需要使用者提供，例如获取支付通道列表函数：



获取支付通道列表





build目录

定义用于构建节点的函数，包括但不限于： 从受信节点获取启动相关参数（位于paramfetch.go），生成内置的创世区块（位于genesis.go）等。 genesis子目录：内置的创世区块数据

cli目录

Lotus命令行工具的实现，依赖于包gopkg.in/urfave/cli.v2，里面的go文件名基本上与Lotus的子命令保持一致。对应于每条子命令及子命令的子命令，都定义了一个Command对象，如相对于Lotus chain 命令：



chain命令

相应的Command定义在文件chain.go中：

chain.go


cmd目录

内含各种不同的命令行项目，Lotus将系统分为不同的进程模块，为每个模块定义一个项目：

lotus	            守护进程	负责公链上数据的同步，与公链进行交互等，是lotus的主要进程之一
lotus-bench	        基准测试工具	 
lotus-storage-miner	挖矿进程	打包信息到区块，存储矿工
lotus-seal-worker	密封数据进程	密封数据是挖矿过程中必不可少的一环，本进程即实现此功能

chain目录

实现与链的交互功能，主要分为如下子目录：

types: 定义Filecoin中的各种数据结构
store: 公链存储相关，处理所有的本地链状态，包括链头、消息和状态等
state:处理Filecoin的状态树，内部包装了HAMT
actors: Filecoin网络内建的各种actor定义
vm:Filecoin虚拟机，这里实现了调用Filecoin内actor的方法的工具

miner目录

定义产出区块逻辑，与全节点通过API进行交互

storage目录

定义存储矿工逻辑，用于实现"lotus-storage-miner"

node目录

定义了lotus节点相关的struct和interface等，各主要子目录如下：
hello:实现hello协议
modules:定义实现节点功能的各种函数，如:
创建钱包

func NewWallet(keystore types.KeyStore) (*Wallet, error) {
w := &Wallet{
keys:     make(map[address.Address]*Key),
keystore: keystore,
}

	return w, nil
}

链存储：
func ChainStore(lc fx.Lifecycle, bs dtypes.ChainBlockstore, ds dtypes.MetadataDS) *store.ChainStore {
chain := store.NewChainStore(bs, ds)

	if err := chain.Load(); err != nil {
		log.Warnf("loading chain state from disk: %s", err)
	}
 
	return chain
}

repo：链上数据在本地的存储仓库，与本地文件系统打交道

documentation目录

Lotus的文档目录，Lotus计划提供中英文两种文字的文档，但目前英文文档在逐步完善中，但cn目录空空如也。目前已有的文档主要涉及以下方面：
新手指导
硬件要求
Lotus在各种系统上的安装
节点运行指导
如何加入开发网
Pond介绍
挖矿指导
向系统存储数据
lib目录
实现lotus项目各模块公用的函数

crypto：实现数据加密，公钥生成，签名与签名验证等
jsonrpc：实现了一个基于json传输数据的rpc包，包括服务端和客户端，可为其它项目提供完整的rpc功能支持
statestore：包装github.com/ipfs/go-datastore，实现状态追踪
sectorbuilder：实现扇区管理
bufstore:：包装github.com/ipfs/go-ipfs-blockstore，集成了Blockstore的读写功能
cborutil：包装github.com/ipfs/go-ipld-cbor，提供操作cbor数据的简便方法
auth： 实现权限认证服务HTTP接口
lotuspond目录
Pond项目目录，Pond是一个用于管理Lotus的UI工具，可用于建立一个独立的本地网络以方便调试。Pond会启动节点，使用指定的拓扑进行连接，启动挖矿，并持续监控节点运行状况。

retrieval目录

实现检索矿工和客户端功能

scripts目录

各种运行脚本，用于布暑节点和矿工等，也包括一些用于启动的配置文件