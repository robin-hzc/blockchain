api
见名知意，这个目下主要存放 API 相关程序，Lotus 是一个大型的区块链项目，模块之间基本都通过 API 来通信，远程 API 的调用走的是 JSONRPC 调用模式，比如 Miner 上链要调用 Lotus Chain 的 API，Worker 和 Miner 之间通信也是相互调用 API。

本目录抽象了节点定义，定义了若干go interface，如 Common（定义节点通用功能）、FullNode （定义一个全节点的行为，继承自Common）、StorageMiner（存储矿工，也从Common继承）和相关的函数。这里有几个重要的接口：

Common： 定义了一些节点通用的功能，在 api_common.go 中定义。

FullNode: 定义一个全节点的所有 API，继承自 Common，在 api_full.go 中定义。

StorageMiner: 定义一个存储矿工的行为，继承自 Common，在 api_storage.go 中定义。

值得一提的是，Lotus 是使用代理模式实现上述接口的，一旦你对接口进行了更改，你需要执行：

make gen
该命令会生成一个 api/proxy_gen.go 文件，自动为上述接口生成一个对应的 struct，分别为CommonStruct，FullNodeStruct，StorageMinerStruct，这些实现只是简单地将请求转发给各自的Internal成员， 具体的功能函数需要使用者提供，比如获取扇区列表 SectorList API 的实现如下：

func (s *StorageMinerStruct) SectorsList(p0 context.Context) ([]abi.SectorNumber, error) {
return s.Internal.SectorsList(p0)
}
切记：每次更改了 API 要记得执行 make gen 重新生成 API 代理文件 proxy_gen.go，否则编译的时候会提示找不到 API 方法。

blockstore
区块存储工具包，包括区块的新增，删除，以及同步相关的 API，这里做了几种实现，有 leveldb 版本，内存（mem.go）版本以及 IPFS 版本的实现。

IPFS 版本实现：

func (i *IPFSBlockstore) DeleteBlock(cid cid.Cid) error

func (i *IPFSBlockstore) Has(cid cid.Cid) (bool, error)

func (i *IPFSBlockstore) Get(cid cid.Cid) (blocks.Block, error)

func (i *IPFSBlockstore) Put(block blocks.Block) error
内存版本实现：

func NewMemory() MemBlockstore {
return make(MemBlockstore)
}

// MemBlockstore is a terminal blockstore that keeps blocks in memory.
type MemBlockstore map[cid.Cid]blocks.Block

func (m MemBlockstore) DeleteBlock(k cid.Cid) error {
delete(m, k)
return nil
}

func (m MemBlockstore) Get(k cid.Cid) (blocks.Block, error) {
b, ok := m[k]
if !ok {
return nil, ErrNotFound
}
return b, nil
}

func (m MemBlockstore) Has(k cid.Cid) (bool, error) {
_, ok := m[k]
return ok, nil
}
build
定义用于构建节点，创建网络相关功能，包括但不限于定义网络相关配置(如 params_main.go)，定义创世节点相关信息(genesis目录)， 以及创世节点的连接信息(bootstrap目录)，复制证明参数下载地址配置等。

chain（二级关注）
顾名思义，就是实现了 Louts 链相关的功能，主要包含了如下子模块：

types: 定义 Louts 链中的各种数据结构。

store: 公链存储相关，处理所有的本地链状态，包括链头、消息和状态等。

messagesinger: 消息签名工具包。

messagepool：消息池，定义消息打包规则。

wallet：实现钱包相关工具。

state: 处理 Filecoin 的状态树，内部包装了HAMT。

actors: 账户体系，定义了各种 actor。

vm: 智能合约虚拟机，这里实现了actor 的方法调用工具包。

cli（一级关注）
Lotus命令行工具的实现，里面的go文件名基本上与Lotus的子命令保持一致，比如：

state.go: 对应 lotus state 命令

sync.go: 对应 lotus sync 命令

wallet.go: 对应 lotus wallet 命令

...

对应于每条子命令及子命令的子命令，都定义了一个Command对象，比如 wallet.go：

// 每个条命定义了若干子命令对象
var walletCmd = &cli.Command{
Name:  "wallet",
Usage: "Manage wallet",
Subcommands: []*cli.Command{
walletNew,
walletList,
walletBalance,
walletExport,
walletImport,
walletGetDefault,
walletSetDefault,
walletSign,
walletVerify,
walletDelete,
walletMarket,
},
}

// 每个子命令对象单独实现一个功能
var walletNew = &cli.Command{
Name:      "new",
Usage:     "Generate a new key of the given type",
ArgsUsage: "[bls|secp256k1 (default secp256k1)]",
Action: func(cctx *cli.Context) error {
api, closer, err := GetFullNodeAPI(cctx)
if err != nil {
return err
}
defer closer()
ctx := ReqContext(cctx)

t := cctx.Args().First()
if t == "" {
t = "secp256k1"
}

nk, err := api.WalletNew(ctx, types.KeyType(t))
if err != nil {
return err
}

fmt.Println(nk.String())

return nil
},
}
cmd（一级关注）
包括 Lotus 项目所有的命令行项目，Lotus 将系统分为不同的进程模块，为每个模块定义一个项目：

模块名称	模块说明
lotus	lotus 守护进程项目，负责与 lotus 公链通信，同步区块，是主要核心进程之一
lotus-miner	存储矿工的核心进程，负责时空证明，爆块，任务调度，是主要核心进程之一
lotus-seal-worker	数据密封主进程，整个复制证明工作由该进程完成，是主要核心进程之一
lotus-bench	基准测试工具项目，如果你要做性能优化，那么这个工具应该要经常使用
lotus-seed	用来生成创世区块的工具，构建网络不可或缺
lotus-shed	非常有用的一个工具包命令集合，里面又各种黑科技命令，比如终止扇区，修复消息池，多重签名工具等。
每个模块都可以单独编译，编译后的程序都可以单独启动一个进程独立运行：

# 编译 lotus 程序
make lotus
# 编译 miner 程序
make lotus-miner
# 编译 worker 程序
make lotus-worker
# 编译 bench 测试工具
make lotus-bench
# 编译钱包工具
make lotus-wallet
你切换到 lotus 代码根目录下，然后输入 make 按 Tab 键就可以呼出各种编译命令：

图片

documentation
Lotus的文档目录，Lotus计划提供中英文两种文字的文档，目前已有的文档主要涉及以下方面：

新手指导.

构建 Lotus.

如何启动 lotus, lotus-miner, lotus-worker.

如何运行本地开发网络.

Pond介绍

Lotus API 接口说明

extern/filecoin-ffi
FFI rust 调用封装，Lotus 的源码分两部分，公链和任务调度这块是 Golang 实现的，另外一些底层计算都是通过调用 Rust 底层库去计算的。比如 PC1，PC2，C1，C2，时空证明等都是用 Rust 实现的。Lotus 通过 filecoin-ffi 这个组件来调用 Rust 底层库。

extern/sector-storage （一级关注）
这里包含了任务调度，资源分配的的核心代码。属于整个项目比较核心的部分。

sealtasks: 定义了一些任务类别常量，如：
const (
TTAddPiece   TaskType = "seal/v0/addpiece"
TTPreCommit1 TaskType = "seal/v0/precommit/1"
TTPreCommit2 TaskType = "seal/v0/precommit/2"
TTCommit1    TaskType = "seal/v0/commit/1" // NOTE: We use this to transfer the sector into miner-local storage for now; Don't use on workers!
TTCommit2    TaskType = "seal/v0/commit/2"
TTFinalize TaskType = "seal/v0/finalize"
TTFetch TaskType = "seal/v0/fetch"
)
stores: 扇区存储相关代码，实现了扇区检索(index.go), 本地存储(local.go)，远程存储(remote.go)等。

storiface: 定义远程调用相关 API

worker: worker 相关功能和属性定义

manager.go: 全局管理对象，调度入口程序，它管理着 Worker，存储，以及调度等对象。

sched.go: 任务调度的具体实现

extern/storage-sealing（一级关注）
这也是 Lotus 的核心模块之一，它包含如下代码：

扇区状态机实现(fsm.go)

扇区的各种事件定义(event.go)

扇区聚合提交上链(precommit_batch.go, commit_batch.go)

sealing 相关的 API 实现

gen
API 代理对象的生成工具，略过即可。

journal
journal 日志工具，略过即可。

lib
实现 lotus 项目各模块公用的函数，主要有下面几个模块：

addrutil： 地址解析工具

backupds： lotus/lotus-miner 元数据备份工具

lotuslog： 定义 lotus 相关程序的日志等级

rpcenc： RPC 调用数据编码工具

sigs： 钱包签名工具包，包括 bls 和 secp256k 两种实现

**ulimit：**lotus 相关程序运行时的 FD 设置工具，主要用来设置 lotus-miner 程序的最大打开文件数。

lotuspond
Pond 项目目录，Pond 是一个用于管理 Lotus 的UI工具，可用于建立一个独立的本地网络以方便调试。Pond会启动节点，使用指定的拓扑进行连接，启动挖矿，并持续监控节点运行状况。不过我个人觉得这个项目没什么卵用 O(∩_∩)O~。

markets（二级关注）
Filecoin 订单市场相关实现，包括订单的过滤，订单价格，以及存储订单和检索订单的适配器等。

miner（二级关注）
爆块主程序，定义产出区块完整逻辑。

node（一级关注）
定义了 lotus 节点相关的 struct 和 interface 等，各主要子目录如下：

config: 定义节点相关配置结构体

hello: 实现 hello 协议

modules: 定义实现节点功能的各种函数，比较重要的有 chain.go, storageminer.go 等。

impl: 节点各种 API 的实现，比如 full.go 实现全节点的相关 API，storminer.go 实现存储节点相关 API 等。

repo: 链上数据在本地的存储仓库，与本地文件系统打交道。

paychmgr
各种支付凭证管理，定义了一些支付通道的 API。

scripts
各种运行脚本，用于部署节点和矿工等，也包括一些用于启动的配置文件。

storage（一级关注）
定义存储矿工逻辑，用于实现"lotus-storage-miner"，时空证明的主程序逻辑也是在这里实现的。

tools
各种打包工具封装，如使用 docker 部署 Lotus 等。

标注 一级关注 的模块是跟任务调度，密封相关的内容，这块的内容通常你最先去触碰并修改的， 通常这些模块你修改的需求最多，同时也是你能发挥的空间最大的地方。

标注 二级关注 的模块是跟链，以及爆块，时空证明相关的，再抽象一点归纳就是：它们都是跟共识有关， 显然相对前面的模块来说，你能修改的不多，因为如果你能随意修改的话，那就不叫共识了。建议有有需要了再去折腾那些，比如优化爆块程序，调试存储订单相关，以及实现多 Miner 方案的时候，通常你需要折腾这几个模块。

其他模块都是工具性的，对你理解 Lotus 原理几乎没有障碍，前期你完全可以直接忽略他们。当然，如果你有特殊需求的，比如你要研究加密原理，多重签名算法等的时候，你可以试着去折腾一下相关工具类模块