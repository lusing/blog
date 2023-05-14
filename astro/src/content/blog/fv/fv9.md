---
title: "操作系统形式化验证实践教程(1) - 证明第一个定理"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

# 操作系统形式化验证实践教程(9) - 规范与证明概述

## 规范与证明的主线

前面铺垫了这么多，下面我们看一下seL4形式化验证的大图：
![seL4证明.png](https://upload-images.jianshu.io/upload_images/1638145-3175418c889795dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

seL4的证明部分主要分为两大部分：规范部分，对应spec目录；证明部分，对应proof目录。

规范分为4种：
- 设计规范：就是从上节所见的haskell代码转换成的，可以运行的对操作系统的建模规范，对应目标ExecSpec
- 抽象规范：是基于硬件规范和设计规范的抽象，对应目标ASpec
- capDL规范：capDL是用于运行时建模的语言，用于系统初始化等动态过程的建模，对应目标DSpec
- C规范：其实是用工具对C代码的解析，就没有列到图上。要不然反汇编的最终代码也得算一个吧。对应目标CSpec

证明上，抽象规范是核心。
首先有抽象规范的不变量验证，然后是refine，抽象规范是否正确被设计规范实现的证明
其次是CRefine，证明C代码的实现与设计规范相符
第三是DRefine，证明运行时建模与抽象规范相符
最后是AsmRefine，证明C语言生成的代码与反汇编出来的相符

除此之外，还有安全相关的一些细节。

我们引用seL4官方论文的图来看一下：
![seL4](https://upload-images.jianshu.io/upload_images/1638145-7848373916e2678a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 规范及其依赖关系

我们再看一张seL4的设计过程图：
![seL4 Design](https://upload-images.jianshu.io/upload_images/1638145-daa15664d05bad80.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在spec目录，执行make ExecSpec，就可以生成设计规范。ExecSpec也是另外几个规范的基础，它们都要引用到ExecSpec，要不然不知道硬件接口等基础信息，上面的高层建筑也就不用玩了。

我们来看ExecSpec的依赖关系，它并不依赖于其它Spec:
```
Session Pure/Pure
Session FOL/FOL
Session Tools/Tools
Session HOL/HOL (main)
Session HOL/HOL-Library (main timing)
Session HOL/HOL-Computational_Algebra (main timing)
Session HOL/HOL-Analysis (main timing)
Session HOL/HOL-Eisbach
Session HOL/HOL-Word (main timing)
Session Lib/Word_Lib (lib)
Session Lib/Lib (lib)
Session Specifications/ExecSpec
```

抽象规范ASpec依赖于ExecSpec:
```
Session Pure/Pure
Session FOL/FOL
Session Tools/Tools
Session HOL/HOL (main)
Session HOL/HOL-Library (main timing)
Session HOL/HOL-Computational_Algebra (main timing)
Session HOL/HOL-Analysis (main timing)
Session HOL/HOL-Eisbach
Session HOL/HOL-Word (main timing)
Session Lib/Word_Lib (lib)
Session Lib/Lib (lib)
Session Specifications/ExecSpec
Session Specifications/ASpec
```

CapDL规范，也就是DSpec，依赖于ExecSpec和ASpec:
```
Session Pure/Pure
Session FOL/FOL
Session Tools/Tools
Session HOL/HOL (main)
Session HOL/HOL-Library (main timing)
Session HOL/HOL-Computational_Algebra (main timing)
Session HOL/HOL-Analysis (main timing)
Session HOL/HOL-Eisbach
Session HOL/HOL-Word (main timing)
Session Lib/Word_Lib (lib)
Session Lib/Lib (lib)
Session Specifications/ExecSpec
Session Specifications/ASpec
Session Specifications/DSpec
```

CapDL用于系统初始化的过程如下图所示：
![CapDL.png](https://upload-images.jianshu.io/upload_images/1638145-328f06ee232c7732.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


C规范是最复杂的，不但依赖ExecSpec，C-Parser工具，还有AsmRefine工具，还有针对C语言本身的CKernel：
```
Session Pure/Pure
Session FOL/FOL
Session Tools/Tools
Session HOL/HOL (main)
Session HOL/HOL-Library (main timing)
Session HOL/HOL-Computational_Algebra (main timing)
Session HOL/HOL-Analysis (main timing)
Session HOL/HOL-Eisbach
Session HOL/HOL-Statespace
Session HOL/HOL-Word (main timing)
Session Lib/Word_Lib (lib)
Session Lib/Lib (lib)
Session Specifications/ExecSpec
Session C-Parser/Simpl-VCG
Session C-Parser/CParser
Session Lib/CLib (lib)
Session Tools/AsmRefine
Session Specifications/CKernel
Session Specifications/CSpec
```

## 证明及其依赖

前面介绍了，Refine过程是检查设计规范与抽象规范的一致性，所以会依赖ExecSpec和ASpec。在proof目录下运行make Refine: 
```
Session Pure/Pure
Session FOL/FOL
Session Tools/Tools
Session HOL/HOL (main)
Session HOL/HOL-Library (main timing)
Session HOL/HOL-Computational_Algebra (main timing)
Session HOL/HOL-Analysis (main timing)
Session HOL/HOL-Eisbach
Session HOL/HOL-Word (main timing)
Session Lib/Word_Lib (lib)
Session Lib/Lib (lib)
Session Specifications/ExecSpec
Session Specifications/ASpec
Session Proofs/AInvs
Session Lib/CorresK
Session Proofs/BaseRefine
Session Proofs/Refine
```

CRefine的过程，依赖上面的Refine过程：
```
Session Pure/Pure
Session FOL/FOL
Session Tools/Tools
Session HOL/HOL (main)
Session HOL/HOL-Library (main timing)
Session HOL/HOL-Computational_Algebra (main timing)
Session HOL/HOL-Analysis (main timing)
Session HOL/HOL-Eisbach
Session HOL/HOL-Statespace
Session HOL/HOL-Word (main timing)
Session Lib/Word_Lib (lib)
Session Lib/Lib (lib)
Session Specifications/ExecSpec
Session Specifications/ASpec
Session Proofs/AInvs
Session Lib/CorresK
Session Proofs/BaseRefine
Session Proofs/Refine
Session C-Parser/Simpl-VCG
Session C-Parser/CParser
Session Lib/CLib (lib)
Session Tools/AsmRefine
Session Unsorted/AutoCorres
Session Specifications/CKernel
Session Specifications/CSpec
Session Proofs/CBaseRefine
Session Proofs/CRefine
```

终极目标是校验最终生成的目标代码```make SimplExportAndRefine```：
```
Session Pure/Pure
Session FOL/FOL
Session Tools/Tools
Session HOL/HOL (main)
Session HOL/HOL-Library (main timing)
Session HOL/HOL-Computational_Algebra (main timing)
Session HOL/HOL-Analysis (main timing)
Session HOL/HOL-Eisbach
Session HOL/HOL-Statespace
Session HOL/HOL-Word (main timing)
Session Lib/Word_Lib (lib)
Session Lib/Lib (lib)
Session Specifications/ExecSpec
Session C-Parser/Simpl-VCG
Session C-Parser/CParser
Session Lib/CLib (lib)
Session Tools/AsmRefine
Session Specifications/CKernel
Session Specifications/CSpec
Session Proofs/SimplExport
Session Proofs/SimplExportAndRefine
```

有同学觉得，能校验C代码和反汇编的代码是不是一致这太神奇了。这其中使用了一个将两者都转换成图结构，然后进行比较的工具：[https://github.com/seL4/graph-refine](https://github.com/seL4/graph-refine)。

详细原理我们还是看下官方的图：
![AsmRefine](https://upload-images.jianshu.io/upload_images/1638145-a851a95ce94edd6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果图还抽象的话我们看个例子：
![Example](https://upload-images.jianshu.io/upload_images/1638145-3f345cfe1312d817.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

左边是C代码，右边是反汇编的结果，中间是它们用图表示的结构。
