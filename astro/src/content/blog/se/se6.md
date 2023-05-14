---
title: "操作系统形式化验证实践教程(1) - 证明第一个定理"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

# 符号执行(6) - clang静态分析器带你一起读代码

符号执行除了可以生成较高的覆盖率的测试用例之外，其实还有很多用途，其中最广为人知的可能是静态扫描代码中的问题。

clang是一个强大的C++编译前端大家都早就很熟悉了，大家在编译中可能都有很多警告没有处理。
其实，编译过程中为了加快编译速度，实现多项式复杂度的时间，并不能覆盖很多路径。数据流分析的时候，有很多时候状态是迭加在一起无法判断的。
如果我们愿意花更多的时间去检查代码的质量，可以使用clang提供的静态分析器。据说clang分析器是与行业两大巨头工具惠普的Fortify和新思的Coverity齐名的三大静态分析工具，而且是三者中唯一免费且开源的。

## 让clang给你讲为什么它认为是bug

我们先来个例子，看看clang分析器发现的bug的报告是什么样的。这是我扫描ctags工具发现的bug列表。

首页平平无齐，就跟你熟悉的静态扫描报告一样：
![ctags-bugs.png](https://upload-images.jianshu.io/upload_images/1638145-21a0eb3cac6287ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们挑两类相对复杂的问题，一类是空指针空引用问题，一类是内存泄漏的问题。

### 空指针路径

先看空指针吧，程序这么复杂，你说我空指针就空指针了？
嗯，clang知道你不服，那就指给你看。我们选一个步骤最少的看下：
![null-7.png](https://upload-images.jianshu.io/upload_images/1638145-a97e0efb7e60b36b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

报告里说，第7步出现了空指针。

如何空了？别急，点击左边的箭头，分析器带我们去看第6步。第6步被桔红色高亮了：
![null-6.png](https://upload-images.jianshu.io/upload_images/1638145-c4d96a277a88453e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

明白了，原来第4步虽然给trash_box进行了判空，但是赋给的defaultTrashBox变量的值，分析器认为是可能为空的，这样就导致空指针的引用。

这样只是猜测，还拿不准。我们再往回找第3步，看看调到第4步的路径是如何来的：
![null-3.png](https://upload-images.jianshu.io/upload_images/1638145-1e59961fa762805b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这张图，我们从第1步的逻辑开始看。假设trash_box为空，第2步走true的分支，然后把defaultTrashBox赋给trash_box，再去调用trashBoxTakeBack。
结果走到trashBoxTakeBack，也就是第4步，defaultTrashBox带来的值还是个空。判空之后，又把这个空赋给trash_box，导致空指针。

clang分析器的这套逻辑看起来是合理的。至少是让我们理解了为什么它是这样认为的。
有了clang分析器的符号执行的结果，等于是帮我们做了笔记，比我们对着电脑去猜测还是有帮助的。也不用去debug去停下来一步步查看变量值。也不用忍受valgrind的慢速度。

### 内存泄漏

我们再看一个内存泄漏的例子。内存泄漏的问题一般复杂在，经常有重复调用的地方，这时候clang分析器帮我们标注出来步骤就特别有用。

![leak-12.png](https://upload-images.jianshu.io/upload_images/1638145-f4f1fe926fc1d4aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上面的图可以看到，第3步和第9步执行的同一行代码，如果clang分析器不给我们指路的话，我们很难脑补第4到第8步是在哪里运行的。

我们往第11步倒，我们认为ctx是发生了error，所以got_error为真。往第10步倒，正是走的false分支，所以在第11步直接return了。

回到第9步与第3步的状态，我们再往第8步回退：
![leak-8.png](https://upload-images.jianshu.io/upload_images/1638145-305cd28a476e4222.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第8步我们假设分配到了内存。第8步与第4步又是共享同一条语句的，如果没有这个标记还真容易乱。

往第7步倒，原来5~7步是PCC_MALLOC，第4步告诉我们，它是真名是pcc_malloc_e：
![leak-7.png](https://upload-images.jianshu.io/upload_images/1638145-6c174c57609dcbad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个没啥意外，是成功的路径，分配了内存。

所以这个是在错误分支上只上报错，没有处理回收内存，clang的分析还是有道理的。

## 如何使用clang分析器

clang为我们提供了一个scan-build工具，会帮我们去做hook编译命令之类的底层操作。

在mac上，默认安装的clang没有带scan-build工具，我们指定自己安装的llvm目录下的就好。

比如，下载好ctags之后，我们直接用scan-build来执行make就好：
```
/usr/local/Cellar/llvm/11.0.0/bin/scan-build make
```
编译成功后，会提示我们如何看报告：
```
scan-build: Analysis run complete.
scan-build: 75 bugs found.
scan-build: Run 'scan-view /var/folders/tx/h74p603j4yqgf95_815cnxg00000gp/T/scan-build-2020-10-27-211852-71001-1' to examine bug reports.
```

照抄命令：
```
/usr/local/Cellar/llvm/11.0.0/bin/scan-view /var/folders/tx/h74p603j4yqgf95_815cnxg00000gp/T/scan-build-2020-10-27-211852-71001-1
```

scan-view会帮我们创建一个文档服务器地址：
```
Starting scan-view at: http://127.0.0.1:8181
```

在浏览器里打开http://127.0.0.1:8181/，就可以看到上面的报告了
