[cpu之DPL,RPL,CPL 之间的联系和区别](https://www.cnblogs.com/buzhunbukaixin/articles/4662004.html)

官方文档[Intel’s software developer manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)，“Privilege Levels” 和 “Checking Caller Access Privileges”这两节。
目前在网上查找的中文资料中，以这篇最为详细，故在此做一步简单的总结以及加入一些个人的理解。有兴趣的同学可以去阅读一下官方文档。
在X86中，存在着3种特权级的描述，分别是CPL，RPL和DPL。


# 1. CPL，RPL，DPL概念
《x86汇编语言从实模式到保护模式》第十一章和第十四章
* CPL(CS.RPL)是CS寄存器里bit 0和bit 1 位组合所得的值.在某一时刻就只有这个值唯一的代表程序的CPL。
* RPL是段选择子里面的bit 0和bit 1位组合所得的值。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image40.png?raw=true" width="70%">

* DPL是段描述符中的特权级, 它的本意是用来代表它所描述的段的特权级。

注意：CS寄存器本身也是存放着段选择子的，但是其bit0和bit1表示的不再是RPL而是CPL。

# 2. 对数据段和堆栈段访问时的特权级控制
处理器对数据段的访问是通过DS寄存器来进行的。DS寄存器中的段选择子的前12个字节表示描述符索引，指向GDT中的一个段描述符，段描述符的DPL表示该地址段的特权级。DS寄存器的后2bit表示其RPL。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image41.png?raw=true" width="70%">

只有当`CPL <= DPL 且 RPL <= DPL`的情况下，才能访问相应的数据段(0表示特权级最高，3表示特权级最低)。即此时CS寄存器的最后2个bit的值以及DS寄存器最后两个bit的值均要小于DS寄存器指向的段描述符的DPL。
当我们通过指令（例如mov，pop，lds，les，lfs，lgs或者lss指令）要将一个段选择子加载到DS寄存器的时候，我们需要对CPL，该段选择子的RPL和段选择子指向的段描述符的DPL进行比较，如下图所示。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image42.png?raw=true" width="70%">

# 3. 对代码段访问的特权级控制
在这里主要解释JMP和CALL指令引起的控制转移。
JMP和CALL指令可以通过4种方式进行控制转移：
* 目标操作数包含目标代码段的段选择子
* 目标操作数指向一个调用门
* 目标操作数指向一个TSS
* 目标操作数指向一个任务门
我们暂时只讨论前面两种引起的控制转移。

## 3.1 JMP和CALL到段选择子
JMP, CALL, 和RET的近跳转是跳转到当前的代码段，所以也不会进行特权级检测。
JMP, CALL, 和RET的远跳转因为可以跳转到其它的代码段，因此需要进行特权级检测，主要是检测这4个字段CPL，RPL，DPL以及段描述符的conforming (C) flag。
* CPL：CPL指的是当前调用CALL指令的这个寄存器的特权级。
* DPL：目标代码段描述符的DPL
* RPL：目标代码段选择子的RPL字段
* conforming (C) flag：目标段描述符的conforming (C) flag标记

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image43.png?raw=true" width="70%">


### 3.1.1 访问Nonconforming代码段
1. 特权级检测规则
* 当前CPL必须与目标代码段选择子指向的段描述符的DPL相等，`CPL == DPL`。
* 目标段选择子的RPL需要小于等于当前的CPL。例如在下图中，C1的RPL可以被设置为0，1，2，但是不能被设置为3。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image44.png?raw=true" width="70%">

2. CPL的变化
当nonconforming代码段的段选择子被加载到CS寄存器的时候，CPL不会被改变，即使目标段选择子的RPL和当前的CPL不同。

### 3.1.2 访问conforming代码段
1. 特权级检测规则
* 当前CPL必须大于等于目标代码段选自指向的段描述符的DPL，`CPL >= DPL`
* 不检测目标段选择子的RPL
在图5-7种，代码段A和B都可以访问D（不论是使用D1或者D2段选择子）。
**For conforming code segments, the DPL represents the numerically lowest privilege level that a calling procedure may be at to successfully make a call to the code segment.**

2. CPL变化
当程序控制转移到conforming代码段的时候，CPL不会发生改变，即使目标代码段的DPL小于当前CPL。这是唯一一种当前代码段的DPL和CPL不相等的情况。由于CPL不会发生改变，因此也不会发生栈切换。

3. 使用场景
Conforming代码段一般用于代码模块，例如数值运算和异常处理程序，它们为应用提供支撑，但是不需要访问受保护的系统模块


## 3.2 门描述符（Gate Descriptor）
对于nonconforming代码段，若想要将控制从低特权转移到高特权级，需要通过门描述符（Gate Descriptor）进行。
总共有4种门描述符：
* 调用门（call gate）
* 中断门（interrupt gate）
* 陷阱门（trap gate）
* 任务门（task gate）

任务门用于task switch，中断门和陷阱门用于对中断异常的处理，这些暂时不讨论。

## 3.3 调用门（Call Gate）
调用门可以完成程序在不同特权级别之间的控制转移。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image45.png?raw=true" width="70%">

P字段表示这个调用门是否有效。如果P为0，会触发#NP异常。因此这个字段可以用来作为一个debug位，统计这个程序被调用的次数。

## 3.4 通过调用门（Call Gate）访问代码段
1.  调用门流程
CALL和JMP指令操作数的段选择子指向调用门描述符，再从门描述符中取得偏移地址以及目标代码的段选择子，这个段选择子指向目标代码段描述符，之后就可以取得目标代码段的段基地址。目标代码段描述符的段及地址加上调用门描述符中的偏移地址，就是目标调用程序的入口了。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image46.png?raw=true" width="70%">


2. 特权检测
访问调用门时，会检测4个特权级。
* CPL：当前程序的CPL
* RPL：指向调用门的段选择子的RPL
* DPL：调用门的DPL字段
* DPL：目标代码段的段描述符的DPL字段
* 目标代码段的段描述符的C flag (conforming)

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image47.png?raw=true" width="70%">


3. 特权检测规则
JMP和CALL指令的特权检测规则是不同的，如下图所示：

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image48.png?raw=true" width="70%">

4. 特权检测示例

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image49.png?raw=true" width="70%">

**首先，当前`CPL`必须要小于等于调用门的`DPL`**。例如，GATE A的DPL是3，所以Code Segment A、B、C都可以访问它。而GATE B的DPL是2，只有Code Segment B和Code Segment C可以访问。
**其次，门选择子的`RPL`也必须要小于等于调用门的`DPL`**。例如，Code Segment C使用Gate Select B1和B2能够访问Gate B，但是不能使用Gate Selector B3来访问Gate B。
**最后，如果CPL和RPL的特权级检测成功之后，处理器还会对目标代码段的DPL与CPL进行特权级检测**。此时JMP和CALL指令对此的处理是不同的：对于nonconforming code segment，调用CALL指令时，如果CPL >= DPL， 就会发生控制转移；而调用JMP指令时，必须CPL == DPL才会发生控制转移。对于conforming code segment，调用CALL指令和JMP指令时，如果CPL >= DPL就会发生控制转移。


5. CPL转化
* 如果CALL指定调用nonconforming destination code segment时，其DPL < CPL，那么CPL的值会变为目的代码段描述符的DPL，然后发生stack switch。
* 如果CALL和JMP指令调用了conforming destination code segment时，CPL的值不会发生改变。





## 3.4 陷阱门和中断门的特权保护机制
参考官方文档 6.12.1.2 Protection of Exception- and Interrupt-Handler Procedures。
* 由于中断和异常向量没有RPL，因此当处理器进入中断或者异常处理程序，或者通过任务门发起任务切换时，不会检测RPL。
* 只有当程序调用`INT n`, `INT 3`, `INTO`指令的时候，处理器才会检测中断门和陷阱门的DPL。此时CPL必须小于等于DPL。
* 对于硬件中断和处理器检测到异常情况，中断门和陷阱门的DPL将会被忽略。

**处理器不允许将高特权级的控制转移到低特权级**

中断发生的时候，CPL怎么转化呢？当中断和异常发生时，任务可能正在特权级别位0的全局空间中执行，也可能正在特权级别位3的局部空间中执行。
**The privilege-level protection for exception- and interrupt-handler procedures is similar to that used for ordinary procedure calls when called through a call gate**。按这段英文所述，CPL的切换应该和CALL指令的调用类似。



