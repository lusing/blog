# 操作系统形式化验证实践教程(5) - 搭建seL4环境

尽管我们有太多的基础知识都还没有讲，我们还是要快速进入到验证操作系统的世界中。先知道有什么用，然后再学习，可能是一种比较高效的方法。

## 搭建seL4验证环境

关于seL4有多牛，我们就不多介绍了，等我们了解了足够多的细节，回头再看，可能有更加鲜活的印象。基本上，seL4是第一个被完整形式化验证的有商用的操作系统。
架构图我们也后面再讲，先搞工程化，搭环境。

### 下载源代码

验证seL4需要三部分， seL4源代码，l4v检验脚本代码，以及isabelle。

我们创建一个验证的目录，然后将seL4代码和l4v代码clone下来：
```
git clone https://github.com/seL4/seL4
git clone https://github.com/seL4/l4v
```
接着还需要将isabelle的路径link到这个验证目录下。

### 安装依赖库

从上一节我们已经了解到，isabelle的依赖环境很复杂。但是，要验证操作系统，我们依赖的工具更多，比如交叉编译链，设备树编译器,cmake, ninja等，还有一系列辅助的python库。
在ubuntu下，可以使用apt-get安装部分依赖库：
```
sudo apt-get install \
    python3 python3-pip python3-dev \
    gcc-arm-none-eabi build-essential libxml2-utils ccache \
    ncurses-dev librsvg2-bin device-tree-compiler cmake \
    ninja-build curl zlib1g-dev texlive-fonts-recommended \
    texlive-latex-extra texlive-metapost texlive-bibtex-extra \
    mlton-compiler haskell-stack
```

### 安装python脚本

基本操作是安装sel4-deps库。
例如：
```
pip3 install sel4-deps --user
```
选择sudo pip install的，还有使用virtual-env的请自助。

### 设置isabelle环境

在l4v目录下，将l4vf的默认配置复制过来：
```
mkdir -p ~/.isabelle/etc
cp -i misc/etc/settings ~/.isabelle/etc/settings
./isabelle/bin/isabelle components -a
./isabelle/bin/isabelle jedit -bf
```

最后检验一下环境是否可以正常工作：
```
./isabelle/bin/isabelle build -bv HOL-Word
```

输出如下：
```
Started at Thu Aug 6 17:45:32 GMT+8 2020 (polyml-5.8.1_x86_64_32-darwin on IT-C02YX0A4LVDQ)
ISABELLE_BUILD_OPTIONS=""

ML_PLATFORM="x86_64_32-darwin"
ML_HOME="/Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/contrib/polyml-5.8.1-20200228/x86_64_32-darwin"
ML_SYSTEM="polyml-5.8.1"
ML_OPTIONS="--minheap 500"

Session Pure/Pure
Session Tools/Tools
Session HOL/HOL (main)
Session HOL/HOL-Library (main timing)
Session HOL/HOL-Word (main timing)

Finished at Thu Aug 6 17:45:35 GMT+8 2020
0:00:02 elapsed time
```

编译HOL-Word，需要依赖HOL/HOL-Libaray, HOL/HOL, Tools/Tools和最基础的Pure/Pure库。

因为没有ROOT文件，大家可能觉得有点抽象，不知道幕后究竟发生了什么。没关系，我们增加一个-l参数列出每个会话的每个文件列表，我们会看到一个长长的列表，我选贴一部分，要不然太长了：
```
Session Pure/Pure
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/contrib/polyml-5.8.1-20200228/x86_64_32-darwin/poly
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/cache.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/counter.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/event_timer.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/future.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/lazy.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/mailbox.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/multithreading.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/par_exn.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/par_list.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/single_assignment.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/standard_thread.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/synchronized.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/task_queue.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/thread_attributes.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/thread_data.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/thread_data_virtual.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/thread_position.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/timeout.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Pure/Concurrent/unsynchronized.ML
...
Session Tools/Tools
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_haskell.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_ml.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_namespace.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_preproc.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_printer.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_runtime.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_scala.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_simp.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_symbol.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_target.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code/code_thingol.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/Code_Generator.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/cache_io.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/Tools/nbe.ML
Session HOL/HOL (main)
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/ATP.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Archimedean_Field.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Argo.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/BNF_Cardinal_Arithmetic.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/BNF_Cardinal_Order_Relation.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/BNF_Composition.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/BNF_Def.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/BNF_Fixpoint_Base.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/BNF_Greatest_Fixpoint.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/BNF_Least_Fixpoint.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/BNF_Wellorder_Constructions.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/BNF_Wellorder_Embedding.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/BNF_Wellorder_Relation.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Basic_BNF_LFPs.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Basic_BNFs.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Binomial.thy
...
Session HOL/HOL-Library (main timing)
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/AList.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/AList_Mapping.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/Adhoc_Overloading.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/BNF_Axiomatization.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/BNF_Corec.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/BigO.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/Boolean_Algebra.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/Bourbaki_Witt_Fixpoint.thy
...
Session HOL/HOL-Word (main timing)
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/Boolean_Algebra.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/Cardinality.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/Numeral_Type.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/Phantom_Type.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/Type_Length.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Library/Z2.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Bit_Comprehension.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Bits.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Bits_Int.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Bits_Z2.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Misc_Arithmetic.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Misc_Auxiliary.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Misc_Typedef.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/More_Word.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Tools/smt_word.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Tools/word_lib.ML
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Word.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Word_Bitwise.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/Word_Examples.thy
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/document/root.bib
  /Applications/Isabelle2020.app/Contents/Resources/Isabelle2020/src/HOL/Word/document/root.tex

Finished at Thu Aug 6 17:59:10 GMT+8 2020
0:00:03 elapsed time
```

## 管理分支

另外需要注意的是管理代码分支与Isabelle/HOL的兼容性。
比如在目前的时间点上，默认分支使用Isabelle 2020是跑不通的，需要切换到isabelle-2020分支。
```
git checkout isabelle-2020
```

下面我们就可以通过跑测试来验证环境的问题了，运行l4v的run_tests脚本。大约需要十几个小时。

运行结果类似下面这样：
```
Testing for L4V_ARCH=ARM:
Testing Orphanage for ARM
Running 49 test(s)...
  Started isabelle ...
  Finished isabelle                  passed      ( 0:00:05 real,  0:00:22 cpu,  0.51GB)
  Started Lib ...
  Finished Lib                       passed      ( 0:00:56 real,  0:05:16 cpu,  3.31GB)
  Started CamkesGlueSpec ...
  Finished CamkesGlueSpec            passed      ( 0:00:31 real,  0:01:42 cpu,  1.83GB)
  Started Sep_Algebra ...
  Finished Sep_Algebra               passed      ( 0:00:39 real,  0:03:37 cpu,  2.91GB)
  Started Concurrency ...
  Finished Concurrency               passed      ( 0:01:09 real,  0:04:24 cpu,  1.87GB)
  Started tests-xml-correct ...
  Finished tests-xml-correct         FAILED *    ( 0:00:00 real,  0:00:00 cpu,  0.01GB)
  Started haskell-translator ...
  Finished haskell-translator        passed      ( 0:00:00 real,  0:00:00 cpu,  0.00GB)
  Started SepTactics ...
  Finished SepTactics                passed      ( 0:01:05 real,  0:05:20 cpu,  3.16GB)
  Started TakeGrant ...
  Finished TakeGrant                 passed      ( 0:00:28 real,  0:01:29 cpu,  1.75GB)
  Started ASpec ...
  Finished ASpec                     passed      ( 0:00:06 real,  0:00:23 cpu,  0.50GB)
  Started AInvs ...
  Finished AInvs                     passed      ( 0:19:53 real,  1:23:31 cpu, 14.11GB)
  Started BaseRefine ...
  Finished BaseRefine                passed      ( 0:04:39 real,  0:10:15 cpu,  5.49GB)
  Started Refine ...
  Finished Refine                    passed      ( 0:27:47 real,  2:25:11 cpu, 17.19GB)
  Started RefineOrphanage ...
  Finished RefineOrphanage           passed      ( 0:01:49 real,  0:05:21 cpu,  3.72GB)
  Started DBaseRefine ...
  Finished DBaseRefine               passed      ( 0:03:00 real,  0:07:30 cpu,  4.96GB)
  Started DRefine ...
  Finished DRefine                   passed      ( 0:08:57 real,  0:50:21 cpu,  5.12GB)
  Started Access ...
  Finished Access                    passed      ( 0:05:06 real,  0:24:18 cpu,  5.42GB)
  Started CamkesAdlSpec ...
  Finished CamkesAdlSpec             passed      ( 0:01:48 real,  0:03:42 cpu,  3.29GB)
  Started InfoFlow ...
  Finished InfoFlow                  passed      ( 0:25:37 real,  1:10:27 cpu,  8.55GB)
  Started DPolicy ...
  Finished DPolicy                   passed      ( 0:02:29 real,  0:07:55 cpu,  3.91GB)
  Started CamkesCdlRefine ...
  Finished CamkesCdlRefine           passed      ( 0:02:40 real,  0:06:38 cpu,  3.32GB)
  Started Bisim ...
  Finished Bisim                     passed      ( 0:00:47 real,  0:02:29 cpu,  2.87GB)
  Started ExecSpec ...
  Finished ExecSpec                  passed      ( 0:04:39 real,  0:11:51 cpu,  4.56GB)
  Started DSpec ...
  Finished DSpec                     passed      ( 0:02:39 real,  0:08:01 cpu,  4.40GB)
  Started SepDSpec ...
  Finished SepDSpec                  passed      ( 0:00:50 real,  0:04:05 cpu,  2.24GB)
  Started DSpecProofs ...
  Finished DSpecProofs               passed      ( 0:01:08 real,  0:06:09 cpu,  2.29GB)
  Started ASepSpec ...
  Finished ASepSpec                  passed      ( 0:00:20 real,  0:01:05 cpu,  1.55GB)
  Started HaskellKernel ...
  Finished HaskellKernel             FAILED *    ( 0:00:00 real,  0:00:00 cpu,  0.02GB)
  Started SysInit ...
  Finished SysInit                   passed      ( 0:02:58 real,  0:13:16 cpu,  3.42GB)
  Started SysInitExamples ...
  Finished SysInitExamples           passed      ( 0:02:19 real,  0:07:46 cpu,  2.20GB)
  Started CParser ...
  Finished CParser                   passed      ( 0:00:06 real,  0:00:25 cpu,  0.55GB)
  Started CLib ...
  Finished CLib                      passed      ( 0:00:42 real,  0:04:26 cpu,  3.65GB)
  Started LibTest ...
  Finished LibTest                   passed      ( 0:04:23 real,  0:13:57 cpu,  6.98GB)
  Started c-kernel ...
  Finished c-kernel                  passed      ( 0:00:01 real,  0:00:00 cpu,  0.02GB)
  Started CKernel ...
  Finished CKernel                   passed      ( 0:00:07 real,  0:00:26 cpu,  0.50GB)
  Started CSpec ...
  Finished CSpec                     passed      ( 0:07:10 real,  0:38:37 cpu,  9.98GB)
  Started CBaseRefine ...
  Finished CBaseRefine               passed      ( 1:07:16 real,  6:04:21 cpu, 17.59GB)
  Started CRefine ...
  Finished CRefine                   passed      ( 1:01:46 real,  4:59:21 cpu, 17.40GB)
  Started InfoFlowCBase ...
  Finished InfoFlowCBase             passed      ( 0:32:30 real,  1:32:45 cpu, 10.14GB)
  Finished SimplExportAndRefine      passed      ( 3:15:33 real,  7:36:36 cpu,  8.88GB)
  Started CParserTest ...
  Finished CParserTest               passed      ( 0:00:06 real,  0:00:23 cpu,  0.51GB)
  Started CParserTools ...
  Finished CParserTools              passed      ( 0:00:01 real,  0:00:00 cpu,  0.02GB)
  Started AutoCorres ...
  Finished AutoCorres                passed      ( 0:01:01 real,  0:05:48 cpu,  3.34GB)
  Started CamkesGlueProofs ...
  Finished CamkesGlueProofs          passed      ( 0:13:20 real,  0:45:57 cpu,  4.98GB)
  Started AutoCorresDoc ...
  Finished AutoCorresDoc             passed      ( 0:00:18 real,  0:00:53 cpu,  1.84GB)
  Started AutoCorresTest ...
  Finished AutoCorresTest            passed      ( 0:05:11 real,  0:32:09 cpu,  9.17GB)
```

其中最长的用例SimplExportAndRefine消耗7个多小时CPU时间。
