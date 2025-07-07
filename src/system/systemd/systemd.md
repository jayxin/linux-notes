<!-- vim-markdown-toc GFM -->

* [Systemd](#systemd)
    * [系统管理工具](#系统管理工具)
        * [systemctl](#systemctl)
        * [systemd-analyze](#systemd-analyze)
        * [hostnamectl](#hostnamectl)
        * [localectl](#localectl)
        * [timedatectl](#timedatectl)
        * [loginctl](#loginctl)
    * [Unit](#unit)
        * [查看 Unit](#查看-unit)
            * [Unit status](#unit-status)
        * [Unit Management](#unit-management)
        * [List Unit Dependencies](#list-unit-dependencies)
        * [Unit Configuration](#unit-configuration)
            * [Introduction](#introduction)
            * [配置文件的状态](#配置文件的状态)
            * [配置文件的格式](#配置文件的格式)
            * [配置文件的区块](#配置文件的区块)
                * [Unit Block](#unit-block)
                * [Install Block](#install-block)
                * [Service Block](#service-block)
                * [Socket Block](#socket-block)
                * [Timer Block](#timer-block)
                * [Path Block](#path-block)
                * [Mount Block](#mount-block)
                * [Swap Block](#swap-block)
                * [Slice Block](#slice-block)
                * [Scope Block](#scope-block)
        * [Override unit configuration](#override-unit-configuration)
    * [Target](#target)
    * [日志管理](#日志管理)

<!-- vim-markdown-toc -->

# Systemd

![systemd architecture](./img/systemd.png)

Systemd 的主要功能:

- 系统引导流程 (init)
- 守护进程管理 (daemon)
- 日志收集 (journal)
- 挂载和自动挂载 (mount/automount)
- 定时任务 (timer)
- 网络管理 (networkd)

每个 systemd 管理的服务, 都有一个对应的 Unit 文件, 后缀
`.service`, 位于 `/etc/systemd/system/` 或
`/lib/systemd/system`.

Help:

```sh
man -k systemd
man systemd.unit
man systemd.service
man systemd.timer
man systemd.exec

# 所有可用字段的索引表
man systemd.directives
```

## 系统管理工具

### systemctl

系统管理.

```sh
# restart
systemctl reboot

# power off
systemctl poweroff

# halt
systemctl halt

# suspend
systemctl suspend

# sleep
systemctl hibernate

# hybrid-sleep
systemctl hybrid-sleep

# rescue mode
systemctl rescue
```

### systemd-analyze

查看启动耗时.

```sh
# 查看启动耗时
systemd-analyze

# 查看每个服务的启动耗时
systemd-analyze blame

# 显示瀑布状的启动过程流
systemd-analyze critical-chain

# 显示指定服务的启动流
systemd-analyze critical-chain atd.service
```

### hostnamectl

查看主机信息.

```sh
hostnamectl

# 设置主机名
hostnamectl set-hostname rocky-linux
```

### localectl

本地化设置.

```sh
# 查看本地化设置
localectl

# 设置本地化参数
localectl set-locale LANG=en_US.utf8
localectl set-keymap en_US
```

### timedatectl

时区设置.

```sh
timedatectl

# 显示所有可用的时区
timedatectl list-timezones

# 设置当前时区
timedatectl set-timezone Asia/Shanghai
timedatectl set-time YYYY-MM-DD
timedatectl set-time HH:MM:SS
```

### loginctl

查看当前登录的用户.

```sh
# 列出当前 session
loginctl list-sessions

# 列出当前登录用户
loginctl list-users

# 列出指定用户的信息
loginctl show-user root
```

## Unit

Systemd 可以管理所有系统资源. 不同的资源统称为 Unit.

不同类型的 Unit:

- Service Unit: 系统服务
- Target Unit: 多个 Unit 构成的一个组
- Device Unit: 硬件设备
- Mount Unit: 文件系统的挂载点
- Automount Unit: 自动挂载点
- Path Unit: 文件或路径
- Scope Unit: 不是由 Systemd 启动的外部进程
- Slice Unit: 进程组
- Snapshot Unit: Systemd 快照, 可以切回某个快照
- Socket Unit: IPC 的 socket
- Swap Unit: Swap Space
- Timer Unit: 定时器

### 查看 Unit

```sh
# 查看正在运行的 Unit
systemctl list-units

# 列出所有 Unit, 包括没有找到配置文件或启动失败的
systemctl list-units --all

# 列出所有没有运行的 Unit
systemctl list-units --all --state=inactive

# 列出所有加载失败的 Unit
systemctl list-units --failed

# 列出所有正在运行的, 类型为 service 的 Unit
systemctl list-units --type=service
```

#### Unit status

查看系统状态和单个 Unit 的状态.

```sh
# 系统状态
systemctl status

# 查看某个 Unit 的状态
systemctl status bluetooth.service

# 显示远程主机的某个 Unit 状态
systemctl -H root@rhel7.example.org status httpd.service

systemctl is-active application.service

systemctl is-failed application.service

# 是否开机自启
systemctl is-enabled application.service
```

### Unit Management

```sh
systemctl start ssh.service

systemctl stop ssh.service

systemctl restart ssh.service

systemctl kill ssh.service

# reload configuration
systemctl reload ssh.service

# reload configuration
systemctl daemon-reload

systemctl show httpd.service

systemctl show -p CPUShares httpd.service

systemctl set-property httpd.service CPUShares=500
```

### List Unit Dependencies

```sh
systemctl list-dependencies ssh.service

systemctl list-dependencies --all ssh.service
```

### Unit Configuration

#### Introduction

每个 Unit 都有一个配置文件, 告诉 Systemd 如何启动 Unit.

Systemd 默认从 `/etc/systemd/system/` 读取配置文件, 里面存放的大部分文件是
符号链接, 指向 `/usr/lib/systemd/system`.

`systemctl enable`(开机启动) 用于在上面两个目录间建立符号链接关系.

```sh
systemctl enable ssh.service
# IFF
ln -s /usr/lib/systemd/system/ssh.service \
    /etc/systemd/system/multi-user.target.wants/ssh.service
```

`systemctl disable` 用于在两个目录之间, 撤销符号链接关系.

```sh
systemctl disable ssh.service
```

配置文件的后缀, 就是该 Unit 的种类, 比如 `sshd.socket`.
若省略, Systemd 默认后缀为 `.service`.

#### 配置文件的状态

```sh
# List all config files
systemctl list-unit-files

systemctl list-unit-files --type=service
```

配置文件的可能状态:

- `enabled`: 已建立启动链接.
- `disabled`: 未建立启动链接.
- `static`: 该配置文件没有 `[Install]` 部分(无法执行), 只能作为其他配置文件的依赖.
- `masked`: 该配置文件被禁止建立启动链接.

从配置文件的状态无法看出, 该 Unit 是否正在运行, 要用 `systemctl status` 查看.

一旦修改配置文件, 就要让 Systemd 重新加载配置文件, 然后重新启动.

```sh
systemctl daemon-reload
systemctl restart httpd.service
```

#### 配置文件的格式

```sh
# 查看配置文件内容
systemctl cat ssh.service
```

配置文件的区块名和字段名, 都是大小写敏感的.

每个区块内部都是一些用等号连接的键值对, 等号两侧不能有空格(类似 `shell` 的语法).

#### 配置文件的区块

##### Unit Block

`[Unit]` 通常是配置文件的第一个区块, 用于定义 Unit 的元数据, 配置和其他 Unit
的关系. 主要有如下字段:

- `Description`: 对该 Unit 的描述, 用于 `systemctl status` 显示.
- `Documentation`: 指定帮助文档链接.
- `Requires`: 当前 Unit 依赖的其他 Unit, 若它们没有运行, 当前 Unit 会启动失败.
- `Wants`: 和当前 Unit 配合的其他 Unit, 若它们没有运行, 当前 Unit 不会启动失败.
- `BindsTo`: 和 `Requires` 类似, 它指定的 Unit 若退出, 会导致当前 Unit 停止运行.
- `Before`: 若该字段指定的 Unit 也要启动, 则必须在当前 Unit 之后启动.
- `After`: 若该字段指定的 Unit 也要启动, 则必须在当前 Unit 之前启动.
- `Conflicts`: 该字段指定的 Unit 不能和当前 Unit 同时运行.
- `Condition...`: 当前 Unit 运行必须满足的条件, 否则不会运行.
- `Assert...`: 当前 Unit 运行必须满足的条件, 否则会报启动失败.

##### Install Block

`[Install]` 通常是配置文件的最后一个区块, 用来定义如何启动, 及是否开机启动.

- `WantedBy`: 它的值是一个或多个 `Target`, 当前 Unit 激活(enable) 时符号链接会
放入 `/etc/systemd/system/` 下面以 `Target` 名 + `.wants` 后缀构成的子目录中.
- `RequiredBy`: 它的值是一个或多个 `Target`, 当前 Unit 激活时, 符号链接会放入
`/etc/systemd/system/` 下面以 `Target` 名 + `.required` 后缀构成的子目录中.
- `Alias`: 当前 Unit 可用于启动的别名.
- `Also`: 当前 Unit 激活时, 会被同时激活的其他 Unit.

##### Service Block

`[Service]` 区块用来配置 `Service`, 只有 `Service` 类型的 Unit 才有这个区块.

- `Type`: 定义启动时的进程行为.
    - `Type=simple`: 默认值, 执行 `ExecStart` 指定的命令, 启动主进程.
    - `Type=forking`: 以 fork 方式从父进程创建子进程, 创建后父进程会立即退出,
        通常用于守护进程.
    - `Type=oneshot`: 一次性进程, Systemd 会等待当前服务退出, 再继续往下执行.
    - `Type=dbus`: 当前服务通过 D-Bus 启动.
    - `Type=notify`: 当前服务启动完毕, 会通知 Systemd, 再继续往下执行.
    - `Type=idle`: 若有其他任务执行完毕, 当前服务才会运行.
- `ExecStart`: 启动当前服务的命令.
- `ExecStartPre`: 启动当前服务之前执行的命令.
- `ExecStartPost`: 启动当前服务之后执行的命令.
- `ExecReload`: 重启当前服务时执行的命令.
- `ExecStop`: 停止当前服务时执行的命令.
- `ExecStopPost`: 停止当前服务之后执行的命令.
- `RestartSec`: 自动重启当前服务间隔的秒数.
- `Restart`: 定义何种情况 Systemd 会自动重启当前服务.
    - `no`: 不重启
    - `always`: 总是重启
    - `on-success`: 成功退出时重启
    - `on-failure`: 非 0 退出时重启
    - `on-abnormal`: 信号或 core dump 时重启
    - `on-abort`
    - `on-watchdog`
- `TimeoutSec`: 定义 Systemd 停止当前服务之前等待的秒数.
- `Environment`: 指定环境变量.
- `EnvironmentFile`: 指定环境变量的文件.
- `RemainAfterExit`: 适用于 `oneshot` 类型, 服务执行后是否保持
    激活状态.
- `TimeoutStartSec`, `TimeoutStopSec`: 服务启停的超时时间.
- `User`, `Group`: 服务运行的用户和组.
- `WorkingDirectory`: 工作目录.
- `StandardOutput`, `StandardError`: 设置日志输出,
    如 `journal`, `syslog`, `null`, `tty` 等.
- `LimitNOFILE`: 文件描述符限制.
- `CapabilityBoundingSet`: 限制服务拥有的 Linux Capabilities.

##### Socket Block

`[Socket]` 用于套接字激活设置.

常用字段:

- `ListenStream`: tcp 或者 unix socket, 如 `8080`, `/run/ssh.sock`.
- `SocketMode`: socket 权限, 如 `0660`.
- `Accept`: 是否接受多个连接.

##### Timer Block

`[Timer]` 用于定时任务调度.

##### Path Block

`[Path]` 是文件或目录监控触发器.

常用字段:

- `PathExists`: 监控文件是否存在.
- `PathChanged`: 监控内容变化.
- `Unit`: 被触发的 unit 名称.

##### Mount Block

`[Mount]` 是文件系统挂载定义.

类似 `/etc/fstab`.

常用字段:

- `What`: 挂载源设备.
- `Where`: 挂载目标路径.
- `Type`: 文件系统类型.
- `Options`: 挂载选项.

##### Swap Block

`[Swap]` 用于 Swap 分区管理.

##### Slice Block

`[Slice]` 用于 CGroup 切片资源限制.

##### Scope Block

`[Scope]` 用于对临时进程组描述.

### Override unit configuration

覆盖某个 Unit 的配置, 比如环境变量.

```sh
systemctl edit docker.service
```

该命令会用文本编辑器打开 `/etc/systemd/system/docker.service.d/override.conf`.
可用来覆盖 Unit 中的配置, 可用于配置代理等环境变量.
需要设定 `$EDITOR` 或 `$SYSTEMD_EDITOR` 环境变量.

## Target

启动计算机时, 需启动大量 Unit. 若每次启动, 都要写明本次启动需要哪些
Unit, 显然不方便. Systemd 的解决方案是 Target.

Target 就是一个 Unit 组, 包含许多相关的 Unit. 启动某个 Target 时,
Systemd 就会启动里面所有的 Unit. 从这个意义上说, Target 这个概念类似于
"状态点", 启动某个 Target 就好比启动到某种状态.

传统的 `init` 启动模式中, 有 RunLevel 的概念, 和 Target 的作用很类似. 不同的是,
RunLevel 是互斥的, 不可能多个 RunLevel 同时启动, 但多个 Target 可以同时启动.

```sh
# 查看当前系统的所有 Target
systemctl list-unit-files --type=target

# 查看一个 Target 包含的所有 Unit
systemctl list-dependencies multi-user.target

# 查看启动时的默认 Target
systemctl get-default

# 设置启动时的默认 Target
systemctl set-default multi-user.target

# 切换 Target 时, 默认不关闭前一个 Target 启动的进程
# systemctl isolate 改变这种行为
# 关闭前一个 Target 中所有不属于后一个 Target 的进程
systemctl isolate multi-user.target
```

Target 和传统 RunLevel 的对应关系:

| Traditional RunLevel | New Target Name   |
|----------------------|-------------------|
| RunLevel 0           | poweroff.target   |
| RunLevel 1           | rescue.target     |
| RunLevel 2           | multi-user.target |
| RunLevel 3           | multi-user.target |
| RunLevel 4           | multi-user.target |
| RunLevel 5           | graphical.target  |
| RunLevel 6           | reboot.target     |

target 和 `init` 进程的主要差别:

- 默认的 RunLevel (在 `/etc/inittab` 文件设置) 现在被默认的 Target 取代,
位置是 `/etc/systemd/system/default.target`, 通常符号链接到
`graphical.target` 或者 `multi-user.target`.
- 启动脚本的位置. 以前是 `/etc/init.d/`, 符号链接到不同的 RunLevel 目录
(比如 `/etc/rc3.d/`, `/etc/rc5.d` 等), 现在放在 `/lib/systemd/system` 和
`/etc/systemd/system` 目录.
- 配置文件的位置. `init` 进程的配置文件是 `/etc/inittab`, 各种服务的配置文件
存放在 `/etc/sysconfig/`. 现在的配置文件主要存放在 `/lib/systemd/`, 在
`/etc/systemd/` 中的修改可以覆盖原始设置.

## 日志管理

`Systemd` 统一管理所有 Unit 的启动日志. 带来的好处是, 可以只用 `journalctl`
一个命令, 查看所有日志 (内核日志和应用日志). 日志的配置文件是
`/etc/systemd/journald.conf`.

```sh
# 查看所有日志 (默认情况, 只保存本次启动的日志)
journalctl

# 查看内核日志(不显示应用的)
journalctl -k

# 查看系统本次启动的日志(boot)
journalctl -b
journalctl -b -0

# 查看上一次启动的日志 (需更改设置)
journalctl -b -1

# 查看指定时间的日志
journalctl --since="2024-02-24 10:24:24"
journalctl --since="20 min ago"
journalctl --since yesterday
journalctl --since "2024-02-24" --until "2024-10-24"
journalctl --since 08:00 --until "1 hour ago"

# 显示尾部的最新 10 行日志
journalctl -n

# 显示尾部的指定行数日志
journalctl -n 20

# 实时查看最新日志
journalctl -f

# 查看指定服务的日志
journalctl /usr/lib/systemd/systemd

# 查看指定进程的日志
journalctl _PID=1

# 查看某个路径的脚本的日志
journalctl /usr/bin/bash

# 查看指定用户的日志
journalctl _UID=1000 --since today

# 查看某个 Unit 的日志
journalctl -u nginx.service
journalctl -u nginx.service --since today

# 实时查看某个 Unit 的最新日志
journalctl -u nginx.service -f

# 合并显示多个 Unit 的日志
journalctl -u nginx.service -u ssh.service --since today

# 查看指定优先级(及以上级别)的日志
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug
journalctl -p err -b

# 日志默认分页输出, 这里改为正常的标准输出
journalctl --no-pager

# JSON 格式输出
journalctl -b -u ssh.service -o json-pretty

# 显示日志占据的硬盘空间
journalctl --disk-usage

# 指定日志占据的最大空间
journalctl --vacuum-size=1G

# 指定日志保存多久
journalctl --vacuum-time=1years
```
