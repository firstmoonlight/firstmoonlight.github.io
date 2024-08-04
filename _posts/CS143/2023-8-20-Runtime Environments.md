---
layout: post
title: Lecture11, about Runtime Environments
tags: [CS143]
---

本文主要是对于CS143的Lecture11的一个简要的笔记, 摘录自己的感想.

# Memory Layout
下面的内存结构描述的是广义上的机器的内存。
但是对于Coolc的generate code阶段，其最上面是High Address，而下面是Low Address。Stack是从High Address向Low Address增长的。


<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_2/Image1.png?raw=true" width="70%">

# Assumptions about Execution
下面是关于执行过程中的两个假设，即coolc语言在runtime的时候所遵循的。
1. 执行是顺序的；控制流在程序中按照定义好的顺序从一个点移动到另一个点。
> Execution is sequential; control moves from one point in a program to another in a well-defined order

但是对于一些支持并发的语言来说，这条假设是不成立的。

2. 当一个过程被调用时，控制最终会返回到调用该过程的点之后的那个位置
> When a procedure is called, control eventually returns to the point immediately after the call

但是对于一些支持Exception或者callcc的语言来说，这条假设是不成立的。

# Activations
## 什么是activation
对于过程P的调用就是P的一个activation。例如P是一个函数的话，那么我们调用P()，这就是一个activation。
> An invocation of procedure P is an activation of P

## P的activation
* 执行 P 中所有的指令
* 以及 P 调用的其它的procedure

# Lifetimes of Variables
• 变量 x 的生命周期是 x 被定义并被执行的那个过程
• 注意：
    – 生命周期是一个动态（run time）的概念 
    – 作用域是一个静态（compile time）概念

# Activation Trees
• Assumption (2) requires that when P calls Q, then Q returns before P does 
• Lifetimes of procedure activations are properly nested 
• Activation lifetimes can be depicted as a tree

这部分的意思是说，由于Assumption（2）, 我们可以将整个程序的Activation用一棵树来表示。</br> 

例如对于下面的code，main函数中调用了g和f，而f有调用了g，因此其activation tree，可以用下面的来表示。
```
Class Main { 
    g() : Int { 1 }; 
    f(): Int { g() }; 
    main(): Int {{ g(); f(); }}; 
}
```

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_2/Image2.png?raw=true" width="70%">


# Activation Records
• The information needed to manage one procedure activation is called an activation record (AR) or frame 
• If procedure F calls G, then G’s activation record contains a mix of info about F and G.

## What is in G's AR when F calls G?
• F is “suspended” until G completes, at which point F resumes. G’s AR contains information needed to resume execution of F. 
• G’s AR may also contain: 
    – G’s return value (needed by F) 
    – Actual parameters to G (supplied by F) 
    – Space for G’s local variables

## The Contents of a Typical AR for G
• Space for G’s return value • Actual parameters 
• Pointer to the previous activation record 
    – The control link; points to AR of caller of G 
• Machine status prior to calling G 
    – Contents of registers & program counter 
    – Local variables 
• Other temporary values


这一部分解释activation record。即如果一个procedure被调用，那么其执行的时候需要一些信息，例如函数调用的时候，需要入参，返回值，当函数结束的时候需要从哪个地方继续开始执行等信息，这部分信息被称为activation record或者frame。</br>
一般存储的信息有4个，result argument， control link， return， address。如图所示，f的frame存在着4个信息，result是第一个返回值，而3是参数，3下面的那个是control link，指向调用者的frame的最开始的地方，而`(*)`和`(**)`则表示下一个需要执行的地址。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_2/Image3.png?raw=true" width="70%">

## Notes
• main has no argument or local variables and its result is never used; its AR is uninteresting 
• `(*)` and `(**)` are return addresses of the invocations of f 
    – The return address is where execution resumes after a procedure call finishes 
• This is only one of many possible AR designs 
    – Would also work for C, Pascal, FORTRAN, etc.


# Globals
• All references to a global variable point to the same object 
    – Can’t store a global in an activation record 
• Globals are assigned a fixed address once 
    – Variables with fixed address are “statically allocated” 
• Depending on the language, there may be other statically allocated values

在讨论global variable的时候，global variable被指定到一个固定的地址。

# Heap Storage
• A value that outlives the procedure that creates it cannot be kept in the AR 
```
method foo() { new Bar } 
The Bar value must survive deallocation of foo’s AR
```
 • Languages with dynamically allocated data use a heap to store dynamic data
 
 使用堆来存储那些在procedure中创建的value。
 
 ## Notes
• The code area contains object code 
    – For most languages, fixed size and read only 
• The static area contains data (not code) with fixed addresses (e.g., global data) 
    – Fixed size, may be readable or writable 
• The stack contains an AR for each currently active procedure 
    – Each AR usually fixed size, contains locals 
• Heap contains all other data 
    – In C, heap is managed by malloc and free

# Alignment
* Most machines are 32 or 64 bit 
    * 8 bits in a byte – 4 bytes in a word 
    * Machines are either byte or word addressable 
* Data is word aligned if it begins at a word boundary 
* Most machines have some alignment restrictions 
    * Or performance penalties for poor alignment

alignment在硬件部分很重要，大部分的硬件是32比特或者64比特。大多数机器都有一些对齐限制。对齐限制是指在内存中存储数据时，数据必须放置在特定的内存地址上，以确保访问效率和正确性。对齐限制的原因通常是为了提高内存访问效率，因为未对齐的内存访问可能会导致额外的内存访问周期，甚至可能导致处理器异常或错误。


# Stack Machines
* A simple evaluation model 
* No variables or registers 
* A stack of values for intermediate results 
* Each instruction:
    * Takes its operands from the top of the stack 
    * Removes those operands from the stack 
    * Computes the required operation on them 
    * Pushes the result on the stack


## Example of Stack Machine Operation
Stack Machine的具体例子如下所示，以`7 + 5`举例，加法有两个操作数，所以我们先将`7`和`5`加入到Stack中，然后我们依次pop这些数字进行操作，再将计算的结果再加入到Stack中。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_2/Image4.png?raw=true" width="70%">

## Why Use a Stack Machine?
* Each operation takes operands from the same place and puts results in the same place 
* This means a uniform compilation scheme 
* And therefore a simpler compiler


* Location of the operands is implicit 
    * Always on the top of the stack 
* No need to specify operands explicitly 
* No need to specify the location of the result 
* Instruction “add” as opposed to “add r1, r2” </br>
⇒ Smaller encoding of instructions </br>
⇒ More compact programs 
* This is one reason why Java Bytecodes and WebAssembly use a stack evaluation model

## 栈机器（Stack Machine）和寄存器机器（Register Machine）的比较
栈机器（Stack Machine）和寄存器机器（Register Machine）是两种不同的计算机体系结构。它们在指令集设计和操作数存储方面有显著差异。
### 栈机器相比于寄存器机器的一些优点：
1. 指令集简洁：
* 栈机器使用后进先出（LIFO）的栈结构来存储操作数和中间结果，指令集相对简单。例如，PUSH 和 POP 指令用于操作栈，指令不需要显式地指定操作数的位置。
* 这种简洁性使得指令解码更为简单，指令的字节码表示也更加紧凑。
2. 编译器简单：
* 栈机器的编译器不需要进行复杂的寄存器分配和管理，因为所有的操作数都通过栈来传递。
* 这可以简化编译器的设计和实现，减少开发难度和时间。
3. 代码密度高：
* 由于栈机器指令不需要指定操作数位置，指令通常较短。这可以提高代码密度，减少程序的存储空间需求。
* 这在嵌入式系统或资源受限的环境中尤为重要。
4. 上下文切换开销低：
* 栈机器在进行上下文切换时，只需保存栈指针和少量状态信息。
* 相比之下，寄存器机器需要保存和恢复大量的寄存器内容，上下文切换的开销较大。5. 硬件实现简单：
* 栈机器的硬件实现通常比寄存器机器简单，因为不需要复杂的寄存器文件和相关的控制逻辑。
* 这可以降低硬件设计和制造的成本，提高系统的可靠性。

### 栈机器的局限性
尽管栈机器有这些优点，但它们也有一些局限性，比如：
* 性能较低：由于频繁的栈操作，栈机器在性能上往往不如寄存器机器。
* 可读性差：栈机器的指令和操作数隐式依赖于栈的位置，程序的可读性和调试性较差。


## Optimizing the Stack Machine
我们将Stack和Register结合来优化我们的Stack Machie，例如我们可以加入一个名为`accumulator`的register来保存计算的结果。
* The add instruction does 3 memory operations 
    * Two reads and one write to the stack 
    * The top of the stack is frequently accessed 
* Idea: keep the top of the stack in a register (called accumulator) 
    * Register accesses are faster 
* The “add” instruction is now 
```
acc ← acc + top_of_stack 
– Only one memory operation!
```
### Stack Machine with Accumulator
Invariants 
* The result of an expression is in the accumulator 
* For `op(e1,…,en)` push the accumulator on the stack after computing each of `e1,…,en-1`
    * After the operation pops n-1 values 
* Expression evaluation preserves the stack


### Stack Machine with Accumulator. Example

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_2/Image5.png?raw=true" width="70%">


### 堆栈机器（Stack Machine）与累加器（Accumulator）架构的组合的优势
1. 简化的指令集：
* 由于操作数隐含在堆栈顶或累加器中，指令集可以更加简洁，减少了操作码的复杂性。
2. 更少的寄存器管理：
* 堆栈机器主要使用堆栈来存储操作数，减少了对寄存器的需求和管理。这简化了编译器的寄存器分配任务。
3. 小巧的代码尺寸：
* 堆栈机器的指令通常更短，因为它们不需要显式地指定操作数位置。累加器架构进一步简化了常见操作，使代码更加紧凑。
4. 简化的表达式求值：
* 堆栈机器特别适合表达式求值，因为它们可以自然地处理嵌套的子表达式。结合累加器，可以有效地处理简单的数学运算。
5. 减少指令解码复杂性：
* 指令集中大部分操作数都隐含在堆栈顶或累加器中，指令解码变得更为简单，硬件实现相对容易。
6. 良好的编译器支持：
* 简单的架构使得为堆栈机器生成代码的编译器相对容易编写和优化，尤其是涉及到表达式和局部变量的处理。
7. 高效的递归函数调用：
* 堆栈结构天然适合处理递归函数调用和返回，减少了处理调用帧的复杂性。
8. 高代码密度：
* 堆栈机器结合累加器架构可以实现较高的代码密度，特别适合内存受限的嵌入式系统和小型设备。

### 总结
堆栈机器与累加器架构的结合提供了简洁、高效的指令集和硬件实现，特别适用于表达式求值、递归函数和内存受限的环境。这种架构在设计和实现上也更为简单，易于理解和维护。
