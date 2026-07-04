# TeXLive 安装与 LaTeX 编译优化

## TeXLive 安装

### 选择

| 条目         | 内容               |
|--------------|--------------------|
| 操作系统     | Linux(Rocky Linux) |
| 文件系统     | ext4               |
| 存储介质     | 16G 的 U 盘        |
| TeXLive 版本 | 2023               |

TeXLive 2023 版和其它年份的发行版只是安装后路径不同，安装步骤一样。

1. 为何选择 Linux 和 ext4?<br>
LaTeX 编译涉及读取大量宏包，即读取大量小文件，
ext4 文件系统针对小文件的读写做了优化，
编译 LaTeX 的速度比 Windows 系统的 NTFS 之类的文件系统要快。
ext4 文件系统在 Windows 上默认是不支持的，
而 Linux 系统对 ext4 文件系统有原生支持。
2. 为何选择 16G 的 U 盘作为存储介质?<br>
    - 方便迁移、携带和备份。TeXLive 中很多脚本都是用 Perl 语言实现的，
    故只要 Linux 系统安装了 Perl 语言的解释器，
    则在大部分情况下 TeXLive 都能正常工作。
    特殊情况比如需要在 LaTeX 中引入 `eps` 文件，则要另外安装 `ghostscript`。
    - 2023 版本的 TeXLive 大小将近 7G，
    选用 16 G 的 U 盘是为了应对 TeXLive 发行版的更新同时又不至于太浪费存储空间。

这里假设已装好 Linux 操作系统(物理机、虚拟机皆可)，
准备好了一个 16G 或更大空间的 U 盘，且 U 盘数据都已备份或无关紧要。

### 分区

假设 U 盘连接后的设备名是 `sdc`。

1. 进入 `fdisk`:
```sh
sudo fdisk /dev/sdc
```

todo figure

常用操作:
- `m`: 查看帮助(menu)
- `d`: 删除分区(delete partition)
- `p`: 打印分区(print partition)
- `n`: 新建分区(new partition)
- `w`: 将内存中的分区结果写入磁盘(write), 必须使用 `w` 写入才能使分区操作生效
- `q`: 退出 `fdisk` 并丢弃内存中所做的修改

2. 删除旧分区:
todo figure

3. 新建分区:
todo figure

4. 将分区结果写入磁盘:
todo figure

### 格式化

格式化即为分区写入文件系统。

查看上面分区的结果:

```sh
lsblk
```

todo figure

写入文件系统:

```sh
sudo mkfs.ext4 -m 0 /dev/sdc1
```

命令说明:
- `-m 0`: 保留 `0%` 的空间给管理员使用。这是在磁盘无法继续写入的情况下(比如磁盘空间已满)允许管理员使用的空间，由于我们是安装 TeXLive，将 U 盘作为数据盘，所以无需保留任何空间，默认是 `5%`。
- `/dev/sdc1`: `sdc` 设备的第 1 个分区。

todo figure

### 挂载

假设安装到 U 盘上的 `latex/texlive/2023` 目录，U 盘挂载点为 `/mnt/ddq`。

```sh
sudo mkdir -p /mnt/ddq
sudo mount /dev/sdc1 /mnt/ddq
sudo mkdir -p /mnt/ddq/latex/texlive
```

### 安装 TeXLive

#### 下载 TeXLive ISO 镜像文件

- 镜像站点列表: [https://ctan.org/mirrors](https://ctan.org/mirrors)
- 阿里云镜像站: [https://mirrors.aliyun.com/CTAN/systems/texlive/Images](https://mirrors.aliyun.com/CTAN/systems/texlive/Images)
- 清华大学镜像站: [https://mirrors.tuna.tsinghua.edu.cn/CTAN/systems/texlive/Images](https://mirrors.tuna.tsinghua.edu.cn/CTAN/systems/texlive/Images)

下载大小以 G 为单位的 `iso` 文件，`md5`、`sha512` 等文件用于 `iso` 文件的校验，
如需校验则请自行查资料，此处不赘述。

#### 挂载 ISO

假设:(根据需要修改和替换路径)
- 挂载点: `/mnt/texlive`
- ISO 位置: `~/Downloads/texlive2023.iso`

```sh
sudo mkdir -p /mnt/texlive
sudo mount ~/Downloads/texlive2023.iso /mnt/texlive
```

#### 执行安装脚本

```sh
sudo /mnt/texlive/install-tl
```

#### 选项和命令说明

```none
======================> TeX Live installation procedure <=====================

======>   Letters/digits in <angle brackets> indicate   <=======
======>   menu items for actions or customizations      <=======
= help>   https://tug.org/texlive/doc/install-tl.html   <=======

 # 检测到的操作系统和 CPU 架构, 这里是 Linux 系统和 x86_64 架构
 Detected platform: GNU/Linux on x86_64

 # 设置要安装的平台的类型, 一般设置成和检测到的平台一样
 <B> set binary platforms: 1 out of 1

 # 选择安装方案, 根据需要选择
 <S> set installation scheme: scheme-minimal

 # 选择安装哪些集合
 <C> set installation collections:
     1 collections out of 25, disk space required: 75 MB (free: 4633 MB)

 # 设置安装位置和环境变量的值
 <D> set directories:
   # 设置安装的主目录
   TEXDIR (the main TeX directory):
     !! default location: /usr/local/texlive/2023
     !! is not writable or not allowed, please select a different one!
   # 系统级本地额外文件存放目录
   TEXMFLOCAL (directory for site-wide local files):
     /usr/local/texlive/texmf-local
   # 系统级可变文件或自动生成的文件目录
   TEXMFSYSVAR (directory for variable and automatically generated data):
     /usr/local/texlive/2023/texmf-var
   # 系统级配置目录
   TEXMFSYSCONFIG (directory for local config):
     /usr/local/texlive/2023/texmf-config
   # 用户级可变文件或自动生成的文件目录
   TEXMFVAR (personal directory for variable and automatically generated data):
     ~/.texlive2023/texmf-var
   # 用户级配置目录
   TEXMFCONFIG (personal directory for local config):
     ~/.texlive2023/texmf-config
   # 用户级文件存放目录
   TEXMFHOME (directory for user-specific files):
     ~/texmf

 # 设置选项
 <O> options:
   # 纸张选择 A4 还是信纸
   [ ] use letter size instead of A4 by default
   # 允许通过 \write18 命令执行外部程序
   [X] allow execution of restricted list of programs via \write18
   # 创建所有格式文件
   [X] create all format files
   # 安装宏包/字体文档
   [X] install macro/font doc tree
   # 安装宏包/字体源码
   [X] install macro/font source tree
   # 创建符号链接到标准目录
   [ ] create symlinks to standard directories
   # 安装后设置 CTAN 为宏包更新源
   [X] after install, set CTAN as source for package updates

 # 选择是否是移动安装, 即是否是安装到 U 盘
 <V> set up for portable installation

# 操作
Actions:
 # 开始安装
 <I> start installation to hard disk
 # 将上面的设置保存到 texlive.profile 文件并退出
 <P> save installation profile to 'texlive.profile' and exit
 # 退出
 <Q> quit

# 输入命令
Enter command:
```

#### 具体步骤

1. 安装平台、方案(scheme)、集合(collection)自行选择和设置。

2. 选择移动安装。

```none
Enter command: V
```

3. 设置安装位置。

进入二级菜单:
```none
Enter command: D
```

设置主目录:
```none
===============================================================================
Directories customization:

 <1> TEXDIR:         /mnt/ddq/latex/texlive/2023
     main tree:      /mnt/ddq/latex/texlive/2023/texmf-dist

Actions:
 <R> return to main menu
 <Q> quit

Enter command: 1
New value for TEXDIR [/mnt/ddq/latex/texlive/2023]: /mnt/ddq/latex/texlive/2023
```

返回一级菜单:
```none
Enter command: R
```

设置后的内容:
```none
 <D> set directories:
   TEXDIR (the main TeX directory):
     /mnt/ddq/latex/texlive/2023
   TEXMFLOCAL (directory for site-wide local files):
     /mnt/ddq/latex/texlive/texmf-local
   TEXMFSYSVAR (directory for variable and automatically generated data):
     /mnt/ddq/latex/texlive/2023/texmf-var
   TEXMFSYSCONFIG (directory for local config):
     /mnt/ddq/latex/texlive/2023/texmf-config
   # 由环境变量决定
   TEXMFVAR (personal directory for variable and automatically generated data):
     $TEXMFSYSVAR
   # 由环境变量决定
   TEXMFCONFIG (personal directory for local config):
     $TEXMFSYSCONFIG
   # 由环境变量决定
   TEXMFHOME (directory for user-specific files):
     $TEXMFLOCAL
```

4. 设置选项。这里使用默认。

```none
<O> options:
   [ ] use letter size instead of A4 by default
   [X] allow execution of restricted list of programs via \write18
   [X] create all format files
   [X] install macro/font doc tree
   [X] install macro/font source tree
   [X] after install, set CTAN as source for package updates
```

5. 开始安装

```none
Enter command: I
```

等待安装完成即可。

#### 取消挂载 ISO

```sh
sudo umount /mnt/texlive
sudo rmdir /mnt/texlive
```

#### 设置环境变量

在 `/mnt/ddq/latex/` 目录新建 `setup.sh` 文件:
```sh
sudo vim /mnt/ddq/latex/setup.sh
```

`setup.sh`文件内容:
```bash
#!/usr/bin/env bash

CWD="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
TEXLIVE_DIST_DIR="$CWD/texlive/2023"

export MANPATH="${TEXLIVE_DIST_DIR}/texmf-dist/doc/man:${MANPATH}"
export INFOPATH="${TEXLIVE_DIST_DIR}/texmf-dist/doc/info:${INFOPATH}"
export PATH="${TEXLIVE_DIST_DIR}/bin/x86_64-linux:${PATH}"
```

- `CWD` 是脚本 `setup.sh` 所在的目录
- `MANPATH`、`INFOPATH` 是文档路径，分别供 `man` 和 `info` 程序读取
- `PATH` 即 TeXLive 中可执行文件的目录

让环境变量生效:
```sh
source /mnt/ddq/latex/setup.sh
```

#### 刷新字体缓存

```sh
sudo cp $TEXLIVE_DIST_DIR/texmf-var/fonts/conf/texlive-fontconfig.conf \
    /etc/fonts/conf.d/09-texlive.conf
# 刷新字体缓存
sudo fc-cache -fsv
```

若挂载路径发生变化，则文件 `09-texlive.conf` 也要更新成对应路径并刷新字体缓存。

比如原来挂载在 `/mnt/ddq`，则内容为:
```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <dir>/mnt/ddq/latex/texlive/2023/texmf-dist/fonts/opentype</dir>
  <dir>/mnt/ddq/latex/texlive/2023/texmf-dist/fonts/truetype</dir>
  <dir>/mnt/ddq/latex/texlive/2023/texmf-dist/fonts/type1</dir>
</fontconfig>
```

现在改成挂载在 `/mnt/usb`，则内容为:
```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <dir>/mnt/usb/latex/texlive/2023/texmf-dist/fonts/opentype</dir>
  <dir>/mnt/usb/latex/texlive/2023/texmf-dist/fonts/truetype</dir>
  <dir>/mnt/usb/latex/texlive/2023/texmf-dist/fonts/type1</dir>
</fontconfig>
```

### 备份和恢复

#### 目的

- 防止 U 盘丢失或突然损坏
- 无需从头安装 TeXLive，只需同步已有的 TeXLive 即可

#### 步骤

假设:
- 有一旧 U 盘和一新 U 盘
- 旧 U 盘的 latex 目录安装了 TeXLive 2023
- 旧 U 盘连接到 Linux 系统后块设备路径为 `/dev/sdc`, 数据分区为 `/dev/sdc1`
- 新 U 盘已按照上面步骤进行分区和格式化
- 新 U 盘连接到 Linux 系统后块设备路径为 `/dev/sdd`, 数据分区为 `/dev/sdd1`

系统块设备及其分区可用 `lsblk` 命令查看。

1. 准备挂载点。
```bash
sudo mkdir -p /mnt/{old,new}
```

2. 挂载 U 盘。
```bash
# 旧 U 盘无写操作，只读挂载
sudo mount -o ro,noload /dev/sdc1 /mnt/old

# 新 U 盘正常挂载，可读写
sudo mount /dev/sdd1 /mnt/new
```

3. 用 `rsync` 同步数据(需已安装 `rsync` 命令)。
```bash
# 将 latex 目录同步到 /mnt/new.
# 注意 latex 后不能有 /, 否则不会为 /mnt/new 建立 latex 目录,
# /mnt/new 目录下直接就是 /mnt/old/latex 目录里的内容
sudo rsync -avPz /mnt/old/latex /mnt/new/
```

等待文件同步完毕即可。因为 `rsync` 是增量同步，故即使同步过程被中断，也只会同步
还未同步过的文件。

4. 卸载 U 盘。
```bash
# 确保数据写入到 U 盘
sudo sync

# 卸载旧 U 盘
sudo umount /mnt/old

# 卸载新 U 盘
sudo umount /mnt/new
```

### 更新 TeXLive

TeXLive 的更新分为宏包级别更新和发行版级别更新。

#### 宏包级别更新

一般不建议把宏包级别更新带入到发行版级别，因为宏包级别更新有可能破坏发行版级别宏包间的依赖关系，
一旦破坏原有依赖关系又无备份的话就无法再回滚到之前状态，最终导致不得不重新安装 TeXLive。

这里把文件存放位置分成 2 处:
- 系统级: 即 TeXLive 发行版的安装位置，如果按照上面的步骤操作，应是在 U
    盘上，只有管理员有权限进行写操作
- 用户级: 用户可读写的位置，一般在用户的家目录 `$HOME` 下

宏包级别的更新我们放在用户级文件位置，这样一旦出错还有机会回滚(删除更新的文件)，系统级文件不受影响。

操作步骤:
```bash
# 准备用户级文件目录
mkdir -p ~/texmf

# 声明用户级文件的根路径, 可将此行持久化至 $HOME/.bashrc 文件
export TEXMFHOME=$HOME/texmf

# 初始化用户级文件树
tlmgr init-usertree

# 设置从哪个站点下载宏包
# 若正在使用的发行版比最新的发行版要旧，
# 则指明的地址必须包含和正在用的发行版同一年份的更新
# tlmgr 不支持跨发行版下载/更新宏包(比如 2023 和 2026 就是跨发行版)
tlmgr --usermode option repository \
    https://ftp.math.utah.edu/pub/tex/historic/systems/texlive/2023/tlnet-final

# 或者还有一种方法就是更新 tlmgr 到最新，但不推荐
wget http://mirror.ctan.org/systems/texlive/tlnet/update-tlmgr-latest.sh
chmod +x update-tlmgr-latest.sh
./update-tlmgr-latest.sh -- --update

# 示例: 安装 booktabs 宏包
tlmgr --usermode install booktabs
```

`tlmgr` 的更多用法见其手册，可用 `man tlmgr` 命令查看。

#### 发行版级别的更新

比如 2023 的发行版更新到 2026。

发行版级更新建议遵循以下步骤:
1. 按照上面安装 TeXLive 的步骤安装新版 TeXLive，注意安装到新路径，不要覆盖旧发行版文件。
2. 测试新发行版是否可用。
3. 若新发行版可用，则可自行选择是否删除旧发行版。

## LaTeX 编译优化

### U 盘挂载优化

```sh
sudo mkdir -p /mnt/ddq
sudo mount -o ro /dev/sdc1 /mnt/ddq
```

命令说明: 把 `sdc` 设备的第 1 个分区以只读的的方式挂载到 `/mnt/ddq` 目录。
- `-o ro`: 只读(Read Only)挂载，等价于 (`-r`, `--read-only`)
- `/dev/sdc1`: 待挂载的分区
- `/mnt/ddq`: 挂载点

在 `mount` 命令的手册 `mount(8)` 中的 `-r` 选项有如下内容:

```
-r, --read-only
    Mount the filesystem read-only. A synonym is -o ro.

    Note that, depending on the filesystem type, state and kernel
    behavior, the system may still write to the device. For example, ext3
    and ext4 will replay the journal if the filesystem is dirty. To
    prevent this kind of write access, you may want to mount an ext3 or
    ext4 filesystem with the ro,noload mount options or set the block
    device itself to read-only mode, see the blockdev(8) command.
```

大意是即使以只读方式挂载，但一些文件系统如ext3、ext4还是有可能会重放日志。
可于挂载时用 `-o ro,noload` 选项或用 `blockdev` 命令来避免此可能。

```sh
# 方法 1
# 使用 blockdev 设置块设备只读
sudo blockdev --setro /dev/sdc
sudo mount /dev/sdc1 /mnt/ddq

# 方法 2
sudo mount -o ro,noload /dev/sdc1 /mnt/ddq
```

只读挂载的好处: **零写入**、**零磨损**、**断电零风险**
- 系统突然断电/U盘突然拔出不会导致 U 盘数据丢失。断电导致数据丢失
本质是写操作被中断，而只读模式下不会有任何写操作发生。
- 极大降低对 U 盘的损耗。只读挂载的情况下操作系统不会对 U 盘的
存储介质进行任何的写入操作，包括文件的 atime(访问时间)，文件写入只在
格式化和装 TeXLive 时发生，在编译 LaTeX 时只是读取 U 盘上
的宏包之类的文件，读操作相对写操作而言，对 U 盘寿命的影响是很小的。

### 区分编辑和编译

- 编辑(edit): 用文本/代码编辑器(text editor)写 LaTeX 代码，这一步是得到 LaTeX
    源码的过程
- 编译(compile): 用 LaTeX 编译器(compiler) 如 `pdflatex`、`xelatex` 等将 LaTeX
    源码转换成 pdf 的过程

### 源码层面优化

源码层面最根本的原则就是 LaTeX 代码越少，编译越快。

#### 注释

我们在编辑 LaTeX 代码时更多时候是关注局部的某处代码的渲染效果，比如某处
公式是否渲染正确、某个图片大小是否合适等，因此可暂时注释已确认
没问题的代码，只编译当前正在编辑的代码。

#### 多文件

多文件即在一个 LaTeX 文档中使用多个 `tex` 源文件来组织源码。

优点:
- 方便维护，结构比单个 `tex` 文件清晰
- 方便定位代码和错误
- 方便注释

```latex
\documentclass{book}

\begin{document}
% 第 1 章
\input{contents/chapter1.tex}

% 第 2 章
\input{contents/chapter2.tex}

% 第 3 章
\input{contents/chapter3.tex}
\end{document}
```

#### 延迟图片嵌入

### 编译层面优化

tmpfs

latexmk

http://tug.ctan.org/info/install-latex-guide-zh-cn/install-latex-guide-zh-cn.pdf
