<!-- vim-markdown-toc GFM -->

* [ELF](#elf)
    * [修改可执行程序或者库的动态链接库的路径](#修改可执行程序或者库的动态链接库的路径)
        * [chrpath](#chrpath)
        * [patchelf](#patchelf)
        * [gcc 指定 rpath 选项](#gcc-指定-rpath-选项)
        * [cmake 指定 rpath](#cmake-指定-rpath)
        * [环境变量 LD_LIBRARY_PATH](#环境变量-ld_library_path)

<!-- vim-markdown-toc -->

# ELF

## 修改可执行程序或者库的动态链接库的路径

`rpath` (library runtime path) 参数用于指定库运行时首先加载系统依赖库的路径,
若找不到则去系统默认路径查找.

```sh
# 查看 ELF 文件的信息
readelf -d test
readelf -d libc.so.6

# 查看程序或者动态库依赖的库路径
ldd test # test 是可执行文件
ldd libc.so.6 # 动态链接库
```

修改 rpath 的方法.

1. chrpath (编译后).
2. patchelf (编译后).
3. gcc 指定 `rpath` 编译选项 (编译时).
4. cmake 指定 `rpath` (编译时).
5. 编译时设置环境变量.

### chrpath

```sh
sudo yum install chrpath -y

# 显示 rpath 路径
chrpath -l xxx.so
chrpath -l test

# 修改 rpath
chrpath -r ./lib xxx.so
chrpath -r ./lib test
```

### patchelf

```sh
sudo apt install patchelf

patchelf --set-rpath ./lib xxx.so
```

### gcc 指定 rpath 选项

```sh
gcc -o test test.c -I. -L. -lc -Wl,-rpath=.
```

### cmake 指定 rpath

```cmake
IF (UNIX)
    set(CMAKE_SKIP_BUILD_RPATH TRUE)
    set(CMAKE_CXX_FLAGS "-Wl,-z,origin,-rpath,$ORIGIN")
ENDIF()
```

or 

```cmake
set(CMAKE_SKIP_BUILD_RPATH FALSE) # 编译时加上 rpath
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE) # 编译时 rpath 不使用安装的 rpath
set(CMAKE_INSTALL_RPATH "./") # 安装 rpath 为当前目录
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH FALSE) # 安装的执行文件不加上 rpath
```

### 环境变量 LD_LIBRARY_PATH

```sh
export LD_LIBRARY_PATH="{$LD_LIBRARY_PATH}:${HOME}/lib"
```

设置完环境变量后进行编译.
