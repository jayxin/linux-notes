<!-- vim-markdown-toc GFM -->

* [GCC](#gcc)
    * [向链接器传参](#向链接器传参)
    * [静态编译案例](#静态编译案例)
    * [动态编译案例](#动态编译案例)
    * [静态库转动态库案例](#静态库转动态库案例)
    * [查看体系结构元组](#查看体系结构元组)

<!-- vim-markdown-toc -->

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

## 静态编译案例

`hello.c`:

```c
/* hello.c */
#include <stdio.h>

void hello(const char * name) {
  printf("Hello %s!\n", name);
}
```

`hello.h`:

```c
/* hello.h */
#ifndef HELLO_H
#define HELLO_H

void hello(const char * name);

#endif
```

`main.c`:

```c
/* main.c */
#include "hello.h"

int main() {
  hello("Tom");
  return 0;
}
```

`cmd.sh`:

```sh
#!/bin/bash

# Build static lib
gcc -c hello.c
ar -crv libhello.a hello.o

# Use static lib
gcc -o hello main.c -L. -lhello
# or
gcc -o hello main.c libhello.a
# or
gcc -c main.c
gcc -o hello main.o libhello.a
```

## 动态编译案例

`A.h`:

```c
/* A.h */
#ifndef A_H
#define A_H

void print1(int);
void print2(char *);

#endif
```

`A1.c`:

```c
/* A1.c */
#include <stdio.h>

void print1(int arg) {
  printf("A1 print arg: %d\n", arg);
}
```

`A2.c`:

```c
/* A2.c */
#include <stdio.h>

void print2(char * arg) {
  printf("A2 print arg: %s\n", arg);
}
```

`test.c`:

```c
/* test.c */
#include <stdlib.h>
#include "A.h"

int main() {
  print1(1);
  print2("test");
  return 0;
}
```

`cmd.sh`:

```sh
#!/bin/bash

# Build dynamic lib
gcc -c -fPIC A1.c A2.c
gcc -shared *.o -o liba.so
# or
gcc -fPIC -shared -o liba.so *.c

# Use dynamic lib
# 1. System path
sudo cp liba.so /usr/lib
gcc -o 'test' 'test.c' liba.so
# ldd test
# or
# 2. Absolute path
gcc -o test -L. -la -Wl,-rpath,$(pwd) 'test.c'
# ldd test
# or
# 3. Relative path
gcc -o test -L. -la -Wl,-rpath,. 'test.c'
# ldd test
```

## 静态库转动态库案例

`hello.h`:

```c
/* hello.h */
#ifndef HELLO_H
#define HELLO_H

void hello(const char * name);

#endif
```

`hello.c`:

```c
/* hello.c */
#include <stdio.h>

void hello(const char * name) {
  printf("Hello %s!\n", name);
}
```

`main.c`:

```c
/* main.c */
#include "hello.h"

int main() {
  hello("Tom");
  return 0;
}
```

`cmd.sh`:

```sh
#!/bin/bash

# Build static lib
gcc -fPIC -c hello.c
ar -r libhello.a hello.o

# From static to dynamic
ar -x libhello.a # Extract
gcc -shared *.o -o libhello.so
nm -D libhello.so

# Use dynamic lib
gcc -o hello -L. -lhello -Wl,-rpath,. main.c
ldd hello
```

## 查看体系结构元组

```sh
# 查看形如 <arch>-<vendor>-<os>-<libc/abi> 的元组
gcc -dumpmachine

clang -dumpmachine

rustc -vV | grep -i host

arch=$(uname -m)
os=$(uname -s | tr '[:upper:]' '[:lower:]')
libc=$(ldd --version | head -n 1 | awk '{print $3}')
echo "$arch-none-$os-$libc"
```
