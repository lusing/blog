---
title: "操作系统形式化验证实践教程(2) - HOL列表与集合"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

在进入相对比较烧脑的证明过程之前，我们先熟悉下HOL语言中处理列表和集合数据结构的部分。
这部分各种函数式语言其实是大同小异的，学习成本比较低。

## 列表类型

HOL的列表类型与别的语言比较像，也是使用[]来表示。类型只要相同就可以，比如我们以整数类型列表为例：

```
value "[uminus int(1),int(0),int(1)]"
```
值和类型如下：
```
"[- 1, 0, 1]"
  :: "int list"
```

[]是Nil，是空列表。
除了用[int1,int2]这样表示外，也可以用int1 # int2 # []的方式来描述。#是Cons操作，为将两个列表连接在一起的操作。

来看个例子：
```
value "nat(2) # nat(10)# []"
``` 
值为：
```
"[2, 10]"
  :: "nat list"
```

可以用".."来生成序列，例：
```
value "[int(1)..int(10)]"
```
值为：
```
"[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]"
  :: "int list"
```

### 求表头

hd函数用于求列表表头。
例：
```
value "hd [int(1),int(3),int(5)]"
```

结果为：
```
"1"
  :: "int"
```

### 除表头外剩余部分

用tl函数来求除表头外剩余部分，比如下表，就是除掉表头1之后，2到10的列表：
```
value "tl [int(1)..int(10)]"
```
结果为：
```
"[2, 3, 4, 5, 6, 7, 8, 9, 10]"
  :: "int list"
```

### 求表尾元素

与tl不同，last用来求最后一个元素，例：
```
value "last [int(1)..int(20)]"
```
结果为20：
```
"20"
  :: "int"
```

### 求列表长度

length函数可以计算列表的长度，例：
```
value "length [int(1)..int(102)]"
```
结果为：
```
"102"
  :: "nat"
```

请注意长度结果是nat自然数，而不是int整数。因为长度不可能为负的。

### 截掉列表的内容

比如截掉头部的两个元素，可以通过drop 2 [列表] 来实现：
```
value "drop 2 [int(1)..int(5)]"
```
输出结果为：
```
"[3, 4, 5]"
  :: "int list"
```

### 列表反序

reverse将现有列表顺序反过来：
```
value "rev [int(1)..int(5)]"
```
结果为：
```
"[5, 4, 3, 2, 1]"
  :: "int list"
```

### 列表值求和

HOL中的sum_list函数可以求数值列表的元素的值的和：

```
value "sum_list [int(1)..int(100)]"
```
结果为：
```
"5050"
  :: "int"
```

### 排序

列表支持sort函数用来排序：
```
value "sort (rev[int(5)..int(10)])"
```
结果为：
```
"[5, 6, 7, 8, 9, 10]"
  :: "int list"
```

### 将列表转换成集合

通过set函数可以将列表转换成集合：
```
value "set [int(1)..int(5)]"
```
结果如下：
```
"{1, 2, 3, 4, 5}"
  :: "int set"
```

上面的内容写得相对比较多，不是想写成手册，而是希望借着操练比较熟悉的内容的机会，让大家尽可能地熟悉Isabelle/HOL的环境，消除恐惧感。

## 集合

与列表类似，HOL支持以{}表示的集合类型。

与列表一样，集合也支持通过序列生成：
```
value "{int(1)..int(10)}"
```
结果为：
```
"{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}"
  :: "int set"
```

### 插入元素

通过insert函数将元素插入到集合中：
```
value "insert (int(100)) {int(1)..int(10)}"
```
结果为：
```
"{100, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10}"
  :: "int set"
```

### 判断元素是否属于集合

比起其它语言用的in操作，HOL直接使用∈符号，输入时写作'\\<in>'
```
value "(int(10)) ∈ {int(1)..int(100)}"
```

源代码实际上为：
```
value "(int(10)) \<in> {int(1)..int(100)}"
```

### 交集与并集

交集和并集直接使用∩和∪。输入时用\\<inter>和\\<union>表示。

我们看下并集的例子：
```
value "{int(1)..int(5)} ∪ {int(100)..int(120)}"
```
输出为：
```
"{5, 4, 3, 2, 1, 100, 101, 102, 103, 104, 105, 106, 107,
  108, 109, 110, 111, 112, 113, 114, 115, 116, 117, 118,
  119, 120}"
  :: "int set"
``` 

![并集](https://upload-images.jianshu.io/upload_images/1638145-6ccc6137cd8e1c8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再来看一个交集的例子：

```
value "{int(1)..int(105)} ∩ {int(100)..int(120)}"
```
结果如下：
```
"{100, 101, 102, 103, 104, 105}"
  :: "int set"
```

### 幂集

幂集是由集合的所有子集所组合的集合。
我们看个最简单的例子，大家就能理解了：
```
value "Pow {int(1)..int(2)}"
```
组合的结果应该为空集，1的集合，2的集合和这个集合本身。
```
"{{2}, {}, {1}, {1, 2}}"
  :: "int set set"
```

巩固一下，如果扩展到5个元素，你还能写全吗？

```
value "Pow {int(1)..int(5)}"
```
结果为：
```
"{{2, 3, 4, 5}, {2, 3, 4}, {2, 3}, {2, 3, 5}, {2, 5}, {2},
  {2, 4}, {2, 4, 5}, {4, 5}, {4}, {}, {5}, {3, 5}, {3},
  {3, 4}, {3, 4, 5}, {1, 3, 4, 5}, {1, 3, 4}, {1, 3},
  {1, 3, 5}, {1, 5}, {1}, {1, 4}, {1, 4, 5}, {1, 2, 4, 5},
  {1, 2, 4}, {1, 2}, {1, 2, 5}, {1, 2, 3, 5}, {1, 2, 3},
  {1, 2, 3, 4}, {1, 2, 3, 4, 5}}"
  :: "int set set"
```

### 量词

HOL终于要开始展示真实本领了，作为一种逻辑语言，它是支持量词的，也就是$\forall$和$\exists$. 

我们还是直接上个例子，对于某个整数集合，所有元素都大于0：
```
value "∀x∈{int(1)..int(10000)}. x>0"
```
这自然是真的：
```
"True"
  :: "bool"
```

![量词](https://upload-images.jianshu.io/upload_images/1638145-44d911de4e1f1c20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这一期我们从比较熟悉的列表和集合操作，引入一点点逻辑相关的操作。其实就是想让大家看到，让这些LaTex符号运行起来，其实也没有什么神秘的。

最后，我们发一下这一节完整的源代码：
```
theory Test2
  imports Main
begin
value "hd [int(1),int(3),int(5)]"
value "nat(2) # nat(10)# []"
value "int(2) # int(3) # []"
value "tl [nat(2),nat(3)]"
value "[uminus int(1),int(0),int(1)]"
value "[int(1)..int(10)]"
value "tl [int(1)..int(10)]"
value "last [int(1)..int(20)]"
value "length [int(1)..int(102)]"
value "sum_list [int(1)..int(100)]"
value "drop 2 [int(1)..int(5)]"
value "rev [int(1)..int(5)]"
value "set [int(1)..int(5)]"
value "sort (rev[int(5)..int(10)])"
value "{int(1)..int(10)}"
value "insert (int(100)) {int(1)..int(10)}"
value "(int(10)) \<in> {int(1)..int(100)}"
value "{int(1)..int(5)} \<union> {int(100)..int(120)}"
value "{int(1)..int(105)} \<inter> {int(100)..int(120)}"
value "\<forall>x\<in>{int(1)..int(10000)}. x>0"
value "Pow {int(1)..int(2)}"
value "Pow {int(1)..int(5)}"
value "-{int(0)..int(10)}"
(* Tests on values *)
end
```
