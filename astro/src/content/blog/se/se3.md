---
title: "符号执行(3) - 与库函数和操作系统交互"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

符号执行有两个主要的问题：一个就是爆炸的状态中进行搜索，也就是上节我们讨论过的时间和策略的问题；另一个就是模拟环境的问题。
一个应用程序总是要调用库函数和操作系统调用的，如果不对这两部分进行建模，那么对于实际的应用程序的限制就太大了。

Klee工具的优势就是不仅在研究上有建树，而且在工程落地上下了很多工夫，比如提供了posix runtime的支持，这样对操作系统的调用就可以被识别出来。

## 编译支持c库函数的klee

针对c库函数，klee基于uclibc库进行建模，对c语言库提供了良好的支持。

为了支持klee-uclibc，我们需要下载和编译它的代码：
```
git clone https://github.com/klee/klee-uclibc.git  
cd klee-uclibc  
./configure --make-llvm-lib  
make 
```

系统会通过wget去下载uclibc的包，所以如果是没有wget工具的需要先安装下。然后会编译成llvm的库。

安装好uclibc的代码之后，我们就可以编译带uclibc库的klee了，通过KLEE_UCLIBC_PATH参数来指定：
```
cmake \
  -DENABLE_SOLVER_STP=ON \
  -DENABLE_POSIX_RUNTIME=ON \
  -DENABLE_KLEE_UCLIBC=ON \
  -DKLEE_UCLIBC_PATH=/workspace/xulun/github/ai/klee-uclibc/ \
  ..
```

系统越来越复杂了，稳妥起见，我们加上单元测试和集成测试来保证我们自己的质量。别测别人的代码，结果自己出问题了就不好玩了。

我们用googletest作为测试框架，先下载解压：
```
wget -c https://github.com/google/googletest/archive/release-1.7.0.zip
unzip release-1.7.0.zip
```

然后我们加上ENABLE_UNIT_TESTS=ON和GTEST_SRC_DIR=
```
cmake \
  -DENABLE_SOLVER_STP=ON \
  -DENABLE_POSIX_RUNTIME=ON \
  -DENABLE_KLEE_UCLIBC=ON \
  -DKLEE_UCLIBC_PATH=/workspace/xulun/github/ai/klee-uclibc/ \
  -DENABLE_UNIT_TESTS=ON \
  -DGTEST_SRC_DIR=/workspace/xulun/github/ai/googletest-release-1.7.0/ \
  ..
```

现在就可以执行编译了：
```
make
```

编译好了之后，我们运行下测试吧。
- 单元测试：make unittests
- 系统测试：make systemtests

单元测试运行结果是这样的：
```
make unittests
[  2%] Built target gtest
[ 10%] Built target kleeSupport
[ 12%] Built target gtest_main
[ 13%] Built target RNGTest
[ 15%] Built target kleeBasic
[ 31%] Built target kleaverExpr
[ 52%] Built target kleaverSolver
[ 53%] Built target AssignmentTest
[ 56%] Built target ExprTest
[ 58%] Built target RefTest
[ 60%] Built target Z3SolverTest
[ 62%] Built target SolverTest
[ 74%] Built target kleeModule
[ 91%] Built target kleeCore
[ 92%] Built target SearcherTest
[ 93%] Built target TreeStreamTest
[ 96%] Built target DiscretePDFTest
[ 98%] Built target TimeTest
Scanning dependencies of target unittests
[100%] Running unittests

Testing Time: 1.70s
  Expected Passes    : 36
[100%] Built target unittests
```

## 从libc到posix的模拟

下面我们以读文件为例，看看有libc和posix与没有他们的差别，和对生成测试用例的差别。

### 没有libc的情况

代码如下：
```c
#include <stdio.h>
#include "klee/klee.h"

int fileo(int n){
    if(n<=0){
        return -1;
    }
    FILE* fp = fopen("data.dat","r+");
    if(fp!=NULL){
        if(n>10){
            return -2;
        }
        char data[10];
        int nCount = fread(data,1,n,fp);
        fclose(fp);
        return nCount;
    }else{
        return -3;
    }
}

int main(){
    int a;
    klee_make_symbolic(&a,sizeof(a),"a");
    return(fileo(a));
}
```

data.dat文件中存了10个字符，n是从1到10的字符数。如果n<=0没意义，返回-1，如果文件可以打开，但是n>10，返回-2，如果打开文件失败返回-3。如果在1到10之间，则返回读取的字节数。

我们还是先编译成llvm指令：
```
clang -emit-llvm -c file.c
```

我们还是跟之前一样，直接无参数用klee去生成测试用例：
```
klee file.bc
KLEE: output directory is "/workspace/xulun/github/libs/klee-out-37"
KLEE: Using STP solver backend
KLEE: WARNING: undefined reference to function: fclose
KLEE: WARNING: undefined reference to function: fopen
KLEE: WARNING: undefined reference to function: fread
KLEE: WARNING ONCE: calling external: fopen(94890914939824, 94890915168664) at [no debug info]
KLEE: ERROR: (location information missing) external call with symbolic argument: fread
KLEE: NOTE: now ignoring this error at this location

KLEE: done: total instructions = 39
KLEE: done: completed paths = 3
KLEE: done: generated tests = 3
```

我们看下生成的测试case，分别是0，1和最大整数：
```
[root@615cce8a2248 libs]# ktest-tool ./klee-out-37/test000001.ktest
ktest file : './klee-out-37/test000001.ktest'
args       : ['file.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 4
object 0: data: b'\x00\x00\x00\x00'
object 0: hex : 0x00000000
object 0: int : 0
object 0: uint: 0
object 0: text: ....
[root@615cce8a2248 libs]# ktest-tool ./klee-out-37/test000002.ktest
ktest file : './klee-out-37/test000002.ktest'
args       : ['file.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 4
object 0: data: b'\x01\x00\x00\x00'
object 0: hex : 0x01000000
object 0: int : 1
object 0: uint: 1
object 0: text: ....
[root@615cce8a2248 libs]# ktest-tool ./klee-out-37/test000003.ktest
ktest file : './klee-out-37/test000003.ktest'
args       : ['file.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 4
object 0: data: b'\xff\xff\xff\x7f'
object 0: hex : 0xffffff7f
object 0: int : 2147483647
object 0: uint: 2147483647
object 0: text: ....
```

另外，我们执行的指令数是39个。因为库函数的调用没有被识别出来。

### 加上libc的情况

下面我们给klee指令libc为前面我们编译的uclibc，命令行为：
```
klee --libc=uclibc file.bc
```

运行结果如下：

```
KLEE: NOTE: Using klee-uclibc : /workspace/xulun/github/ai/klee/build/Debug+Asserts/lib/klee-uclibc.bca
KLEE: output directory is "/workspace/xulun/github/libs/klee-out-38"
KLEE: Using STP solver backend
KLEE: WARNING: undefined reference to function: __syscall_rt_sigaction
KLEE: WARNING: undefined reference to function: close
KLEE: WARNING: undefined reference to function: fcntl
KLEE: WARNING: undefined reference to function: fstat
KLEE: WARNING: undefined reference to function: ioctl
KLEE: WARNING: undefined reference to function: lseek64
KLEE: WARNING: undefined reference to function: mkdir
KLEE: WARNING: undefined reference to function: open
KLEE: WARNING: undefined reference to function: open64
KLEE: WARNING: undefined reference to function: read
KLEE: WARNING: undefined reference to function: sigprocmask
KLEE: WARNING: undefined reference to function: stat
KLEE: WARNING: undefined reference to function: write
KLEE: WARNING: undefined reference to function: kill (UNSAFE)!
KLEE: WARNING: executable has module level assembly (ignoring)
KLEE: WARNING ONCE: calling external: ioctl(0, 21505, 94240056457072) at libc/termios/tcgetattr.c:43 12
KLEE: WARNING ONCE: calling __user_main with extra arguments.
KLEE: WARNING ONCE: Alignment of memory from call "malloc" is not modelled. Using alignment of 8.
KLEE: WARNING ONCE: calling external: open(94240043179840, 2, 438) at libc/stdio/_fopen.c:146 8
KLEE: ERROR: libc/stdio/_READ.c:47: external call with symbolic argument: read
KLEE: NOTE: now ignoring this error at this location

KLEE: done: total instructions = 12625
KLEE: done: completed paths = 3
KLEE: done: generated tests = 3
```

我们从日志中看到，虽然还是只生成3个用例，但是执行的指令数从之前的39个变成了现在的12625个。

导致libc调用分支不能被识别的原因是external call with symbolic argument: read，就是系统调用read没有被实现。

也就是说，光有libc库还不够，因为libc的实现依赖操作系统的调用。

### 增加对操作系统的建模

最终的终极大招，klee帮我们提供了一个符合posix标准的运行时，我们通过--posix-runtime参数来调用posix runtime。
我们看下效果，命令行如下：
```
klee --libc=uclibc --posix-runtime file.bc
```

运行结果如下：
```
KLEE: NOTE: Using POSIX model: /workspace/xulun/github/ai/klee/build/Debug+Asserts/lib/libkleeRuntimePOSIX.bca
KLEE: NOTE: Using klee-uclibc : /workspace/xulun/github/ai/klee/build/Debug+Asserts/lib/klee-uclibc.bca
KLEE: output directory is "/workspace/xulun/github/libs/klee-out-41"
KLEE: Using STP solver backend
KLEE: WARNING: executable has module level assembly (ignoring)
KLEE: WARNING ONCE: calling external: syscall(16, 0, 21505, 94333716546160) at runtime/POSIX/fd.c:1007 10
KLEE: WARNING ONCE: Alignment of memory from call "malloc" is not modelled. Using alignment of 8.
KLEE: WARNING ONCE: calling __klee_posix_wrapped_main with extra arguments.

KLEE: done: total instructions = 15619
KLEE: done: completed paths = 3
KLEE: done: generated tests = 3
```

我们看到，比起只有uclibc库时的12625条指令，现在指令数增加到15619个，而且对于posix系统调用也可以模拟了。

## 调用其它库

除了系统调用和libc，程序可能还用到了其它库。
没有关系，只要我们把相关库的llvm字节码通过-link-llvm-lib参数引入进来就好。link-llvm-lib参数可以使用多次，有一个库算一个就好。

例：
```
klee -link-llvm-lib=libhelper.so.bc -link-llvm-lib=libhelper2.so.bc test.bc
```
