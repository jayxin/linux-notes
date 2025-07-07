<!-- vim-markdown-toc GFM -->

* [Tricks About Accounting](#tricks-about-accounting)
    * [查看系统是何时安装的](#查看系统是何时安装的)
    * [find 权限匹配](#find-权限匹配)
    * [查看命令执行后那些文件被修改](#查看命令执行后那些文件被修改)
    * [查看内存占用](#查看内存占用)
    * [可视化查看系统启动时的性能](#可视化查看系统启动时的性能)
    * [正确修改系统时间](#正确修改系统时间)
    * [查看操作系统位数](#查看操作系统位数)
    * [查看 CPU 型号](#查看-cpu-型号)
    * [查看系统架构](#查看系统架构)

<!-- vim-markdown-toc -->

# Tricks About Accounting

## 查看系统是何时安装的

```bash
ls -lct /etc/ | tail -1 | awk '{print $6, $7, $8}'
```

## find 权限匹配

```bash
# 精确匹配指定的权限
find . -perm 111

# 3个权限中的任意一个匹配指定的权限(or)
find . -perm /111

# 3个权限位都必须至少匹配指定的权限(and)
find . -perm -111
```

## 查看命令执行后那些文件被修改

```bash
T="$(date "+%F %T.%N")"; ./rsync.sh; find / -xdev -newermt "$T"
```

## 查看内存占用

```bash
ps -eo pid,ppid,%mem,%cpu,cmd --sort=-%mem | head
ps aux --sort -rss | head -n 10
ps aux --sort=-%mem | awk '{print $6/1024 " MB", $11}' | head -n 11
ps aux --sort=-%mem | awk '{print $2, $6/1024 " MB", $11}' | head -n 11
```

## 可视化查看系统启动时的性能

```bash
systemd-analyze plot >boot.svg
```

## 正确修改系统时间

三种时钟:

- Real Time Clock(RTC): 硬件时钟
- Local Time: 系统本地时钟, 使用 `date` 修改的是本地时钟, 不会改变硬件时钟, 重启后系统时间会被硬件时钟覆盖.
本地时间由 UTC 时间加上时区偏移量计算得到.
- UTC Time: 系统UTC时钟

```bash
# 同时查看三种时间, timedatectl 是 systemd 的管理工具
timedatectl

# Method 1
# 设置系统时间的同时同步到硬件时钟
date -s "2024-01-01 00:00:00" && hwclock -w

# Method 2
# 该命令会自动同步 RTC 时间
timedatectl set-time "2024-01-01 00:00:00"
```

标准:

- Local time 时间正确
- UTC 和 RTC 时间一致

## 查看操作系统位数

```sh
getconf LONG_BIT
```

## 查看 CPU 型号

```sh
lscpu | grep -i 'model name'
```

## 查看系统架构

```sh
# print all info
uname -a
uname --all

# kernel name
uname -s
uname --kernel-name

# network node hostname
uname -n
uname --nodename

# kernel release
uname -r
uname --kernel-release

# kernel version
uname -v
uname --kernel-version

# machine hardware name
uname -m
uname --machine

# processor type (non-portable)
uname -p
uname --processor

# hardware platform (non-portable)
uname -i
uname --hardware-platform

# OS
uname -o
uname --operating-system
```
