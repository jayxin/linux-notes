# GCC

## 向链接器传参

gcc 能编译, 也能链接. 链接通过 `ld` 实现.

```sh
gcc -o test test.c -I. -L. -lc -Wl,-rpath=.
# -I: search path for header
# -L: search path for lib
# -lc: link libc
# -Wl... : for linker
```

格式:

```sh
gcc -Wl,param1,param2,param3,...
# ld 收到如下参数
ld param1 param2 param3

# 链接首选静态库
-Wl,-Bstatic

# 链接首选动态库
-Wl,-Bdynamic

-Wl,-Bstatic -la -lb -lc -Wl,-Bdynamic -ld -le
# IFF
ld liba.a libb.a libc.a libd.so libe.so

-Wl,-rpath -Wl,$HOME/lib
# IFF
ld -rpath $HOME/lib
```
