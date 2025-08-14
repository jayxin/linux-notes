# TeXLive

## 编译 TeXLive

<details>
<summary>从源码编译 TeXLive</summary>

```sh
# 克隆源代码仓库
svn co svn://tug.org/texlive/trunk/Build/source

# 进入 source 目录

# compile
TL_MAKE_FLAGS=-j`nproc` ./Build -C --without-x CFLAGS=-g CXXFLAGS=-g --prefix=$HOME/texlive/2025

# install
cd Work
make install

# config
cat >> $HOME/.bashrc <<'EOF'
# TeXLive
export TEXMFROOT=/mnt/texlive/2024
export TEXMFCNF=$TEXMFROOT/texmf-dist/web2c
export PATH=$HOME/texlive/2025/bin/x86_64-pc-linux-gnu:$PATH
EOF

# check
which luatex

# 生成 .fmt 文件, 位于 $TEXMFVAR 目录下
fmtutil-user --all
# 或生成某个特定类型
fmtutil-user -byfmt lualatex
```

</details>
