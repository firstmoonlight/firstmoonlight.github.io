---
layout: post
title: Mit6.828 HW7:xv6 locking
tags: [Mit6.828]
---

HW7:xv6 locking的简要说明。

## DeadLock
```
  struct spinlock lk;
  initlock(&lk, "test lock");
  acquire(&lk);
  acquire(&lk);
```
要求我们用一句话说明，上述的代码出现的问题。

</br>答：由于在第一次`acquire`之后，并未`release`就进行了`acquire`操作，导致第二次的`acuqire`会永远获取不到锁，即死锁。


## Interrupt in ide.c
在xv6的代码中，`acquire`函数会通过`cli`指令，屏蔽中断；`release`函数通过`sti`指令，将中断打开。但是，如果在`ide.c`的`iderw`函数中，我们在`acquire`函数的后面加上`sti`指令，而在`release`函数的前面加上`cli`指令，并重新编译内核，那么xv6就有可能boot失败，请找出原因。

答：
1. `panic`函数
</br>如下面的代码显示，`getcallerpcs`函数会将栈中存储的`eip`信息存放到`pcs`数组中，然后打印`pcs`数组。

```
void
panic(char *s)
{
  int i;
  uint pcs[10];
  cli();
  cons.locking = 0;
  // use lapiccpunum so that we can call panic from mycpu()
  cprintf("lapicid %d: panic: ", lapicid());
  cprintf(s);
  cprintf("\n");
  getcallerpcs(&s, pcs);
  for(i=0; i<10; i++)
    cprintf(" %p", pcs[i]);
  panicked = 1; // freeze other CPU
  for(;;)
    ;
}


// Record the current call stack in pcs[] by following the %ebp chain.
void
getcallerpcs(void *v, uint pcs[])
{
  uint *ebp;
  int i;
  ebp = (uint*)v - 2;
  for(i = 0; i < 10; i++){
    if(ebp == 0 || ebp < (uint*)KERNBASE || ebp == (uint*)0xffffffff)
      break;
    pcs[i] = ebp[1];     // saved %eip
    ebp = (uint*)ebp[0]; // saved %ebp
  }
  for(; i < 10; i++)
    pcs[i] = 0;
}
```

2. 从下图中可以看出，xv6在`sched locks`的地方被`panic`。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image43.png?raw=true" width="70%">

* 在`kernel.asm`中，依次查找`eip`中的值，就可以查出调用栈了，调用栈为：</br>
`trapasm.S: trapret` -> `proc.c: forkret` ->` fs.c: iinit` -> `fs.c: readsb` ->` bio.c: bread `->` ide.c: iderw` -> `trapasm.S: alltraps` -> `trap.c: trap` -> `proc.c: yield` ->` proc.c: sched`
* 从调用栈中可以看出，在`ide.c:iderw`函数中，调用到了`acquire(&idelock)`之后，由于`sti`指令被加入，所以中断又重新可以被触发，因此触发了`trapams.S: alltraps`，此时需要当前进程放弃CPU，因此调用到了`yield`函数，判断`if(mycpu()->ncli != 1)`的时候出错。


## interrupts in file.c
* 将上述的`sti()`和`cli()`去除，重新编译`kernel`，并保证`kernel`能够重新正常运行。
* 现在和上述的实验一样
  * 在获得`file_table_lock`的时候，通过`sti`，不再屏蔽中断。(`file_table_lock`用来保护文件描述符表，当我们在打开和关闭文件的时候，会修改这个文件描述符表。)
  * 在`file.c`文件中的`filealloc()`中，`acquire()`函数被调用的后面加入`sti()`，然后在每一处调用到`release()`的前面，加入`cli()`。
* 重新编译`kernel`，然后在`qemu`中将它启动，`kernel`并不会`panic`。

* 解释为什么我们对`file_table_lock`和`ide_lock`做同样的操作，但是却出现了不同的表现: 因为从`kernel`对`file_table_lock`和`ide_lock`的使用情况来看，在` xv6 `中没有中断处理函数会争` ftable.lock` 保护的资源。自然也就不会` acquire(&ftable.lock)`。


## xv6 lock implementation
>**Why does release() clear lk->pcs[0] and lk->cpu before clearing lk->locked? Why not wait until after?**

答：因为在`acquire`的时候，会调用`getcallerpcs`来记录debug信息，即lk->pcs[0]和lk->cpu。因此当线程1在`release`的时候，如果lk->locked先被释放，会导致线程2通过`getcallerpcs`来修改lk->pcs[0]和lk->cpu，同时线程1自己也在修改lk->pcs[0]和lk->cpu，形成竞争条件。
