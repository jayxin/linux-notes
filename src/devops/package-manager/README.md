# Package Manager

## Red Hat Series

### How to extract files from a RPM package?

```sh
mkdir packagecontents; cd packagecontents

# -i extract
# -d make directories
# -m preserve modification time
# -v verbose
rpm2cpio ../foo.rpm | cpio -idmv

find .
```

### 查看 yum 安装的包包含哪些文件

```sh
yum install yum-utils -y
repoquery -l <package name>
```
