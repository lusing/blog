---
title: "操作系统形式化验证实践教程(1) - 证明第一个定理"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

# 操作系统形式化验证实践教程(7) - C代码的自动验证

上一节教程不知道大家看晕了没有，其实虽然细节很多还没有讲清楚，但是从结构上大家可以看到，其实是很模式化的工作。
那么能不能让这个模式化的工作自动化起来，也能降低一点入门的学习门槛？这时就该AutoCorres工具出马了。

## AutoCorres

既然回到人间，不用再看着一排的simp, vcg之类的，咱们的难度又回到第一讲加减法的时代。

先实现一个C语言实现加法的函数：
```c
unsigned int plus(unsigned int a, unsigned int b)
{
    return a + b;
}
```

下面开始写HOL，先引入AutoCorres：
```
imports
  "AutoCorres.AutoCorres"
```

然后还是把C源文件读进来，并且install了：
```
external_file "plus.c"
install_C_file "plus.c"
```

然后给c文件autocorres一下：
```
autocorres "plus.c"
```

按照惯例locale一下：
```
context plus begin
```
实际上执行的是：
```
locale Plus.plus
  fixes symbol_table :: "char list ⇒ 32 word"
```

万事俱备，我们现在已经能读懂C代码了。

### 通过by eval验证具体例子

先写个case验证下：
```
lemma "plus' 128 127 = 255"
```
C代码既然懂了，就直接unfold一下：
```
  unfolding plus'_def
```
验证对不对，执行一下by eval，完整代码如下：

```
lemma "plus' 128 127 = 255"
  unfolding plus'_def
  by eval
```

### 演绎验证

例子只能证明在这一种情况下是正确的，但无法证明在所有情况下都正确。我们有没有办法证明所有情况下都正确呢？
可以的，我们不用by eval了，换个apply的规则就好了：
```
lemma plus_correct: "plus' a b = a + b"
  unfolding plus'_def
  apply (rule refl)
  done
```

我们来看下目标：
```
proof (prove)
goal (1 subgoal):
 1. a + b = a + b
```
这真的也没啥可以证的了。。。

## 自动生成定理和证明

我们乘胜追击，再来搞个最大值的：
```
unsigned max(unsigned a, unsigned b)
{
    if (a <= b) {
        return b;
    }
    return a;
}
```

后面没啥惊喜的，一条龙：imports AutoCorres, external_file, install_C_file, autocorres, unfolding: 
```
imports
  "AutoCorres.AutoCorres"
begin

external_file "simple.c"

install_C_file "simple.c"

autocorres "simple.c"

context simple begin

lemma "max' a b = max a b"
  unfolding max'_def max_def
  by (rule refl)
```

下面我们来个更强大的，自动生成定理和证明：
```
thm simple.max'_def simple.max'_ac_corres
```

生成的结果如下：
```
  simple.max' ?a ?b ≡ if ?a ≤ ?b then ?b else ?a
  ac_corres (simple.lift_global_heap ∘ globals) True
   simple_global_addresses.Γ ret__unsigned_'
   ((λs. a_' s = ?a') and (λs. b_' s = ?b') and
    (λx. abs_var ?a id ?a' ∧ abs_var ?b id ?b') ∘
    simple.lift_global_heap ∘
    globals)
   (L2_gets (λ_. simple.max' ?a ?b) [''ret''])
   (Call max_'proc)
```

虽然autocorres不会知道max跟库中的max有啥关系，但是可以生成```if ?a ≤ ?b then ?b else ?a```这样的定理。

## 验证并非是重复代码逻辑

刚才我们看到加法还有最大值，已经都被autocorres轻松消解掉了，连定理和证明都可以自动生成了。
但是，实际上，形式化验证并没有这么简单。
我们看个求最大公约数的例子：
```c
unsigned gcd(unsigned a, unsigned b)
{
    unsigned c;
    while (a != 0) {
        c = a;
        a = b % a;
        b = c;
    }
    return b;
}
```

这个如何验证？

自动生成下：
```
thm simple.gcd'_def simple.gcd'_ac_corres
```
结果如下：
```
  simple.gcd' ?a ?b ≡
  do (a, b) <- whileLoop (λ(a, b) b. a ≠ 0)
                 (λ(a, b). return (b mod a, a))
                (?a, ?b);
     return b
  od
  ac_corres (simple.lift_global_heap ∘ globals) True
   simple_global_addresses.Γ (unat ∘ ret__unsigned_')
   ((λs. a_' s = ?a') and (λs. b_' s = ?b') and
    (λx. abs_var ?a unat ?a' ∧ abs_var ?b unat ?b') ∘
    simple.lift_global_heap ∘
    globals)
   (liftE (simple.gcd' ?a ?b)) (Call gcd_'proc)
```

这重复了下实现逻辑没错，但是没有体验中最大公约数的本质。

我们来看下seL4中的实现吧：
```
lemma gcd_to_return [simp]:
    "gcd' a b = return (gcd a b)"
  apply (subst monad_to_gets [where v="λ_. gcd a b"])
    apply (wp gcd_wp)
    apply simp
   apply (clarsimp simp: gcd'_def)
   apply (rule empty_fail_bind)
    apply (rule empty_fail_whileLoop)
    apply (clarsimp simp: split_def)
   apply (clarsimp simp: split_def)
  apply (clarsimp simp: split_def)
  done
```

其中，monad_to_gets是一个辅助定理：
```
lemma monad_to_gets:
    "⟦ ⋀P. ⦃ P ⦄ f ⦃ λr s. P s ∧ r = v s ⦄!; empty_fail f ⟧ ⟹ f = gets v"
  apply atomize
  apply (monad_eq simp: validNF_def valid_def no_fail_def empty_fail_def)
  apply (rule conjI)
   apply clarsimp
   apply (drule_tac x="λs'. s = s'" in spec)
   apply clarsimp
   apply force
  apply clarsimp
  apply (drule_tac x="λs'. s' = t" in spec)
  apply clarsimp
  apply force
  done
```

求最弱前置条件gcd_wp为：
```
lemma gcd_wp [wp]:
    "⦃ P (gcd a b) ⦄ gcd' a b ⦃ P ⦄!"
  (* Unfold definition of "gcd'". *)
  apply (unfold gcd'_def)

  (* Annoate the loop with an invariant and measure. *)
  apply (subst whileLoop_add_inv [where
     I="λ(a', b') s. gcd a b = gcd a' b' ∧ P (gcd a b) s"
     and M="λ((a', b'), s). a'"])

  (* Solve using weakest-precondition. *)
  apply (wp; clarsimp)
   apply (metis gcd.commute gcd_red_nat)
  using gt_or_eq_0 by fastforce
```
这种水平的验证我们暂时还写不出来，大家只要有个概念就好，后面针对常见算法的验证方法我们用到再讲。
