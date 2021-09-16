git clone https://github.com/filecoin-project/lotus.git
git checkout lastTags 
git switch -c lastTags
go mod tidy遇到问题：0
go: github.com/filecoin-project/filecoin-ffi@v0.30.4-0.20200910194244-f640612a1a1f (replaced by ./extern/filecoin-ffi): reading extern/filecoin-ffi/go.mod: open /home/ly/codespace/lotus/extern/filecoin-ffi/go.mod: no such file or directory

这是因为子模块被抽取出来作为一个新的仓库，使用以下命令：
cd lotus
git submodule update --init --recursive

#推荐开启翻墙，不然真心难受