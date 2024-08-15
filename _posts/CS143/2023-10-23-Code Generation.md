---
layout: post
title: Lecture12, about Code Generation
tags: [CS143]
---

这节课主要描述了俩个主题，一是code generate的简介，介绍了MIPS汇编语言，并自定义了一种简单的编程语言，通过这种简单语言，我们介绍了code generation的stack machine的实现方式。二是Cool语言的Object layout。

# Lecture Outline
* Topic 1: Basic Code Generation 
    *  The MIPS assembly language 
    *  A simple source language 
    *  Stack-machine implementation of the simple language 
* Topic 2: Code Generation for Objects

# From Stack Machines to MIPS
* The compiler generates code for a stack machine with accumulator 
* We want to run the resulting code on the MIPS processor (or simulator) 
* We simulate stack machine instructions using MIPS instructions and registers

我们将会利用Stack Machine来生成MIPS汇编语言，我们采用优化版本的Stack Machine，即`Stack Machine + accumulator`。


## Simulating a Stack Machine…
* The accumulator is kept in MIPS register $a0 
* The stack is kept in memory 
    * The stack grows towards lower addresses 
    * Standard convention on the MIPS architecture 
* The address of the next location on the stack is kept in MIPS register $sp 
    * The top of the stack is at address $sp + 4

### 以下是对上述的解释：
* 累加器保存在 MIPS 寄存器 $a0 中：
    * 在这个堆栈机器架构中，累加器的值被存储在 MIPS 寄存器 $a0 中。累加器是一个临时存储单元，用于保存计算过程中间的结果。
* 堆栈保存在内存中：
    * 堆栈数据结构用于存储临时变量、返回地址等。堆栈中的数据存储在内存中，而不是在寄存器中。
* 堆栈向下增长：
    * 在 MIPS 架构中，堆栈的增长方向是从高地址向低地址增长。这意味着每次将新数据推入堆栈时，堆栈指针（$sp）的值会减小。
* 堆栈指针地址保存在 MIPS 寄存器 $sp 中：
    * MIPS 寄存器 $sp 保存了堆栈的当前地址，指向堆栈的顶部位置。
* 堆栈顶的位置在 $sp + 4：
    * 堆栈的顶部实际在 $sp + 4 位置。这是因为堆栈指针 $sp 指向的是下一个可用的内存位置，而堆栈顶是当前存储数据的最后一个位置。
    
### 示例
假设堆栈当前的堆栈指针 $sp 指向 0x1000：
1. 初始状态：
* $sp = 0x1000
* 堆栈顶部在 $sp + 4 = 0x1004
2. 推入一个值：
* 假设我们推入一个整数值 42。
* 首先，$sp 减少 4，使其指向 0x0FFC。
* 然后，值 42 被存储在内存地址 0x0FFC。
3. 堆栈状态：
* $sp = 0x0FFC
* 堆栈顶部在 $sp + 4 = 0x1000

### 总结
在这种堆栈机器架构中，累加器存储在寄存器 $a0 中，而堆栈数据存储在内存中，堆栈通过寄存器 $sp 进行管理，并且堆栈顶在 $sp + 4 位置。通过这种方式，可以有效地管理和访问堆栈数据。

## MIPS Assembly
MIPS architecture</br> 
* Prototypical Reduced Instruction Set Computer (RISC) architecture 
* Arithmetic operations use registers for operands and results 
* Must use load and store instructions to use operands and results in memory
* 32 general purpose registers (32 bits each) 
* We will use $sp, $a0 and $t1 (a temporary register)

### A Sample of MIPS Instructions 
1. lw reg1 offset(reg2) 
    * Load 32-bit word from address reg2 + offset into reg1 
2. add reg1 reg2 reg3 
    * reg1 ← reg2 + reg3 
3. sw reg1 offset(reg2) 
    * Store 32-bit word in reg1 at address reg2 + offset 
4. addiu reg1 reg2 imm 
    * reg1 ← reg2 + imm 
    * u” means overflow is not checked 
5. li reg imm 
    * reg ← imm

### MIPS Assembly. Example.
* The stack-machine code for 7 + 5 in MIPS:
```
// stack-machine code
acc ← 7 
push acc 
acc ← 5 
acc ← acc + top_of_stack
pop

// MIPS code
li $a0, 7               # 将 7 加载到累加器
sw $a0, 0($sp)     # 将累加器的值推入堆栈
addiu $sp, $sp, -4 # 减少堆栈指针，堆栈向下增长

li $a0, 5          # 将 5 加载到累加器
lw $t1, 4($sp)     # 加载堆栈顶的值到临时寄存器 $t1
add $a0, $a0, $t1  # 将累加器和堆栈顶的值相加
addiu $sp, $sp, 4  # 增加堆栈指针，弹出堆栈顶部的值

```

###  Stack Machine code 以及 MIPS code 的解释
#### 堆栈机器代码
1. acc ← 7
* 将累加器 acc 设置为 7。
2. push acc
* 将累加器 acc 的值推入堆栈。
3. acc ← 5
* 将累加器 acc 设置为 5。
4. acc ← acc + top_of_stack
* 将累加器 acc 的值加上堆栈顶部的值。
5. pop
* 弹出堆栈顶部的值。

#### MIPS code
```
li $a0, 7               # 将 7 加载到累加器
sw $a0, 0($sp)     # 将累加器的值推入堆栈
addiu $sp, $sp, -4 # 减少堆栈指针，堆栈向下增长

li $a0, 5          # 将 5 加载到累加器
lw $t1, 4($sp)     # 加载堆栈顶的值到临时寄存器 $t1
add $a0, $a0, $t1  # 将累加器和堆栈顶的值相加
addiu $sp, $sp, 4  # 增加堆栈指针，弹出堆栈顶部的值
```

# A Small Language
* A language with integers and integer operations
```
P → D; P 
        | D 
D → def id(ARGS) = E; 
ARGS → id, ARGS 
                | id 
E → int 
        | id 
        | if E1 = E2 then E3 else E4 
        | E1 + E2 
        | E1 – E2 
        | id(E1,…,En)
```

* The first function definition f is the “main” routine 
* Running the program on input i means computing f(i) 
* Program for computing the Fibonacci numbers: 
```
def fib(x) = if x = 1 then 0 else if x = 2 then 1 else fib(x - 1) + fib(x – 2)
```

上述是我们所描述的一个简单的语言，用于接下来的code generation的举例。该语言是函数式语言，不包含oo的特性。

# Code Generation Strategy
* For each expression e we generate MIPS code that: 
    * Computes the value of e in $a0 
    * Preserves $sp and the contents of the stack 
* We define a code generation function cgen(e) whose result is the code generated for e

这部分描述了如何为每个表达式生成 MIPS 代码的过程。具体来说，它描述了代码生成的目标和一个用于生成代码的函数 cgen(e)。以下是详细解释：
目标</br>
1. 计算表达式 e 的值并存储在 $a0 中：
* 每个表达式 e 的计算结果都存储在 MIPS 寄存器 $a0 中。这意味着在计算任何表达式时，最终结果都需要保存在 $a0 中。
2. 保持堆栈指针 $sp 和堆栈的内容不变：
* 计算表达式 e 时，不能改变堆栈指针 $sp 和堆栈中的现有数据。这是为了确保在计算其他表达式时，堆栈的状态是一致的，不会因为某个表达式的计算而导致堆栈内容的破坏。

代码生成函数 cgen(e)</br>
* 定义：
    * cgen(e) 是一个函数，它接收一个表达式 e 并返回生成的 MIPS 代码。
    * 这个函数的目标是生成用于计算表达式 e 的 MIPS 代码，并且代码的执行结果应该在 $a0 中。

## Code Generation for Constants
* The code to evaluate a constant simply copies it into the accumulator: 
```
cgen(i) = li $a0 i 
```
* This preserves the stack, as required 

这部分描述了如何为常量生成 MIPS 代码，并确保堆栈的状态不变。
1. 常量的代码生成
* 将常量的值复制到累加器（accumulator），也就是寄存器 $a0 中。
* 在生成代码时，需要确保堆栈的内容和堆栈指针 $sp 保持不变。
2. 代码生成规则cgen(i)：
* 这个函数生成的代码将立即数 i 加载到 $a0 中。

## Code Generation for Add
```
cgen(e1 + e2) = 
        cgen(e1) 
        sw $a0 0($sp) 
        addiu $sp $sp -4 
        cgen(e2) 
        lw $t1 4($sp) 
        add $a0 $t1 $a0 
        addiu $sp $sp 4
```

这部分描述了如何为表达式 e1 + e2 生成 MIPS 代码。</br>
具体步骤如下：
1. 计算表达式 e1 的值并存储在 $a0 中。
2. 将 $a0 的值推入堆栈。
3. 更新堆栈指针 $sp。
4. 计算表达式 e2 的值并存储在 $a0 中。
5. 从堆栈中取出之前保存的 e1 的值，并将其加载到 $t1 中。
6. 将 $t1 和 $a0 的值相加，并将结果存储在 $a0 中。
7. 更新堆栈指针 $sp，恢复堆栈的状态。


### Code Generation for Add. Wrong!
* Optimization: Put the result of e1 directly in $t1? 
```
cgen(e1 + e2) = 
        cgen(e1) 
        move $t1 $a0 
        cgen(e2) 
        add $a0 $t1 $a0
```

这个slice提出一个优化方案，将 e1 的结果直接存储在 $t1 中，而不是将其推入堆栈。这样可以减少堆栈操作，提高效率。
但是这个优化方案是存在问题的，这种优化假设特定寄存器（如 $t1）可用，并且不会被其他部分的代码覆盖。但是如果我们对`3 + (7 + 5)`进行gen code，那么$t1，会被覆盖，导致错误。

### Code Generation Notes
* The code for + is a template with “holes” for code for evaluating e1 and e2 
    * 即， 加法运算的代码实际上是一个模板，其中有一些占位符（“空白”）用于插入计算 e1 和 e2 的代码。也就是说，生成加法运算代码时，需要将计算 e1 和 e2 的具体代码填充到模板中的特定位置。
* Stack machine code generation is recursive --- 栈机器代码的生成过程是递归的。这意味着，在生成某个表达式的代码时，可能会递归地生成它的子表达式的代码。
    *  – Code for e1 + e2 is code for e1 and e2 glued together --- e1 + e2 的代码是通过将 e1 和 e2 的代码拼接在一起得到的。这种拼接通常是先生成 e1 的代码，再生成 e2 的代码，然后将它们组合成一个整体的代码序列

*  Code generation can be written as a recursivedescent of the AST --- 代码生成过程可以写成对抽象语法树 (AST) 的递归下降。抽象语法树是一种表示程序结构的树状图，代码生成器可以通过递归遍历这棵树来生成每个节点（表达式、语句等）对应的代码。
    *  – At least for expressions --- 
至少对于表达式来说，上述递归生成代码的方式是适用的。这意味着，虽然可能有些复杂的语法结构需要特殊处理，但对于表达式部分，递归下降的方法通常是有效的。


## Code Generation for Sub and Constants
* New instruction: sub reg1 reg2 reg3 
    * Implements reg1 ← reg2 - reg3 
```
cgen(e1 - e2) = 
        cgen(e1) 
        sw $a0 0($sp) 
        addiu $sp $sp -4 
        cgen(e2) 
        lw $t1 4($sp) 
        sub $a0 $t1 $a0 
        addiu $sp $sp 4
```


上述解释了这样的一个过程，生成 MIPS 汇编代码来计算表达式 e1 - e2，并将结果放在 $a0 寄存器中。我们会在下面逐行进行解析：

1. `cgen(e1)`
* 首先生成计算表达式 e1 的代码，并将结果放在 $a0 寄存器中。
2. `sw $a0 0($sp)`

* 将 $a0 中的值存储到栈顶位置（$sp 指针指向的内存地址）。
3. `addiu $sp $sp -4`

* 将栈指针 $sp 减小 4 个字节，以便为下一个值腾出空间。这是因为 MIPS 架构通常以字为单位，4 个字节是一个字的大小。
4. `cgen(e2)`

* 生成计算表达式 e2 的代码，并将结果放在 $a0 寄存器中。
5. `lw $t1 4($sp)`

* 从栈顶（之前存储的 e1 的结果）加载值到 $t1 寄存器中。这里是 4($sp)，因为之前将 $sp 减小了 4。
6. ` sub $a0 $t1 $a0`

* 计算 $t1 - $a0，即 e1 - e2，并将结果存储到 $a0 寄存器中。
7. `addiu $sp $sp 4`
* 恢复栈指针 $sp，将其增加 4 个字节，恢复到存储 e1 之前的状态


## Code Generation for Conditional

* We need flow control instructions 
* New instruction: `beq reg1 reg2 label` 
    * Branch to label if reg1 = reg2 
* New instruction: `b label` 
    * Unconditional jump to label

```
cgen(if e1 = e2 then e3 else e4) = 
            cgen(e1) 
            sw $a0 0($sp) 
            addiu $sp $sp -4 
            cgen(e2) lw $t1 4($sp) 
            addiu $sp $sp 4 
            beq $a0 $t1 true_branch 
            false_branch: 
            cgen(e4) 
            b end_if 
            true_branch: 
            cgen(e3) 
            end_if:
```



上述是一个生成 MIPS 汇编代码来处理 if e1 = e2 then e3 else e4 表达式的模板。</br>
它先计算 e1 并将其结果存储在栈中，然后计算 e2 并与 e1 的结果进行比较。如果相等，则跳转到 true_branch 并执行 e3，否则执行 e4。最后，无论哪个分支执行，都跳转到 end_if 继续执行后续代码。解释如下：

1. `cgen(e1)`
* 生成计算表达式 e1 的代码，并将结果放在 $a0 寄存器中。
2. `sw $a0 0($sp)`
* 将 `$a0` 中的值存储到栈顶位置（$sp 指针指向的内存地址）。
3. `addiu $sp $sp -4`
*  将栈指针 $sp 减小 4 个字节，以便为下一个值腾出空间。
5. `cgen(e2)`
*  生成计算表达式 e2 的代码，并将结果放在 $a0 寄存器中。
7. `lw $t1 4($sp)`
*  从栈顶（之前存储的 e1 的结果）加载值到 $t1 寄存器中。
8. `addiu $sp $sp 4`
*  恢复栈指针 $sp，将其增加 4 个字节，恢复到存储 e1 之前的状态。
10. `beq $a0 $t1 true_branch`
*  如果 $a0 和 $t1 相等，则跳转到 true_branch 标签。
12. `false_branch:`
*  false_branch 标签的起始位置。
14. `cgen(e4)`
*  生成计算表达式 e4 的代码，并将结果放在 $a0 寄存器中。
16. `b end_if`
*  无条件跳转到 end_if 标签。
18. `true_branch:`
*  true_branch 标签的起始位置。
20. `cgen(e3)`
*  生成计算表达式 e3 的代码，并将结果放在 $a0 寄存器中。
22. `end_if:`
*  end_if 标签的起始位置。



## The Activation Record
* Code for function calls and function definitions depends on the layout of the AR 
* A very simple AR suffices for this language: 
    * The result is always in the accumulator 
        * No need to store the result in the AR 
    * The activation record holds actual parameters 
        * For f(x1,…,xn) push xn,…,x1 on the stack 
        * These are the only variables in this language

* The stack discipline guarantees that on function exit $sp is the same as it was on function entry ---- 它保证在函数退出时，栈指针 $sp 恢复到函数入口时的状态。这确保了函数调用和返回期间，栈的内容被正确管理，不会破坏程序的正确性。
* We need the return address -- 在函数调用期间，需要保存返回地址，以便在函数执行完毕后返回到正确的调用点。返回地址通常存储在链接寄存器 $ra 中，函数返回时使用 jr $ra 指令跳转回调用点。
* A pointer to the current activation is useful 
    * This pointer lives in register $fp (frame pointer) 
    * Reason for frame pointer will be clear shortly

* $fp 的作用
在函数入口时，帧指针 $fp 被保存，并设置为当前栈指针 $sp 的值。这保证了帧指针始终指向当前激活记录的起始位置，使得局部变量和参数的访问不受栈指针移动的影响。帧指针在函数退出时恢复，确保函数的栈空间能够正确回收，不会干扰其他函数的栈帧。
 
* Summary: For this language, an AR with the caller’s frame pointer, the actual parameters, and the return address suffices

* Picture: Consider a call to f(x,y), the AR is:
<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_15/Image1.png?raw=true" width="70%">



这部分描述function call的gen code的准备阶段。如果我们要处理function call和function definition，那么我们必须先确定其AR的layout。

在这个简单的语言中，我们确保AR遵循下面的规律：

1. 结果总是保存在累加器 $a0 中。
2. 不需要将结果存储在AR中。
3. AR保存实际参数
    * 即，对于函数 f(x1, ..., xn)，将参数 xn, ..., x1 依次压入栈中。
    * 参数是这个简单语言中唯一的变量。



## Code Generation for Function Call

* The calling sequence is the instructions (of both caller and callee) to set up a function invocation 
* New instruction: ` jal label` 
    * Jump to label, save address of next instruction in $ra 
    * On other architectures the return address is stored on the stack by the “call” instruction

* code generation for function call
```
cgen(f(e1,…,en)) = 
        sw $fp 0($sp) 
        addiu $sp $sp -4 
        cgen(en) 
        sw $a0 0($sp) 
        addiu $sp $sp -4 
        … 
        cgen(e1) 
        sw $a0 0($sp) 
        addiu $sp $sp -4 
        jal f_entry
```

**上述的代码解释如下：**
1. 保存当前帧指针
* 将当前帧指针 $fp 保存到栈中。
* 更新栈指针 $sp，为保存帧指针腾出空间。
2. 生成参数表达式的代码并压栈
* 按照从右到左的顺序生成每个参数表达式 e1, e2, ..., en 的代码，并将它们的结果保存到栈中。注意生成代码时，每次都会更新栈指针。
3. 跳转到函数入口
* 使用 jal 指令跳转到函数 f 的入口地址 f_entry。

**如上所示在调用一个函数时，调用者需要进行一系列的操作来保存上下文和传递参数。**

1. 保存帧指针 (Frame Pointer) 的值
* 调用者首先将当前的帧指针 $fp 的值保存到栈中，以便在函数返回时恢复。
2. 逆序保存实际参数 (Actual Parameters)
* 调用者将实际参数按从右到左的顺序压入栈中。这是因为栈在内存中是向下增长的。3. 保存返回地址 (Return Address)
* 调用者通过 jal 指令调用函数时，返回地址会自动保存到寄存器 $ra 中。
4. AR的大小
AR 是存储函数调用相关信息的数据结构。在 MIPS 架构中，激活记录包含以下内容：保存的帧指针， 实际参数。对于 n 个参数的函数，激活记录的长度为 `4*n + 4 `字节。每个参数占 4 字节，帧指针占 4 字节。 


## Code Generation for Function Definition
* New instruction: jr reg 
    * Jump to address in register reg
介绍了新指令`jr reg`，用于跳转到寄存器 reg 中存储的地址。

```
cgen(def f(x1,...,xn) = e) = 
    move $fp, $sp       # 将当前栈指针保存到帧指针中
    sw $ra, 0($sp)      # 保存返回地址
    addiu $sp, $sp, -4  # 更新栈指针，为返回地址腾出空间
    cgen(e)             # 生成并计算表达式 e 的代码
    lw $ra, 4($sp)      # 恢复返回地址
    addiu $sp, $sp, z   # 更新栈指针，恢复到函数调用前的状态
    lw $fp, 0($sp)      # 恢复帧指针
    jr $ra              # 返回到调用函数的位置
```
* Note: The frame pointer points to the top, not bottom of the frame 
* The callee pops the return address, the actual arguments and the saved value of the frame pointer 
* `z = 4*n + 8`

上面描述了下面的几个事实：
* 在MIPS架构中，帧指针（frame pointer，$fp）通常指向当前栈帧的顶部，而不是底部。这意味着帧指针的值是在栈帧的起始位置。
*  当被调用函数完成其任务时，它需要恢复调用者的上下文环境。这包括恢复返回地址、实际参数和保存的帧指针。

* z 的计算
`z = 4*n + 8` 表示在恢复栈指针时需要调整的字节数。这是因为：
    * 每个参数占用 4 字节，共 n 个参数，占用 4*n 字节。
    * 返回地址占用 4 字节。
    * 保存的帧指针占用 4 字节。
    
    
## Calling Sequence: Example for f(x,y)
下面是调用函数 f(x, y) 的完整过程，包括调用者（caller）和被调用者（callee），
我们假设 f 是一个接受两个参数 x 和 y 的函数。
### 调用者的流程
调用者负责将参数压入栈中，保存当前的帧指针和返回地址，然后跳转到被调用函数的入口。如图`On entry`阶段。
### 被调用者的流程
被调用者负责保存返回地址和帧指针，然后执行函数体的代码，并在返回前恢复调用者的上下文。如图`Before exit`流程。
<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_15/Image2.png?raw=true" width="70%">


## Code Generation for Variables
* Variable references are the last construct 
* The “variables” of a function are just its parameters 
    * They are all in the AR – Pushed by the caller 
* Problem: Because the stack grows when intermediate results are saved, the variables are not at a fixed offset from $sp
* Solution: use a frame pointer 
    * Always points to the return address on the stack 
    * Since it does not move it can be used to find the variables 
* Let xi be the ith (i = 1,…,n) formal parameter of the function for which code is being generated
```
cgen(xi ) = 
        lw $a0 z($fp) ( z = 4*i )
```


在函数调用中，变量引用的处理涉及到从栈中正确地访问函数的参数。由于在栈上存储中间结果会导致栈指针（$sp）移动，函数的参数不会总是位于固定的偏移量。
### 问题描述
* 变量：
在函数中，变量实际上就是函数的参数。
* 存储位置：这些参数都存储在 AR 中，是由调用者在调用函数时压入栈中的。
* 问题：由于在函数执行过程中，栈指针会因保存中间结果而变化，参数在栈中的位置相对于栈指针是不固定的。

### 解决方法
#### 使用帧指针（Frame Pointer）
为了能够正确地访问参数，可以使用帧指针（$fp）。帧指针在函数调用时会指向当前激活记录的起始位置，因此参数的偏移量相对于帧指针是固定的。


### Example: 
For a function def f(x,y) = e the activation and frame pointer are set up as follows:
<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_15/Image3.png?raw=true" width="70%">


## Summary
* The activation record must be designed together with the code generator * Code generation can be done by recursive traversal of the AST 
* We recommend you use a stack machine for your Cool compiler (it’s simple)

* Production compilers do different things 
    * Emphasis is on keeping values (esp. current stack frame) in registers 
    * Intermediate results are laid out in the AR, not pushed and popped from the stack


### 在实际的工程中编译器会做不同的事情
商用编译器与教学编译器或研究编译器相比，通常具有更高的性能要求和更复杂的优化策略。以下是一些常见的差异和优化策略：
1. 寄存器分配：
* 商用编译器会更智能地分配寄存器，以尽可能减少内存访问的次数。寄存器访问比内存访问要快得多，因此优化寄存器分配可以显著提高程序运行速度。
2. 中间代码优化：
* 在生成最终的机器代码之前，生产编译器会对中间代码进行各种优化，例如常量折叠、死代码消除和循环优化。
3. 内联扩展：
* 为了减少函数调用的开销，编译器可以将一些小的函数内联展开，即直接将函数体插入调用处，而不是生成一个实际的调用指令。



# An Improvement
* Idea: Keep temporaries in the AR 
* The code generator must assign a location in the AR for each temporary

```
def fib(x) = if x = 1 then 0 else if x = 2 then 1 else fib(x - 1) + fib(x – 2)
```
对于上面的函数，我们可以在编译阶段就确定它的临时变量的个数。因此我们不需要对每个临时变量分配空间进行存储，由于变量是临时的，因此我们可以多个临时变量共用一个空间。所以对这个函数，我们最多只需要使用2个临时变量的空间即可。

## How Many Temporaries?
* Let NT(e) = # of temps needed to evaluate e 
* NT(e1 + e2) – Needs at least as many temporaries as NT(e1) 
    * Needs at least as many temporaries as NT(e2) + 1 
    * Space used for temporaries in e1 can be reused for temporaries in e2
```
NT(e1 + e2) = max(NT(e1), 1 + NT(e2)) 
NT(e1 - e2) = max(NT(e1), 1 + NT(e2)) 
NT(if e1 = e2 then e3 else e4) = max(NT(e1),1 + NT(e2), NT(e3), NT(e4)) 
NT(id(e1,…,en) = max(NT(e1),…,NT(en)) 
NT(int) = 0 
NT(id) = 0
```


在编译器设计中，计算表达式所需的临时变量数量（temps）是优化和资源管理的重要方面。让我们详细解释并扩展这几条规则：

**NT(e) = # of temps needed to evaluate e**

这条规则的意思是，NT(e) 表示计算表达式 e 所需的临时变量的数量。临时变量用于存储中间计算结果，特别是在更复杂的表达式中。

**NT(e1 + e2) – Needs at least as many temporaries as NT(e1)**

当我们有一个表达式 e1 + e2 时，计算这个表达式至少需要和计算 e1 一样多的临时变量。原因是，我们首先需要计算 e1 并存储其结果，可能需要若干临时变量。

**Needs at least as many temporaries as NT(e2) + 1**

此外，计算 e1 + e2 时，还需要计算 e2 的结果。为了将 e1 和 e2 的结果相加，至少需要一个额外的临时变量来存储 e1 的结果。因此，计算 e1 + e2 需要至少 NT(e2) + 1 个临时变量。

**Space used for temporaries in e1 can be reused for temporaries in e2**

然而，编译器可以优化内存使用。在计算完 e1 后，e1 使用的临时变量空间可以被重用来计算 e2。这意味着，尽管总的临时变量需求可能较高，但通过重用空间，实际所需的内存量可以减少。这是编译器优化的一个重要技术，称为<u>临时变量重用</u>。

## The Revised AR
* For a function definition f(x1,…,xn) = e the AR has 2 + n + NT(e) elements 
    * – Return address 
    * – Frame pointer 
    * – n arguments 
    * – NT(e) locations for intermediate results


下面是经过我们优化之后的AR，存储返回地址 (Return address)， 帧指针 (Frame pointer)， n 个参数 (n arguments)， NT(e) 个临时变量空间 (NT(e) locations for intermediate results)。
<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_15/Image4.png?raw=true" width="70%">


## Revised Code Generation
* Code generation must know how many temporaries are in use at each point 
* Add a new argument to code generation: the position of the next available temporary
* The temporary area is used like a small, fixedsize stack

下图展示了经过优化后的`e1 + e2`的gen code流程。
```
cgen(e1 + e2) = 
            cgen(e1) 
            sw $a0 0($sp) 
            addiu $sp $sp -4 
            cgen(e2) 
            lw $t1 4($sp) 
            add $a0 $t1 $a0 
            addiu $sp $sp 4

cgen(e1 + e2, nt) = 
            cgen(e1, nt) 
            sw $a0 nt($fp) 
            cgen(e2, nt + 4) 
            lw $t1 nt($fp) 
            add $a0 $t1 $a0
```


# Code Generation for OO Languages
## Object Layout
* OO implementation = Stuff from last part + more stuff 
* OO Slogan: If B is a subclass of A, then an object of class B can be used wherever an object of class A is expected 
* This means that code in class A works unmodified for an object of class B

### Example
```
Class A { 
        a: Int; 
        d: Int; 
        f(): Int { a ← a + d }; 
};

Class B inherits A { 
        b: Int; 
        f(): Int { a }; 
        g(): Int { a ← a + b }; 
};

Class C inherits A { 
        c: Int; 
        h(): Int { a ← a + c }; 
};
```

* Attributes a and d are inherited by classes B and C 
* All methods in all classes refer to a 
* For A methods to work correctly in A, B, and C objects, attribute a must be in the same “place” in each object

上述描述了A，B，C的结构类型，B继承A，C继承A。

### Cool Object Layout
* The first 3 words of Cool objects contain header information:
<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_15/Image5.png?raw=true" width="70%">

上面描述Cool Object的layout，我们将逐个描述其属性。

* Class tag is an integer 
    * Identifies class of the object 
* Object size is an integer 
    * Size of the object in words
*  Dispatch ptr is a pointer to a table of methods 
    *  More later 
* Attributes in subsequent slots 
* Lay out in contiguous memory

### 对象的组成部分
1. 类标签 (Class tag) :
* 类标签是一个唯一的整数标识符，用于标识对象所属的类。例如，不同的类可能有不同的整数标签，用于区分对象的类型。
2. 对象大小 (Object size) :
* 对象大小表示对象在内存中占用的大小，以字（word）为单位。这使得在内存分配和对象访问时，可以快速确定对象的大小。
3. Dispatch ptr 是一个指向方法表的指针:
* Dispatch ptr指向一个方法表，该表存储了对象所属类的所有方法。方法表用于动态分派方法调用，使得对象可以正确调用其类或父类的方法。
4. 属性 (Attributes):
* 对象的各个属性存储在内存中的后续槽位中。
* 属性按照声明的顺序排列在内存中，使得属性访问变得高效和简单。
5. 内存布局:
* 对象的所有组成部分在内存中是连续布局的。这种连续的内存布局使得对象的管理和访问更加高效


### Subclass

**Given a layout for class A, a layout for subclass B can be defined by extending the layout of A with additional slots for the additional attributes of B**

**Leaves the layout of A unchanged (B is an extension)** 




如下图所示，子类继承父类时，子类的内存布局可以通过扩展父类的内存布局来定义。这种方式保留了父类的内存布局不变，同时在子类的布局中增加额外的槽位来存储子类的新属性。这种方法确保了继承机制的正确性和效率。
<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_15/Image6.png?raw=true" width="70%">


* The offset for an attribute is the same in a class and all of its subclasses 
    * Any method for an A1 can be used on a subclass A2
* Consider layout for An < … < A3 < A2 < A1
<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_15/Image7.png?raw=true" width="70%">


## Dynamic Dispatch

### Example 
我们用同样的例子来说明，如何Dynamic Dispatch如何实现
```
Class A { 
        a: Int; 
        d: Int; 
        f(): Int { a ← a + d }; 
};

Class B inherits A { 
        b: Int;
        f() : Int { a }; 
        g(): Int { a ← a + b }; 
};

Class C inherits A { 
        c: Int; 
        h(): Int { a ← a + c }; 
};
```
说明：

* e.g() 
    * g refers to method in B if e is a B 
* e.f() 
    * f refers to method in A if e is an A or C (inherited in the case of C) 
    * f refers to method in B if e is a B 
* The implementation of methods and dynamic dispatch strongly resembles the implementation of attributes


#### 方法和动态分派的实现与属性的实现相似
在面向对象编程中，方法的实现和动态分派机制与属性的实现方式有很多相似之处。这种相似性体现在内存布局、指针的使用以及访问方式上。
##### 内存布局中的方法
1. 方法表（Dispatch Table）:
* 每个类都有一个方法表，包含类的所有方法的地址。
* 方法表是一个指针数组，每个指针指向一个方法的实现。
* 对于每个对象，其方法表指针存储在对象的固定位置（通常是对象的前三个槽位之一）。
2.  对象的内存布局:
* 对象的内存布局包括一个指向方法表的指针（dispatch ptr）、类标签、对象大小和属性槽位。
##### 动态分派机制
动态分派（Dynamic Dispatch）是指在运行时决定调用哪个方法。具体步骤如下：
1. 获取方法表指针:
* 当调用一个方法时，首先从对象的内存中获取方法表指针。
2. 查找方法地址:
* 使用方法的名称或其偏移量在方法表中查找对应的方法地址。
3. 调用方法:
* 跳转到方法地址并执行该方法。

### Dispatch Tables
* Every class has a fixed set of methods (including inherited methods) 
* A dispatch table indexes these methods 
    * An array of method entry points 
    * A method f lives at a fixed offset in the dispatch table for a class and all of its subclasses

在面向对象编程中，每个类都有一个固定的方法集合，包括继承的方法。为了实现方法的动态分派，每个类都会维护一个分派表（dispatch table），该表用于索引这些方法。

如下图所示的是每个类的dispatch table。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_8_15/Image8.png?raw=true" width="70%">
* The dispatch table for class A has only 1 method 
* The tables for B and C extend the table for A to the right 
* Because methods can be overridden, the method for f is not the same in every class, but is always at the same offset

类 A 只有一个方法 f，而 B 和 C 可以重写 f，也可以扩展 A，添加新的方法。
1. 方法位置固定:
* 在分派表中，每个方法的位置（偏移量）是固定的。例如，方法 f 在类 A、B 和 C 的分派表中的位置都是 0。
2. 方法重写:
* 当子类重写父类的方法时，新的方法地址会覆盖父类分派表中的相应位置。例如，类 B 和 C 重写了 f 方法，分派表中的偏移量 0 处会存储 B::f 和 C::f 的地址。
3. 分派表扩展:
* 子类的分派表通过在父类分派表的基础上扩展来实现。例如，类 B 在类 A 的分派表右侧新增了方法 g 的地址。


### Using Dispatch Tables

* The dispatch pointer in an object of class X points to the dispatch table for class X 
* Every method f of class X is assigned an offset Of in the dispatch table at compile time


* To implement a dynamic dispatch e.f() we 
    * Evaluate e, giving an object x 
    * Call D[Of ]
        *  D is the dispatch table for x 
        *  In the call, self is bound to x


#### 动态分派的实现
当我们在 Cool 语言中调用一个方法时，例如 `e.f()`，动态分派的过程如下：
1. 计算表达式 e:
* 首先，计算表达式 e，得到一个对象 x。
2. 获取dispatch table D:
* 从对象 x 的分派指针获取对应的分派表 D。这个分派表包含了类 X 的所有方法的入口地址。
3. 调用方法 f:
* 在分派表 D 中，通过偏移量 Of 找到方法 f 的入口地址并进行调用。
* 在调用过程中，将 self 绑定到对象 x，以确保方法调用中的 self 引用的是当前对象 x。
