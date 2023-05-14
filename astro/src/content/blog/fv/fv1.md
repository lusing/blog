---
title: "操作系统形式化验证实践教程(1) - 证明第一个定理"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

形式化方法分为三个主要部分：系统建模(System Modeling)、形式规约(Formal Specification)和形式化验证(Formal Verification)。
其中系统建模用形式化的模型来描述系统及其行为模式。建模完成后，需要用形式规约来精确描述建模出来的需求。有了规约，如何检验是否符合规约呢?这就需要形式化验证方法。

形式化验证方法主要分为两类：一类是以穷尽搜索为基础的模型检测，另一类是以逻辑推理为基础的演绎逻辑。相对于前者，后者既可以验证有穷的状态系统，也可以使用归纳法来处理无限状态的问题。

演绎逻辑在早期发展很快，但是后来在大规模软件验证上成本较高，所以发展一直不快。但是，最近十几年，以seL4为代表的操作系统和CompCert为代表的编译器的正确性证明的完成，给形式化验证带来了重要的突破性进展。
这也带来了两大流派：seL4所使用的Isabelle/HOL工具和CompCert所使用的Coq工具。
Isabelle是基于Standard ML语言开发的，支持生成Standard ML, ocaml, Haskell和Scala语言的代码。而Coq是基于ocaml语言的。

波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。

## Isabelle/HOL简介

Isabelle/HOL可以通过下面的网址下载和安装：[https://www.cl.cam.ac.uk/research/hvg/Isabelle/installation.html](https://www.cl.cam.ac.uk/research/hvg/Isabelle/installation.html)。支持Linux/mac/Windows平台。

以Linux为例，我们可以先通过wget工具下载tar.gz包：
```
wget -c https://www.cl.cam.ac.uk/research/hvg/Isabelle/dist/Isabelle2020_linux.tar.gz
```

HOL是High Order Logic，即高阶逻辑的缩写。

废话不多说，Isabelle封装在jEdit中，有完整的集成开发环境。我们安装好了之后直接打开看一眼：

![Isabelle/HOL](https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有几点不跟普通编程语言的不同之处：
- HOL的代码以theory组织在一起
- theory之下有函数fun,值value, 有公式lemma, theorem等
- 当我们把光标放到lemma或theorem中时，发现系统可以自动帮我们做一些推理的验证
- HOL代码中大量使用非ASCII符号，可能给用惯了ASCII字符的程序员们带来一定的不适应，但是可读性绝对比用ASCII表示要上一个层次

## 自动证明第一个定理

同其它编程语言类似，HOL也有自己的数据类型系统。我们先从最简单的布尔类型开始。

HOL支持bool来表示布尔类型。用True表示真值，False表示假值。
取反为$\lnot$, 取交为$\land$，取并为$\lor$。

下面我们写一个求布尔交的函数。
交函数的输入为两个bool，输出也是一个bool. 
我们用"bool ⇒ bool ⇒ bool"来描述这个函数的类型。
“⇒”的输入方法是用TeX值：<\Rightarrow>.
HOL语句要用字符串符号""来括起来，原理我们后面再讲。
代码逻辑上，除了输入为True和True返回True之外，其它都返回False:
```hol
fun conj2 :: "bool ⇒ bool ⇒ bool" where
"conj2 True True = True"|
"conj2 _ _ = False"
```

下面高光时刻来了，我们来用Isabelle/HOL证明我们学习之旅中的第一个定理False与另一个布尔值取conj2操作，结果一定为False。

```hol
lemma conj_02: "conj2 False m = False"
  apply(induction m)
   apply(auto)
  done
```
apply(induction m)和apply(auto)的作用是让HOL自动帮我们推断证明。
把光标放到apply(auto)语句，我们从output窗口看到：
```
proof (prove)
goal (2 subgoals):
 1. conj2 False True = False
 2. conj2 False False = False
```
如图所示：
![conj02](https://upload-images.jianshu.io/upload_images/1638145-5d5d1df63ba78554.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

趁热打铁，我们再写一个练习下：
```
fun not2 :: "bool ⇒ bool" where
"not2 True = False" |
"not2 False = True"
lemma not02 : "not2 x = (¬ x)"
  apply(induction x)
  apply(auto)
  done
```
![lnot](https://upload-images.jianshu.io/upload_images/1638145-cbbae96b576c8bac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 从有限到无限

恭喜大家，已经学会证明有限情况下的定理证明了。但是这还不够，我们还要能证明无限的情况。
下面我们就从有限的布尔类型，前进到自然数类型nat和整数类型int. 

自然数的定义带着浓浓的数学归纳法的味道。这里用到了求后继函数Suc：自然数nat = 0 或 Suc nat。也就是说，自然数定义为0，Suc(0), Suc(Suc(0))...

有了这个定义，我们可以重新定义一个加法了：
```
fun add2 ::"nat ⇒ nat ⇒nat" where 
"add2 0 n = n" | 
"add2 (Suc m) n = Suc(add2 m n)"
```
首先是0和n进行add2等于n，然后是m的后继与n进行add2操作，等于m与n进行add2操作后再求后继。

这么说有点抽象，我们来看例子。

第一个例子：add2 0 1，根据定义，这个就等于1。这个容易理解。
第二个例子：add2 1 2。1是Suc 0，于是这个式子为Suc(add2 0 2)，add2 0 2根据第一条定义式结果为2。
第三个例子：add2 2 3。2是Suc 1，于是式子为Suc(add2 1 3)，再递推为Suc(Suc(add2 0 3)，结果为5.

这样，通过这样一种递推关系，我们重新定义了加法。

那么，这个我们新定义的加法，归纳出来的加法，跟真实的加法是不是一致呢？我们写个定理去进行自动证明：
```
lemma add_02: "add2 m n = m+n"
  apply(induction m)
  apply(auto)
  done
```

下面是证明的结果：
```
proof (prove)
goal (2 subgoals):
 1. add2 0 n = 0 + n
 2. ⋀m. add2 m n = m + n ⟹ add2 (Suc m) n = Suc m + n
```
根据两个初始条件，有两个子目标需要证明：
第一个是初始条件，add2 0 n = 0+n。
第二个是基于后继的递推条件。

### 皮亚诺公理

上面的证明方法叫做归纳原理，也叫做皮亚诺第五公理。
我们来从头看下这5条公理：
1. 0是一个自然数
2. 任何自然数n都有一个自然数Suc(n)作为它的后继
3. 0不是任何自然数的后继
4. 后继函数是单一的，即，如果Suc(m)=Suc(n)，则m=n. 
5. 令Q为关于自然数的一个性质。如果
- 0具有性质Q
- 并且 如果自然数n具有性质Q，则Suc(n)也具有性质Q
- 那么所有自然数n都有性质Q

在此基础上，我们可以定义加法和乘法。

- 加法：对于任何自然数n和m：
  - n + 0 = n
  - 并且 n + Suc(m) = Suc(n + m)
- 乘法：对任何自然数n和m:
  - n * 0 = 0
  - n * Suc(m) = (n * m) + n 

乘法的定理证明如下：
```
fun mul2 ::"nat ⇒ nat ⇒nat" where 
"mul2 0 n = 0" | 
"mul2 (Suc m) n = n * m + n"

lemma add_02: "mul2 m n = m * n"
  apply(induction m)
  apply(auto)
  done
```

## 求值

在过程中，难免有些值我们希望看到输出结果，这时可以通过value语句来实现。

例：
```
value "Suc(0)"
```
输出结果为：
```
"1"
  :: "nat"
```
1为结果的值，nat是类型。

调用我们自己定义的函数也没有问题：
```
value "add2 1 (Suc 4)"
```

输出结果为：
```
"6"
  :: "nat"
```

恭喜你，我们已经在定理证明的世界里证明了第一个定理。
后面的路很长，坡也有点陡，我们继续前进。
