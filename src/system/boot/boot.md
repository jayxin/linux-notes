# Boot

## BIOS 切换到 UEFI

### 如何判断系统是 BIOS 还是 UEFI 启动

方法1:
```bash
# 判断文件系统的挂载点, 若只有 /boot 则为 BIOS
lsblk
```

方法2:
```bash
# 判断目录是否存在, 存在则为 UEFI, 不存在为 BIOS
ls /sys/firmware/efi
```

方法3:
```bash
# 查看命令输出
efibootmgr
# BootCurrent, BootOrder --> UEFI
# EFI variables are not supported --> BIOS
```

方法4:
```bash
dmesg | grep -i efi
```

方法5:
```bash
ls /boot/efi
```

方法6:
```bash
# 使用 systemd 提供的工具
sudo bootctl status
# Boot Loader: systemd-boot/Boot Loader: EFI --> UEFI
# Boot Loader: BIOS --> BIOS
```

### 切换到 UEFI

**先备份数据**!

1. 确认磁盘是否有剩余可分配空间(Free Space), 至少 100MB, 因为 GPT 格式的
    的分区表需要额外的预留空间.
    ```bash
    parted /dev/sda unit MB print free
    ```
2.  分区.
    ```bash
    dnf install gdisk -y
    gdisk /dev/sda
    # n 新增一个分区
    # 分区号默认递增
    # 第一个扇区的位置默认
    # 最后一个扇区设置为分区大小: +100M
    # 分区类型(EFI分区)代码为: ef00
    # 写入分区表: w
    ```
3.  查看分区结果.
    ```bash
    # 刷新分区信息
    partprobe

    # 查看结果
    parted /dev/sda print | grep -i "partition table"
    ```
4. 将新增的分区(假设为`/dev/sda3`)格式化为 EFI 分区所支持的文件系统格式:
    ```bash
    dnf install dosfstools -y
    mkfs.vfat /dev/sda3

    # 查看结果
    lsblk --fs
    ```
5. 分区挂载:
    ```bash
    mkdir -p /boot/efi # 这个目录由于存放 EFI 的所有组件
    mount /dev/sda3 /boot/efi
    ```
6. 生成 UEFI 启动所需文件:
    ```bash
    # 安装完成后, 组件就会被释放到 /boot/efi 下
    dnf install shim-* grub2-efi-*
    ```
7. 安装引导加载程序:
    ```bash
    grub2-install --target=x86_64-efi /dev/sda
    # 此处会看到报错, 忽略即可
    ```
8. 重新生成 `grub.cfg` 配置文件到UEFI对应目录下:
    ```bash
    grub2-mkconfig >/boot/efi/EFI/rocky/grub.cfg
    ```
9. 为新增分区配置持久化挂载:
    ```bash
    echo 'UUID=AAF8-7DB3 /boot/efi vfat defaults 0 0' >>/etc/fstab
    ```
10. 将BIOS固件切换到UEFI, 重启.
11. 验证是否切换成功:
    ```bash
    ls /sys/firmware/efi/
    # or
    efibootmgr
    ```
