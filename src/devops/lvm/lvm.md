<!-- vim-markdown-toc GFM -->

* [LVM](#lvm)
    * [Introduction](#introduction)
        * [不使用 LVM 的扩容思路](#不使用-lvm-的扩容思路)
        * [使用 LVM 的扩容思路](#使用-lvm-的扩容思路)
        * [术语](#术语)
    * [安装 LVM](#安装-lvm)
    * [PV](#pv)
    * [VG](#vg)
    * [LV](#lv)
        * [LV 的扩充和缩小](#lv-的扩充和缩小)
            * [lvextend, lvreduce, lvresize, resize2fs](#lvextend-lvreduce-lvresize-resize2fs)
            * [xfs_growfs](#xfs_growfs)
        * [LV Snapshot](#lv-snapshot)
    * [删除 LVM](#删除-lvm)
    * [LVM 的优缺点](#lvm-的优缺点)
    * [挂载 LVM 类型的设备](#挂载-lvm-类型的设备)

<!-- vim-markdown-toc -->

# LVM

## Introduction

Logical Volume Manager.

将一至多个硬盘的分区在逻辑上进行组合, 当成一个大硬盘来使用.

当硬盘空间不足时, 可以动态地添加其它硬盘的分区到已有的卷组中 (磁盘空间的动态管理).

```none
App <=> File System <=> Physical Disk

App <=> File System <=> Logical Volume <=> Physical Disk
```

### 不使用 LVM 的扩容思路

传统的文件系统是基于分区的, 一个文件系统对应一个分区, 这种方式比较直观,
但不易改变.

- 不同的分区相互独立, 单独的文件不能跨分区存储, 容易出现硬盘的利用率不均衡;
- 当一个文件系统/分区装满时, 是不能对其进行扩容的, 只能采用重新分区/建立文件系统,
重新分区会丢失数据, 就要:
    - 做数据的迁移和备份;
    - 或者把分区中的数据移到另一个更大的分区中;
    - 或者采用符号链接的方式使用其它分区的空间 - 都非常麻烦.
- 若要把硬盘上的多个分区合并在一起使用, 只能采用重新分区的方式,
需要做好数据的备份与恢复.

### 使用 LVM 的扩容思路

- 硬盘的多个分区由 LVM 统一管理为卷组, 可以很轻松地加入或移走某个分区,
也就是扩大或减小卷组的可用容量, 充分利用硬盘空间.
- 文件系统建立在逻辑卷上, 而逻辑卷可以根据需要改变大小(在卷组容量范围内)以满足要求.
- 文件系统建立在 LVM 上, 可以跨分区存储访问, 更加方便.

### 术语

- **PV** (Physical Volume, 物理卷) :
PV 在逻辑卷管理中处于最底层, 它可以是实际物理硬盘上的分区,
也可以是整个物理硬盘.
- **VG** (Volume Group, 卷组):
VG 建立在 PV 之上, 一个 VG 至少包括一个 PV, 在 VG 建立之后
可动态添加 PV 到 VG 之中. 一个逻辑卷管理系统中可以只有一个
VG, 也可以有多个 VG.
- **LV** (Logical Volume, 逻辑卷):
LV 建立在 VG 之上, VG 中的未分配空间可以用于建立新的 LV,
LV 建立后可以动态地扩展和缩小空间. 系统中的多个 LV 可属于同一个
VG, 也可属于不同的多个 VG. LV 相当于原来分区的概念, 但是可以动态增减空间.
- **PE** (Physical Extent):
每一个 PV 被划分为称为 PE 的基本单元, 具有唯一编号的 PE 是可被
LVM 寻址的最小单元. PE 的大小是可配置的, 默认 4 MB.
- **LE** (Logical Extent):
LV 也被划分为称为 LE (Logical Extent) 的可被寻址的基本单元.
在同一个 VG 中, LE 的大小和 PE 是相同的, 并且一一对应.

![LVM](./img/lvm.jpg)

常用命令:


- PV: pvcreate, pvs, pvdisplay, pvremove, pvmove, pvscan
- VG: vgcreate, vgs, vgdisplay, vgremove, vgrename, vgreduce, vgextent, vgscan
- LV: lvcreate, lvs, lvdisplay, lvremove, lvextend, lvresize, lvscan, lvrename

假设有两块磁盘:
1. `/dev/sdb`
2. `/dev/sdc`

先分区, 设置分区类型:

```sh
# 使用 fdisk 对 sdb 和 sdc 进行分区, 将分区类型设置为 'Linux LVM'
fdisk ...

# 让内核重新识别分区表
partprobe /dev/sd{b,c}
partx -a /dev/sdc
partx -s /dev/sdc

# 查看分区信息表
cat /proc/partitions

# 查看
ls /dev/sdb*
ls /dev/sdc*

# 验证磁盘分区结果
fdisk -l | grep "LVM$"
```

## 安装 LVM

```sh
yum install lvm2 -y
rpm -qa | grep lvm
```

## PV

- **pvcreate**: create pv
- **pvs**: pv summary
- **pvdisplay**: display detailed information
- **pvremove**: remove pv
- **pvscan**: scan for pv

```sh
# 创建 2 个 PV
pvcreate /dev/sdb{1,2}

# summary
pvs

# 查找已存在的 PV
pvscan

# 移除 /dev/sdb2
pvremove /dev/sdb2

# detailed info
pvdisplay /dev/sdb1

pvcreate /dev/sdb2

pvs -o +pv_uuid

pvs -v
```

## VG

- **vgcreate**: create vg
- **vgs**: vg summary
- **vgextend**: 动态扩展 LVM VG, 通过向 VG 中添加 PV 来增加 VG 的容量
- **vgreduce**: 通过删除 VG 中的 PV 减少 VG 容量, 不能删除 VG 中的最后一个
PV, 即一个 VG 至少有一个 PV
- **vgdisplay**: display vg
- **vgscan** : detailed info 
- **vgremove** : 删除 VG, 其上的 LV 必须处于离线状态

```sh
# 创建 datavg 卷组
vgcreate datavg /dev/sdb{1,2}

# display vg
vgdisplay datavg

vgs

# 向 datavg VG 中添加 PV
vgextend datavg /dev/sdc1

pvs

vgs

# 从 datavg VG 中移除 PV
vgreduce datavg /dev/sdc1

# 移除 datavg
vgremove datavg

# 显示系统中所有 VG
vgscan

vgs -o +pv_name

vgs -v
```

## LV

- lvcreate: create lv
    - `lvcreate -L #[mMgGtT] -n NAME VG`
    - `-L`: 指定大小
    - `-l`: 指定大小 (LE 数)
    - `-n`: name
    - `-s`: 创建快照
    - `-p r`: 设置为只读 (一般用于创建快照)
- lvs: summary of lv
- lvscan: scan lv
- lvdisplay: display lv
- lvextend: 可在线扩展 LV 空间
- lvreduce: 缩减 LV 空间, 一般离线使用
- lvremove: remove lv, 需要处于离线 (卸载) 状态

```sh
# 创建 datalv LV
lvcreate -L 5G -n datalv datavg
ls /dev/mapper

mkfs.xfs -L Backup /dev/datavg/datalv

mkdir /dbbackup
mount /dev/mapper/datavg-datalv /dbbackup/
df -hT | grep datalv

lvs

lvscan

lvdisplay

lvcreate -L 3G -n weblv datavg
mkfs.ext4 -L webapp /dev/datavg/weblv
mkdir /webapp
mount /dev/mapper/datavg-datalv /webapp/
df -hT | grep mapper
lvs
lvscan
```

### LV 的扩充和缩小

- lvextend: 扩展 LV 空间. <br>
    `lvextend -L [+]#[mMgGtT] /dev/VG_NAME/LV_NAME`.
- lvreduce: 缩减 LV 空间
    1. 先确定缩减后的目标大小, 确保对应的目标逻辑卷大小中
    有足够的空间可容纳原有所有数据.
    2. 卸载文件系统, 执行强制检测: `e2fsck -f`.
    3. 缩减逻辑边界: `resize2fs DEVICE`.
    4. 缩减物理边界: `lvreduce`.
    5. 新挂载 `mount`.
- lvresize: 扩展或缩小 LV 空间.
    - resize2fs: 针对的是 ext2, ext3, ext4 文件系统.
    它可以增加或缩减没有挂载的文件系统的大小.
    若文件系统已经挂载, 它可以扩大文件系统的大小,
    前提是内核支持在线调整大小.
    - xfs_growfs: 针对 xfs 文件系统.

#### lvextend, lvreduce, lvresize, resize2fs

```sh
# 增大至 5G
lvextend -L 5G /dev/mapper/datavg-weblv

# 增加 2G
lvextend -L +2G /dev/mapper/datavg-weblv

# 调整逻辑大小
resize2fs /dev/mapper/datavg-weblv

df -hT | grep weblv

# 卸载
umount -f /dev/mapper/datavg-weblv

# 检查
e2fsck -f /dev/datavg/weblv

# 调整至 5G
resize2fs /dev/datavg/weblv 5G

# 缩减至 5G
lvreduce -L 5G /dev/datavg/weblv

# 挂载
mount /dev/mapper/datavg-weblv /webapp/
df -hT | grep weblv
```

#### xfs_growfs

xfs 文件系统只支持增大分区空间的情况, 不支持减小的情况.

```sh
# 增大至 8G
lvextend -L 8G /dev/mapper/datavg-datalv

# 增加 2G
lvextend -L +2G /dev/mapper/datavg-datalv

# 扩容
xfs_growfs /dev/mapper/datavg-datalv

vgs

lvs

lvscan
```

### LV Snapshot

当创建一个snapshot的时候, 仅拷贝原始卷里数据的元数据(meta-data).
创建的时候, 并不会有数据的物理拷贝, 因此 snapshot 的创建几乎是实时的,
当原始卷上有写操作执行时, snapshot 跟踪原始卷块的改变,
这个时候原始卷上将要改变的数据在改变之前被拷贝到 snapshot 预留的空间里,
因此这个原理的实现叫做写时复制 (copy-on-write).

在写操作写入块之前, 原始数据被移动到 snapshot 空间里, 
这样就保证了所有的数据在 snapshot 创建时保持一致.
而对于 snapshot 的读操作, 如果是没有修改过的块, 那么会将读操作直接重定向到原始卷上,
如果是已经修改过的块, 那么就读取拷贝到snapshot中的块.

创建快照前需将针对的逻辑卷临时改为只读, 创建完毕后再改为读写.

1. 创建快照前:
    ```sh
    mount -o remount,ro /dev/datavg/weblv /webapp
    ```
2. 创建快照后:
    ```sh
    mount -o remount,rw /dev/datavg/weblv /webapp
    ```

```sh
df -h | grep datavg
cd /webapp/
# 复制一些文件
cp /etc/passwd /etc/group /etc/redhat-release .

# 只读挂载
mount -o remount,ro /dev/datavg/weblv /webapp

# 对 weblv 做快照为 weblvsnap, 大小 2G, 只读
lvcreate -s -L 2G -n weblvsnap -p r /dev/datavg/weblv

# 读写挂载
mount -o remount,rw /dev/datavg/weblv /webapp

mkdir /mnt/web
# 挂载快照
mount /dev/datavg/weblvsnap /mnt/web
df -hT | grep weblv

lvs
lvscan
lvdisplay /dev/datavg/weblvsnap
```

## 删除 LVM

1. 卸载
2. lvremove
3. vgremove
4. pvremove

```sh
umount /dev/mapper/datavg-weblvsnap
umount /dev/mapper/datavg-weblv
umount /dev/mapper/datavg-datalv

# 删除 LV, 需要处于离线(卸载)状态
lvremove /dev/mapper/datavg-weblvsnap

lvs

# 删除 VG
vgremove datavg

# 删除 PV
pvremove /dev/sdc1
```

## LVM 的优缺点

优点:

1. 文件系统可以跨多个磁盘，因此文件系统大小不会受物理磁盘的限制.
2. 可以在系统运行的状态下动态的扩展文件系统的大小.
3. 可以增加新的磁盘到LVM的存储池中.
4. 可以以镜像的方式冗余重要的数据到多个物理磁盘.
5. 可以方便的导出整个卷组到另外一台机器.

缺点:

1. 在从卷组中移除一个磁盘的时候必须使用 reducevg 命令
(这个命令要求 root 权限, 并且不允许在快照卷组中使用).
2. 当卷组中的一个磁盘损坏时, 整个卷组都会受到影响.
3. 因为加入了额外的操作, 存储性能受到影响.
4. 不能减小文件系统大小 (受文件系统类型限制).

使用 LVM 将获得更好的可扩展性和可操作性, 但却损失了可靠性和存储性能,
总的说来就是在这两者间选择.

## 挂载 LVM 类型的设备

需要先激活再挂载.

```sh
# 直接挂载会出错
mount /dev/sdb1 /mnt
#mount: unknown filesystem type 'LVM2_member'

# 查看 PV
pvs

# 查看 VG
vgs

# 查看 LV
lvdisplay
# LV Status 为 unenable (表示未激活)

# 激活 LV
vgchange -ay /dev/lvname
# or
lvm vgchange -ay

# 挂载 LV
mkdir -v /data
mount /dev/vgname/lvname /data

# 或使用 lv 的 UUID 来挂载
blkid /dev/vgname/lvname
vim /etc/fstab
```
