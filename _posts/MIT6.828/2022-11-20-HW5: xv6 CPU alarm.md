---
layout: post
title: Mit6.828 HW5:xv6 CPU alarm
tags: [Mit6.828]
---

这个HW的主要任务就是在进程使用CPU的时候，定期向其发送alert。

* 添加这个功能的意义在于，一是可以限制进程占用CPU的时间，二是可以当进程想要执行定时任务的时候，可以利用这个alert。
* 添加一个新的alarm(interval，handler）系统调用。 如果应用程序调用alarm（n，fn），那么在程序消耗每个n“ticks”的CPU时间之后，内核将调用应用程序函数fn。 当fn返回时，应用程序将从中断处继续。 (tick是xv6中相当随意的时间单位，由硬件定时器产生中断的频率决定).

添加系统调用的流程和HW3是一样的，流程如下：
## 1 添加系统调用流程
### 1.1 创建`alarmtest.c`，用于生成可执行文件`alarmtest`。
* 创建`alarmtest.c`文件，其包含如下代码：

```
#include "types.h"
#include "stat.h"
#include "user.h"

void periodic();

int
main(int argc, char *argv[])
{
  int i;
  printf(1, "alarmtest starting\n");
  alarm(10, periodic);
  for(i = 0; i < 25*500000; i++){
    if((i % 250000) == 0)
      write(2, ".", 1);
  }
  exit();
}

void
periodic()
{
  printf(1, "alarm!\n");
}
```

* 程序调用`alarm(10, periodic)`，要求内核每隔10个ticks强制调用periodic()，然后自旋一段时间。其输出应该如下所示：

```
$ alarmtest
alarmtest starting
.....alarm!
....alarm!
.....alarm!
......alarm!
.....alarm!
....alarm!
....alarm!
......alarm!
.....alarm!
...alarm!
...$ 
```

### 1.2 修改Makefile

```
  169 UPROGS=\
  170     _cat\
  171     _echo\
  172     _forktest\
  173     _grep\
  174     _init\
  175     _kill\
  176     _ln\
  177     _ls\
  178     _mkdir\
  179     _rm\
  180     _sh\
  181     _stressfs\
  182     _usertests\
  183     _wc\
  184     _zombie\
  185     _date\
++ 186     _alarmtest\

  253 EXTRA=\
  254     mkfs.c ulib.c user.h cat.c echo.c forktest.c grep.c kill.c\
-- 255     ln.c ls.c mkdir.c rm.c stressfs.c usertests.c wc.c zombie.c date.c
++ 255     ln.c ls.c mkdir.c rm.c stressfs.c usertests.c wc.c zombie.c date.c alarmtest.c\
  256     printf.c umalloc.c\
  257     README dot-bochsrc *.pl toc.* runoff runoff1 runoff.list\
  258     .gdbinit.tmpl gdbutil\
```

### 1.3 修改user.h

```
int uptime(void);
int date(struct rtcdate* r);
++ int alarm(int ticks, void (*handler)());
```

### 1.4 修改syscall.h和usys.S

```
//syscall.h
#define SYS_close  21
#define SYS_date   22
++ #define SYS_alarm  23

//usys.S
SYSCALL(uptime)
SYSCALL(date)
++ SYSCALL(alarm)
```

### 1.5 修改proc.h
```
++ typedef void (*ALARMHANDLER)();


// Per-process state
struct proc {
  uint sz;                     // Size of process memory (bytes)
  pde_t* pgdir;                // Page table
  char *kstack;                // Bottom of kernel stack for this process
  enum procstate state;        // Process state
  int pid;                     // Process ID
  struct proc *parent;         // Parent process
  struct trapframe *tf;        // Trap frame for current syscall
  struct context *context;     // swtch() here to run process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
++   int alarmticks;
++   int alarmtickscnt;
++   ALARMHANDLER alarmhandler;
};
```

### 1.6 修改syscall.c

```
++ int
++ sys_alarm(void)
++ {
++   int ticks;
++   void (*handler)();
++   if(argint(0, &ticks) < 0)
++     return -1;
++   if(argptr(1, (char**)&handler, 1) < 0)
++     return -1;
++   myproc()->alarmticks = ticks;
++   myproc()->alarmhandler = handler;
++   return 0;
++ }
```

### 1.7 修改trap.c

```
  case T_IRQ0 + IRQ_TIMER:
    if(cpuid() == 0){
      acquire(&tickslock);
      ticks++;
      wakeup(&ticks);
      release(&tickslock);
    }
    if(myproc() != 0 && (tf->cs & 3) == 3) {
      myproc()->alarmtickscnt++;
      if (myproc()->alarmtickscnt >= myproc()->alarmticks) {
        tf->esp -= 4;
        // eip压栈
        *(uint *)(tf->esp) = tf->eip;
        tf->eip = (uint)myproc()->alarmhandler;
        myproc()->alarmtickscnt = 0；
      }
    }
     
    lapiceoi();
    break;
```

### 1.8 执行结果

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image25.png?raw=true" width="70%">


## 2 个人遇到的难点
### 2.1 当一个进程的报警间隔到期时，如何让它执行handler程序？
> In your IRQ_TIMER code, when a process's alarm interval expires, you'll want to cause it to execute its handler. How can you do that?

答：如下代码所示，当alarmticketscnt大于设定的alarmticks时，就可以执行回调函数alarmhandler。

```
  case T_IRQ0 + IRQ_TIMER:
    if(cpuid() == 0){
      acquire(&tickslock);
      ticks++;
      wakeup(&ticks);
      release(&tickslock);
    }
    if(myproc() != 0 && (tf->cs & 3) == 3) {
      myproc()->alarmtickscnt++;
      if (myproc()->alarmtickscnt >= myproc()->alarmticks) {
        //这样调用不能够执行到回调函数
        //myproc()->alarmhandler();
        tf->esp -= 4;
        // eip压栈
        *(uint *)(tf->esp) = tf->eip;
        tf->eip = (uint)myproc()->alarmhandler;
        myproc()->alarmtickscnt = 0；
      }
    }
     
    lapiceoi();
    break;
```

`myproc()->alarmhandler();`内核态下这段代码无法跳转到回调函数的原因：回调函数是在用户态下定义的，其特权级因该为3，而内核态的特权级为0，由于安全性，特权级高的无法直接调用特权低的代码，因此这部分代码不起作用。

### 2.2当handler程序返回之后，如何让当前进程继续执行呢？
> You need to arrange things so that, when the handler returns, the process resumes executing where it left off. How can you do that?

答：因为handler函数是在ineterrupt的时候调用的，因此当handler返回，interrupt结束的时候，就会自动执行process。

tf->esp保存着的是用户态栈的栈顶地址，而用户态栈顶中保存着的是其下一条执行的语句。因此当我们将tf->eip替换成回调函数的地址之后，表示我们在中断结束之后，将会执行回调函数，而执行完回调函数之后，cpu会继续执行用户态栈顶保存的地址所对应的指令，因此只需要我们将用户态栈顶扩展4个字节，并将原先想要执行的指针放入到这4个字节中，就实现了从回调函数返回后再继续执行process的功能了。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image26.png?raw=true" width="70%">

