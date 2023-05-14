---
title: "操作系统形式化验证实践教程(1) - 证明第一个定理"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

# 操作系统形式化验证实践教程(3) - 自动证明工具

## 归纳推理的推广

在第一节，我们学习了自然性的归纳法推理。大家都学过数学归纳法，对此应该或清晰或模糊有个概念。

其实，大家打破思维限制，归纳推理其实可以应用在更广阔的领域。

比如，我们想一想，基于第二节所讲的列表，有哪些定理可以通过归纳法证明？
我们举个例子，大家有没有想到，如何证明：将一个列表反转两次，所得的列表与原列表是相等的？

我们来试着用之前的apply(induction)和apply(auto)的方法证明一下：
```hol
lemma "rev (rev xs) = xs"
  apply(induction xs)
   apply(auto)
  done
```
输出如下，首先是针对空列表[]的证明，然后是个递推证明a # xs：
```
proof (prove)
goal (2 subgoals):
 1. rev (rev []) = []
 2. ⋀a xs.
       rev (rev xs) = xs ⟹ rev (rev (a # xs)) = a # xs
```

这还不过瘾，咱们再折腾下，将列表反转4次试试：
```hol
lemma "rev(rev(rev(rev xs))) = xs"
  apply(induction xs)
   apply(auto)
  done
```

一样可以自动证明出来哈：
```
proof (prove)
goal (2 subgoals):
 1. rev (rev (rev (rev []))) = []
 2. ⋀a xs.
       rev (rev (rev (rev xs))) = xs ⟹
       rev (rev (rev (rev (a # xs)))) = a # xs
```

我膨胀了，咱们来个直觉上不太好想的，假设有两个列表，它们正向连接和反向连接相等，它们的长度也相等，求证：这两个列表相等：
```
lemma "⟦ xs @ ys = ys @ xs; length xs = length ys ⟧ ⟹ xs = ys"
  apply(induction xs)
  apply(auto)
  done
```

上一条里面生词比较多哈，左右括号分别是lbrakk和rbrakk，推出是Longrightarrow。我们看一下其源代码：
```
lemma "\<lbrakk> xs @ ys = ys @ xs; length xs = length ys \<rbrakk> \<Longrightarrow> xs = ys"
  apply(induction xs)
  apply(auto)
  done
```

这下确实auto推不出来了：
![conj](https://upload-images.jianshu.io/upload_images/1638145-dc9a88edf60d8ff5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

自动推理不出来，我们可以通过增加辅助证明的方式来帮助它。这个有点小复杂，我们后面再讲。
这里，我们先推介出大杀器，可以用于自动定理证明的大杀器 - sledgehammer。

## sledgehammer

sledgehammer是一个帮助我们搜索证明方法的利器。我们要证明的问题，很多可能是前人早已经证明过的。
我们可以有三种方式：
1. 重造轮子，自己再证明一遍
2. 去库里里面找一个
3. 工具帮我们找一个
实现第三种操作的工具就是sledgehammer。

我们从output区域切换到sledgehammer, Prover证明器先选cvc4，点击Apply，看到的结果如下：
![sledgehammer](https://upload-images.jianshu.io/upload_images/1638145-07bc1d699c390c37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

输出为：
```
Proof found... 
"cvc4": Try this: using append_eq_append_conv by blast (35 ms)
```
我们就用cvc4让我们try的这条，来替代apply那几句：
```
lemma "⟦ xs @ ys = ys @ xs; length xs = length ys ⟧ ⟹ xs = ys"
using append_eq_append_conv by blast
```

恭喜你，证明这就算做完了。
![append_eq_append_conv](https://upload-images.jianshu.io/upload_images/1638145-c32be7f23a02ed3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 更多证明器

cvc4只是众多证明器中的一个，我们还有很多种选择，比如系统默认给我们提供了一个更多选择：
![more prover](https://upload-images.jianshu.io/upload_images/1638145-55c7af23b9eda4ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

找到之后，我们把鼠标移到相应的建议上点击，就可以直接输入到源代中：
```
lemma "⟦ xs @ ys = ys @ xs; length xs = length ys ⟧ ⟹ xs = ys"
  by (metis append_eq_conv_conj)
```

![metis](https://upload-images.jianshu.io/upload_images/1638145-f887e2c22fcd9fef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

sledgehammer的背后，是一系列的自动定理证明器和SMT solvers。关于它们的原理，后面有机会详细介绍下，这是我喜欢的话题。

## 定理搜索器

Isabelle IDE中也有针对定理的搜索功能，如果各种证明器都找不到的话，可以人肉去研究下：

![find theorems](https://upload-images.jianshu.io/upload_images/1638145-da43e2b0adf1faea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有了上面的自动定理证明工具和定理搜索工具，在后面我们学习原理的同时，特别鼓励大家能自动的就自动，能借用的就借用。原理可以在使用中慢慢加深，用高深理论把大家吓跑不是本系列的目的，能够帮助大家在工作中把方法和工具用起来才是。
