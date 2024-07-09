最近在看《effective modern C++》，到条款二十五，看到了C++返回值拷贝的说明，故做一下总结。并参考了网上的资料。[深入理解C++中的RVO](https://zhuanlan.zhihu.com/p/341680064)

## 概述
我将以下面的类为例子，说明C++在RVO的情况下以及关闭RVO情况下，其函数返回值是如何返回给调用者的，并给出其汇编代码的说明。环境是Ubuntu Linux，编译器为GCC7.5.0。
```
class Obj {
public:
    Obj() { // 构造函数
    std::cout << "in Obj() " << " " << this << std::endl;
    }
    Obj(int n) {
    std::cout << "in Obj(int) " << " " << this << std::endl;
    }
    Obj(const Obj &obj) { // 拷贝构造函数
    std::cout << "in Obj(const Obj &obj) " << &obj << " " << this << std::endl;
    }
    Obj(const Obj &&obj) { // 移动构造函数
    std::cout << "in Obj(const Obj &&obj) " << &obj << " " << this << std::endl;
    }
    Obj &operator=(const Obj &obj) { // 赋值构造函数
    std::cout << "in operator=(const Obj &obj)" << std::endl;
    return *this;
    }
    Obj &operator=(const Obj &&obj) { // 移动赋值构造函数
    std::cout << "in operator=(const Obj &&obj)" << std::endl;
    return *this;
    }
    ~Obj() { // 析构函数
    std::cout << "in ~Obj() " << this << std::endl;
    }
private:
    int n;
};
```

## 关闭RVO
编译命令加上`-fno-elide-constructors`表示关闭RVO 优化开关。定义函数`fun1`，并调用。
```
//test28_RVO.cpp
Obj fun1() {
  Obj obj;
  // do sth;
  return obj;
}

int main() {
    Obj a = fun1();
    return 0;
}
```
编译：`g++ -fno-elide-constructors -g -o test28_RVO test28_RVO.cpp`。
结果打印：如下图所示。
* 定义拷贝构造函数以及移动构造函数的情况下：

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image11.png?raw=true" width="70%">

* 定义拷贝构造函数而未定义移动构造函数的情况下：

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image12.png?raw=true" width="70%">

结果分析：从打印可以分析出，在关闭RVO优化的情况下，C++返回一个对象会调用拷贝构造函数（或者移动构造函数）2次。分析其汇编代码，可知，在`main`函数调用函数`fun1()`的时候，调用者会将一个栈的地址传递给`fun1()`。在函数`fun1()`内，当调用构造函数创建`obj`之后，会将`obj`拷贝到这个栈的地址，作为一个临时对象。函数`fun()`调用结束之后，会再调用一次拷贝构造函数，临时对象拷贝到接收者即`a`中。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image13.png?raw=true" width="70%">

## 在RVO下
代码和上面的相同。
```
//test28_RVO.cpp
Obj fun1() {
  Obj obj;
  // do sth;
  return obj;
}

int main() {
    Obj a = fun1();
    return 0;
}
```
编译：`g++  -g -o test28_RVO test28_RVO.cpp`。
结果打印：如下图所示。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image14.png?raw=true" width="70%">

结果分析：在启用RVO优化的情况下，只调用了一次构造函数，拷贝构造函数没有被调用。
从汇编代码结果可以看出，`main`函数将地址`-0x1c(%rbp)`传递给函数`fun1()`，之后，`fun1()`在调用`Obj`构造函数的时候，直接在这个地址上进行构造，免去了临时对象的创建以及拷贝。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image15.png?raw=true" width="70%">

## RVO的条件
《modern effective c++》中指出，在开启RVO之后，返回局部对象只有在满足以下两个条件的情况下，编译器才会进行RVO优化。
（1）局部对象与函数返回值的类型相同。
（2）局部对象就是要返回的东西。函数形参不满足要求。
注意：当函数不同控制路径返回不同局部变量时，不会进行拷贝消除的操作，因为其不知道要返回哪个对象。


## 返回值为std::move的情况
代码如下：
```
Obj fun2() {
  Obj obj;
  // do sth;
  return std::move(obj);
}

int main() {
    Obj a = fun2();
    return 0;
}
```
编译：`g++  -g -o test28_RVO test28_RVO.cpp`。
结果打印：如下图所示
* 定义拷贝构造函数以及移动构造函数的情况下：

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image16.png?raw=true" width="70%">

* 定义拷贝构造函数，未移动构造函数的情况下(之所以编译器未报错，是因为**const修饰的左值形参能够接收一个右值实参**)：

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image17.png?raw=true" width="70%">

结果分析：从汇编代码中可以看出，main将变量`a`的地址传递给`fun2()`。`fun2()`创建`Obj obj`，然后调用移动构造函数将`obj`移动到`a`中。

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/2024_7_8/Image18.png?raw=true" width="70%">

