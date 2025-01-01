<!-- vim-markdown-toc GFM -->

* [Tricks About Log](#tricks-about-log)
    * [记录服务器操作](#记录服务器操作)
    * [为日志添加时间戳](#为日志添加时间戳)

<!-- vim-markdown-toc -->

# Tricks About Log

## 记录服务器操作

```bash
ssh root@server | tee ssh-$(date "+%F_%T").log
```

## 为日志添加时间戳

```bash
tail -f demo.log | awk '{now=strftime("%F %T%z\t");sub(/^/,now);print}'
```
