---
title: "符号执行(7) - clang静态扫描进阶"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---


通过前面的学习，我们了解到符号执行技术其实是有很多限制的。
为了提高准确率，减少误报，我们有三件事情可以做：
- 第一是收集信息了解内部状态，找到哪里薄弱哪里有限制，我们才好有针对性地去改进
- 第二是了解有哪些checker，根据情况配置合适checker.
- 第三是调整一些参数。比如默认clang分析器为了节省时间，循环只执行4次，我们可以根据情况适当扩展

## 收集更多信息

 scan-build命令是可以支持很多参数的。

### stats参数：数据流信息

首先我们可以给scan-build增加-stats参数。
例：
```
scan-build -stats make
```

这个参数的结果是可以显示块的信息，分析了多少个块，有多少块不可达等：
```
closeInputFile -> Total CFGBlocks: 6 | Unreachable CFGBlocks: 0 | Exhausted Block: no | Empty WorkList: yes
writerWriteTag -> Total CFGBlocks: 3 | Unreachable CFGBlocks: 0 | Exhausted Block: no | Empty WorkList: yes
processListLangdefFlagsOptions -> Total CFGBlocks: 3 | Unreachable CFGBlocks: 0 | Exhausted Block: no | Empty WorkList: yes
(isSubroutineDeclaration): The analyzer generated a sink at this point
(cxxParserExtractVariableDeclarations): The analyzer generated a sink at this point
(findBetaTags): The analyzer generated a sink at this point
processIf0Option -> Total CFGBlocks: 6 | Unreachable CFGBlocks: 0 | Exhausted Block: no | Empty WorkList: yes
localLet -> Total CFGBlocks: 20 | Unreachable CFGBlocks: 0 | Exhausted Block: no | Empty WorkList: yes
findErlangTags -> Total CFGBlocks: 14 | Unreachable CFGBlocks: 0 | Exhausted Block: no | Empty WorkList: no
isConverting -> Total CFGBlocks: 3 | Unreachable CFGBlocks: 0 | Exhausted Block: no | Empty WorkList: yes
Analyzed 17720 blocks in 1437 functions in 158 files
322 functions aborted early (22.41%)
255 had aborted blocks (17.75%)
139 had unfinished worklists (9.67%)
1634 blocks were never reached (9.22%)
scan-build: 46 bugs found.
```

### internal-stats参数: 耗时信息

internal-stats参数会打印处理每个文件所花的时间：路径探索的时间，语法相关分析时间，路径相关后处理时间。

```
  CC       main/libctags_a-keyword.o
===-------------------------------------------------------------------------===
                                Analyzer timers
===-------------------------------------------------------------------------===
  Total Execution Time: 0.0597 seconds (0.0600 wall clock)

   ---User Time---   --System Time--   --User+System--   ---Wall Time---  --- Name ---
   0.0584 ( 98.7%)   0.0000 (  0.0%)   0.0584 ( 97.9%)   0.0587 ( 97.9%)  Path exploration time
   0.0007 (  1.3%)   0.0005 (100.0%)   0.0013 (  2.1%)   0.0013 (  2.1%)  Syntax-based analysis time
   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)  Path-sensitive report post-processing time
   0.0592 (100.0%)   0.0005 (100.0%)   0.0597 (100.0%)   0.0600 (100.0%)  Total

  CC       parsers/libctags_a-itcl.o
parsers/itcl.c:240:5: warning: Value stored to 'protection' is never read
                                protection = KEYWORD_NONE;
                                ^            ~~~~~~~~~~~~
===-------------------------------------------------------------------------===
                                Analyzer timers
===-------------------------------------------------------------------------===
  Total Execution Time: 2.1349 seconds (2.1474 wall clock)

   ---User Time---   --System Time--   --User+System--   ---Wall Time---  --- Name ---
   2.0878 ( 99.9%)   0.0455 ( 99.8%)   2.1333 ( 99.9%)   2.1458 ( 99.9%)  Path exploration time
   0.0015 (  0.1%)   0.0001 (  0.2%)   0.0016 (  0.1%)   0.0016 (  0.1%)  Syntax-based analysis time
   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)  Path-sensitive report post-processing time
   2.0893 (100.0%)   0.0456 (100.0%)   2.1349 (100.0%)   2.1474 (100.0%)  Total
```

### -v三连

除了上面两种统计信息之外，Clang静态分析器CSA还为我们准备了丰富的过程中的信息，包括路径探索的过程，语法分析的过程等。

使用方法很简单，加-v，最多可以加三个，加得越多信息越丰富。

例：
```
scan-build -v -v -v make
```

比如我们以emacs里的terminfo.c的分析为例，人家原本的命令行是这样的：
```
  CC       terminfo.o
gcc -c -Demacs -I. -I. -I../lib -I../lib -isystem /usr/include/gtk-3.0 -isystem /usr/include/pango-1.0 -isystem /usr/include/glib-2.0 -isystem /usr/lib/glib-2.0/include -isystem /usr/include/harfbuzz -isystem /usr/include/freetype2 -isystem /usr/include/libpng16 -isystem /usr/include/libmount -isystem /usr/include/blkid -isystem /usr/include/fribidi -isystem /usr/include/cairo -isystem /usr/include/pixman-1 -isystem /usr/include/gdk-pixbuf-2.0 -isystem /usr/include/gio-unix-2.0 -isystem /usr/include/cloudproviders -isystem /usr/include/atk-1.0 -isystem /usr/include/at-spi2-atk/2.0 -isystem /usr/include/dbus-1.0 -isystem /usr/lib/dbus-1.0/include -isystem /usr/include/at-spi-2.0 -pthread -isystem /usr/include/librsvg-2.0 -isystem /usr/include/glib-2.0 -isystem /usr/lib/glib-2.0/include -isystem /usr/include/libmount -isystem /usr/include/blkid -isystem /usr/include/gdk-pixbuf-2.0 -pthread -isystem /usr/include/cairo -isystem /usr/include/pixman-1 -isystem /usr/include/freetype2 -isystem /usr/include/libpng16 -isystem /usr/include/harfbuzz -isystem /usr/include/libpng16 -isystem /usr/include/libxml2 -isystem /usr/include/dbus-1.0 -isystem /usr/lib/dbus-1.0/include -isystem /usr/include/glib-2.0 -isystem /usr/lib/glib-2.0/include -pthread -isystem /usr/include/libmount -isystem /usr/include/blkid -isystem /usr/include/glib-2.0 -isystem /usr/lib/glib-2.0/include -isystem /usr/include/freetype2 -isystem /usr/include/libpng16 -isystem /usr/include/harfbuzz -isystem /usr/include/glib-2.0 -isystem /usr/lib/glib-2.0/include -isystem /usr/include/freetype2 -isystem /usr/include/libpng16 -isystem /usr/include/harfbuzz -isystem /usr/include/glib-2.0 -isystem /usr/lib/glib-2.0/include -isystem /usr/include/harfbuzz -isystem /usr/include/freetype2 -isystem /usr/include/libpng16 -isystem /usr/include/glib-2.0 -isystem /usr/lib/glib-2.0/include -isystem /usr/include/freetype2 -isystem /usr/include/libpng16 -isystem /usr/include/harfbuzz -isystem /usr/include/glib-2.0 -isystem /usr/lib/glib-2.0/include -MMD -MF deps/terminfo.d -MP -isystem /usr/include/p11-kit-1 -isystem /usr/include/cairo -isystem /usr/include/glib-2.0 -isystem /usr/lib/glib-2.0/include -isystem /usr/include/pixman-1 -isystem /usr/include/freetype2 -isystem /usr/include/libpng16 -isystem /usr/include/harfbuzz -fno-common -Wall -Warith-conversion -Wdate-time -Wdisabled-optimization -Wdouble-promotion -Wduplicated-cond -Wextra -Wformat-signedness -Winit-self -Winvalid-pch -Wlogical-op -Wmissing-declarations -Wmissing-include-dirs -Wmissing-prototypes -Wnested-externs -Wnull-dereference -Wold-style-definition -Wopenmp-simd -Wpacked -Wpointer-arith -Wstrict-prototypes -Wsuggest-attribute=format -Wsuggest-attribute=noreturn -Wsuggest-final-methods -Wsuggest-final-types -Wtrampolines -Wuninitialized -Wunknown-pragmas -Wunused-macros -Wvariadic-macros -Wvector-operation-performance -Wwrite-strings -Warray-bounds=2 -Wattribute-alias=2 -Wformat=2 -Wformat-truncation=2 -Wimplicit-fallthrough=5 -Wshift-overflow=2 -Wvla-larger-than=4031 -Wredundant-decls -Wno-missing-field-initializers -Wno-override-init -Wno-sign-compare -Wno-type-limits -Wno-unused-parameter -Wno-format-nonliteral -g3 -O2 terminfo.c
```

被scan-build魔改之后变成这样了：
```
[LOCATION]: /workspace/xulun/github/lang/emacs/src
#SHELL (cd '/workspace/xulun/github/lang/emacs/src' && '/usr/bin/clang-10' '-cc1' '-triple' 'x86_64-pc-linux-gnu' '-analyze' '-disable-free' '-disable-llvm-verifier' '-discard-value-names' '-main-file-name' 'terminfo.c' '-analyzer-store=region' '-analyzer-opt-analyze-nested-blocks' '-analyzer-checker=core' '-analyzer-checker=apiModeling' '-analyzer-checker=unix' '-analyzer-checker=deadcode' '-analyzer-checker=security.insecureAPI.UncheckedReturn' '-analyzer-checker=security.insecureAPI.getpw' '-analyzer-checker=security.insecureAPI.gets' '-analyzer-checker=security.insecureAPI.mktemp' '-analyzer-checker=security.insecureAPI.mkstemp' '-analyzer-checker=security.insecureAPI.vfork' '-analyzer-checker=nullability.NullPassedToNonnull' '-analyzer-checker=nullability.NullReturnedFromNonnull' '-analyzer-output' 'plist' '-w' '-setup-static-analyzer' '-mrelocation-model' 'pic' '-pic-level' '2' '-pic-is-pie' '-mthread-model' 'posix' '-mframe-pointer=none' '-fmath-errno' '-fno-rounding-math' '-masm-verbose' '-mconstructor-aliases' '-munwind-tables' '-target-cpu' 'x86-64' '-dwarf-column-info' '-fno-split-dwarf-inlining' '-debugger-tuning=gdb' '-resource-dir' '/usr/lib/clang/10.0.1' '-isystem' '/usr/include/gtk-3.0' '-isystem' '/usr/include/pango-1.0' '-isystem' '/usr/include/glib-2.0' '-isystem' '/usr/lib/glib-2.0/include' '-isystem' '/usr/include/harfbuzz' '-isystem' '/usr/include/freetype2' '-isystem' '/usr/include/libpng16' '-isystem' '/usr/include/libmount' '-isystem' '/usr/include/blkid' '-isystem' '/usr/include/fribidi' '-isystem' '/usr/include/cairo' '-isystem' '/usr/include/pixman-1' '-isystem' '/usr/include/gdk-pixbuf-2.0' '-isystem' '/usr/include/gio-unix-2.0' '-isystem' '/usr/include/cloudproviders' '-isystem' '/usr/include/atk-1.0' '-isystem' '/usr/include/at-spi2-atk/2.0' '-isystem' '/usr/include/dbus-1.0' '-isystem' '/usr/lib/dbus-1.0/include' '-isystem' '/usr/include/at-spi-2.0' '-isystem' '/usr/include/librsvg-2.0' '-isystem' '/usr/include/glib-2.0' '-isystem' '/usr/lib/glib-2.0/include' '-isystem' '/usr/include/libmount' '-isystem' '/usr/include/blkid' '-isystem' '/usr/include/gdk-pixbuf-2.0' '-isystem' '/usr/include/cairo' '-isystem' '/usr/include/pixman-1' '-isystem' '/usr/include/freetype2' '-isystem' '/usr/include/libpng16' '-isystem' '/usr/include/harfbuzz' '-isystem' '/usr/include/libpng16' '-isystem' '/usr/include/libxml2' '-isystem' '/usr/include/dbus-1.0' '-isystem' '/usr/lib/dbus-1.0/include' '-isystem' '/usr/include/glib-2.0' '-isystem' '/usr/lib/glib-2.0/include' '-isystem' '/usr/include/libmount' '-isystem' '/usr/include/blkid' '-isystem' '/usr/include/glib-2.0' '-isystem' '/usr/lib/glib-2.0/include' '-isystem' '/usr/include/freetype2' '-isystem' '/usr/include/libpng16' '-isystem' '/usr/include/harfbuzz' '-isystem' '/usr/include/glib-2.0' '-isystem' '/usr/lib/glib-2.0/include' '-isystem' '/usr/include/freetype2' '-isystem' '/usr/include/libpng16' '-isystem' '/usr/include/harfbuzz' '-isystem' '/usr/include/glib-2.0' '-isystem' '/usr/lib/glib-2.0/include' '-isystem' '/usr/include/harfbuzz' '-isystem' '/usr/include/freetype2' '-isystem' '/usr/include/libpng16' '-isystem' '/usr/include/glib-2.0' '-isystem' '/usr/lib/glib-2.0/include' '-isystem' '/usr/include/freetype2' '-isystem' '/usr/include/libpng16' '-isystem' '/usr/include/harfbuzz' '-isystem' '/usr/include/glib-2.0' '-isystem' '/usr/lib/glib-2.0/include' '-isystem' '/usr/include/p11-kit-1' '-isystem' '/usr/include/cairo' '-isystem' '/usr/include/glib-2.0' '-isystem' '/usr/lib/glib-2.0/include' '-isystem' '/usr/include/pixman-1' '-isystem' '/usr/include/freetype2' '-isystem' '/usr/include/libpng16' '-isystem' '/usr/include/harfbuzz' '-D' 'emacs' '-I' '.' '-I' '.' '-I' '../lib' '-I' '../lib' '-internal-isystem' '/usr/local/include' '-internal-isystem' '/usr/lib/clang/10.0.1/include' '-internal-externc-isystem' '/include' '-internal-externc-isystem' '/usr/include' '-O2' '-Wwrite-strings' '-Wno-missing-field-initializers' '-Wno-override-init' '-Wno-sign-compare' '-Wno-type-limits' '-Wno-unused-parameter' '-Wno-format-nonliteral' '-fconst-strings' '-fdebug-compilation-dir' '/workspace/xulun/github/lang/emacs/src' '-ferror-limit' '19' '-fmessage-length' '0' '-stack-protector' '2' '-fgnuc-version=4.2.1' '-fobjc-runtime=gcc' '-fno-common' '-fdiagnostics-show-option' '-vectorize-loops' '-vectorize-slp' '-analyzer-display-progress' '-analyzer-output=html' '-faddrsig' '-o' '/tmp/scan-build-2020-10-29-033331-24628-1' '-x' 'c' 'terminfo.c')
```

主要差异的部分我们择出来：
```
'/usr/bin/clang-10' '-cc1' '-triple' 'x86_64-pc-linux-gnu' '-analyze' '-disable-free' '-disable-llvm-verifier' '-discard-value-names' '-main-file-name' 'terminfo.c' '-analyzer-store=region' '-analyzer-opt-analyze-nested-blocks' '-analyzer-checker=core' '-analyzer-checker=apiModeling' '-analyzer-checker=unix' '-analyzer-checker=deadcode' '-analyzer-checker=security.insecureAPI.UncheckedReturn' '-analyzer-checker=security.insecureAPI.getpw' '-analyzer-checker=security.insecureAPI.gets' '-analyzer-checker=security.insecureAPI.mktemp' '-analyzer-checker=security.insecureAPI.mkstemp' '-analyzer-checker=security.insecureAPI.vfork' '-analyzer-checker=nullability.NullPassedToNonnull' '-analyzer-checker=nullability.NullReturnedFromNonnull' '-analyzer-output' 'plist' '-w' '-setup-static-analyzer'
```

使用最多的是通过-analyzer-checker来指定checker，先按下不表。

然后后面就是对于语法和路径的针对每个函数的分析过程:

```
ANALYZE (Syntax): ./lisp.h will_dump_p
ANALYZE (Syntax): ./lisp.h will_bootstrap_p
ANALYZE (Syntax): ./lisp.h will_dump_with_pdumper_p
ANALYZE (Syntax): ./lisp.h dumped_with_pdumper_p
ANALYZE (Syntax): ./lisp.h will_dump_with_unexec_p
ANALYZE (Syntax): ./lisp.h dumped_with_unexec_p
ANALYZE (Syntax): ./lisp.h definitely_will_not_unexec_p
...
ANALYZE (Syntax): ./lisp.h maybe_gc
ANALYZE (Syntax): terminfo.c tparam
ANALYZE (Path,  Inline_Regular): ./tparam.h tparam
```

每一项的数量跟代码本身相关，比如etags.c中的路径分析就比较多：
```
ANALYZE (Path,  Inline_Regular): etags.c Erlang_functions
ANALYZE (Path,  Inline_Regular): etags.c Prolog_functions
ANALYZE (Path,  Inline_Regular): etags.c HTML_labels
ANALYZE (Path,  Inline_Regular): etags.c Texinfo_nodes
ANALYZE (Path,  Inline_Regular): etags.c TeX_commands
ANALYZE (Path,  Inline_Regular): etags.c Scheme_functions
ANALYZE (Path,  Inline_Regular): etags.c Forth_words
ANALYZE (Path,  Inline_Regular): etags.c PS_functions
ANALYZE (Path,  Inline_Regular): etags.c Lua_functions
ANALYZE (Path,  Inline_Regular): etags.c Lisp_functions
ANALYZE (Path,  Inline_Regular): etags.c Pascal_functions
ANALYZE (Path,  Inline_Regular): etags.c Makefile_targets
ANALYZE (Path,  Inline_Regular): etags.c Cobol_paragraphs
ANALYZE (Path,  Inline_Regular): etags.c PHP_functions
ANALYZE (Path,  Inline_Regular): etags.c Ruby_functions
ANALYZE (Path,  Inline_Regular): etags.c Python_functions
ANALYZE (Path,  Inline_Regular): etags.c Perl_functions
ANALYZE (Path,  Inline_Regular): etags.c Asm_labels
ANALYZE (Path,  Inline_Regular): etags.c Ada_funcs
ANALYZE (Path,  Inline_Regular): etags.c Go_functions
ANALYZE (Path,  Inline_Regular): etags.c Fortran_functions
ANALYZE (Path,  Inline_Regular): etags.c just_read_file
ANALYZE (Path,  Inline_Regular): etags.c Yacc_entries
ANALYZE (Path,  Inline_Regular): etags.c Cstar_entries
ANALYZE (Path,  Inline_Regular): etags.c Cjava_entries
ANALYZE (Path,  Inline_Regular): etags.c Cplusplus_entries
ANALYZE (Path,  Inline_Regular): etags.c plain_C_entries
ANALYZE (Path,  Inline_Regular): etags.c default_C_entries
ANALYZE (Path,  Inline_Regular): etags.c C_entries
ANALYZE (Path,  Inline_Regular): etags.c consider_token
ANALYZE (Path,  Inline_Regular): etags.c write_classname
ANALYZE (Path,  Inline_Regular): etags.c main
ANALYZE (Path,  Inline_Regular): etags.c analyze_regex
ANALYZE (Path,  Inline_Regular): etags.c xnrealloc
ANALYZE (Path,  Inline_Regular): etags.c error
ANALYZE (Path,  Inline_Regular): etags.c xnmalloc
ANALYZE (Path,  Inline_Regular): etags.c fatal
ANALYZE (Path,  Inline_Regular): etags.c print_help
```

对照前面的统计结果，我们就可以比较精确地排查问题。

## checker配置

现在我们回头来看checker. 
我们可以通过clang -cc1 -analyzer-checker-help命令来查看目前支持的checker列表：
```
clang -cc1 -analyzer-checker-help
OVERVIEW: Clang Static Analyzer Checkers List

USAGE: -analyzer-checker <CHECKER or PACKAGE,...>

CHECKERS:
  core.CallAndMessage           Check for logical errors for function calls and Objective-C message expressions (e.g., uninitialized arguments, null function pointers)
  core.DivideZero               Check for division by zero
  core.DynamicTypePropagation   Generate dynamic type information
  core.NonNullParamChecker      Check for null pointers passed as arguments to a function whose arguments are references or marked with the 'nonnull' attribute
  core.NullDereference          Check for dereferences of null pointers
  core.StackAddressEscape       Check that addresses to stack memory do not escape the function
  core.UndefinedBinaryOperatorResult
                                Check for undefined results of binary operators
  core.VLASize                  Check for declarations of VLA of undefined or zero size
  core.uninitialized.ArraySubscript
                                Check for uninitialized values used as array subscripts
  core.uninitialized.Assign     Check for assigning uninitialized values
  core.uninitialized.Branch     Check for uninitialized values used as branch conditions
  core.uninitialized.CapturedBlockVariable
                                Check for blocks that capture uninitialized values
  core.uninitialized.UndefReturn Check for uninitialized values being returned to the caller
  cplusplus.InnerPointer        Check for inner pointers of C++ containers used after re/deallocation
  cplusplus.Move                Find use-after-move bugs in C++
  cplusplus.NewDelete           Check for double-free and use-after-free problems. Traces memory managed by new/delete.
  cplusplus.NewDeleteLeaks      Check for memory leaks. Traces memory managed by new/delete.
  cplusplus.PureVirtualCall     Check pure virtual function calls during construction/destruction
  deadcode.DeadStores           Check for values stored to variables that are never read afterwards
  fuchsia.HandleChecker         A Checker that detect leaks related to Fuchsia handles
  nullability.NullPassedToNonnull
                                Warns when a null pointer is passed to a pointer which has a _Nonnull type.
  nullability.NullReturnedFromNonnull
                                Warns when a null pointer is returned from a function that has _Nonnull return type.
  nullability.NullableDereferenced
                                Warns when a nullable pointer is dereferenced.
  nullability.NullablePassedToNonnull
                                Warns when a nullable pointer is passed to a pointer which has a _Nonnull type.
  nullability.NullableReturnedFromNonnull
                                Warns when a nullable pointer is returned from a function that has _Nonnull return type.
  optin.cplusplus.UninitializedObject
                                Reports uninitialized fields after object construction
  optin.cplusplus.VirtualCall   Check virtual function calls during construction/destruction
  optin.mpi.MPI-Checker         Checks MPI code
  optin.osx.OSObjectCStyleCast  Checker for C-style casts of OSObjects
  optin.osx.cocoa.localizability.EmptyLocalizationContextChecker
                                Check that NSLocalizedString macros include a comment for context
  optin.osx.cocoa.localizability.NonLocalizedStringChecker
                                Warns about uses of non-localized NSStrings passed to UI methods expecting localized NSStrings
  optin.performance.GCDAntipattern
                                Check for performance anti-patterns when using Grand Central Dispatch
  optin.performance.Padding     Check for excessively padded structs.
  optin.portability.UnixAPI     Finds implementation-defined behavior in UNIX/Posix functions
  osx.API                       Check for proper uses of various Apple APIs
  osx.MIG                       Find violations of the Mach Interface Generator calling convention
  osx.NumberObjectConversion    Check for erroneous conversions of objects representing numbers into numbers
  osx.OSObjectRetainCount       Check for leaks and improper reference count management for OSObject
  osx.ObjCProperty              Check for proper uses of Objective-C properties
  osx.SecKeychainAPI            Check for proper uses of Secure Keychain APIs
  osx.cocoa.AtSync              Check for nil pointers used as mutexes for @synchronized
  osx.cocoa.AutoreleaseWrite    Warn about potentially crashing writes to autoreleasing objects from different autoreleasing pools in Objective-C
  osx.cocoa.ClassRelease        Check for sending 'retain', 'release', or 'autorelease' directly to a Class
  osx.cocoa.Dealloc             Warn about Objective-C classes that lack a correct implementation of -dealloc
  osx.cocoa.IncompatibleMethodTypes
                                Warn about Objective-C method signatures with type incompatibilities
  osx.cocoa.Loops               Improved modeling of loops using Cocoa collection types
  osx.cocoa.MissingSuperCall    Warn about Objective-C methods that lack a necessary call to super
  osx.cocoa.NSAutoreleasePool   Warn for suboptimal uses of NSAutoreleasePool in Objective-C GC mode
  osx.cocoa.NSError             Check usage of NSError** parameters
  osx.cocoa.NilArg              Check for prohibited nil arguments to ObjC method calls
  osx.cocoa.NonNilReturnValue   Model the APIs that are guaranteed to return a non-nil value
  osx.cocoa.ObjCGenerics        Check for type errors when using Objective-C generics
  osx.cocoa.RetainCount         Check for leaks and improper reference count management
  osx.cocoa.RunLoopAutoreleaseLeak
                                Check for leaked memory in autorelease pools that will never be drained
  osx.cocoa.SelfInit            Check that 'self' is properly initialized inside an initializer method
  osx.cocoa.SuperDealloc        Warn about improper use of '[super dealloc]' in Objective-C
  osx.cocoa.UnusedIvars         Warn about private ivars that are never used
  osx.cocoa.VariadicMethodTypes Check for passing non-Objective-C types to variadic collection initialization methods that expect only Objective-C types
  osx.coreFoundation.CFError    Check usage of CFErrorRef* parameters
  osx.coreFoundation.CFNumber   Check for proper uses of CFNumber APIs
  osx.coreFoundation.CFRetainRelease
                                Check for null arguments to CFRetain/CFRelease/CFMakeCollectable
  osx.coreFoundation.containers.OutOfBounds
                                Checks for index out-of-bounds when using 'CFArray' API
  osx.coreFoundation.containers.PointerSizedValues
                                Warns if 'CFArray', 'CFDictionary', 'CFSet' are created with non-pointer-size values
  security.FloatLoopCounter     Warn on using a floating point value as a loop counter (CERT: FLP30-C, FLP30-CPP)
  security.insecureAPI.DeprecatedOrUnsafeBufferHandling
                                Warn on uses of unsecure or deprecated buffer manipulating functions
  security.insecureAPI.UncheckedReturn
                                Warn on uses of functions whose return values must be always checked
  security.insecureAPI.bcmp     Warn on uses of the 'bcmp' function
  security.insecureAPI.bcopy    Warn on uses of the 'bcopy' function
  security.insecureAPI.bzero    Warn on uses of the 'bzero' function
  security.insecureAPI.decodeValueOfObjCType
                                Warn on uses of the '-decodeValueOfObjCType:at:' method
  security.insecureAPI.getpw    Warn on uses of the 'getpw' function
  security.insecureAPI.gets     Warn on uses of the 'gets' function
  security.insecureAPI.mkstemp  Warn when 'mkstemp' is passed fewer than 6 X's in the format string
  security.insecureAPI.mktemp   Warn on uses of the 'mktemp' function
  security.insecureAPI.rand     Warn on uses of the 'rand', 'random', and related functions
  security.insecureAPI.strcpy   Warn on uses of the 'strcpy' and 'strcat' functions
  security.insecureAPI.vfork    Warn on uses of the 'vfork' function
  unix.API                      Check calls to various UNIX/Posix functions
  unix.Malloc                   Check for memory leaks, double free, and use-after-free problems. Traces memory managed by malloc()/free().
  unix.MallocSizeof             Check for dubious malloc arguments involving sizeof
  unix.MismatchedDeallocator    Check for mismatched deallocators.
  unix.Vfork                    Check for proper usage of vfork
  unix.cstring.BadSizeArg       Check the size argument passed into C string functions for common erroneous patterns
  unix.cstring.NullArg          Check for null pointers being passed as arguments to C string functions
  valist.CopyToSelf             Check for va_lists which are copied onto itself.
  valist.Uninitialized          Check for usages of uninitialized (or already released) va_lists.
  valist.Unterminated           Check for va_lists which are not released by a va_end call.
```

那么，我们用scan-build默认用了哪些checker呢？
它们是，core的全部：
 + core.CallAndMessage：检查函数调用和ObjectC的消息
 + core.DivideZero：检查除0错
 + core.DynamicTypePropagation：生成动态类型信息
 + core.NonNullParamChecker：检查作为参数传递给函数的空指针      
 + core.NullDereference：检查空指针的解引用
 + core.StackAddressEscape：检查堆栈越界
 + core.UndefinedBinaryOperatorResult：检查二进制运算符的未定义结果
 + core.VLASize：检查未定义的或零大小的VLA的声明                  
 + core.uninitialized.ArraySubscript：检查用作数组下标的未初始化值
 + core.uninitialized.Assign：检查是否分配了未初始化的值     
 + core.uninitialized.Branch：检查是否将未初始化的值用作分支条件     
 + core.uninitialized.CapturedBlockVariable：检查捕获未初始化值的块
 + core.uninitialized.UndefReturn：检查是否有未初始化的值返回给调用者

C++的全部：
 + cplusplus.InnerPointer: 检查重新分配或释放后使用的c++容器的内部指针
 + cplusplus.Move：查找c++中移动后使用的bug                
 + cplusplus.NewDelete：检查双重释放和释放后使用的问题。
 + cplusplus.NewDeleteLeaks: 检查new/delete的内存泄漏
 + cplusplus.PureVirtualCall：在构造/析构期间检查纯虚函数调用     

死代码目前就这一条：

 + deadcode.DeadStores: 检查存储到变量的值是否永远不会读取

空指针和引用的两条：
 + nullability.NullPassedToNonnull 一个空指针被传递给一个具有_Nonnull类型的指针时发出警告。
 + nullability.NullReturnedFromNonnull: 当返回类型为_Nonnull的函数返回空指针时发出警告。

安全中的一部分：

 + security.insecureAPI.UncheckedReturn: 在使用返回值必须始终检查的函数时发出警告
 + security.insecureAPI.getpw: 使用'getpw'函数的警告
 + security.insecureAPI.gets: 警告使用'get '函数
 + security.insecureAPI.mkstemp: 当“mkstemp”在格式字符串中传递的值小于6时发出警告
 + security.insecureAPI.mktemp: 使用'mktemp'函数的警告
 + security.insecureAPI.vfork: 使用“vfork”功能的警告

unix兼容api的全部：
 + unix.API：检查对各种UNIX/Posix函数的调用                      
 + unix.Malloc：检查内存泄漏、双重释放和释放后使用的问题。跟踪由malloc()/free()管理的内存。
 + unix.MallocSizeof: 检查涉及sizeof的可疑malloc参数
 + unix.MismatchedDeallocator:检查不匹配的Deallocator
 + unix.Vfork: 检查是否正确使用vfork
 + unix.cstring.BadSizeArg:检查传递给C字符串函数的size参数是否存在常见的错误模式
 + unix.cstring.NullArg:检查作为参数传递给C字符串函数的空指针

## 循环参数

为了节省资源，CSA在遇到循环的时候，默认执行4次。如果确认代码跟循环有关的话，可以尝试将循环次数加大。
通过 -maxloop可以指定循环次数，比如我们改成10：
```
scan-build -maxloop 10 make
```
