---
title: "操作系统形式化验证实践教程(1) - 证明第一个定理"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

# 符号执行(1) - 自动生成覆盖率用例之利器

对于安全性要求比较高的软件，为了防止出现安全漏洞，我们不得不花大量时间写更多的测试用例来提升覆盖率。尤其是高可靠性软件需要的修正条件判定覆盖MC/DC(Modified Condition/Decision Coverage)，更是要多花不少心思。

全靠手工写，工作量太大，而且重复性工作不少。靠模糊测试命中的话效率又比较低。
那么，有没有什么办法可以将这些机械的工作做得自动化一点，机器能够帮我们设计一些测试用例呢？符号执行就是一种可用的利器。

## 什么是符号执行

为了避免有同学望文生义，我们先解释下符号执行的含义。符号执行是借助程序的形式化语义来分析代码的一种方法，具体地说，不考虑循环的情况下，符号执行就是求解霍尔逻辑的最弱前置条件。
这里面主要的工具除了霍尔逻辑的公理外，主要还会用到可满足性模理论SMT工具。后面我们讲符号执行工具klee时大家就会看到，相当多的步骤其实我们是在准备SMT工具。

有个简单的概念之后，我们迅速进入通过例子学习的阶段。对于跟安全性打交道不多的同学来说，完全不懂Hoare Logic, SMT, SAT这些概念不影响使用符号执行工具来帮我们找出一些测试用例。

## 通过例子学习klee符号执行

下面我们就以klee为例来讲解下如何在工作中使用符号执行来帮我们生成测试用例。

klee是用于C/C++的符号执行工具，也有达人研究中应用于Rust等语言的用法。通过后面的例子可以看到，只要能生成llvm byte code，应该都有办法来执行。

我们先写个待测函数，将百分制的分数映射成ABCD等级：
```c
char testscore(int score)
{
    if (score > 100)
    {
        return 'E';
    }
    else if (score < 0)
    {
        return 'E';
    }
    else if (score >= 90)
    {
        return 'A';
    }
    else if (score >= 80)
    {
        return 'B';
    }
    else if (score >= 60)
    {
        return 'C';
    }
    else
    {
        return 'D';
    }
}
```

既然是符号执行，我们写测试用例时不给具体值，只给一个符号，然后让klee帮我们去找该测什么值。这通过klee_make_symbolic函数来实现，我们给上面的testscore写个main函数来调用：
```c
int main()
{
    int score;
    klee_make_symbolic(&score, sizeof(score), "score");
    return (testscore(score));
}
```

我们用clang来编译它，生成llvm中间代码testscore.bc:
```
clang -emit-llvm -c testscore.c
```

然后我们就调用klee去自动执行上一步编译出的字节码:
```
klee testscore.bc
```

输出如下：

```
KLEE: output directory is "/workspace/xulun/github/libs/klee-out-3"
KLEE: Using STP solver backend

KLEE: done: total instructions = 61
KLEE: done: completed paths = 6
KLEE: done: generated tests = 6
```

以上说明，总共61条指令，klee为我们发现了6个分支，并生成了覆盖这6个分支的测试用例。

我们通过klee-stats工具来看下生成的用例的覆盖率：
```
----------------------------------------------------------------------------
|    Path     |  Instrs|  Time(s)|  ICov(%)|  BCov(%)|  ICount|  TSolver(%)|
----------------------------------------------------------------------------
|./klee-out-3/|      61|     0.05|   100.00|   100.00|      41|       96.17|
----------------------------------------------------------------------------
```

我们看到，语句和分支的覆盖率都是100%，干得不错。

下面我们用ktest-tool工具来看下生成的6个测试用例的值是什么：
```
ktest-tool ./klee-out-3/test000001.ktest
ktest file : './klee-out-3/test000001.ktest'
args       : ['testscore.bc']
num objects: 1
object 0: name: 'score'
object 0: size: 4
object 0: data: b'\xff\xff\xff\x7f'
object 0: hex : 0xffffff7f
object 0: int : 2147483647
object 0: uint: 2147483647
object 0: text: ....
[root@7a5293f64325 libs]# ktest-tool ./klee-out-3/test000002.ktest
ktest file : './klee-out-3/test000002.ktest'
args       : ['testscore.bc']
num objects: 1
object 0: name: 'score'
object 0: size: 4
object 0: data: b'\x00\x00\x00\x80'
object 0: hex : 0x00000080
object 0: int : -2147483648
object 0: uint: 2147483648
object 0: text: ....
[root@7a5293f64325 libs]# ktest-tool ./klee-out-3/test000003.ktest
ktest file : './klee-out-3/test000003.ktest'
args       : ['testscore.bc']
num objects: 1
object 0: name: 'score'
object 0: size: 4
object 0: data: b'\x00\x00\x00\x00'
object 0: hex : 0x00000000
object 0: int : 0
object 0: uint: 0
object 0: text: ....
[root@7a5293f64325 libs]# ktest-tool ./klee-out-3/test000004.ktest
ktest file : './klee-out-3/test000004.ktest'
args       : ['testscore.bc']
num objects: 1
object 0: name: 'score'
object 0: size: 4
object 0: data: b'Z\x00\x00\x00'
object 0: hex : 0x5a000000
object 0: int : 90
object 0: uint: 90
object 0: text: Z...
[root@7a5293f64325 libs]# ktest-tool ./klee-out-3/test000005.ktest
ktest file : './klee-out-3/test000005.ktest'
args       : ['testscore.bc']
num objects: 1
object 0: name: 'score'
object 0: size: 4
object 0: data: b'<\x00\x00\x00'
object 0: hex : 0x3c000000
object 0: int : 60
object 0: uint: 60
object 0: text: <...
[root@7a5293f64325 libs]# ktest-tool ./klee-out-3/test000006.ktest
ktest file : './klee-out-3/test000006.ktest'
args       : ['testscore.bc']
num objects: 1
object 0: name: 'score'
object 0: size: 4
object 0: data: b'P\x00\x00\x00'
object 0: hex : 0x50000000
object 0: int : 80
object 0: uint: 80
object 0: text: P...
```

通过读取test000001.ktest到test000006.ktest这6个文件，我们发现，系统帮我们找到的score值分别为：2147483647，-2147483648，0，90，60，80。最大正值，最小负值和0都被考虑到了，还有代码中区分不同分支的60,80,90都被自动找到了。

## 两个参数的例子

单找一个参数不过瘾，我们再来试试两个参数的。
先来个最简单的，求两个数的最大值吧：

```c
#include "klee/klee.h"

int max2(int a, int b){
    if(a>b){
        return a;
    }else{
        return b;
    }
}

int main()
{
    int a,b;
    klee_make_symbolic(&a, sizeof(a), "a");
    klee_make_symbolic(&b, sizeof(b), "b");
    return (max2(a,b));
}
```

老办法，编译成bc字节码：
```
clang -emit-llvm -c max.c
```

然后运行klee max.bc:
```
klee max.bc
KLEE: output directory is "/workspace/xulun/github/libs/klee-out-4"
KLEE: Using STP solver backend

KLEE: done: total instructions = 32
KLEE: done: completed paths = 2
KLEE: done: generated tests = 2
```

这个只有两个分支，所以klee给我们也就找到两个。
再用klee-stats看下覆盖率：
```
# klee-stats ./klee-out-4/
----------------------------------------------------------------------------
|    Path     |  Instrs|  Time(s)|  ICov(%)|  BCov(%)|  ICount|  TSolver(%)|
----------------------------------------------------------------------------
|./klee-out-4/|      32|     0.00|   100.00|   100.00|      29|       65.62|
----------------------------------------------------------------------------
```

## 带循环的例子

下面我们再挑战个复杂点的例子，带循环结构的例子。
我们以辗转相除法求最大公约数为例子吧：
```c
#include "klee/klee.h"

short gcd(short a, short b){
    short a0 = a;
    short b0 = b;
    short c0 = 0;
    if(a<=0 || b<=0){
        return 0;
    }

    if(a<b){
        a0 = b;
        b0 = a;
    }

    for(;;){
        c0 = a0 % b0;
        if(c0==0){
            return b0;
        }else{
            a0 = b0;
            b0 = c0;
        }
    }

    return 1;
}

int main()
{
    short a,b;
    klee_make_symbolic(&a, sizeof(a), "a");
    klee_make_symbolic(&b, sizeof(b), "b");
    return (gcd(a,b));
}
```

大家看到，这个例子我们没用int类型，而是使用的short，这样是因为int需要运行的时间较长，光short类型klee就为我们发现了44个case，大约会占满一个CPU核几分钟左右。
```
klee gcd2.bc
KLEE: output directory is "/workspace/xulun/github/libs/klee-out-5"
KLEE: Using STP solver backend

KLEE: done: total instructions = 1431
KLEE: done: completed paths = 44
KLEE: done: generated tests = 44
```

我们看下coverage:
```
# klee-stats ./klee-out-5/
----------------------------------------------------------------------------
|    Path     |  Instrs|  Time(s)|  ICov(%)|  BCov(%)|  ICount|  TSolver(%)|
----------------------------------------------------------------------------
|./klee-out-5/|    1431|   212.26|    98.75|    90.00|      80|       99.99|
----------------------------------------------------------------------------
```

因为这个是测试最大公约数，klee能够帮助我们生成例子能帮我们省不少事。
因为a和b都是0的case已经可以cover到a<=0 || b<=0这一分支，所以klee除了第一个case是0，0之外，后面全是正的有效例子。
第一个是0,0:
```
ktest-tool ./klee-out-6/test000001.ktest
ktest file : './klee-out-6/test000001.ktest'
args       : ['gcd3.bc']
num objects: 2
object 0: name: 'a'
object 0: size: 2
object 0: data: b'\x00\x00'
object 0: hex : 0x0000
object 0: int : 0
object 0: uint: 0
object 0: text: ..
object 1: name: 'b'
object 1: size: 2
object 1: data: b'\x00\x00'
object 1: hex : 0x0000
object 1: int : 0
object 1: uint: 0
object 1: text: ..
```

全部44次的值如下：

|轮数|a|b|
|---|---|---|
|1|0|0|
|2|1|0|
|3|1|1|
|4|1|3|
|5|2|3|
|6|2559|2|
|7|5|7|
|8|31317|65|
|9|4102|16405|
|10|20482|12289|
|11|6573| 8204
|12|32601| 143
|13|2589| 6038
|14|32511| 4192
|15|3783| 16384
|16|20610| 11838
|17|21524| 28799
|18|22774| 1444
|19|24704| 31937
|20|24587| 16044
|21|25961| 26699
|22|19805| 17422
|23|23516| 25899
|24|13077| 7540
|25|18653| 28171
|26|32577| 20000
|27|19932| 32107
|28|32622| 26737
|29|19290| 26657
|30|27009| 16386
|31|19074| 29917
|32|23467| 32422
|33|28266| 17443
|34|19584| 31685
|35|31505| 19452
|36|29779| 18406
|37|15413| 24939
|38|25829| 15969
|39|28655| 17709
|40|20041| 32428
|41|15841|25631
|42|25633|15842
|43|17711|28657
|44|28657|17711

## 数组和字符串的例子

我们来个字符串的例子，因为不是测库函数，所以没有调用库函数。其实klee_make_symbolic本来就是为数组设计的，我们只要把数组大小传给第2个参数就可以了：

```c
#include "klee/klee.h"

int check_pass(char* a){
    if(a==NULL){
        return -1;
    }else if(a[0]=='\0'){
        return -2;
    }else{
        if(a[0]=='C' && a[1]=='+' && a[2]=='+'){
            return 1;
        }else{
            return 0;
        }
    }
}

#define LEN 3

int main()
{
    char secret[LEN];
    klee_make_symbolic(&secret, LEN, "secret");
    return (check_pass(secret));
}
```

klee为我们找到5个分支，分别是：0开头的，非0非C开头的，C开头的, C+开头的, C++ 5种：
```
[root@7a5293f64325 libs]# ktest-tool ./klee-out-7/test000001.ktest
ktest file : './klee-out-7/test000001.ktest'
args       : ['strc.bc']
num objects: 1
object 0: name: 'secret'
object 0: size: 3
object 0: data: b'\x00\x00\x00'
object 0: hex : 0x000000
object 0: text: ...
[root@7a5293f64325 libs]# ktest-tool ./klee-out-7/test000002.ktest
ktest file : './klee-out-7/test000002.ktest'
args       : ['strc.bc']
num objects: 1
object 0: name: 'secret'
object 0: size: 3
object 0: data: b'\x01\xff\xff'
object 0: hex : 0x01ffff
object 0: text: ...
[root@7a5293f64325 libs]# ktest-tool ./klee-out-7/test000003.ktest
ktest file : './klee-out-7/test000003.ktest'
args       : ['strc.bc']
num objects: 1
object 0: name: 'secret'
object 0: size: 3
object 0: data: b'C\x00\xff'
object 0: hex : 0x4300ff
object 0: text: C..
[root@7a5293f64325 libs]# ktest-tool ./klee-out-7/test000004.ktest
ktest file : './klee-out-7/test000004.ktest'
args       : ['strc.bc']
num objects: 1
object 0: name: 'secret'
object 0: size: 3
object 0: data: b'C+\x00'
object 0: hex : 0x432b00
object 0: text: C+.
[root@7a5293f64325 libs]# ktest-tool ./klee-out-7/test000005.ktest
ktest file : './klee-out-7/test000005.ktest'
args       : ['strc.bc']
num objects: 1
object 0: name: 'secret'
object 0: size: 3
object 0: data: b'C++'
object 0: hex : 0x432b2b
object 0: text: C++
```

## 编译安装klee

展示了klee的能力之后，很多同学跃跃欲试想一试身手了。建议初试身手时在ubuntu Linux上尝试。

klee因为其复杂性，依赖比较多。除了llvm,cmake之类的通用依赖之外，我们还需要为其编译SAT求解器minisat和SMT求解器stp。

为了让文章简单，下面的步骤都取了极简的步骤。后面我们还会编译更复杂依赖的klee。

### 下载编译minisat

minisat是一种SAT-布尔可满足性理论求解器。后面会介绍SAT的原理，包括DPLL, CDCL方法等。

```
git clone https://github.com/stp/minisat.git
cd minisat
mkdir build
cd build
cmake ..
make
sudo make install
```

### 下载编译stp

STP是基于minisat的SMT-可满足性模理论求解器。

```
git clone https://github.com/stp/stp.git
cd stp
mkdir build
cd build
cmake ..
make
sudo make install
```

### 下载编译klee

下面是极简步骤，后面我们还会增加库和加上测试。

```
git clone https://github.com/klee/klee.git
mkdir build
cd build
cmake -DENABLE_UNIT_TESTS=OFF -DENABLE_SYSTEM_TESTS=OFF ..
make
sudo make install
```

好了，现在klee命令可用了，大家就可以用自己的代码做实验了：）
