---
layout: post
title: Mit6.828 HW6：Threads and Locking
tags: [Mit6.828]
---

基于Mit6.828 HW6：Threads and Locking的题解。

## 参考资料
[MIT6.828_HW6_Threads and Locking](https://blog.csdn.net/Small_Pond/article/details/92838852)


## `ph.c`程序
这个实验给了我们一个实例`ph.c`，这个程序会创建一个线程，`thread`，如下所示：
```
static void *
thread(void *xa)
{
  long n = (long) xa;
  int i;
  int b = NKEYS/nthread;
  int k = 0;
  double t1, t0;
  //  printf("b = %d\n", b);
  t0 = now();
  for (i = 0; i < b; i++) {
    // printf("%d: put %d\n", n, b*n+i);
    put(keys[b*n + i], n);
  }
  t1 = now();
  printf("%ld: put time = %f\n", n, t1-t0);
  // Should use pthread_barrier, but MacOS doesn't support it ...
  __sync_fetch_and_add(&done, 1);
  while (done < nthread) ;
  t0 = now();
  for (i = 0; i < NKEYS; i++) {
    struct entry *e = get(keys[i]);
    if (e == 0) k++;
  }
  t1 = now();
  printf("%ld: get time = %f\n", n, t1-t0);
  printf("%ld: %d keys missing\n", n, k);
  return NULL;
}
```
这个线程先是通过`put`操作，将数据放入到`table`中，`table`是一个数组，大小为`NBUCKET`，每个元素为一个链表。`put`操作的时候每个线程会`insert`同样数量的数据，例如有2个线程，那么线程0会将keys[0]到keys[49999]的值`insert`到`table`中，线程·会将keys[50000]到keys[99999]`insert`到`table`中。通过`while(down < nthread)`语句，保证所有线程先进行了`put`操作之后，才会进行`get`操作。而`get`操作，则是遍历`keys[100000]`数组，找到每个`keys[i]`对应的value，如果找不到，那么计数`k++`。


## `missing keys`的原因
因为在多线程的时候，由于`insert`并不是原子操作，导致两个线程冲突，有部分数据并没有`insert`到`table`中。因此可以对`insert`操作加锁。
```
static
void put(int key, int value)
{
  int i = key % NBUCKET;
  pthread_mutex_lock(&lock);  // acquire lock
  insert(key, value, &table[i], table[i]);
  pthread_mutex_unlock(&lock);  // release lock
}
```
**加锁之后，put的时间会变长**
* 未加锁时

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image41.png?raw=true" width="70%">

* 加锁之后

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/MIT6.828/Image42.png?raw=true" width="70%">



## 注意
下面这种加锁方式会存在问题，原因在于`put`函数中，其`&table[i]`和`table[i]`的地址会因为其他线程插入数据导致这两部分的地址失效。
```
static void
insert(int key, int value, struct entry **p, struct entry *n)
{
  pthread_mutex_lock(&lock);  // acquire lock
  struct entry *e = malloc(sizeof(struct entry));
  e->key = key;
  e->value = value;
  e->next = n;
  *p = e;
  pthread_mutex_unlock(&lock);  // release lock
}
static
void put(int key, int value)
{
  int i = key % NBUCKET;
  insert(key, value, &table[i], table[i]);
}
```
