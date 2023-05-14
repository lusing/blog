---
title: "符号执行(4) - 幕后英雄SMT"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

前面几讲，我们基本上了解了klee的主要用法。下面我们进入更深入的原理部分，不仅要掌握搜索方法，还要理解每个分支逻辑的具体内容。

为了看到klee背后的逻辑，我们增加--write-smt2s参数，生成每个case对应的smt文件。

## 初识SMT

### 生成SMT文件

为了简化理解，我们选取一个判断符号的简单例子：
```c
#include "klee/klee.h"

char sign2(char a){
    if(a>0){
        return 1;
    }else if(a==0){
        return 0;
    }else{
        return -1;
    }
}

int main(void){
    char a;
    klee_make_symbolic(&a,sizeof(a),"a");
    return sign2(a);
}
```

然后还是用clang -emit-llvm -c来编译成bitcode：
```
clang -emit-llvm -c simpbyte.c
```
这次运行klee时，我们加上--write-smt2s参数:
```
klee --write-smt2s simpbyte.bc
```

klee为我们生成的测试case为1，-128，0：
```
[root@615cce8a2248 klee-out-46]# ktest-tool test000001.ktest
ktest file : 'test000001.ktest'
args       : ['simpbyte.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 1
object 0: data: b'\x01'
object 0: hex : 0x01
object 0: int : 1
object 0: uint: 1
object 0: text: .
[root@615cce8a2248 klee-out-46]# ktest-tool test000002.ktest
ktest file : 'test000002.ktest'
args       : ['simpbyte.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 1
object 0: data: b'\x80'
object 0: hex : 0x80
object 0: int : -128
object 0: uint: 128
object 0: text: .
[root@615cce8a2248 klee-out-46]# ktest-tool test000003.ktest
ktest file : 'test000003.ktest'
args       : ['simpbyte.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 1
object 0: data: b'\x00'
object 0: hex : 0x00
object 0: int : 0
object 0: uint: 0
object 0: text: .
```

### SMT的内容与运行

同时，我们发现对于每个case，都生成了一个.smt文件，例如test000001.smt2。
我们来看看它们的内容吧。
第一个case:test000001.smt2
```smt
(set-logic QF_AUFBV )
(declare-fun a () (Array (_ BitVec 32) (_ BitVec 8) ) )
(assert (bvslt  (_ bv0 32) ((_ sign_extend 24)  (select  a (_ bv0 32) ) ) ) )
(check-sat)
(exit)
```

第二个：
```smt
(set-logic QF_AUFBV )
(declare-fun a () (Array (_ BitVec 32) (_ BitVec 8) ) )
(assert (let ( (?B1 (select  a (_ bv0 32) ) ) ) (and  (=  false (bvslt  (_ bv0 32) ((_ sign_extend 24)  ?B1 ) ) ) (=  false (=  (_ bv0 8) ?B1 ) ) ) ) )
(check-sat)
(exit)
```

第三个：
```smt
(set-logic QF_AUFBV )
(declare-fun a () (Array (_ BitVec 32) (_ BitVec 8) ) )
(assert (=  (_ bv0 8) (select  a (_ bv0 32) ) ) )
(check-sat)
(exit)
```

虽然每个文件都不长，但是是不是看了有一种头大的感觉？

别怕！首先，它们的功能非常有限；其次，它们是可运行的。也就是说我们可以通过打印中间件等办法去辅助理解。

还记得我们之前安装的z3，编译安装的stp等工具吗？它们就是执行smt文件的求解器。

比如，我们运行stp test000001.smt2，它会显示：
```
sat
```
sat是可满足的意思，也就是有解。
我们要想求得符合条件的一个解，就可以通过smt的(get-model)方法去获取。
我们给三个smt文件分别加上(get-model)调用，我们以第一个为例进行说明，修改后test000001.smt2文件如下：
```
(set-logic QF_AUFBV )
(declare-fun a () (Array (_ BitVec 32) (_ BitVec 8) ) )
(assert (bvslt  (_ bv0 32) ((_ sign_extend 24)  (select  a (_ bv0 32) ) ) ) )
(check-sat)
(get-model)
(exit)
```

我们还是通过stp test000001.smt2来执行这个修改后的smt文件，返回值如下：
```
sat
(model
( define-fun |a|  (_ BitVec 32) (_ BitVec 8) #x00000000 #x01 )
)
```

BitVec表示位组成的数组。a返回的是只有一个单元的数组，数组0位置的结果是1。
这个1，就是我们第一个测试用例的结果。klee通过搜索路径，帮我们生成了SMT求解器可以求解的公式，最后通过SMT求解器来进行求得结果。

我们再修改下test000002.smt2：
```
(set-logic QF_AUFBV )
(declare-fun a () (Array (_ BitVec 32) (_ BitVec 8) ) )
(assert (let ( (?B1 (select  a (_ bv0 32) ) ) ) (and  (=  false (bvslt  (_ bv0 32) ((_ sign_extend 24)  ?B1 ) ) ) (=  false (=  (_ bv0 8) ?B1 ) ) ) ) )
(check-sat)
(get-model)
(exit)
```

运行stp test000002.smt2:
```
sat
(model
( define-fun |a|  (_ BitVec 32) (_ BitVec 8) #x00000000 #x80 )
)
```

结果果然是0x80，也就是-128。

最后一个case应该是0，我们确认下，还是先改smt:
```
(set-logic QF_AUFBV )
(declare-fun a () (Array (_ BitVec 32) (_ BitVec 8) ) )
(assert (=  (_ bv0 8) (select  a (_ bv0 32) ) ) )
(check-sat)
(get-model)
(exit)
```

运行stp test000003.smt2：
```
sat
(model
( define-fun |a|  (_ BitVec 32) (_ BitVec 8) #x00000000 #x00 )
)
```

果然是0x00。

## SMT的求解不只一个值

以上是我们用stp求解器求得的值。对于SMT公式来说，只要找到任意一个符合的值都可以，不同的SMT求解器的策略不同，所获取的值也可能不一样。

比如我们换成用z3求解器求解，获取的值就是64，-128和0，如下：
```
[root@615cce8a2248 klee-out-46]# z3 test000001.smt2
sat
(model
  (define-fun a () (Array (_ BitVec 32) (_ BitVec 8))
    ((as const (Array (_ BitVec 32) (_ BitVec 8))) #x40))
)
[root@615cce8a2248 klee-out-46]# z3 test000002.smt2
sat
(model
  (define-fun a () (Array (_ BitVec 32) (_ BitVec 8))
    ((as const (Array (_ BitVec 32) (_ BitVec 8))) #x80))
)
[root@615cce8a2248 klee-out-46]# z3 test000003.smt2
sat
(model
  (define-fun a () (Array (_ BitVec 32) (_ BitVec 8))
    ((as const (Array (_ BitVec 32) (_ BitVec 8))) #x00))
)
```

毫不意外的，如果我们使用z3作为klee的smt求解器，求解出的结果就是64,-128,0：
```
[root@615cce8a2248 libs]# klee --solver-backend=z3 simpbyte.bc
KLEE: output directory is "/workspace/xulun/github/libs/klee-out-50"
KLEE: Using Z3 solver backend

KLEE: done: total instructions = 35
KLEE: done: completed paths = 3
KLEE: done: generated tests = 3
[root@615cce8a2248 libs]# ktest-tool ./klee-out-50/test000001.ktest
ktest file : './klee-out-50/test000001.ktest'
args       : ['simpbyte.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 1
object 0: data: b'@'
object 0: hex : 0x40
object 0: int : 64
object 0: uint: 64
object 0: text: @
[root@615cce8a2248 libs]# ktest-tool ./klee-out-50/test000002.ktest
ktest file : './klee-out-50/test000002.ktest'
args       : ['simpbyte.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 1
object 0: data: b'\x80'
object 0: hex : 0x80
object 0: int : -128
object 0: uint: 128
object 0: text: .
[root@615cce8a2248 libs]# ktest-tool ./klee-out-50/test000003.ktest
ktest file : './klee-out-50/test000003.ktest'
args       : ['simpbyte.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 1
object 0: data: b'\x00'
object 0: hex : 0x00
object 0: int : 0
object 0: uint: 0
object 0: text: .
```

### z3直接打印模型值

如果嫌修改smt文件麻烦，可以通过给z3加-model参数来直接获取。

比如这是klee生成的smt文件：
```
[root@615cce8a2248 klee-out-51]# cat test000001.smt2
(set-logic QF_AUFBV )
(declare-fun a () (Array (_ BitVec 32) (_ BitVec 8) ) )
(assert (bvslt  (_ bv0 32) ((_ sign_extend 24)  (select  a (_ bv0 32) ) ) ) )
(check-sat)
(exit)
```

我们通过z3 -model test000001.smt2命令来求解：
```
[root@615cce8a2248 klee-out-51]# z3 -model test000001.smt2
sat
(model
  (define-fun a () (Array (_ BitVec 32) (_ BitVec 8))
    ((as const (Array (_ BitVec 32) (_ BitVec 8))) #x40))
)
```

## SMT求解就像是求解联立方程组

如果看了上面的感性认识，大家还是因为细节没讲而感到困惑的话，我们不妨从纯SMT的角度讲个例子，大家就明白了。

### 使用python z3-solver求解方程组

我们首先求解一个联立方程组：
```
x+y=1
x+z=2
```

为了更容易理解，我们先学习用python来调用z3。

我们首先安装z3-solver库：
```
pip install z3-solver --user
```

下面我们开始编程：
```python
import z3
x = z3.Int('x')
y = z3.Int('y')
z = z3.Int('z')
s = z3.Solver()
s.add(x+y==1)
s.add(x+z==2)
print(s.check())
print(s.model())
```

程序很好懂，不用解释大家都能明白。

输出的值为：
```
sat
[y = 0, z = 1, x = 1]
```

sat就是说明有解。后面的s.model()就是smt中的(get-model)，输出一组值。

### 翻译成smt语言

Python的版本大家应该是看懂了。下面我们就原封不动翻译成smtlib的语言：
```smt
(declare-const x Int)
(declare-const y Int)
(declare-const z Int)
(assert (= (+ x y) 1))
(assert (= (+ x z) 2))
(check-sat)
(get-model)
(exit)
```

z3运行结果如下：
```
sat
(model 
  (define-fun y () Int
    0)
  (define-fun z () Int
    1)
  (define-fun x () Int
    1)
)
```

虽然格式与Python不同，但是求解的值是相同的。

除了将x,y,z当成常量，我们也可以将它们当成函数，写成这样，效果是相同的：
```
(declare-fun x () Int)
(declare-fun y () Int)
(declare-fun z () Int)
(assert (=(+ x y)1))
(assert (=(+ x z)2))
(check-sat)
(get-model)
(exit)
```

## 小结

klee生成测试case的主要步骤是通过第二节介绍的搜索逻辑生成SMT表达式，然后通过求解SMT表达式来获取可取的值做为测试用例的过程。
SMT表达式可以通过write-smt2s参数来生成。我们可以通过生成的smt2文件来了解这个case的取值范围和逻辑结果。
