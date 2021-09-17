代码版本：v1.11.3-rc1

lotus是Lotus项目中的全节点命令行二进制。启动全节点的命令是：lotus daemon

daemon进程的功能是全节点的完整功能，概述一下主要包括如下

1.通过P2P网络发现其他节点，并进行通信

2.从其它全节点获取区块数据，并对每一次区块的消息及区块打包信息进行校验并保存到本地存储

3.为其它全节点提供区块同步的服务

4.钱包管理服务，包括转帐、消息签名等

5.RPC API服务

daemon命令的Action代码逻辑(lotus/cmd/daemon.go 78行)

var DaemonCmd = &cli.Command{

Name:  "daemon",

Usage: "Start a lotus daemon process",

Flags: []cli.Flag{
...
},

Action: func(cctx *cli.Context) error {
...
...

// 创建FsRepo,即lotus的Repo

r, err := repo.NewFS(cctx.String("repo"))
​

// 初始化Repo,例如创建初始文件等

if err := r.Init(repo.FullNode); err != nil && err != repo.ErrRepoExists {

return xerrors.Errorf("repo init error: %w", err)

}
​

// 获取参数文件

if err := paramfetch.GetParams(build.ParametersJson(), 0); err != nil {

return xerrors.Errorf("fetching proof parameters: %w", err)

}

​
​
// 如果是Genesis启动方式,则加载Genesis文件

genesis := node.Options()

if len(genBytes) > 0 {

genesis = node.Override(new(modules.Genesis), modules.LoadGenesis(genBytes))

}
​
...

var api api.FullNode

// 创建全节点, 从第二个参数起都是Option

stop, err := node.New(ctx,

node.FullAPI(&api),

// 上线, 多数初始化逻辑在此实现

node.Online(),

// Repo相关的初始化

node.Repo(r),

​
genesis,

​
node.ApplyIf(func(s *node.Settings) bool { return cctx.IsSet("api") },

node.Override(node.SetApiEndpointKey, func(lr repo.LockedRepo) error {

apima, err := multiaddr.NewMultiaddr("/ip4/127.0.0.1/tcp/" +
cctx.String("api"))

if err != nil {

return err

}
return lr.SetAPIEndpoint(apima)

})),
node.ApplyIf(func(s *node.Settings) bool { return !cctx.Bool("bootstrap") },

node.Unset(node.RunPeerMgrKey),

node.Unset(new(*peermgr.PeerMgr)),

),

)
...

// 根据repo内的配置创建APIEndpoint, 即网络监听对象

endpoint, err := r.APIEndpoint()

...

// 启动RPC服务

return serveRPC(api, stop, endpoint)

},
