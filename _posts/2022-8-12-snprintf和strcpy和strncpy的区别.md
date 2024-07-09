---
layout: post
title: snprintf和strcpy和strncpy的区别
tags: [c++语法]
---

`snprintf`，`strcpy`，`strncpy`这几个函数的功能都是将原字符串拷贝到目的字符串中。但是在细节部分还是存在着一些细微的差别。主要参考`man`说明。


# `snprintf`
## 格式
```
int snprintf(char *str, size_t size, const char *format, ...);
```

## 说明
>**The functions snprintf() and vsnprintf() write at most size bytes (including the terminating null byte ('\0')) to str.**

即，`snprintf`会把源字符串拷贝到目的地址。如果源字符串的长度大于约束条件`size`，那么会拷贝`size - 1`个字符到目的地址，并将最后一个字符赋值为`'\0'`。

## 例子
```
#include <iostream>
#include <string.h>
using namespace std;
int main() {
    char buf[10] = {0};
    snprintf(buf, 10, "hello word");
    cout << buf << ", " << strlen(buf) << ", " << sizeof(buf) <<  endl;
    return 0;
}
```
输出结果如下：

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image3.png?raw=true" width="70%">

可以看出，目的内存`buf`只能存放10个字符，而`"hello word"`加上最后的`'\0'`，总共有11个字符，因此`snprintf`最多拷贝9个字符到`buf`中，并将最后一个字符设置为`'\0'`。

# `strcpy`
## 格式
```
 char *strcpy(char *dest, const char *src);
```

## 说明
>**The  strcpy()  function  copies the string pointed to by src, including the terminating null byte ('\0'), to the buffer pointed to by dest.  The strings may not overlap, and the destination string dest must be large enough to receive the copy.**

`strcpy`将`src`整个字符串（包括`“\0”`)拷贝到`dest`中。如果`src`字符串（包括`“\0”`)的长度大于`dest`的空间大小，会导致数组越界的`bug`。

## 例子
```
#include <iostream>
#include <string.h>
using namespace std;
int main() {
    char buf[8] = {0};
    char buf1[8] = {0};
    char buf2[8] = {0};
    //snprintf(buf, 10, "hello word");
    //strncpy(buf1,  "hello wrod_nice", 10);
    strcpy(buf1,  "hello word");
    //cout << buf << ", " << strlen(buf) << ", " << sizeof(buf) <<  endl;
    cout << buf1 << ", " << strlen(buf1) << ", " << sizeof(buf1) <<  endl;
    cout << buf2 << ", " << strlen(buf2) << ", " << sizeof(buf2) <<  endl;
    return 0;
}
```
输出结果如下：

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image4.png?raw=true" width="70%">

可以看出，原本拷贝到`buf1`中的数据，覆盖了`buf2`中的数据。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image5.png?raw=true" width="70%">

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image6.png?raw=true" width="70%">

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image7.png?raw=true" width="70%">




# `strncpy`

## 格式
```
char *strncpy(char *dest, const char *src, size_t n);
```
## 说明
>**The  strncpy()  function is similar, except that at most n bytes of src are copied.  Warning: If there is no null byte among the first n bytes of src, the string placed in dest will not be null-terminated. If the length of src is less than n, strncpy() writes additional null bytes to dest to ensure that a total  of  n  bytes are written.**

`strncpy`和`snprintf`的区别在于，如果`src`的`n`个字符没有包含`"\0"`，那么`strncpy`不会将`"\0"`拷贝到`dst`中。

## 例子
```
int main() {
    char buf[8] = {0};
    char buf1[8] = {0};
    char buf2[8] = {0};
    //snprintf(buf, 10, "hello word");
    strncpy(buf1,  "hello word", 8);
    //strcpy(buf1,  "hello word");
    //cout << buf << ", " << strlen(buf) << ", " << sizeof(buf) <<  endl;
    cout << buf1 << ", " << strlen(buf1) << ", " << sizeof(buf1) <<  endl;
    cout << buf2 << ", " << strlen(buf2) << ", " << sizeof(buf2) <<  endl;
    return 0;
}
```

输出如下图所示

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image8.png?raw=true" width="70%">

同样的，我们看一下`buf`，`buf1`，`buf2`的内容。`buf2`的内容并未被覆盖，但是`buf1`的最后一个字符是`o`而不是`"\0"`。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image9.png?raw=true" width="70%">

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image10.png?raw=true" width="70%">

