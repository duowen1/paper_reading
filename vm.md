# 论文介绍
## 论文题目
Protecting Cloud Virtual Machines from Commodity Hypervisor and Host Operating System Exploits

——USENIX 2019
## 摘要
本文实现了HypSec，利用微内核减少可信计算基的规模以保证虚拟机的机密性和完整性。HypSec将虚拟机监视器分为了两个部分，分别为不受信的主机——主要在不访问虚拟机内的数据时实现大部分复杂的虚拟机功能；以及一个受信的核提供对虚拟机数据的访问控制以及实现对CPU和内存的模拟。

# 威胁模型
## 假设
1. 虚拟机采用端对端加密信道保护IO数据；
2. 硬件虚拟化支持IOMMU；
3. 硬件支持TEE；
4. 硬件环境是可信的；
5. 无法暴力破解加密算法，通信协议的设计没有缺陷；
6. 系统初始化时为良性的。
## 威胁
攻击者拥有远程访问hypervisor和虚拟机的权限（包括管理员权限），攻击者的目标是破坏虚拟机数据的私密性和完整性，包括虚拟机启动镜像、虚拟机内存数据、虚拟机内存缓冲区、硬盘中的数据、CPU寄存器的数据。攻击者可以利用漏洞攻击hostvisor和虚拟机。攻击者可以通过DMA攻击虚拟机内存。

# 设计
![HypSec架构图](https://s2.ax1x.com/2020/02/20/3ePN6g.png "HypSec架构图")

虚拟机监视器被划分为相对独立的两个部分，分别为corevisor和hostvisor，corevisor属于TCB的一部分，拥有更高的权限。如上图所示，HypSec的提供了多个API接口，VM CREATE, VM BOOT, VM ENTER, IOMMU OPS, GET VM STATE。

## 启动与初始化
HypSec保证可信的corevisor代码启动并且bootsrapping代码本身是安全的。Hostvisor和corevisor被链接成二进制代码并且经过签名，经过TEE中的秘钥验证。为了保证boostrapping代码的安全，HypeSec利用hostvisor的bootstrapping代码安装corevisor，这样做基于hostviosr首先是良性的，而且此时网络和串行输入设备服务还不能利用。在此之后corevisor将完全控制硬件。
对于虚拟机启动，hostvisor调用VM CREATE请求corevisor在其内存中分配VM state，包括NPT和VCPU状态。然后调用VM BOOT请求corevisor验证虚拟机镜像。如果验证成功，调用VM ENTER执行VM。

## CPU
HypSec保护VM CPU状态，corevisor处理VM的所有trap、指令模拟和上下文转换。VCPU调度由hostvisor完成。Corevisor处理CPU的执行流是hostvisor或者虚拟机。虚拟机退出时，corevisor保存VM执行上下文（从CPU硬件保存到VCPU状态），然后将hostvisor的状态恢复到CPU中。Hostvisor处理VCPU的调度，这不需要VM CPU的状态。但是虚拟机有时候需要执行请求分享值的指令，参数将会保存到通用寄存器中。这样corevisor就可以判断是否参数可否被hostvisor访问。

## 内存
![内存虚拟化](https://s2.ax1x.com/2020/02/20/3m1G7t.png  "Hypec内存虚拟化")
### 内存保护
对于hostvisor，地址是hVA(host虚拟地址)，通过页表HostPT转化为vhPA(虚拟物理地址)，然后在corevisor中的hNPT转化为hPA。Corevisor采用平坦化的寻址，vhPA实际与hPA相同。这也就导致hostvisor和vm的地址空间相互隔离，hostvisor想要访问vm一定会陷入corevisor。
### 内存分配
内存分配主要由hostvisor完成，hostvisor为每个VM维护一个vNPT，corevisor维护一个sNPT，sNPT是vNPT的影子列表。Corevisor控制NPT基址寄存器指向hNPT或者sNPT。
如图所示，内存虚拟化策略如下：
1. 当虚拟机想要寻址发生缺页中断时，会陷入corevisor。
2. 如果corevisor发现gPA是可利用的虚拟机内存地址，NPT基址寄存器会指向hNPT。
3. 控制流转到hostvisor，为gPA分配一个物理页面。
4. Hostvisor分配一个虚拟物理页面和vhPA，然后更新vNPT，会陷入到corevisor
5. Corevisor验证vhPA没有被占用，然后更新sNPT。修改NPT基址寄存器。
6. 控制流回到虚拟机，VM有权限访问新分配的内存。
### 内存回收
HypSec引入了“气球”机制实现hostviosr的回收，在虚拟机中会安装一个“气球”的半虚拟化设备，当主机的内存低是，hostvisor会让“气球膨胀”。虚拟机中的OS将因此开始回收页面，“气球”驱动通知corevisor将要回收的页面，corevisor在sNPT取消映射，然后清除内存中的内容，并把页面分配给hostvisor，最后“气球放气”。
## I/O
对IO的保护将由虚拟机完成，虚拟机采用端对端的IO加密信道。

# 实现
![](https://s2.ax1x.com/2020/02/21/3n8MHs.png "KVM/ARM")
作者实现在ARM架构下Linux平台上，基于开源的KVM，如图所示，Corevisor运行在EL2下，hostvisor运行在EL1下，虚拟机VM运行在EL0下，三者相互隔离。


# 安全分析
## 1.HypSec的corevisor全生命期受信
1. HypeSec利用硬件安全启动经过签名和验证的HypSec代码，防止在启动时的攻击。
2. Hostvisor在网络和串口输入未启动时安全安装corevisor，防止安装阶段攻击。
3. Corevisor有独立地地址空间，并且完全控制硬件和页表，通过隔离防止攻击。

## 2.HypSec保证只有受信的虚拟机镜像启动
Corevisor验证虚拟机镜像的签名，公钥和签名储存在TEE中，所以被篡改的hostvisor不能替换虚拟机

## 3.HypSec隔离虚拟机之间以及虚拟机和hostvisor的地址空间
Corevisor记录物理页表的使用者、利用嵌套页表硬件保证物理空间隔离。Corevisor控制IOMMU以及它的页表，保证被篡改的hostviosr不能通过DMA方式访问corevisor以及VM中的内存。Corevisor管理虚拟机的影子页表，MMU必须经过影子页表才能在虚拟机中寻址。

## 4.HypSec保护虚拟机的CPU寄存器
只有Corevisor才能访问虚拟机的寄存器，hostvisor不能再未经授权的情况下访问VM寄存器。攻击者不能通过劫持控制流的方式因为寄存器受corevisor保护。

## 5.HypSec保护VMIO的私密性
HypeSec保护IO，通过将加密秘钥保存到VM的CPU寄存器或者内存中。

## 6.HypSec保护VMIO的完整性
如果在永久修改虚拟机IO数据之前可以验证IO数据，则可以保证VMIO的完整性。

## 7.HypSec保护所有VM数据的私密性
基于1,3,4，远程攻击者不能篡改corevisor，对任何虚拟机以及hostvisor的攻击不能获取到其他虚拟机的数据。

## 8.HypSec保护所有VM数据的完整性
基于1,3,4，如果在永久修改虚拟机IO数据之前可以验证IO数据，则可以保证VM数据的完整性。

## 如果虚拟机监视器是良性的而且负责处理IO，HypSec可以保证所有虚拟机数据的私密性和完整性
如果hostvisor和corevisor都是良性的，根据3,4可以保证所有虚拟机数据的私密性和完整性。

# 实验结果
作者针对多个角度进行了对比试验，包括运行时代价（包括基线和负载）、可信计算基础的简化、针对攻击的防御。具体结果可详见论文，在以后的试验中可以参考作者的角度。