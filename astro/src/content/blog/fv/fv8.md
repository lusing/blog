---
title: "操作系统形式化验证实践教程(1) - 证明第一个定理"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

# 操作系统形式化验证实践教程(8) - 用Haskell做系统建模

到上节为止，我们验证的虽然已经是C语言源代码了，但是跟操作系统的关系还基本没有。
从这一节开始，我们开始进入操作系统的部分。

操作系统涉及到硬件，也涉及到整体的功能的设计。这部分在seL4中是使用Haskell语言实现的。作为本系列教程的配套，我们有《Haskell快餐教程》，请没有haskell语言基础的同学先移步学习第一节，然后我们回来看Haskell代码。另外说一句，我们也有《Standard ML快餐教程》来介绍Standard ML语言的，不知道大家看到了没有。

大家刚学习Haskell，对Haskell还不熟悉。所以我们先从简单的开始，边学语言边验证系统。

## Haskell字长处理

我们看来seL4验证中我们要学习的第一个Haskell代码，字长处理：
```lhs
> module Data.WordLib where
>
> import Data.Word
> import Data.Bits
>
> wordBits :: Int
> wordBits = finiteBitSize (undefined::Word)
>
> wordSize :: Int
> wordSize = wordBits `div` 8
>
> wordSizeCase :: a -> a -> a
> wordSizeCase a b = case wordBits of
>         32 -> a
>         64 -> b
>         _ -> error "Unknown word size"
>
> wordRadix :: Int
> wordRadix = wordSizeCase 5 6
```

首先是引入了对字和位处理的两个包：
```
> import Data.Word
> import Data.Bits
```

wordBits是计算一个Word中包含了多少个比特。通过finiteBitSize函数来计算，函数定义参考可以参见：[https://www.stackage.org/haddock/lts-16.10/base-4.13.0.0/Data-Bits.html#v:finiteBitSize](https://www.stackage.org/haddock/lts-16.10/base-4.13.0.0/Data-Bits.html#v:finiteBitSize)

我们可以在[https://www.stackage.org/](https://www.stackage.org/)
中搜索API的文档：
![haskell search](https://upload-images.jianshu.io/upload_images/1638145-9baec177400ffa00.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以在ghci中去尝试下相关代码：
```hs
Prelude> import Data.Bits
Prelude Data.Bits> wordBits = finiteBitSize (undefined::Word)
Prelude Data.Bits> :set +t
Prelude Data.Bits> wordBits
64
it :: Int
```
可知，我用的是64位系统，所以wordBits为64. 
换算成字节，是8个字节：
```hs
Prelude Data.Bits> wordSize = wordBits `div` 8
wordSize :: Int
Prelude Data.Bits> wordSize
8
it :: Int
```

再看后面的case表达式：
```lhs
> wordSizeCase :: a -> a -> a
> wordSizeCase a b = case wordBits of
>         32 -> a
>         64 -> b
>         _ -> error "Unknown word size"
>
> wordRadix :: Int
> wordRadix = wordSizeCase 5 6
```
32位对应5，64位对应6，所以我这个环境下wordRadix为6. 

## 寄存器

下面我们开始进入硬件建模的第一个元素：寄存器。

我们以ARM和X64为例，看下寄存器的基本结构：
![寄存器.png](https://upload-images.jianshu.io/upload_images/1638145-cf58e3f74718c466.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

寄存器信息都定义于SEL4.Machine.RegisterSet中：

```lhs
\item[The message information register] contains metadata about the contents of an IPC message, such as the length of the message and whether a capability is attached.

> msgInfoRegister :: Register
> msgInfoRegister = Register Arch.msgInfoRegister

\item[Message registers] are used to hold the message being sent by an object invocation.

This list may be empty, though it should contain as many registers as possible. Message words that do not fit in these registers will be transferred in a buffer in user-accessible memory.

> msgRegisters :: [Register]
> msgRegisters = map Register Arch.msgRegisters

\item[The capability register] is used when performing a system call, to specify the location of the invoked capability.

> capRegister :: Register
> capRegister = Register Arch.capRegister

\item[The badge register] is used to return the badge of the capability from which a message was received. This is typically the same as "capRegister".

> badgeRegister :: Register
> badgeRegister = Register Arch.badgeRegister

\item[The frame registers] are the registers that are used by the architecture's function calling convention. They generally include the current instruction and stack pointers, and the argument registers. They appear at the beginning of a "ReadRegisters" or "WriteRegisters" message, and are one of the two subsets of the integer registers that can be copied by "CopyRegisters".

> frameRegisters :: [Register]
> frameRegisters = map Register Arch.frameRegisters

\item[The general-purpose registers] include all registers that are not in "frameRegisters", except any kernel-reserved or constant registers (such as the MIPS "zero", "k0" and "k1" registers). They appear after the frame registers in a "ReadRegisters" or "WriteRegisters" message, and are the second of two subsets of the integer registers that can be copied by "CopyRegisters".

> gpRegisters :: [Register]
> gpRegisters = map Register Arch.gpRegisters

\item[An exception message] is sent by the kernel when a hardware exception is raised by a user-level thread. The message contains registers from the current user-level state, as specified by this list. Two architecture-defined words, specifying the type and cause of the exception, are appended to the message. The reply may contain updated values for these registers.

> exceptionMessage :: [Register]
> exceptionMessage = map Register Arch.exceptionMessage

\item[A syscall message] is sent by the kernel when a user thread performs a system call that is not recognised by seL4. The message contains registers from the current user-level state, as specified by this list. A word containing the system call number is appended to the message. The reply may contain updated values for these registers.

> syscallMessage :: [Register]
> syscallMessage = map Register Arch.syscallMessage

> tlsBaseRegister :: Register
> tlsBaseRegister = Register Arch.tlsBaseRegister

\item[The fault register] holds the instruction which was being executed when the fault occured.

> faultRegister :: Register
> faultRegister = Register Arch.faultRegister

\item[The next instruction register] holds the instruction that will be executed upon resumption.

> nextInstructionRegister :: Register
> nextInstructionRegister = Register Arch.nextInstructionRegister
```

具体的实现定义在架构相关的实现文件中。

ARM架构的所有寄存器为：
```lhs
> data Register =
>     R0 | R1 | R2 | R3 | R4 | R5 | R6 | R7 | R8 | R9 | SL | FP | IP | SP |
>     LR | NextIP | FaultIP | CPSR | TPIDRURW | TPIDRURO
```

下面是ARM对于逻辑寄存器实现的定义：
```lhs
> capRegister = R0
> msgInfoRegister = R1
> msgRegisters = [R2 .. R5]
> badgeRegister = R0
> faultRegister = FaultIP
> nextInstructionRegister = NextIP
> frameRegisters = FaultIP : SP : CPSR : [R0, R1] ++ [R8 .. IP]
> gpRegisters = [R2, R3, R4, R5, R6, R7, LR, TPIDRURW, TPIDRURO]
> exceptionMessage = [FaultIP, SP, CPSR]
> syscallMessage = [R0 .. R7] ++ [FaultIP, SP, LR, CPSR]
> tlsBaseRegister = TPIDRURW
```

而X64的所有寄存器如下：
```lhs
> data Register =
>     RAX | RBX | RCX | RDX | RSI | RDI | RBP |
>     R8 | R9 | R10 | R11 | R12 | R13 | R14 | R15 |
>     FaultIP | -- "FaultIP"
>     FS_BASE | GS_BASE |
>     ErrorRegister | NextIP | CS | FLAGS | RSP | SS
```

映射到逻辑寄存器上：
```lhs
> capRegister = RDI
> msgInfoRegister = RSI
> msgRegisters = [R10, R8, R9, R15]
> badgeRegister = capRegister
> faultRegister = FaultIP
> nextInstructionRegister = NextIP
> frameRegisters = FaultIP : RSP : FLAGS : [RAX .. R15]
> gpRegisters = [FS_BASE, GS_BASE]
> exceptionMessage = [FaultIP, RSP, FLAGS]
> tlsBaseRegister = FS_BASE

> syscallMessage = [RAX .. R15] ++ [FaultIP, RSP, FLAGS]
```

这些只是通用的，每种架构还有自己的特色。

比如ARM支持Hypervisor的话，支持VCPU相关寄存器：
```lhs
> data VCPUReg =
>       VCPURegSCTLR
>     | VCPURegACTLR
>     | VCPURegTTBCR
>     | VCPURegTTBR0
>     | VCPURegTTBR1
>     | VCPURegDACR
>     | VCPURegDFSR
>     | VCPURegIFSR
>     | VCPURegADFSR
>     | VCPURegAIFSR
>     | VCPURegDFAR
>     | VCPURegIFAR
>     | VCPURegPRRR
>     | VCPURegNMRR
>     | VCPURegCIDR
>     | VCPURegTPIDRPRW
>     | VCPURegFPEXC
>     | VCPURegLRsvc
>     | VCPURegSPsvc
>     | VCPURegLRabt
>     | VCPURegSPabt
>     | VCPURegLRund
>     | VCPURegSPund
>     | VCPURegLRirq
>     | VCPURegSPirq
>     | VCPURegLRfiq
>     | VCPURegSPfiq
>     | VCPURegR8fiq
>     | VCPURegR9fiq
>     | VCPURegR10fiq
>     | VCPURegR11fiq
>     | VCPURegR12fiq
>     | VCPURegSPSRsvc
>     | VCPURegSPSRabt
>     | VCPURegSPSRund
>     | VCPURegSPSRirq
>     | VCPURegSPSRfiq
>     | VCPURegCNTV_CTL
>     | VCPURegCNTV_CVALhigh
>     | VCPURegCNTV_CVALlow
>     | VCPURegCNTVOFFhigh
>     | VCPURegCNTVOFFlow
```

对于X64 CPU，咱们有GDT需要处理：
```lhs
> data GDTSlot
>     = GDT_NULL
>     | GDT_CS_0
>     | GDT_DS_0
>     | GDT_TSS
>     | GDT_TSS_Padding
>     | GDT_CS_3
>     | GDT_DS_3
>     | GDT_FS
>     | GDT_GS
>     | GDT_ENTRIES
>     deriving (Eq, Show, Enum, Ord, Ix)

> gdtToSel :: GDTSlot -> Word
> gdtToSel g = (fromIntegral (fromEnum g) `shiftL` 3 ) .|. 3

> gdtToSel_masked :: GDTSlot -> Word
> gdtToSel_masked g = gdtToSel g .|. 3

> selCS3 = gdtToSel_masked GDT_CS_3
> selDS3 = gdtToSel_masked GDT_DS_3
> selFS = gdtToSel_masked GDT_FS
> selGS = gdtToSel_masked GDT_GS
> selCS0 = gdtToSel_masked GDT_CS_0
> selDS0 = gdtToSel GDT_DS_0
```

## 小结

至此，我们的画卷基本上都徐徐展开了，做操作系统的验证，不光需要Isabelle/HOL, Standard ML, 还有Haskell，还需要硬件相关知识。后面我们还会涉及到操作系统软件相关的知识。
