# Systemd 定时器

定时任务, 就是未来的某个或多个时间点, 预定执行的任务, 比如每 5 分钟收一次邮件.

Linux 通常用 `cron` 设置定时任务, 但 Systemd 也能做到, 但 Systemd 有如下优点:

- 自动生成日志, 配合 Systemd 的日志工具, 方便排错.
- 可以设置内存和 CPU 使用额度, 比如最多使用 50% 的 CPU.
- 任务可以拆分, 依赖其他 Systemd 单元, 完成非常复杂的任务.

## 例子-每小时发送一封电子邮件

### 邮件脚本

需要安装 `ssmtp` 或 `msmtp`.

```sh
#!/usr/bin/env bash

# mail.sh
echo "Body content" | /usr/bin/mail -s "Subject" someone@example.org
```

### Systemd Unit

每个单元都有一个单元描述文件, 它们分散在 3 个目录.

- `/lib/systemd/system`: 系统默认的单元文件.
- `/etc/systemd/system`: 用户安装的软件的单元文件.
- `/usr/lib/systemd/system`: 用户自定义的单元文件.

```sh
# 查看所有 timer unit
systemctl list-unit-files --type timer
```

#### Service Unit

```sh
cd /usr/lib/systemd/system/

vim mytimer.service
#[Unit]
#Description=MyTimer
#
#[Service]
#ExecStart=/bin/bash /path/to/mail.sh
```

定义时, 所有路径都要写成绝对路径, 否则 Systemd 会找不到.

```sh
# 启动 Unit
systemctl start mytimer.service
```

#### Timer Unit

Service Unit 只定义了如何执行任务, 要定时执行, 还需要定义 Timer Unit.

```sh
cd /usr/lib/systemd/system/

vim mytimer.timer
#[Unit]
#Description=Runs my timer every hour
#
#[Timer]
#OnUnitActiveSec=1h
#Unit=mytimer.service
#
#[Install]
#WantedBy=multi-user.target
```

`[Timer]` 部分可以有的字段:

- `OnActiveSec` 定时器生效后, 多少时间开始执行任务.
- `OnBootSec` 系统启动后, 多少时间开始执行任务.
- `OnStartupSec` Systemd 进程启动后, 多少时间开始执行任务.
- `OnUnitActiveSec` 该 Unit 上次执行后, 等多少时间再次执行.
    - `OnUnitActiveSec=1h` 1小时执行一次任务.
    - `OnUnitActiveSec=*-*-* 02:00:00` 每天凌晨两点执行.
    - `OnUnitActiveSec=Mon *-*-* 02:00:00` 每周一凌晨两点执行.
- `OnUnitInactiveSec` 该 Unit 上次关闭后多少时间, 再次执行.
- `OnCalendar` 基于绝对时间, 而非相对时间执行.
- `AccuracySec` 若因为各种原因, 任务必须推迟执行, 推迟的最大秒数, 默认 60 秒.
- `Unit`: 要执行的任务, 默认是同名的带有 `.service` 后缀的单元.
- `Persistent`: 若设置了该字段, 即使定时器到时没有启动, 也会自动执行相应的单元.
- `WakeSystem`: 若系统休眠, 是否自动唤醒系统.

`[Instal]` 定义开机自启 (`systemctl enable`) 和关闭开机自启 (`systemctl
disable`) 这个单元时, 执行的命令.

执行 `systemctl enable mytimer.timer` 命令时, 相当于在 `multi-user.target.wants`
目录中创建一个符号链接, 指向 `mytimer.timer`.

### 定时器管理

```sh
# 启动定时器
systemctl start mytimer.timer

# 查看定时器状态
systemctl status mytimer.timer

# 查看所有正在运行的定时器
systemctl list-timers

# 关闭定时器
systemctl stop mytimer.timer

# 开机自启
systemctl enable mytimer.timer

# 开机不自启
systemctl disable mytimer.timer

# 查看定时器日志
journalctl -u mytimer.timer

# 查看 mytiemr.timer 和 mytimer.service 的日志
journalctl -u mytimer

# 查看最新日志 (f: follow)
journalctl -f -u mytimer.timer
```
