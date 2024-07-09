---
layout: post
title: Mit6.828 Lab 3:User Environments PARTA
tags: [Mit6.828]
---

Mit6.828 Lab 3:User Environments PARTA的做题思路，以及答案。

>**In this lab, the terms environment and process are interchangeable - both refer to an abstraction that allows you to run a program. We introduce the term "environment" instead of the traditional term "process" in order to stress the point that JOS environments and UNIX processes provide different interfaces, and do not provide the same semantics.**

本实验指的用户环境和`UNIX`中的进程是一个概念，之所有没有使用进程是强调`JOS`的用户环境和`UNIX`进程将提供不同的接口。

[Lab 3: User Environments实验报告](https://www.cnblogs.com/gatsby123/p/9838304.html)
# Part A: User Environments and Exception Handling
在`kern/env.c`中，kernel维护3个全局变量来管理整个运行空间中的`enviroment`。
```
struct Env *envs = NULL;                // All environments
struct Env *curenv = NULL;              // The current env
static struct Env *env_free_list;       // Free environment list
```

1. `envs` 
在JOS刚启动的时候，`envs`指向一个数组，这个数组保存着系统中的所有`environment`。
1. `curenv`
`curenv`表示当前正常运行的`environment`，在启动阶段，`curenv`等于`NULL`。
The kernel uses the curenv symbol to keep track of the currently executing environment at any given time. During boot up, before the first environment is run, curenv is initially set to NULL.

1. `env_free_list`
The JOS kernel keeps all of the inactive Env structures on the env_free_list. This design allows easy allocation and deallocation of environments, as they merely have to be added to or removed from the free list.

## Environment State
解释`ENV`的各个字段的含义，直接看英文吧。
```
struct Env {
        struct Trapframe env_tf;        // Saved registers
        struct Env *env_link;           // Next free Env
        envid_t env_id;                 // Unique environment identifier
        envid_t env_parent_id;          // env_id of this env's parent
        enum EnvType env_type;          // Indicates special system environments
        unsigned env_status;            // Status of the environment
        uint32_t env_runs;              // Number of times environment has run

        // Address space
        pde_t *env_pgdir;               // Kernel virtual address of page dir
};
```


**env_tf**:
    This structure, defined in `inc/trap.h`, holds the saved register values for the environment while that environment is not running: i.e., when the kernel or a different environment is running. The kernel saves these when switching from user to kernel mode, so that the environment can later be resumed where it left off.
**env_link**:
    This is a link to the next ` Env` on the `env_free_list`. `env_free_list ` points to the first free environment on the list.
**env_id**:
The kernel stores here a value that uniquely identifiers the environment currently using this `Env `structure (i.e., usi1ng this particular slot in the `envs` array). After a user environment terminates, the kernel may re-allocate the same `Env` structure to a different environment - but the new environment will have a different `env_id` from the old one even though the new environment is re-using the same slot in the `envs` array.
**env_parent_id**:
The kernel stores here the `env_id` of the environment that created this environment. In this way the environments can form a “family tree,” which will be useful for making security decisions about which environments are allowed to do what to whom.
**env_type**:
This is used to distinguish special environments. For most environments, it will be `ENV_TYPE_USER`. We'll introduce a few more types for special system service environments in later labs.
**env_status**:
This variable holds one of the following values:
* **ENV_FREE**:   
Indicates that the `Env` structure is inactive, and therefore on the `env_free_list`.
* **ENV_RUNNABLE**:    
Indicates that the `Env` structure represents an environment that is waiting to run on the processor.
* **ENV_RUNNING**: 
Indicates that the `Env `structure represents the currently running environment.
* **ENV_NOT_RUNNABLE**:
Indicates that the `Env` structure represents a currently active environment, but it is not currently ready to run: for example, because it is waiting for an interprocess communication (IPC) from another environment.
* **ENV_DYING**:Indicates that the Env structure represents a zombie environment. A zombie environment will be freed the next time it traps to the kernel. We will not use this flag until Lab 4.

**env_pgdir**:
    This variable holds the kernel virtual address of this environment's page directory.

## Allocating the Environments Array

### Exercise 1. 
**解答：**
要求创建页表，以此来存放`envs `数组。代码可以参考`pages`数组的创建。进行空间映射之后，相当于在内存开辟一块空间用来存储每个`Environment`的信息。
```
//////////////////////////////////////////////////////////////////////
// Map the 'envs' array read-only by the user at linear address UENVS
// (ie. perm = PTE_U | PTE_P).
// Permissions:
//    - the new image at UENVS  -- kernel R, user R
//    - envs itself -- kernel RW, user NONE
// LAB 3: Your code here.
envs = (struct Env *)boot_alloc(NENV * sizeof(struct Env));
memset(envs, 0, sizeof(NENV * sizeof(struct Env)));
boot_map_region(kern_pgdir, UENVS, PTSIZE, PADDR(envs), PTE_U);
```
**故障排查：**
运行程序后，发现了下面的报错。。。。。
原来是下面这两条语句的位置错了。这两条语句的位置应该在`page_init`之前。之前没有注意到，汗。。。。。。。
```
envs = (struct Env *)boot_alloc(NENV * sizeof(struct Env));
memset(envs, 0, sizeof(NENV * sizeof(struct Env)));
```

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image27.png?raw=true" width="70%">




## Creating and Running Environments
开始执行一个用户`environment`。需要在`env.c`文件中，完善以下的函数。
```
env_init()
        Initialize all of the Env structures in the envs array and add them to the env_free_list. Also calls env_init_percpu, which configures the segmentation hardware with separate segments for privilege level 0 (kernel) and privilege level 3 (user).
env_setup_vm()
        Allocate a page directory for a new environment and initialize the kernel portion of the new environment's address space.
region_alloc()
        Allocates and maps physical memory for an environment.
load_icode()
        You will need to parse an ELF binary image, much like the boot loader already does, and load its contents into the user address space of a new environment.
env_create()
        Allocate an environment with env_alloc and call load_icode to load an ELF binary into it.
env_run()
        Start a given environment running in user mode.
```

### env_init
设置`envs`中的链表结构，并将`env_free_list`指向`envs[0]`
```
void
env_init(void)
{
    // Set up envs array
    // LAB 3: Your code here.
    // Set up envs array
    // LAB 3: Your code here.
       //int i = 0;
    //env_free_list = NULL;
    env_free_list = NULL;
    for (int i = NENV - 1; i >= 0; i--) {   //前插法构建链表
        envs[i].env_id = 0;
        envs[i].env_link = env_free_list;
        env_free_list = &envs[i];
    }
     // Per-CPU part of the initialization
     env_init_percpu();
}
```
**需要注意函数`env_init_percpu()`。**
1. 首先它加载全局描述符表并且初始化段寄存器`gs`, `fs`, `es`, `ds`, `ss`。`GDT`定义在kern/env.c中。总共加载了6个描述符，其中最开始的描述符必须为空。之后4个分别为内核代码段，内核数据段，用户代码段，用户数据段，以及tss段，其中tss段为空，`trap_init_percpu()`函数会对其进行初始化。

* 全局描述符格式如下所示：

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image28.png?raw=true" width="70%">

* 内核代码段

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image29.png?raw=true" width="70%">

* 内核数据段

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image30.png?raw=true" width="70%">

* 用户代码段

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image31.png?raw=true" width="70%">

* 用户数据段

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image32.png?raw=true" width="70%">

xv6的内存模型是平坦模型，即全局段描述符映射到全部的线性地址空间。用户段和内核段的区别在于其特权级别DPL不同，内核的DPL为0，用户的DPL为3.
2.  初始化段寄存器`gs`, `fs`, `es`, `ds`, `ss`
其中`gs`和`fs`段寄存区的内容指向用户数据段(0x20>>3正好为4，表示GDT中第4个描述符，第0个描述符为空)。
`es`, `ds`, `ss`段寄存区的内容指向内核数据段(0x10>>3正好为2，表示GDT中第2个描述符，第0个描述符为空)。
最后通过`ljmp`的指令，设置`cs`寄存器的值。
```
asm volatile("ljmp %0,$1f\n 1:\n" : : "i" (GD_KT));
//这句汇编语句的意思是：跳转到 CS:0x8, IP: 标签1处的汇编地址。即跳转到“1：”这个标签位置。使用ljmp语句，会使CS寄存器赋值为0x8
//$1f的意思是向后跳转到标签为1的地方，f表示forward，相应的$1b表示向前跳转到标签为1的地方，b表示backward
```

### env_setup_vm
分配一个页目录给`environment`。
为当前的进程分配一个页，用来存放页目录，同时将内核部分的内存的映射完成。
总的思路就是给e指向的Env结构分配页目录，并且继承内核的页目录结构。
设置完页目录也就确定了当前用户环境线性地址空间到物理地址空间的映射。同过这种方式将用户空间和内核空间隔离开来。
```
static int
env_setup_vm(struct Env *e)
{
    int i;
    struct PageInfo *p = NULL;
    // Allocate a page for the page directory
    if (!(p = page_alloc(ALLOC_ZERO)))
        return -E_NO_MEM;
    // Now, set e->env_pgdir and initialize the page directory.
    //
    // Hint:
    //    - The VA space of all envs is identical above UTOP
    //  (except at UVPT, which we've set below).
    //  See inc/memlayout.h for permissions and layout.
    //  Can you use kern_pgdir as a template?  Hint: Yes.
    //  (Make sure you got the permissions right in Lab 2.)
    //    - The initial VA below UTOP is empty.
    //    - You do not need to make any more calls to page_alloc.
    //    - Note: In general, pp_ref is not maintained for
    //  physical pages mapped only above UTOP, but env_pgdir
    //  is an exception -- you need to increment env_pgdir's
    //  pp_ref for env_free to work correctly.
    //    - The functions in kern/pmap.h are handy.
    
    // LAB 3: Your code here.
    p->pp_ref++;
    e->env_pgdir = (pde_t *)page2kva(p);
    memcpy(e->env_pgdir, kern_pgdir, PGSIZE);  //继承内核页目录
    // UVPT maps the env's own page table read-only.
    // Permissions: kernel R, user R
    e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;
    return 0;


}

```

### region_alloc
为虚拟地址`va`分配`len`长度的内存空间。
首先分配足够的物理内存页，之后，将`va`地址映射到这些物理内存页，注意，其修改的地址空间是`Environment`的，而不是内核的空间，也就是说，如果`cr3`存储了`kern_pgdir`的话，访问相同的虚拟地址是无法找到这个被分配的物理空间的，在`kern_pgdir`中，这部分地址是未分配的。
```
static void
region_alloc(struct Env *e, void *va, size_t len)
{
    // LAB 3: Your code here.
    // (But only if you need it for load_icode.)
    //
    // Hint: It is easier to use region_alloc if the caller can pass
    //   'va' and 'len' values that are not page-aligned.
    //   You should round va down, and round (va + len) up.
    //   (Watch out for corner-cases!)
    void *begin = ROUNDDOWN(va, PGSIZE), *end = ROUNDUP(va+len, PGSIZE);
    pte_t   *pte = NULL;                // 页表项指针
    while (begin < end) {
        struct PageInfo *pg = page_alloc(0); //分配一个物理页
        if (!pg) {
            panic("region_alloc failed\n");
        }
        page_insert(e->env_pgdir, pg, begin, PTE_W | PTE_U);   //修改e->env_pgdir，建立线性地址begin到物理页pg的映射关系
        begin += PGSIZE;    //更新线性地址
    }
}

```
### load_icode
这个函数加载`binary`地址处的ELF文件的各个段。`ELF`文件已经从磁盘拷贝到到内存`binary`处，之后，我们需要读取`ELF`头部，获取各个`segment`的大小以及需要被加载到的虚拟地址。注意：在执行for循环前，需要加载`e->env_pgdir`，也就是这句`lcr3(PADDR(e->env_pgdir));`因为我们要将`Segment`拷贝到用户的线性地址空间内，而不是内核的线性地址空间。
```
static void
load_icode(struct Env *e, uint8_t *binary)
{
	// LAB 3: Your code here.
	struct Elf *ELFHDR = (struct Elf *) binary;
	struct Proghdr *ph;				//Program Header
	int ph_num;						//Program entry number
	if (ELFHDR->e_magic != ELF_MAGIC) {
		panic("binary is not ELF format\n");
	}
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
	ph_num = ELFHDR->e_phnum;

	lcr3(PADDR(e->env_pgdir));			//这步别忘了，虽然到目前位置e->env_pgdir和kern_pgdir除了PDX(UVPT)这一项不同，其他都一样。
										//但是后面会给e->env_pgdir增加映射关系

	for (int i = 0; i < ph_num; i++) {
		if (ph[i].p_type == ELF_PROG_LOAD) {		//只加载LOAD类型的Segment
			region_alloc(e, (void *)ph[i].p_va, ph[i].p_memsz);
			memset((void *)ph[i].p_va, 0, ph[i].p_memsz);		//因为这里需要访问刚分配的内存，所以之前需要切换页目录
			memcpy((void *)ph[i].p_va, binary + ph[i].p_offset, ph[i].p_filesz); //应该有如下关系：ph->p_filesz <= ph->p_memsz。搜索BSS段
		}
	}

	lcr3(PADDR(kern_pgdir));

	e->env_tf.tf_eip = ELFHDR->e_entry;
	// Now map one page for the program's initial stack
	// at virtual address USTACKTOP - PGSIZE.

	// LAB 3: Your code here.
	region_alloc(e, (void *) (USTACKTOP - PGSIZE), PGSIZE);
}
```
### env_create
从`env_free_list`链表拿一个`Env`结构，加载从`binary`地址开始处的`ELF`可执行文件到该`Env`结构。
```
void
env_create(uint8_t *binary, enum EnvType type)
{
    // LAB 3: Your code here.
    struct Env *env_store;
    if (env_alloc(&env_store, 0) < 0) {
        panic("env_create : env_alloc error!");
        return;
    }
    load_icode(env_store, binary);
    env_store->env_type = type;
}
```

### env_run(struct Env * e)
执行e指向的`Environment`
该函数首先设置`curenv`，然后修改`e->env_status`，`e->env_runs`两个字段。接着加载线性地址空间，最后将`e->env_tf`结构中的寄存器快照弹出到寄存器，这样就会从新的`%eip`地址处读取指令进行解析。

**`Trapframe`结构和`env_pop_tf()`函数**
汇编指令说明：
[ret,retf,iret的区别](https://blog.csdn.net/lucas_w17/article/details/75055782)
`popal` --- 弹出所有的通用寄存器，
`ret` --- 也可以叫做近返回，即段内返回。处理器从堆栈中弹出IP或者EIP，然后根据当前的CS：IP跳转到新的执行地址。如果之前压栈的还有其余的参数，则这些参数也会被弹出。
`iret` --- 用于从中断返回，会弹出IP/EIP，然后CS，以及一些标志。然后从CS：IP执行。
`retf `--- 也叫远返回，从一个段返回到另一个段。先弹出堆栈中的IP/EIP，然后弹出CS，有之前压栈的参数也会弹出。
```
struct PushRegs {
	/* registers as pushed by pusha */
	uint32_t reg_edi;
	uint32_t reg_esi;
	uint32_t reg_ebp;
	uint32_t reg_oesp;		/* Useless */
	uint32_t reg_ebx;
	uint32_t reg_edx;
	uint32_t reg_ecx;
	uint32_t reg_eax;
} __attribute__((packed));

struct Trapframe {
	struct PushRegs tf_regs;
	uint16_t tf_es;
	uint16_t tf_padding1;
	uint16_t tf_ds;
	uint16_t tf_padding2;
	uint32_t tf_trapno;
	/* below here defined by x86 hardware */
	uint32_t tf_err;
	uintptr_t tf_eip;
	uint16_t tf_cs;
	uint16_t tf_padding3;
	uint32_t tf_eflags;
	/* below here only when crossing rings, such as from user to kernel */
	uintptr_t tf_esp;
	uint16_t tf_ss;
	uint16_t tf_padding4;
} __attribute__((packed));

// Restores the register values in the Trapframe with the 'iret' instruction.
// This exits the kernel and starts executing some environment's code.
//
// This function does not return.
//
void
env_pop_tf(struct Trapframe *tf)
{
	asm volatile(
		"\tmovl %0,%%esp\n"				//将%esp指向tf地址处
		"\tpopal\n"						//弹出Trapframe结构中的tf_regs值到通用寄存器
		"\tpopl %%es\n"					//弹出Trapframe结构中的tf_es值到%es寄存器
		"\tpopl %%ds\n"					//弹出Trapframe结构中的tf_ds值到%ds寄存器
		"\taddl $0x8,%%esp\n" /* skip tf_trapno and tf_errcode */
		"\tiret\n"						//中断返回指令，具体动作如下：从Trapframe结构中依次弹出tf_eip,tf_cs,tf_eflags,tf_esp,tf_ss到相应寄存器
		: : "g" (tf) : "memory");
	panic("iret failed");  /* mostly to placate the compiler */
}
```
从代码以及注释中也可以梳理出其流程：首先pop出所有的通用寄存器，其次手动赋值`ds`, `es`寄存器，最后通过`iret`，将保存的`eip`, `cs`, `eflags`, `esp`, `ss`的值赋值到对应的寄存器中，开始执行`eip`指向的地址。

### 启动`ENVIROMNMENT`流程梳理
`environment`初始化分为两步，第一步是`environment`的初始化，创建`environment`启动所需要的各种资源；第二步是启动第一个`environment`，然后将控制转移到用户程序。
#### env_init
1.  初始化`env_free_list`数据结构，长度为1024，表示链表支持1024个`environment`。
2.  `env_init_percpu`加载gdt以及设置段描述符寄存器。gdt装载了4个段，内核代码段，内核数据段，用户代码段，用户数据段，其范围都指向了0~4G，只是用户段的DPL和内核段的DPL。之所以只使用了这4个段，一是因为开启页模式之后我们可以使用页模型来进行内存的管理，而不必再使用段来进行管理；二是，定义用户和内核的代码段与数据段是为了对内核和用户的代码进行特权级管理。

#### ENV_CREATE(user_hello, ENV_TYPE_USER);
这个宏展开后的形式`env_create(_binary_obj_user_hello_start, ENV_TYPE_USER)`。
1. 首先通过`env_alloc`创建`struct ENV`大小的空间，并初始化。主要设置的部分是`e->env_pgdir`，`e->env_tf`, `e->env_status`。
2. 加载`_binary_obj_user_hello_start`指向的二进制文件。通过读取`_binary_obj_user_hello_start`的ELF头部，将`ELF`文件的各个可加载段加载到`e->env_pgdir`指向的用户空间，并暂时分配4096字节栈空间。
3. 最后设置`e->env_tf.tf_eip`指向`ELF`文件的入口。

#### env_run
1. 将`env[0]`作为入参传递给`env_run`，作为第一个`environment`进行启动。
2. 设置`curenv`指向将要运行的`environment`，即`env[0]`。
3. 通过`lcr3`将`env[0]`中的页基地址加载到`crr3`寄存器中，从而通过虚拟内存的映射可以取出`environment`中缓存的一些数据。
4. `env_pop_tf`将缓存在`struct Trapframe`结构体中的各个寄存器的值取出，通过将`struct Trapframe *tf` 的地址赋值给`esp`，之后调用`popal`以及`pop`指令，将`tf`中的各个寄存器的值取出，最后使用`iret`，弹出`eip`和`cs`寄存器，并从这个寄存器开始执行。



### 调试
将控制转移到`hello（user/hello.c）`程序的时候，`hello`使用了`cprintf`函数，而`cprintf`通过`int 48` 触发了系统调用。
```
cprintf --> vcprintf --> vprintfmt --> putch --> cputchar --> "int 0x30"
```
但是由于我们并未设置系统调用的IDT，因此此处不起作用，会导致`Triple fault`。


## Handling Interrupts and Exceptions
### Basic of Proctected Control Transfer
* 中断和异常是一种控制流的突变，如下图所示(引用自《深入理解计算机系统》)。interrupt和exception的区别在于，interrupt是由处理器的外部事件引起的，比如I/O输入等，而exception是处理处理器自身在执行指令过程中产生的错误，比如执行div指令时，除数为0的情况。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image33.png?raw=true" width="70%">

* Interrupt分为两种
    * 可屏蔽中断 ： 由INTR引脚触发
    * 不可屏蔽中断 ： 由NMI引脚触发
* 深入理解计算机系统中，将异常分为4类。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image34.png?raw=true" width="70%">



* 为了防止中断发生时，当前运行的代码不会跳转到内核的任意位置执行，x86提供了两种机制：
    * **中断描述符表**：处理器确保异常或中断发生时，只会跳转到由内核定义的代码点处执行。x86允许256种不同的中断或异常进入点，每一个都有一个向量号，从0到255。CPU使用向量号作为IDT的索引，取出一个IDT描述符，根据IDT描述符可以获取中断处理函数cs和eip的值，从而进入中断处理函数执行。
    * **任务状态段(TSS)**：当x86异常发生，并且发生了从用户模式到内核模式的转换时，处理器也会进行栈切换。一个叫做task state segment (TSS)的结构指定了栈的位置。TSS是一个很大的数据结构，由于JOS中内核模式就是指权限0，所以处理器只使用TSS结构的ESP0和SS0两个字段来定义内核栈，其它字段不使用。那么内核如何找到这个TSS结构的呢？JOS内核维护了一个static struct Taskstate ts;的变量，然后在trap_init_percpu()函数中，设置TSS选择子（使用ltr指令）。

### Excercise 4
修改trapentry.S和trap.c建立异常处理函数，在trap_init()中建立并且加载IDT。
#### add entry point in trapentry.S
1. 如何确定异常是否自动压入`error code`
从[Chapter 9, Exceptions and Interrupts](https://pdos.csail.mit.edu/6.828/2018/readings/i386/s09_10.htm)中，查找到每个异常的信息：Interrupt号，处理器是否自动压入error code.

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image35.png?raw=true" width="70%">

2. 代码
```
  //Divide error
 TRAPHANDLER_NOEC(th0, 0) 
  //Debug exceptions
 TRAPHANDLER_NOEC(th1, 1) 
  //Breakpoint
 TRAPHANDLER_NOEC(th3, 3) 
  //Overflow
 TRAPHANDLER_NOEC(th4, 4) 
  //Bounds check
 TRAPHANDLER_NOEC(th5, 5) 
  //Invalid opcode   
 TRAPHANDLER_NOEC(th6, 6) 
  //Coprocessor not available 
 TRAPHANDLER_NOEC(th7, 7) 
  //System error    
 TRAPHANDLER(th8, 8) 
  //Coprocessor Segment Overrun 
 TRAPHANDLER_NOEC(th9, 9) 
  //Invalid TSS   
 TRAPHANDLER(th10, 10) 
  //Segment not present 
 TRAPHANDLER(th11, 11) 
  //Stack exception        
 TRAPHANDLER(th12, 12) 
  //General protection fault  
 TRAPHANDLER(th13, 13) 
  //Page fault  
 TRAPHANDLER(th14, 14) 
  //Coprocessor error   
 TRAPHANDLER_NOEC(th16, 16) 
  //Two-byte SW interrupt  
 TRAPHANDLER_NOEC(th_syscall, T_SYSCALL)
 
 
 
 
 //参考inc/trap.h中的Trapframe结构。tf_ss，tf_esp，tf_eflags，tf_cs，tf_eip，tf_err在中断发生时由处理器压入，所以现在只需要压入剩下寄存器（%ds,%es,通用寄存器） 
 //切换到内核数据段
 _alltraps: 
    pushl %ds 
    pushl %es 
    pushal 
    pushl $GD_KD 
    popl %ds 
    pushl $GD_KD 
    popl %es 
    pushl %esp      //压入trap()的参数tf，%esp指向Trapframe结构的起始地址 
    call trap          //调用trap()函数
```












#### 安装IDT表
要解决几个问题：
1、中断描述符表的格式
2、中断描述符表存放在哪个位置


[x86的DPL,RPL,CPL](https://blog.csdn.net/FirMoonLight/article/details/127129493?spm=1001.2014.3001.5502)
[cpu之DPL,RPL,CPL 之间的联系和区别](https://www.cnblogs.com/buzhunbukaixin/articles/4662004.html)

1. 格式问题
`STAGE`宏用来设置中断门和陷阱门。《x86汇编语言：从实模式到保护模式》中说明,通过门的概念，我们可以实现低特权级代码到高特权级代码的控制转移，例如从用户态转移到内核态。
在保护模式下，处理器对中断的管理和实模式是不同的，它并非使用传统的中断向量表来保存中断处理过程的地址，而是中断描述符表(IDT)，顾名思义，在这个表中，保存的是和中断处理过程有关的描述符，包括中断门，陷阱门和任务门。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image36.png?raw=true" width="70%">

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image37.png?raw=true" width="70%">

中断门和陷阱门在使用上的区别不在于中断是外部产生的还是有CPU本身产生的，而在于通过中断门进入中断服务程序时CPU会自动将中断关闭（将EFLAGS寄存器中IF标志位置0），以防止嵌套中断产生，而通过陷阱门进入服务程序时则维持IF标志位不变。这是二者唯一的区别。

`STAGE`参数说明：
`gate`：中断门或者陷阱门存放的内存地址
`istrap`：表示是中断门还是陷阱门
`sel`：段选择子，应该选择内核代码段的GDT条目
`off`：处理程序入口地址
`dpl`：描述符特权级，应该为0，最高特权级

2. 中断描述符表位置
存放`trap.c`文件中的`idt`数组处。

3. 代码
```
void
trap_init(void)
{
	extern struct Segdesc gdt[];

	// LAB 3: Your code here.
    void th0();
	void th1();
	void th3();
	void th4();
	void th5();
	void th6();
	void th7();
	void th8();
	void th9();
	void th10();
	void th11();
	void th12();
	void th13();
	void th14();
	void th16();
	void th_syscall();
    SETGATE(idt[0], 0, GD_KT, th0, 0);
    SETGATE(idt[1], 0, GD_KT, th1, 0);
    SETGATE(idt[3], 0, GD_KT, th3, 3);
    SETGATE(idt[4], 0, GD_KT, th4, 0);
	SETGATE(idt[5], 0, GD_KT, th5, 0);
	SETGATE(idt[6], 0, GD_KT, th6, 0);
	SETGATE(idt[7], 0, GD_KT, th7, 0);
	SETGATE(idt[8], 0, GD_KT, th8, 0);
	SETGATE(idt[9], 0, GD_KT, th9, 0);
	SETGATE(idt[10], 0, GD_KT, th10, 0);
	SETGATE(idt[11], 0, GD_KT, th11, 0);
	SETGATE(idt[12], 0, GD_KT, th12, 0);
	SETGATE(idt[13], 0, GD_KT, th13, 0);
	SETGATE(idt[14], 0, GD_KT, th14, 0);
	SETGATE(idt[16], 0, GD_KT, th16, 0);

    SETGATE(idt[T_SYSCALL], 0, GD_KT, th_syscall, 3);
	// Per-CPU setup 
	trap_init_percpu();
}
```
该函数会在进入内核时由`i386_init()`调用。我们添加的代码就是建立`IDT`，`trap_init_percpu()`中的`lidt(&idt_pd);`正式加载`IDT`。
`trap_init_percpu()`中除了加载`IDT`，同时会设置`TSS`描述符，当异常或者陷阱触发的时候，会切换到这个`TSS`，因此`TSS`中栈是内核栈。
```
ts.ts_esp0 = KSTACKTOP;
ts.ts_ss0 = GD_KD;
```

### Questions
1. What is the purpose of having an individual handler function for each exception/interrupt? (i.e., if all exceptions/interrupts were delivered to the same handler, what feature that exists in the current implementation could not be provided?)
解答：不同的中断/异常需要使用不同的中断函数进行处理。比如有些异常是代表指令有错误，则不会返回被中断的命令。而有些中断可能只是为了处理外部IO事件，此时执行完中断函数还要返回到被中断的程序中继续运行。
2. Did you have to do anything to make the user/softint program behave correctly? The grade script expects it to produce a general protection fault (trap 13), but softint's code says int $14. Why should this produce interrupt vector 13? What happens if the kernel actually allows softint's int $14 instruction to invoke the kernel's page fault handler (which is interrupt vector 14)?
因为当前的系统正在运行在用户态下，CPL为3。通过INT指令触发14之后，处理器会调用到14的中断门，该中断门的DPL为0，因此会触发异常13（General Protection Exception）。配置14中断门的特权级为0是为了防止用户态代码通过INT指令来调用到该处理函数。

## 执行结果
此时虽然IDT配置了中断向量表，在遇到`INT 0x30`时可以进行跳转，但是具体的处理流程还未完成，因此无法看到`hello.c`中想要打印的字符串。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image38.png?raw=true" width="70%">

# Part B: Page Faults, Breakpoints Exceptions, and System Calls

## Handling Page Fault
当处理器遇到Page Fault的时候，它将导致错误的虚拟地址存储在寄存器CR2中，在trap.c中，通过`page_fault_handler`函数处理页面错误异常。

### Exercise 5
Modify `trap_dispatch()` to dispatch page fault exceptions to `page_fault_handler()`. You should now be able to get make grade to succeed on the faultread, faultreadkernel, faultwrite, and faultwritekernel tests. If any of them don't work, figure out why and fix them. Remember that you can boot JOS into a particular user program using make run-x or make run-x-nox. For instance, make run-hello-nox runs the hello user program.
在`trap_dispatch()`添加如下代码，此时`page_fault`会交给`page_fault_handler`进行处理，但是`page_fault_handler`的代码还是不完整，需要继续完善。
```
    int trapno = tf->tf_trapno;
    if (tf->tf_trapno == T_PGFLT) {
        page_fault_handler(tf);
        return;
    }
```

## The Breakpoint Exception
break exception通常被调试器用来向程序插入断点，调试器（例如gdb）会将指定位置的指令替换为`int 3`。在JOS中，我们将这个异常作为一个伪系统调用，这样任何`user environment`都可以利用这个异常来触发JOS kernel monitor。

### Exercise 6
Modify `trap_dispatch() `to make breakpoint exceptions invoke the kernel monitor. You should now be able to get make grade to succeed on the breakpoint test.
breakpoint exception的处理函数是哪个？`monitor(tf);`。那么代码如下：
```
    if (tf->tf_trapno == T_BRKPT) {
        monitor(tf);
        return;
    }
```

### challenge

### Questions
[MIT6.828 Lab3](https://www.cnblogs.com/fatsheep9146/p/5451579.html)

3. The break point test case will either generate a break point exception or a general protection fault depending on how you initialized the break point entry in the IDT (i.e., your call to SETGATE from trap_init). Why? How do you need to set it up in order to get the breakpoint exception to work as specified above and what incorrect setup would cause it to trigger a general protection fault?
这段的翻译是，如果你在设置IDT时，对break point exception采用不同的方式进行设置，可能会产生触发不同的异常，有可能是break point exception，有可能是 general protection exception。这是为什么？
解答：
在设置IDT表中的breakpoint exception的表项时，如果我们把表项中的DPL字段设置为3，则会触发break point exception，如果设置为0，则会触发general protection exception。因为当程序调用`INT n, INT 3, INTO`指令的时候，处理器才会检测中断门和陷阱门的DPL。此时CPL必须小于等于DPL。因为用户态的CPL为3，所以中断门和陷阱门的DPL必须被设置为3。

4. What do you think is the point of these mechanisms, particularly in light of what the user/softint test program does?


## System Calls
用户`environment`通过`system call`来使用内核功能。当用户程序触发系统调用，系统进入内核态。处理器和操作系统将保存该用户程序当前的上下文状态，由内核完成系统调用，然后回到用户程序继续执行。而用户程序到底是如何得到操作系统的注意，以及它如何说明它希望操作系统做什么事情的方法是有很多不同的实现方式的。

在JOS中，我们会采用int指令，这个指令会触发一个处理器的中断。特别的，我们用`int $0x30`来代表系统调用中断。注意，中断0x30不是通过硬件产生的。

应用程序会把系统调用号以及系统调用的参数放到寄存器中。通过这种方法，内核就不需要去查询用户程序的堆栈了。系统调用号存放到` %eax `中，参数则存放在 `%edx, %ecx, %ebx, %edi`和 `%esi `中。内核会把返回值送到` %eax`中。在`lib/syscall.c`中已经写好了触发一个系统调用的代码。
```
    asm volatile("int %1\n"
             : "=a" (ret)
             : "i" (T_SYSCALL),
               "a" (num),
               "d" (a1),
               "c" (a2),
               "b" (a3),
               "D" (a4),
               "S" (a5)
             : "cc", "memory");
```
"__volatile__"表示编译器不要优化代码，后面的指令 保留原样，其中`"=a"`表示输出变量ret要从`"eax"`取出;` "i"`表示int指令的参数为48，触发系统调用;最后一个子句告诉汇编器这可能会改变条件代码和任意内存位置。memory强制gcc编译器假设所有内存单元均被汇编指令修改，这样cpu中的registers和cache中已缓存的内存单元中的数据将作废。cpu将不得不在需要的时候重新读取内存中的数据。这就阻止了cpu又将registers，cache中的数据用于去优化指令，而避免去访问内存。

### Exercise 7
Add a handler in the kernel for interrupt vector T_SYSCALL.
在内核中为中断向量T_SYSCALL添加一个handler。你需要编辑`kern/trapentry.S`和`kern/trap.c`的`trap_init()`，同时需要修改`trap_dispatch()`来处理`system call interrupt`，并保证返回值能够通过`%eax`传给用户。最后，你需要使用`kern/syscall.c`文件中的`syscall()`函数，来处理每一种系统调用。

答：这个Excercise要求我们对系统调用的整个流程有一个全面的理解。
第一步：编辑`kern/trapentry.S`和`kern/trap.c`，需要始化的时候，设置system call的idt条目。
```
//kern/trapentry.S增加

TRAPHANDLER_NOEC(th_syscall, T_SYSCALL)

//kern/trap.c增加

SETGATE(idt[T_SYSCALL], 0, GD_KT, th_syscall, 3);
```

第二步：修改`trap_dispatch()`函数

1、 `trap_dispatch()`被调用的顺序是： `trapentry.s:_alltraps` --> `void trap(struct Trapframe *tf)` --> `void trap_dispatch(struct Trapframe *tf)`。
2、 也就是说，当`int 0x30`被触发之后，`idt[0x30]`处的中断向量表被找到，然后从该中断向量中找到处理函数的入口，即`
SETGATE(idt[T_SYSCALL], 0, GD_KT, th_syscall, 3);`函数中，`th_syscall`处的汇编代码。而`th_syscall`部分的汇编代码是由宏`TRAPHANDLER`创建的，正好调用到了`_alltrap`函数。

3、 汇编指令`pushal`的寄存器的顺序
从下图中可以看出，入栈顺序为`EAX，ECX，EDX，EBX，ESP（压栈前的值），EBP，ESI，EDI`。所以`PushRegs`的结构体如下所示。
```
struct PushRegs {
    /* registers as pushed by pusha */
    uint32_t reg_edi;
    uint32_t reg_esi;
    uint32_t reg_ebp;
    uint32_t reg_oesp;      /* Useless */
    uint32_t reg_ebx;
    uint32_t reg_edx;
    uint32_t reg_ecx;
    uint32_t reg_eax;
} __attribute__((packed));

```

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image39.png?raw=true" width="70%">


4、 每个系统调用对应的处理函数
其处理函数在`kern/syscall.c `中，目前总共有4个处理函数，`sys_cputs`，`sys_cgetc`，`sys_getenvid`，`sys_env_destroy`。对比`kern/syscall.c`和`lib/syscall.c`，可以发现，两者都有`syscall`函数，但是`lib/syscall.c`中的`syscall`函数其功能是通过汇编代码`int 0x30`来触发系统中段，因此可以断定，`lib/syscall.c`中的代码是提供用户调用的，而`kern/syscall.c`中的代码，是内核真正的处理逻辑。

代码如下：
```
//trap.c

    if (tf->tf_trapno == T_SYSCALL) {
        ret_code = syscall(tf->tf_regs.reg_eax, tf->tf_regs.reg_edx,
                    tf->tf_regs.reg_ecx,
                    tf->tf_regs.reg_ebx,
                    tf->tf_regs.reg_edi,
                    tf->tf_regs.reg_esi);
        tf->tf_regs.reg_eax = ret_code;
        return;
    }

//kern/syscall.c

int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
    // Call the function corresponding to the 'syscallno' parameter.
    // Return any appropriate return value.
    // LAB 3: Your code here.
    //panic("syscall not implemented");
    int32_t res = 0;
    switch (syscallno) {
        case SYS_cputs:
            sys_cputs((char *)a1, a2);
            break;
        case SYS_cgetc:
            res = sys_cgetc();
            break;
        case SYS_env_destroy:
            res = sys_env_destroy(a1);
            break;
        case SYS_getenvid:
            res = sys_getenvid();
            break;
        default:
            return -E_INVAL;
    }
    return res;
}


```


5、 `make run-hello`后打印出了`hello world`。并且出现了如练习中所说的page fault。
为什么出现`page fault`，在下一节中将做出说明 -- 在hello.c中it tries to access thisenv->env_id，而thisenv并没有初始化。
```
He110 World6828 decimal is XXX octal!
Physical memory: 131072K available, base = 640K, extended = 130432K
check_page_free_list() succeeded!
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_free_list() succeeded!
check_page_installed_pgdir() succeeded!
[00000000] new env 00001000
Incoming TRAP frame at 0xefffffbc
hello, world
Incoming TRAP frame at 0xefffffbc
[00001000] user fault va 00000048 ip 0080005f
TRAP frame at 0xf01d3000
  edi  0x00000000
  esi  0x00000000
  ebp  0xeebfdfd0
  oesp 0xefffffdc
  ebx  0x00802000
  edx  0xeebfde78
  ecx  0x0000000d
  eax  0x00000000
  es   0x----0023
  ds   0x----0023
  trap 0x0000000e Page Fault
  cr2  0x00000048
  err  0x00000004 [user, read, not-present]
  eip  0x0080005f
  cs   0x----001b
  flag 0x00000082
  esp  0xeebfdfc8
  ss   0x----0023
[00001000] free env 00001000
Destroyed the only environment - nothing more to do!

```


## user-mode startup
用户程序首先从`lib/entry.S`开始运行。 经过一些初始化后，此代码在`lib/libmain.c`中调用`libmain()`。 我们应该修改`libmain()`以初始化全局指针`thisenv`以指向`envs[]`数组中此环境对应的`struct Env`。 程序是如何执行到这里来的？ 应该问题还是会回到之前提出的`env[0]`到底是什么时候初始化的。

修改libmain.c,添加一条语句即可。
```
  // set thisenv to point at our Env structure in envs[].
    // LAB 3: Your code here.
    thisenv = &envs[ENVX(sys_getenvid())];
```
重新执行`make run-hello`，发现能正常打印`i am environment ...`字段。
```
check_page_free_list() succeeded!
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_free_list() succeeded!
check_page_installed_pgdir() succeeded!
[00000000] new env 00001000
Incoming TRAP frame at 0xefffffbc
Incoming TRAP frame at 0xefffffbc
hello, world
Incoming TRAP frame at 0xefffffbc
i am environment 00001000
Incoming TRAP frame at 0xefffffbc
[00001000] exiting gracefully
[00001000] free env 00001000
Destroyed the only environment - nothing more to do!

```

## Page faults and memory protection
内存保护是操作系统的一个重要特性，可确保一个程序中的错误不会破坏其他程序或破坏操作系统本身。
操作系统通常依靠硬件支持来实现内存保护。 操作系统会通知硬件哪些虚拟地址有效，哪些虚拟地址无效。 当程序试图访问无效地址或无权地址时，处理器会在导致故障的指令处停止程序，然后携带者有关操作的信息陷入内核。

故障分为可修复故障（例如自动栈扩展，缺页）和不可修复故障。

系统调用为内存保护提出了一个有趣的问题。 大多数系统调用接口允许用户程序传递指向内核的指针。 这些指针指向要读取或写入的用户缓冲区。 然后内核在执行系统调用时对这些指针进行解引用操作。 
这有两个问题：
1. 内核中的页错误可能比用户程序中的页面错误严重得多。如果内核在处理自己的数据结构时发生页错误，那就是内核错误，并且错误处理程序应该使内核（进而整个系统）panic。 但是，当内核解引用用户程序传递的指针时，它需要一种方法来记住这些解引用操作引起的任何页错误实际上都代表用户程序。
2. 内核通常比用户程序拥有更多的内存权限。用户程序可能传递一个指向系统调用的指针，该系统调用指向内核可以读写但程序不能读写的内存。内核必须小心不要被诱骗去解引用这样的指针，因为这可能会暴露私有信息或者破坏内核的完整性。

由于上诉两个原因，内核在处理用户通过system call传过来的指针的时候，需要格外小心。

我们只需要仔细检查每个从用户空间传入到内核空间的指针，就能够很完美的handle这两个问题。当用户程序向内核传递了一个指针时，我们需要判断这个地址是否是属于用户空间的地址，如果是，那个这个地址就能够被正常操作。因此，内核就不会因为解引用指针，而导致内存的page fault了。

### Excercise 9
**Firstly，Change kern/trap.c to panic if a page fault happens in kernel mode.
Secondly, Read user_mem_assert in kern/pmap.c and implement user_mem_check in that same file. Change kern/syscall.c to sanity check arguments to system calls.
Finally, change debuginfo_eip in kern/kdebug.c to call user_mem_check on usd, stabs, and stabstr.**

1. 修改`kern/trap.c`，一旦在内核态发生`page fault`就直接panic，因此在`page_fault_handler()`中添加如下代码：
```
	if ((tf->tf_cs & 3) == 0) //内核态发生缺页中断直接panic
		panic("page_fault_handler():page fault in kernel mode!\n");

```

2. 修改`kern/pmap.c`，实现函数`user_mem_check`。
```
int
user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
    // LAB 3: Your code here.
    cprintf("user_mem_check va: %x, len: %x\n", va, len);
    uint32_t begin = (uint32_t) ROUNDDOWN(va, PGSIZE);
    uint32_t end = (uint32_t) ROUNDUP(va+len, PGSIZE);
    uint32_t i;
    for (i = begin; i < end; i += PGSIZE) {
        pte_t *pte = pgdir_walk(env->env_pgdir, (void*)i, 0);
        if ((i >= ULIM) || !pte || !(*pte & PTE_P) || ((*pte & perm) != perm)) {        //具体检测规则
            user_mem_check_addr = (i < (uint32_t)va ? (uint32_t)va : i);                //记录无效的那个线性地址
            return -E_FAULT;
        }
    }
    cprintf("user_mem_check success va: %x, len: %x\n", va, len);
    return 0;
}
```

3. 修改` kern/syscall.c `, 测用户环境是否有权限访问线性地址区域`[va, va+len)`。然后对在`kern/syscall.c`中的系统调用函数使用`user_mem_check()`工具函数进行内存访问权限检查。


4. 执行`make run-buggyhello-nox`，会对`sys_puts`的入参进行检测，发现地址为非法。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image40.png?raw=true" width="70%">

5. 在`kern/kdebug.c` 中修改函数` debuginfo_eip` ，在`usd, stabs, stabstr`中调用` user_mem_check `。

```
if (user_mem_check(curenv, usd, sizeof(struct UserStabData), PTE_U)) {
    return -1;
}

if (user_mem_check(curenv, stabs, stab_end - stabs, 0) || user_mem_check(curenv, stabstr, stabstr_end - stabstr, 0)) {
    return -1;
}
```


**Excersice9**做完之后，**Excersice10**也一并做完了。

