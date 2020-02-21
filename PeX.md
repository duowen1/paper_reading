# 论文介绍
Pex: A Permission Check Analysis Framework for Linux Kernel

——Usenix 2019

文章主要介绍了一种检测Linux内核中的权限检查的方法，作者实现了内核权限静态检测工具Pex，其可以探测内核权限检查的缺失、重复和不一致。工具的输入为Linux内核经过编译器前端的中间代码IR。经过测试，在4.18.5稳定版Linux内核下，共发现36处权限检测错误，其中的14处已经被开发者证实。


# 实验背景
权限检查是操作系统安全的重要组成部分，但是内核设计者很难做到对于每一个特权函数都进行正确的检查。工具在设计时有两个重要的挑战：
## 1. 频繁的间接调用
内核设计者在设计内核时，为了实现模块化的设计通常采用分层，层之间的接口通常抽象为函数指针。由于内核的庞大的代码量，现有的指针分析工具很难起作用。如果采用基于类型的指针分析，其可以适应Linux内核的规模。但是基于类型的间接跳转分析会导致误判。
## 2. 缺少权限检查和特权函数的对应关系
权限检查的要求并没有被文档化，不能确定特权函数需要进行什么样的权限检查。现有的静态特权错误检测工具不能很好地解决这个问题，它们通常会有无法检测特权函数的上层函数、检测开销过大、误报率高等问题。

# 实现原理
PeX实现了一个准确的间接调用分析技巧——KIRIN。KIRIN基于LLVM框架，其设计思想来源如下：
1. Linux内核中95%以上的间接跳转都来自于内核接口
2. 内核接口的类型被保存到两个地方——初始化位置（函数指针定义的地方）和LLVM IR形式的间接调用点（函数指针使用的地方）

基于上述内容，PeX首先收集从内核接口初始化位置间接调用目标，然后在间接调用处解决。KIRIN线性扫描Linux内核代码已得到所有静态分配以及含有函数指针域的结构体变量。对于每个结构体，KIRIN跟踪被分配到函数指针的函数地址，以偏移量作为域的值。对于其余动态初始化的内核接口，KIRIN对其进行了数据流分析。
KIRIN将第一pass保存到一个map中，key为接口类型和偏移量组成的pair，value为调用目标的set。在每一个间接跳转点，KIRIN从LLVM IR中取回内核接口的类型和偏移量，然后在map中查询跳转目标。

# 模型架构
![Tex静态分析架构图](https://s2.ax1x.com/2020/02/13/1LitbV.png "Tex静态分析架构图")

上图展示了PeX的架构图，PeX接受Linux内核的IR以及常规权限检查，分析并报告所有所有检测到的错误。PeX首先利用KIRIN分析所有间接调用，然后绘制增强调用图并仅保留从用户空间可以到达的调用点。基于调用图，Pex会产生过程间控制流图。从小部分用户提供的权限检查开始，PeX会自动检测wrappers（封装函数，wrappers调用了特权函数）

PeX利用KIRIN的结果产生调用图，将其分为两个部分，首先时用户空间可以调用的函数（含有前缀Sys_），PeX便利调用图，标记所有已经访问的函数，把他们标记为用户空间可以调用的函数。对于内核初始化函数，PeX将遍历调用图通过start_kernel和每个__init作为前缀的函数。

PeX利用了过程间的dominator分析：在不访问dominator的情况下，没有路径可以直接执行call指令。文章给出了寻找特权函数的算法：

```
INPUT:
    pcfuncs - all permission checking functions 
OUTPUT:
    pvfuncs - privileged functions 
procedure PRIVILEGED FUNCTION DETECTION 
    for f ← pcfuncs do 
        for u←User(f) do 
            CallInst←CallInstDominatedBy(u)
            callee←getCallee(CallInst) 
            pvfuncs.insert(callee) 
        endfor 
    endfor 
    return pvfuncs 
end procedure
```

# 实验效果

![输入情况](https://s2.ax1x.com/2020/02/13/1LbPT1.png "输入情况")

![实验结果](https://s2.ax1x.com/2020/02/13/1LbspT.png "实验结果")

作者将KIRIN与其他类型的间接跳转分析工具进行了对比实验，通过结果可知，在开销和准确率上有显著的提升。