---
title: "操作系统形式化验证实践教程(6) - 解析C源代码"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

从这一讲我们跨出了Isabelle/HOL的领域，开始进入操作系统的领域。
目前的操作系统主要是由C语言和汇编语言写成的，所以我们的第一步先从解析C语言代码开始。

## 构造C解析器

我们需要一个能够解析C源代码，并且能在HOL操作C源代码的工具。seL4为我们提供了c-parser. 

我们首先进入l4v/tools/c-parser目录，接着构造c-parser-deps:
```
make c-parser-deps
```
依赖构建好之后，我们就构建c-parser，并且运行相关的回归测试。直接输入make命令就好。

实际执行的命令是isabelle build：
```
isabelle/bin/isabelle build -d ../.. -b -v CParser
```

构建过程如下，首先是我们上一讲构造Word-Lib时已经见过的基础库：
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
```

然后是C-Parser部分，主要分为两部分: Simpl-VCG和CParser本身。我们先看Simpl-VCG部分：

```
Session C-Parser/Simpl-VCG
Session C-Parser/CParser
Building Simpl-VCG ...
Simpl-VCG: theory HOL-Library.Old_Recdef
Simpl-VCG: theory HOL-Statespace.DistinctTreeProver
Simpl-VCG: theory Simpl-VCG.Language
Simpl-VCG: theory HOL-Statespace.StateFun
Simpl-VCG: theory Simpl-VCG.Generalise
Simpl-VCG: theory HOL-Statespace.StateSpaceLocale
Simpl-VCG: theory Simpl-VCG.Semantic
Simpl-VCG: theory Simpl-VCG.HoarePartialDef
Simpl-VCG: theory Simpl-VCG.Termination
Simpl-VCG: theory Simpl-VCG.HoarePartialProps
Simpl-VCG: theory Simpl-VCG.HoareTotalDef
Simpl-VCG: theory Simpl-VCG.SmallStep
Simpl-VCG: theory Simpl-VCG.HoarePartial
Simpl-VCG: theory Simpl-VCG.HoareTotalProps
Simpl-VCG: theory Simpl-VCG.HoareTotal
Simpl-VCG: theory Simpl-VCG.Hoare
Simpl-VCG: theory Simpl-VCG.StateSpace
Simpl-VCG: theory Simpl-VCG.Vcg
Timing Simpl-VCG (6 threads, 23.240s elapsed time, 72.501s cpu time, 4.949s GC time, factor 3.12)
Finished Simpl-VCG (0:00:35 elapsed time, 0:01:36 cpu time, factor 2.74)
```

simpl-vcg在C语言的解析证明中，将会被反复用到。
simpl是Sequential Imperative programming language的缩写，是我们推理证明时对编程语言的一种抽象表示形式。
而VCG是Verification condition generator，用于自动生成验证条件。

最后则是CParser本身：
```
Building CParser ...
CParser: theory CParser.MapExtra
CParser: theory CParser.PrettyProgs
CParser: theory CParser.Arrays
CParser: theory CParser.StaticFun
CParser: theory CParser.Padding
CParser: theory CParser.Addr_Type
CParser: theory CParser.IndirectCalls
CParser: theory Lib.MLUtils
CParser: theory CParser.CTypesBase
CParser: theory CParser.MapExtraTrans
CParser: theory CParser.CTypesDefs
CParser: theory CParser.CTypes
CParser: theory CParser.HeapRawState
CParser: theory CParser.Vanilla32_Preliminaries
CParser: theory CParser.Word_Mem_Encoding
CParser: theory CParser.Vanilla32
CParser: theory CParser.CompoundCTypes
CParser: theory CParser.ArraysMemInstance
CParser: theory CParser.ArchArraysMemInstance
CParser: theory CParser.TypHeap
CParser: theory CParser.Separation
CParser: theory CParser.SepCode
CParser: theory CParser.SepInv
CParser: theory CParser.SepTactic
CParser: theory CParser.SepFrame
CParser: theory CParser.StructSupport
CParser: theory CParser.ArrayAssertion
CParser: theory CParser.CProof
CParser: theory CParser.CLanguage
CParser: theory CParser.ModifiesProofs
CParser: theory CParser.PackedTypes
CParser: theory CParser.CTranslation
Timing CParser (6 threads, 79.482s elapsed time, 190.601s cpu time, 12.588s GC time, factor 2.40)
Finished CParser (0:01:35 elapsed time, 0:03:42 cpu time, factor 2.34)
```

最后，会运行一些回归测试。这些测试项目也是我们学习如何验证C程序的好例子，因为毕竟更上一个层次有更上一个层次的关注点:

```
Session Unsorted/CParserTest
Running CParserTest ...
CParserTest: theory CParserTest.MachineWords
CParserTest: theory CParserTest.analsignedoverflow
CParserTest: theory CParserTest.asm_stmt
CParserTest: theory CParserTest.attributes
CParserTest: theory CParserTest.array_of_ptr
CParserTest: theory CParserTest.arrays
CParserTest: theory CParserTest.basic_char
CParserTest: theory CParserTest.bigstruct
CParserTest: theory CParserTest.breakcontinue
CParserTest: theory CParserTest.bug20060707
CParserTest: theory CParserTest.bug_mvt20110302
CParserTest: theory CParserTest.bugzilla180
CParserTest: theory CParserTest.bugzilla181
CParserTest: theory CParserTest.bugzilla182
CParserTest: theory CParserTest.builtins
CParserTest: theory CParserTest.charlit
CParserTest: theory CParserTest.codetests
CParserTest: theory CParserTest.dc_20081211
CParserTest: theory CParserTest.dc_embbug
CParserTest: theory CParserTest.decl_only
CParserTest: theory CParserTest.dont_translate
CParserTest: theory CParserTest.dupthms
CParserTest: theory CParserTest.emptystmt
CParserTest: theory CParserTest.extern_builtin
CParserTest: theory CParserTest.extern_dups
CParserTest: theory CParserTest.factorial
CParserTest: theory CParserTest.fncall
CParserTest: theory CParserTest.fnptr
CParserTest: theory CParserTest.gcc_attribs
CParserTest: theory CParserTest.ghoststate1
CParserTest: theory CParserTest.ghoststate2
CParserTest: theory CParserTest.globals_fn
CParserTest: theory CParserTest.globals_in_record
CParserTest: theory CParserTest.globinits
CParserTest: theory CParserTest.guard_while
CParserTest: theory CParserTest.hexliteral
CParserTest: theory CParserTest.initialised_decls
CParserTest: theory CParserTest.inner_fncalls
CParserTest: theory CParserTest.int_promotion
CParserTest: theory CParserTest.isa2014
CParserTest: theory CParserTest.jiraver039
CParserTest: theory CParserTest.jiraver092
CParserTest: theory CParserTest.jiraver105
CParserTest: theory CParserTest.jiraver110
CParserTest: theory CParserTest.jiraver1241
CParserTest: theory CParserTest.jiraver150
CParserTest: theory CParserTest.jiraver224
CParserTest: theory CParserTest.jiraver253
CParserTest: theory CParserTest.jiraver254
CParserTest: theory CParserTest.jiraver307
CParserTest: theory CParserTest.jiraver310
CParserTest: theory CParserTest.jiraver313
CParserTest: theory CParserTest.jiraver315
CParserTest: theory CParserTest.jiraver332
CParserTest: theory CParserTest.jiraver336
CParserTest: theory CParserTest.jiraver337
CParserTest: theory CParserTest.jiraver344
CParserTest: theory CParserTest.jiraver345
CParserTest: theory CParserTest.jiraver384
CParserTest: theory CParserTest.jiraver400
CParserTest: theory CParserTest.jiraver422
CParserTest: theory CParserTest.jiraver426
CParserTest: theory CParserTest.jiraver429
CParserTest: theory CParserTest.jiraver432
CParserTest: theory CParserTest.jiraver434
CParserTest: theory CParserTest.jiraver439
CParserTest: theory CParserTest.jiraver440
CParserTest: theory CParserTest.jiraver443
CParserTest: theory CParserTest.jiraver443a
CParserTest: theory CParserTest.jiraver456
CParserTest: theory CParserTest.jiraver464
CParserTest: theory CParserTest.jiraver473
CParserTest: theory CParserTest.jiraver54
CParserTest: theory CParserTest.jiraver550
CParserTest: theory CParserTest.jiraver808
CParserTest: theory CParserTest.jiraver881
CParserTest: theory CParserTest.kmalloc
CParserTest: theory CParserTest.list_reverse
CParserTest: theory CParserTest.list_reverse_norm
CParserTest: theory CParserTest.locvarfncall
CParserTest: theory CParserTest.longlong
CParserTest: theory CParserTest.many_local_vars
CParserTest: theory CParserTest.modifies_assumptions
CParserTest: theory CParserTest.modifies_speed
CParserTest: theory CParserTest.multi_deref
CParserTest: theory CParserTest.multidim_arrays
CParserTest: theory CParserTest.mutrec_modifies
CParserTest: theory CParserTest.parse_addr
CParserTest: theory CParserTest.parse_c99block
CParserTest: theory CParserTest.parse_complit
CParserTest: theory CParserTest.parse_dowhile
CParserTest: theory CParserTest.parse_enum
CParserTest: theory CParserTest.parse_fncall
CParserTest: theory CParserTest.parse_forloop
CParserTest: theory CParserTest.parse_include
CParserTest: theory CParserTest.parse_protos
CParserTest: theory CParserTest.parse_retfncall
CParserTest: theory CParserTest.parse_sizeof
CParserTest: theory CParserTest.parse_someops
CParserTest: theory CParserTest.parse_struct
CParserTest: theory CParserTest.parse_struct_array
CParserTest: theory CParserTest.parse_switch
CParserTest: theory CParserTest.parse_typecast
CParserTest: theory CParserTest.parse_voidfn
CParserTest: theory CParserTest.phantom_mstate
CParserTest: theory CParserTest.populate_globals
CParserTest: theory CParserTest.postfixOps
CParserTest: theory CParserTest.protoparamshadow
CParserTest: theory CParserTest.ptr_auxupd
CParserTest: theory CParserTest.ptr_diff
CParserTest: theory CParserTest.really_simple
CParserTest: theory CParserTest.relspec
CParserTest: theory CParserTest.retprefix
CParserTest: theory CParserTest.selection_sort
CParserTest: theory CParserTest.shortcircuit
CParserTest: theory CParserTest.signed_div
CParserTest: theory CParserTest.signedoverflow
CParserTest: theory CParserTest.simple_annotated_fn
CParserTest: theory CParserTest.simple_constexpr_sizeof
CParserTest: theory CParserTest.simple_fn
CParserTest: theory CParserTest.sizeof_typedef
CParserTest: theory CParserTest.spec_annotated_fn
CParserTest: theory CParserTest.spec_annotated_voidfn
CParserTest: theory CParserTest.swap
CParserTest: theory CParserTest.switch_unsigned_signed
CParserTest: theory CParserTest.test_shifts
CParserTest: theory CParserTest.ummbug20100217
CParserTest: theory CParserTest.untouched_globals
CParserTest: theory CParserTest.variable_munge
CParserTest: theory CParserTest.varinit
CParserTest: theory CParserTest.void_ptr_init
CParserTest: theory CParserTest.volatile_asm
CParserTest: theory CParserTest.ptr_modifies
Timing CParserTest (6 threads, 497.276s elapsed time, 2153.041s cpu time, 948.943s GC time, factor 4.33)
Finished CParserTest (0:08:19 elapsed time, 0:36:01 cpu time, factor 4.33)

Finished at Fri Aug 7 20:14:24 GMT+8 2020
0:08:25 elapsed time, 0:36:01 cpu time, factor 4.28

```

## vcg的感性认识

我们首先对vcg来个感性认识。
比如我们要验证下面的C函数：
```c
int h(int e)
{
  while (e < 10)
    /** INV: "\<lbrace> True \<rbrace>" */
  {
    if (e < -10) { continue; }
    if (e < 0) { break; }
    e = e - 1;
  }
  return e;
}
```

写出来的验证函数如下：
```
lemma h:
  "Γ ⊢ ⦃ -10 <=s ´e & ´e <s 0 ⦄
  ´ret__int :== PROC h(´e)
  ⦃ ´ret__int = ´e ⦄"
apply (hoare_rule HoarePartial.ProcNoRec1)
apply (hoare_rule HoarePartial.Catch [where R = "⦃ ´ret__int = ´e ⦄"])
  defer
  apply vcg
apply (hoare_rule HoarePartial.conseq
           [where P' = "λe. ⦃ ´e = e & e <s 0 & -10 <=s e ⦄"
            and Q' = "λe. ⦃ ´e = e & ´ret__int = e ⦄"
            and A' = "λe. ⦃ ´e = e & ´ret__int = e ⦄"])
  defer
  apply (simp add: subset_iff)
apply clarsimp
apply (rule_tac R="{}" in HoarePartial.Seq)
  defer
  apply vcg
apply (rule_tac R="⦃ ´e = Z ⦄" in HoarePartial.Seq)
  defer
  apply vcg
apply (rule_tac R = "⦃ ´e = Z & ´global_exn_var = Break ⦄" in HoarePartial.Catch)
  defer
  apply vcg
  apply simp
apply (rule_tac P' = "⦃ ´e = Z & Z <s 0 & -10 <=s Z ⦄"
            and Q' = "⦃ ´e = Z & Z <s 0 & -10 <=s Z ⦄ ∩ - ⦃ ´e <s 10 ⦄"
            and A' = "⦃ ´e = Z & ´global_exn_var = Break ⦄"
         in HoarePartial.conseq_no_aux)
  defer
  apply simp
apply (simp add: whileAnno_def)
apply (rule HoarePartialDef.While)
apply vcg
apply (simp add: subset_iff)
done
```

我们看到基本上推理都是apply vcg.
如果vcg自动推导失败，刚不得不写出更多的人工推理。
就可能变成这样的推导式：
```
lemma dotest:
  "Γ ⊢ ⦃ ´x = 4 ⦄ ´ret__int :== PROC dotest(´x)
       ⦃ ´ret__int = 4 ⦄"
apply (hoare_rule HoarePartial.ProcNoRec1)
apply (hoare_rule HoarePartial.Catch [where R="⦃ ´ret__int = 4 ⦄"])
  apply (hoare_rule HoarePartial.Seq [where R="{}"])
    apply (hoare_rule HoarePartial.Seq [where R="⦃ ´x = 4 ⦄"])
      apply (hoare_rule HoarePartial.Catch [where R="⦃ ´x = 4 & ´global_exn_var = Break ⦄"])
        apply (hoare_rule HoarePartial.Seq [where R="⦃ False ⦄"])
          apply (vcg, simp)
        apply (hoare_rule HoarePartial.conseq_exploit_pre, simp)
      apply (vcg, simp)
    apply vcg
  apply vcg
apply vcg
done
```

## 受限C语言子集与语法分析工具

seL4的验证只支持C语言的一个受限子集。所幸的是，限制并不太多：
- 不能使用goto语句
- switch语句中每个case结束时不能fall-through进入下个case，一定要有break, continue哪怕是return都行
- 不能使用union
- 结构体、枚举、typedef必须是全局的

可以使用c-parser工个来检查是否有违反规范的部分。
c-parser位于l4v/tools/c-parser/standalone-parser目录下，命令格式为：
```
c-parser 架构 c源代码
```
例：
```
l4v/tools/c-parser/standalone-parser/c-parser ARM asm_stmt.c
```

## 在HOL中读取C代码

有了CParser之后，我们就可以在HOL中引入CParser.CTranslation来将C源代码解析成可以在HOL中处理的结构：
```
imports "CParser.CTranslation"
```
引入了工具类之后，我们就可以将C源代码引入进来了：
```
external_file "breakcontinue.c"
```
下一步我们就可以用install_C_file来将这个c文件进行转换
```
install_C_file "breakcontinue.c"
```

具体的过程类似于下面这样：
```
Created locale for globals ("breakcontinue_global_addresses")- with 1 globals elements 
-- Fixes: symbol_table
 
1597124608: There are 0 globals: 
 
1597124608: There are 0 addressed variables: 
 
Defining record: myvars =
  global_exn_var :: c_exntype
  ret__int :: 32 signed word
  ret__int :: 32 signed word
  ret__int :: 32 signed word
  ret__int :: 32 signed word
  ret__int :: 32 signed word
  y___int :: 32 signed word
  x___int :: 32 signed word
  x___int :: 32 signed word
  e___int :: 32 signed word
  d___int :: 32 signed word
  c___int :: 32 signed word 
1597124611: Ignoring initialisations of modified globals (if any)
 
1597124611: Beginning function translation for all functions
 
1597124611: Translating function dotest
 
1597124611: Translating function i
 
1597124611: Translating function h
 
1597124611: Translating function g
 
1597124611: Translating function f
 
1597124611: Translated all functions
 
1597124611: Adding body_def for dotest_body
 
1597124611: Adding body_def for i_body
 
1597124611: Adding body_def for h_body
 
1597124611: Adding body_def for g_body
 
1597124611: Adding body_def for f_body
 
1597124611: Proving automatically calculated modifies proofs
 
1597124611: Globals_all_addressed mode = false
 
1597124611: Beginning modifies proof for singleton f
 
1597124611: Beginning modifies proof for singleton g
 
1597124611: Beginning modifies proof for singleton h
 
1597124611: Beginning modifies proof for singleton i
 
1597124611: Beginning modifies proof for singleton dotest
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.02
 
1597124611: modifies vcg-time:0.02
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.02
 
1597124611: modifies vcg-time:0.02
 
1597124611: modifies vcg-time:0.04
 
1597124611: modifies vcg-time:0.02
 
1597124611: modifies:g:0.14s:completed
 
1597124611: modifies vcg-time:0.02
 
1597124611: modifies:f:0.14s:completed
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.03
 
1597124611: modifies vcg-time:0.02
 
1597124611: modifies vcg-time:0.02
 
1597124611: modifies vcg-time:0.01
 
1597124611: modifies:h:0.16s:completed
 
1597124611: modifies vcg-time:0.02
 
1597124611: modifies vcg-time:0.01
 
1597124611: modifies:dotest:0.17s:completed
 
1597124611: modifies vcg-time:0.02
 
1597124611: modifies:i:0.18s:completed
```

## 完整例子骨架解析

### 例子源代码

再进入细节之前，我们把完整的C源代码和HOL源代码展示下：
```c
int f(int c)
{
  while (c < 10) {
    if (c < 0) { break; }
    c = c - 1;
  }
  return 3;
}

int g(int d)
{
  while (d < 10) {
    if (d < 0) { continue; }
    d = d - 1;
  }
  return 3;
}

int h(int e)
{
  while (e < 10)
    /** INV: "\<lbrace> True \<rbrace>" */
  {
    if (e < -10) { continue; }
    if (e < 0) { break; }
    e = e - 1;
  }
  return e;
}

int i(int x)
{
  int y;
  y = 10;
  while (x < 10) {
    while (x + y > 15) {
      if (x < 3) { break; }
      y = y - 2;
    }
    if (x < 3) { continue; }
  }
}

int dotest(int x)
{
  do {
    if (x < 10) break;
    x--;
  } while (x >= 10);
  return x;
}

int f(int c)
{
  while (c < 10) {
    c = c - 1;
  }
  return 3;
}
```

对应的验证代码如下：

```
theory breakcontinue
imports "CParser.CTranslation"
begin

declare sep_conj_ac [simp add]

external_file "breakcontinue.c"
install_C_file "breakcontinue.c"

context breakcontinue_global_addresses
begin

thm f_body_def
thm g_body_def
thm h_body_def
thm i_body_def
thm dotest_body_def

lemma h:
  "Γ ⊢ ⦃ -10 <=s ´e & ´e <s 0 ⦄
  ´ret__int :== PROC h(´e)
  ⦃ ´ret__int = ´e ⦄"
apply (hoare_rule HoarePartial.ProcNoRec1)
apply (hoare_rule HoarePartial.Catch [where R = "⦃ ´ret__int = ´e ⦄"])
  defer
  apply vcg
apply (hoare_rule HoarePartial.conseq
           [where P' = "λe. ⦃ ´e = e & e <s 0 & -10 <=s e ⦄"
            and Q' = "λe. ⦃ ´e = e & ´ret__int = e ⦄"
            and A' = "λe. ⦃ ´e = e & ´ret__int = e ⦄"])
  defer
  apply (simp add: subset_iff)
apply clarsimp
apply (rule_tac R="{}" in HoarePartial.Seq)
  defer
  apply vcg
apply (rule_tac R="⦃ ´e = Z ⦄" in HoarePartial.Seq)
  defer
  apply vcg
apply (rule_tac R = "⦃ ´e = Z & ´global_exn_var = Break ⦄" in HoarePartial.Catch)
  defer
  apply vcg
  apply simp
apply (rule_tac P' = "⦃ ´e = Z & Z <s 0 & -10 <=s Z ⦄"
            and Q' = "⦃ ´e = Z & Z <s 0 & -10 <=s Z ⦄ ∩ - ⦃ ´e <s 10 ⦄"
            and A' = "⦃ ´e = Z & ´global_exn_var = Break ⦄"
         in HoarePartial.conseq_no_aux)
  defer
  apply simp
apply (simp add: whileAnno_def)
apply (rule HoarePartialDef.While)
apply vcg
apply (simp add: subset_iff)
done

(* another example where vcg fails, generating impossible sub-goals *)
lemma dotest:
  "Γ ⊢ ⦃ ´x = 4 ⦄ ´ret__int :== PROC dotest(´x)
       ⦃ ´ret__int = 4 ⦄"
apply (hoare_rule HoarePartial.ProcNoRec1)
apply (hoare_rule HoarePartial.Catch [where R="⦃ ´ret__int = 4 ⦄"])
  apply (hoare_rule HoarePartial.Seq [where R="{}"])
    apply (hoare_rule HoarePartial.Seq [where R="⦃ ´x = 4 ⦄"])
      apply (hoare_rule HoarePartial.Catch [where R="⦃ ´x = 4 & ´global_exn_var = Break ⦄"])
        apply (hoare_rule HoarePartial.Seq [where R="⦃ False ⦄"])
          apply (vcg, simp)
        apply (hoare_rule HoarePartial.conseq_exploit_pre, simp)
      apply (vcg, simp)
    apply vcg
  apply vcg
apply vcg
done

end

end
```

### 函数体定义

前面几句我们已经了解了，是对C源码的解析：
```
theory breakcontinue
imports "CParser.CTranslation"
begin

declare sep_conj_ac [simp add]

external_file "breakcontinue.c"
install_C_file "breakcontinue.c"
```

下面一句
```
context breakcontinue_global_addresses
```
实际上对应的是一个locale语句：
```
locale breakcontinue_global_addresses
  fixes symbol_table :: "char list ⇒ 32 word"
```
关于locale，我们后面会详细介绍，这里先跳过，我们先要有个全局的印象。

下面就是函数定义体，以*_body_def开头的，跟C源代码中的函数一一对应：
```
thm f_body_def
thm g_body_def
thm h_body_def
thm i_body_def
thm dotest_body_def
```

这些thm是个什么东西呢？

我们展开其中一个来看一下：
```
f_body ≡
TRY
  TRY
    WHILE ´c <s 0xA DO
      IF ´c <s 0 THEN
        cbreak global_exn_var_'_update
      FI;;
      Guard SignedArithmetic
       ⦃- 2147483648 ≤ sint ´c - sint 1 ∧
        sint ´c - sint 1 ≤ 2147483647⦄
       (´c :== ´c - 1)
    OD
  CATCH ccatchbrk global_exn_var_'
  END;;
  creturn global_exn_var_'_update ret__int_'_update
   (λs. 3);;
  Guard DontReach {} SKIP
CATCH SKIP
END
```

我们跟f函数的代码对照着看，就会发现这基本上就是对这个函数C的逻辑的翻译：
```c
int f(int c)
{
  while (c < 10) {
    if (c < 0) { break; }
    c = c - 1;
  }
  return 3;
}
```

比如```while(c<10){...}```就被翻译成```WHILE ´c <s 0xA DO```
有一点不同是，因为HOL的数值类型与C的数值类型并不一样，是没有值的字节数表示的限制的，所以需要增加对值的范围的检查：
```
      Guard SignedArithmetic
       ⦃- 2147483648 ≤ sint ´c - sint 1 ∧
        sint ´c - sint 1 ≤ 2147483647⦄
       (´c :== ´c - 1)
```

### 验证函数功能

我们通过前面的工具翻译好C的逻辑之后，我们就可以来进行验证了。
从源代码看的话有一点tricky，我们将其展开之后，虽然有不少细节不理解，但是大致上我们可以理解验证的过程了。

我们要验证的h定理定义如下：
```
lemma h:
  "Γ ⊢ ⦃ -10 <=s ´e & ´e <s 0 ⦄
  ´ret__int :== PROC h(´e)
  ⦃ ´ret__int = ´e ⦄"
```

应用第一条规则HoarePartial.ProcNoRec1：
```
apply (hoare_rule HoarePartial.ProcNoRec1)
```
此时建立了第一个目标：
```
proof (prove)
goal (1 subgoal):
 1. Γ⊢ ⦃- 0xA <=s ´e ∧ ´e <s 0⦄
        TRY
          TRY
            WHILE ´e <s 0xA
            INV ⦃True⦄
            DO
              TRY
                Guard SignedArithmetic
                 ⦃- 2147483648 ≤ - sint 0xA ∧
                  - sint 0xA ≤ 2147483647⦄
                 (IF ´e <s - 0xA THEN
                    ´global_exn_var :== Continue;;
                    THROW
                  FI);;
                IF ´e <s 0 THEN
                  cbreak global_exn_var_'_update
                FI;;
                Guard SignedArithmetic
                 ⦃- 2147483648 ≤ sint ´e - sint 1 ∧
                  sint ´e - sint 1 ≤ 2147483647⦄
                 (´e :== ´e - 1)
              CATCH IF ´global_exn_var = Continue THEN
                      SKIP
                    ELSE
                      THROW
                    FI
              END
            OD
          CATCH ccatchbrk global_exn_var_'
          END;;
          creturn global_exn_var_'_update ret__int_'_update
           e_';;
          Guard DontReach {} SKIP
        CATCH SKIP
        END
        ⦃´ret__int = ´e⦄
```
我们加入一个Catch规则：
```
apply (hoare_rule HoarePartial.Catch [where R = "⦃ ´ret__int = ´e ⦄"])
```
现在变成两个目标，如下：
```
proof (prove)
goal (2 subgoals):
 1. Γ⊢ ⦃- 0xA <=s ´e ∧ ´e <s 0⦄
        TRY
          WHILE ´e <s 0xA
          INV ⦃True⦄
          DO
            TRY
              Guard SignedArithmetic
               ⦃- 2147483648 ≤ - sint 0xA ∧
                - sint 0xA ≤ 2147483647⦄
               (IF ´e <s - 0xA THEN
                  ´global_exn_var :== Continue;;
                  THROW
                FI);;
              IF ´e <s 0 THEN
                cbreak global_exn_var_'_update
              FI;;
              Guard SignedArithmetic
               ⦃- 2147483648 ≤ sint ´e - sint 1 ∧
                sint ´e - sint 1 ≤ 2147483647⦄
               (´e :== ´e - 1)
            CATCH IF ´global_exn_var = Continue THEN
                    SKIP
                  ELSE
                    THROW
                  FI
            END
          OD
        CATCH ccatchbrk global_exn_var_'
        END;;
        creturn global_exn_var_'_update ret__int_'_update
         e_';;
        Guard DontReach {} SKIP
        ⦃´ret__int = ´e⦄,⦃´ret__int = ´e⦄
 2. Γ⊢ ⦃´ret__int = ´e⦄ SKIP ⦃´ret__int = ´e⦄
```

然后我们apply一次vcg，将目标化简成一个：
```
proof (prove)
goal (1 subgoal):
 1. Γ⊢ ⦃- 0xA <=s ´e ∧ ´e <s 0⦄
        TRY
          WHILE ´e <s 0xA
          INV ⦃True⦄
          DO
            TRY
              Guard SignedArithmetic
               ⦃- 2147483648 ≤ - sint 0xA ∧
                - sint 0xA ≤ 2147483647⦄
               (IF ´e <s - 0xA THEN
                  ´global_exn_var :== Continue;;
                  THROW
                FI);;
              IF ´e <s 0 THEN
                cbreak global_exn_var_'_update
              FI;;
              Guard SignedArithmetic
               ⦃- 2147483648 ≤ sint ´e - sint 1 ∧
                sint ´e - sint 1 ≤ 2147483647⦄
               (´e :== ´e - 1)
            CATCH IF ´global_exn_var = Continue THEN
                    SKIP
                  ELSE
                    THROW
                  FI
            END
          OD
        CATCH ccatchbrk global_exn_var_'
        END;;
        creturn global_exn_var_'_update ret__int_'_update
         e_';;
        Guard DontReach {} SKIP
        ⦃´ret__int = ´e⦄,⦃´ret__int = ´e⦄
```

然后我们再加上一层约束规则：
```
apply (hoare_rule HoarePartial.conseq
           [where P' = "λe. ⦃ ´e = e & e <s 0 & -10 <=s e ⦄"
            and Q' = "λe. ⦃ ´e = e & ´ret__int = e ⦄"
            and A' = "λe. ⦃ ´e = e & ´ret__int = e ⦄"])
```
现在又变成两个子目标：
```
proof (prove)
goal (2 subgoals):
 1. ∀Z. Γ⊢ ⦃´e = Z ∧ Z <s 0 ∧ - 0xA <=s Z⦄
            TRY
              WHILE ´e <s 0xA
              INV ⦃True⦄
              DO
                TRY
                  Guard SignedArithmetic
                   ⦃- 2147483648 ≤ - sint 0xA ∧
                    - sint 0xA ≤ 2147483647⦄
                   (IF ´e <s - 0xA THEN
                      ´global_exn_var :== Continue;;
                      THROW
                    FI);;
                  IF ´e <s 0 THEN
                    cbreak global_exn_var_'_update
                  FI;;
                  Guard SignedArithmetic
                   ⦃- 2147483648 ≤ sint ´e - sint 1 ∧
                    sint ´e - sint 1 ≤ 2147483647⦄
                   (´e :== ´e - 1)
                CATCH IF ´global_exn_var = Continue THEN
                        SKIP
                      ELSE
                        THROW
                      FI
                END
              OD
            CATCH ccatchbrk global_exn_var_'
            END;;
            creturn global_exn_var_'_update
             ret__int_'_update e_';;
            Guard DontReach {} SKIP
            ⦃´e = Z ∧ ´ret__int = Z⦄,
            ⦃´e = Z ∧ ´ret__int = Z⦄
 2. ⋀s. - 0xA <=s e_' s ∧ e_' s <s 0 ⟹
         ∃Z. (e_' s = Z ∧ Z <s 0 ∧ - 0xA <=s Z) ∧
             ⦃´e = Z ∧ ´ret__int = Z⦄ ⊆ ⦃´ret__int = ´e⦄ ∧
             ⦃´e = Z ∧ ´ret__int = Z⦄ ⊆ ⦃´ret__int = ´e⦄
```

我们apply(simp add: subset_iff)来化简一下，现在只剩一个子目标了：
```
proof (prove)
goal (1 subgoal):
 1. ∀Z. Γ⊢ ⦃´e = Z ∧ Z <s 0 ∧ - 0xA <=s Z⦄
            TRY
              WHILE ´e <s 0xA
              INV ⦃True⦄
              DO
                TRY
                  Guard SignedArithmetic
                   ⦃- 2147483648 ≤ - sint 0xA ∧
                    - sint 0xA ≤ 2147483647⦄
                   (IF ´e <s - 0xA THEN
                      ´global_exn_var :== Continue;;
                      THROW
                    FI);;
                  IF ´e <s 0 THEN
                    cbreak global_exn_var_'_update
                  FI;;
                  Guard SignedArithmetic
                   ⦃- 2147483648 ≤ sint ´e - sint 1 ∧
                    sint ´e - sint 1 ≤ 2147483647⦄
                   (´e :== ´e - 1)
                CATCH IF ´global_exn_var = Continue THEN
                        SKIP
                      ELSE
                        THROW
                      FI
                END
              OD
            CATCH ccatchbrk global_exn_var_'
            END;;
            creturn global_exn_var_'_update
             ret__int_'_update e_';;
            Guard DontReach {} SKIP
            ⦃´e = Z ∧ ´ret__int = Z⦄,
            ⦃´e = Z ∧ ´ret__int = Z⦄
```

下面我们再通过apply clarsimp来化简一下：
```
proof (prove)
goal (1 subgoal):
 1. ⋀Z. Γ⊢ ⦃´e = Z ∧ Z <s 0 ∧ 0xFFFFFFF6 <=s Z⦄
            TRY
              WHILE ´e <s 0xA
              INV UNIV
              DO
                TRY
                  Guard SignedArithmetic UNIV
                   (IF ´e <s 0xFFFFFFF6 THEN
                      ´global_exn_var :== Continue;;
                      THROW
                    FI);;
                  IF ´e <s 0 THEN
                    cbreak global_exn_var_'_update
                  FI;;
                  Guard SignedArithmetic
                   ⦃- 2147483648 < sint ´e ∧
                    sint ´e ≤ 2147483648⦄
                   (´e :== ´e - 1)
                CATCH IF ´global_exn_var = Continue THEN
                        SKIP
                      ELSE
                        THROW
                      FI
                END
              OD
            CATCH ccatchbrk global_exn_var_'
            END;;
            creturn global_exn_var_'_update
             ret__int_'_update e_';;
            Guard DontReach {} SKIP
            ⦃´e = Z ∧ ´ret__int = Z⦄,
            ⦃´e = Z ∧ ´ret__int = Z⦄
```

后面的思路类似，都是apply rule_tac，然后再vcg加化简，最后整个目标被证明。

### 再看一个例子

前面堆的有点多，大家可能有点晕了。别担心，我们换个例子，大家再试图理解下：
```
lemma dotest:
  "Γ ⊢ ⦃ ´x = 4 ⦄ ´ret__int :== PROC dotest(´x)
       ⦃ ´ret__int = 4 ⦄"
apply (hoare_rule HoarePartial.ProcNoRec1)
apply (hoare_rule HoarePartial.Catch [where R="⦃ ´ret__int = 4 ⦄"])
  apply (hoare_rule HoarePartial.Seq [where R="{}"])
    apply (hoare_rule HoarePartial.Seq [where R="⦃ ´x = 4 ⦄"])
      apply (hoare_rule HoarePartial.Catch [where R="⦃ ´x = 4 & ´global_exn_var = Break ⦄"])
        apply (hoare_rule HoarePartial.Seq [where R="⦃ False ⦄"])
```

通过上一个例子，我们已经理解了，这些hoare_rule都是加规则，并没有进行任何化简与推理，所以这6条语句会形成6个子目标：
```
proof (prove)
goal (6 subgoals):
 1. Γ⊢ ⦃´x = 4⦄
        IF ´x <s 0xA THEN
          cbreak global_exn_var_'_update
        FI;;
        Guard SignedArithmetic
         ⦃- 2147483648 ≤ sint ´x - sint 1 ∧
          sint ´x - sint 1 ≤ 2147483647⦄
         (´x :== ´x - 1)
        ⦃False⦄,⦃´x = 4 ∧ ´global_exn_var = Break⦄
 2. Γ⊢ ⦃False⦄
        WHILE 0xA <=s ´x DO
          IF ´x <s 0xA THEN
            cbreak global_exn_var_'_update
          FI;;
          Guard SignedArithmetic
           ⦃- 2147483648 ≤ sint ´x - sint 1 ∧
            sint ´x - sint 1 ≤ 2147483647⦄
           (´x :== ´x - 1)
        OD
        ⦃´x = 4⦄,⦃´x = 4 ∧ ´global_exn_var = Break⦄
 3. Γ⊢ ⦃´x = 4 ∧ ´global_exn_var = Break⦄
        ccatchbrk global_exn_var_' ⦃´x = 4⦄,⦃´ret__int = 4⦄
 4. Γ⊢ ⦃´x = 4⦄
        creturn global_exn_var_'_update ret__int_'_update
         x_'
        {},⦃´ret__int = 4⦄
 5. Γ⊢ {} Guard DontReach {} SKIP ⦃´ret__int = 4⦄,
        ⦃´ret__int = 4⦄
 6. Γ⊢ ⦃´ret__int = 4⦄ SKIP ⦃´ret__int = 4⦄
```
应用一次apply(simp, vcg)，减少到5个：
```
proof (prove)
goal (5 subgoals):
 1. Γ⊢ ⦃False⦄
        WHILE 0xA <=s ´x DO
          IF ´x <s 0xA THEN
            cbreak global_exn_var_'_update
          FI;;
          Guard SignedArithmetic
           ⦃- 2147483648 ≤ sint ´x - sint 1 ∧
            sint ´x - sint 1 ≤ 2147483647⦄
           (´x :== ´x - 1)
        OD
        ⦃´x = 4⦄,⦃´x = 4 ∧ ´global_exn_var = Break⦄
 2. Γ⊢ ⦃´x = 4 ∧ ´global_exn_var = Break⦄
        ccatchbrk global_exn_var_' ⦃´x = 4⦄,⦃´ret__int = 4⦄
 3. Γ⊢ ⦃´x = 4⦄
        creturn global_exn_var_'_update ret__int_'_update
         x_'
        {},⦃´ret__int = 4⦄
 4. Γ⊢ {} Guard DontReach {} SKIP ⦃´ret__int = 4⦄,
        ⦃´ret__int = 4⦄
 5. Γ⊢ ⦃´ret__int = 4⦄ SKIP ⦃´ret__int = 4⦄
```
加一条规则，同时化简：```apply (hoare_rule HoarePartial.conseq_exploit_pre, simp)```
化简成4条子目标了，而且我们看到，跟代码结构密切相关的第一条子目标已经被证明了：
```
proof (prove)
goal (4 subgoals):
 1. Γ⊢ ⦃´x = 4 ∧ ´global_exn_var = Break⦄
        ccatchbrk global_exn_var_' ⦃´x = 4⦄,⦃´ret__int = 4⦄
 2. Γ⊢ ⦃´x = 4⦄
        creturn global_exn_var_'_update ret__int_'_update
         x_'
        {},⦃´ret__int = 4⦄
 3. Γ⊢ {} Guard DontReach {} SKIP ⦃´ret__int = 4⦄,
        ⦃´ret__int = 4⦄
 4. Γ⊢ ⦃´ret__int = 4⦄ SKIP ⦃´ret__int = 4⦄
```
再来一次apply(simp, vcg)，带Break的第一条也被消解掉了：
```
proof (prove)
goal (3 subgoals):
 1. Γ⊢ ⦃´x = 4⦄
        creturn global_exn_var_'_update ret__int_'_update
         x_'
        {},⦃´ret__int = 4⦄
 2. Γ⊢ {} Guard DontReach {} SKIP ⦃´ret__int = 4⦄,
        ⦃´ret__int = 4⦄
 3. Γ⊢ ⦃´ret__int = 4⦄ SKIP ⦃´ret__int = 4⦄
```
Again，一个vcg，creturn也被解决掉了：
```
proof (prove)
goal (2 subgoals):
 1. Γ⊢ {} Guard DontReach {} SKIP ⦃´ret__int = 4⦄,
        ⦃´ret__int = 4⦄
 2. Γ⊢ ⦃´ret__int = 4⦄ SKIP ⦃´ret__int = 4⦄
```
再来两次apply vcg，目标得证。

## 小结

这一节列了这么多，其实就是想让大家先有一个全局的框架：
1. 通过CParser.CTranslation来将C代码翻译成中间表示形式
2. 通过external_file来指定具体的c代码
3. 通过install_C_file完成翻译过程
4. 通过thm \*_body_def来获取对函数\*_body的表示
5. 通过定理中调用PROC去引用C源代码的中间表示
6. 通过hoare_rule来增加要验证的规则
7. 其中HoarePartial.ProcNoRec1是将C代码变成子目标的方法
8. 主要的证明和化简工具是simp的各种变种还有vcg
