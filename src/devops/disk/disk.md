<!-- vim-markdown-toc GFM -->

* [Tricks About Disk](#tricks-about-disk)
    * [多个磁盘对拷](#多个磁盘对拷)

<!-- vim-markdown-toc -->

# Tricks About Disk

## 多个磁盘对拷

```bash
dd if=/dev/sdb bs=128k | tee >(dd of=/dev/sdc bs=128k) tee >(dd of=/dev/sde bs=128k) | dd of=/dev/sdd bs=128k
```
