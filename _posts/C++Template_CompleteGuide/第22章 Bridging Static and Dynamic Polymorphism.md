---
layout: post
title: 第22章 Bridging Static and Dynamic Polymorphism
tags: [c++语法]
---

本章介绍如何在 C++ 中桥接静态和动态多态性。


## 22.1 Function Objects, Pointers, and std::function<>
1、 下面这段代码虽然可以实现接收一个可调用对象(function pointer, functor, lambda)的功能，但是会增加代码的大小。
```c++
template<typename F>
void forUpTo(int n, F f)
{
    for (int i = 0; i != n; ++i)
    {
        f(i); // call passed function f for i
    }
}
```

2、而下面这段代码，只能接收function pointer，对于lambda，functor不起作用
```c++
void forUpTo(int n, void (*f)(int))
{
    for (int i = 0; i != n; ++i)
    {
         f(i); // call passed function f for i
    }
}
```
3、因此标准库引入了`std::function<>`，它可以接收function pointer, functor, lambda，而不增加代码的大小。
```c++
#include <functional>
void forUpTo(int n, std::function<void(int)> f)
{
    for (int i = 0; i != n; ++i)
    {
        f(i); // call passed function f for i
    }
}
```
> std::function<> 的模板参数是一个函数类型，它描述了函数对象将接收的参数类型以及它应该产生的返回类型，就像函数指针描述参数和结果类型一样。

> 它使用一种称为类型擦除的技术来实现这一点，这种技术弥合了静态和动态多态性之间的差距。


## 22.2 Generalized Function Pointers
本章将会实现`FunctionPtr`框架，实现类似于`std::function`的功能。
* 它可用于调用函数，调用者无需了解函数本身。
* 它可以被复制、移动和赋值。
* 它可以从另一个函数（具有兼容签名）初始化或赋值。
* 它具有“null”状态，表示没有函数与其绑定。

```c++
// primary template:
template<typename Signature>
class FunctionPtr;
// partial specialization:
template<typename R, typename... Args>
class FunctionPtr<R(Args...)>
{
private:
    FunctorBridge<R, Args...>* bridge;
public:
    // constructors:
    FunctionPtr() : bridge(nullptr) {
    }
    FunctionPtr(FunctionPtr const& other); // see functionptr-cpinv.hpp
    FunctionPtr(FunctionPtr& other)
    : FunctionPtr(static_cast<FunctionPtr const&>(other)) {
    }
    FunctionPtr(FunctionPtr&& other) : bridge(other.bridge) {
        other.bridge = nullptr;
    }
// construction from arbitrary function objects:
    template<typename F> FunctionPtr(F&& f); // see functionptr-init.hpp
    // assignment operators:
    FunctionPtr& operator=(FunctionPtr const& other) {
        FunctionPtr tmp(other);
        swap(*this, tmp);
        return *this;
    }
    FunctionPtr& operator=(FunctionPtr&& other) {
        delete bridge;
        bridge = other.bridge;
        other.bridge = nullptr;
        return *this;
    }
    // construction and assignment from arbitrary function objects:
    template<typename F> FunctionPtr& operator=(F&& f) {
        FunctionPtr tmp(std::forward<F>(f));
        swap(*this, tmp);
        return *this;
    }
    // destructor:
    ~FunctionPtr() {
        delete bridge;
    }
    friend void swap(FunctionPtr& fp1, FunctionPtr& fp2) {
        std::swap(fp1.bridge, fp2.bridge);
    }
    explicit operator bool() const {
        return bridge == nullptr;
    }
    // invocation:
    R operator()(Args... args) const; // see functionptr-cpinv.hpp

};
```
该实现包含一个非静态成员变量 bridge，它将负责存储和操作存储的函数对象。该指针的所有权与 FunctionPtr 对象绑定，因此提供的大部分实现仅管理该指针。

## 22.3 Bridge Interface
本节实现了Bridge的接口。
FunctorBridge 类模板负责底层函数对象的所有权和操作。它被实现为一个抽象基类，构成了 FunctionPtr 动态多态性的基础。注意clone和invoke函数是const标识的。
```c++
template<typename R, typename... Args>
class FunctorBridge
{
public:
    virtual ~FunctorBridge() {
    }
    virtual FunctorBridge* clone() const = 0;
    virtual R invoke(Args... args) const = 0;
};
```

通过调用FunctorBridge，FunctionPtr实现了相应的功能。
```c++
template<typename R, typename... Args>
FunctionPtr<R(Args...)>::FunctionPtr(FunctionPtr const& other)
: bridge(nullptr)
{
    if (other.bridge) {
    bridge = other.bridge->clone();
}
}
template<typename R, typename... Args>
R FunctionPtr<R(Args...)>::operator()(Args... args) const
{
    return bridge->invoke(std::forward<Args>(args)...);
}
```

## 22.4 Type Erasure
通过模板技术来为任意类型的函数对象提供多态支持

1. FunctorBridge 是一个抽象类：
* 它定义了一组虚函数接口（例如调用、拷贝、析构等），但不提供具体实现。
* 这样做的目的是提供一种“类型擦除”机制：允许我们以统一方式存储和操作各种函数对象。
2. 派生类负责实现具体行为：
* 每个函数对象（可能是函数指针、lambda、函数对象等）都需要一个具体的实现类来包装。
* 因为函数对象的类型是无穷无尽的，所以理论上需要无穷多个派生类来支持它们。
3. 如何解决“派生类数量无穷”的问题？
* 使用模板派生类（parameterized derived class），即用一个类模板 SpecificFunctorBridge<F> 来包装任意类型的函数对象 F。
* 模板实例化的机制可以在编译期根据函数对象的类型自动生成所需的派生类。

```c++
template<typename Functor, typename R, typename... Args>
class SpecificFunctorBridge : public FunctorBridge<R, Args...> {
    Functor functor;
public:
    template<typename FunctorFwd>
    SpecificFunctorBridge(FunctorFwd&& functor)
         : functor(std::forward<FunctorFwd>(functor)) {
    }
    virtual SpecificFunctorBridge* clone() const override {
        return new SpecificFunctorBridge(functor);
    }
    virtual R invoke(Args... args) const override {
        return functor(std::forward<Args>(args)...);
    }
};
```
**type erase的本质：**
```c++
template<typename R, typename... Args>
template<typename F>
FunctionPtr<R(Args...)>::FunctionPtr(F&& f)
    : bridge(nullptr)
{
    using Functor = std::decay_t<F>;
    using Bridge = SpecificFunctorBridge<Functor, R, Args...>;
    bridge = new Bridge(std::forward<F>(f));
}
```

* SpecificFunctorBridge 是一个模板派生类，持有真正的可调用对象 f。
* `bridge` 是一个` FunctorBridge<R, Args...>* ，`指向的是基类指针。
* 分配了 Bridge 类型（含实际对象）的实例，并将它指针存到 bridge。

在这个过程中，从 `Bridge*` 转为 `FunctorBridge<R, Args...>*`，即：
```cpp
Bridge* → FunctorBridge<R, Args...>* // 派生类指针 → 基类指针 
```
这样一来，<u>除了基类中定义的虚函数接口，其他关于 F 的具体类型信息完全丢失了</u>。这是类型擦除的核心目的：保留行为（通过虚函数），抹除类型（通过指针转换）。


✅ **为什么使用 std::decay_t<F>？**

std::decay_t<F> 主要是为了得到适合存储的类型。考虑下面这些例子：
```cpp
FunctionPtr<void()> f1([]{});
FunctionPtr<void()> f2(std::ref(some_functor));
FunctionPtr<void()> f3(&some_function);
```
如果我们直接使用 F，可能会得到这些情况：
* 匿名 lambda，可能是 const 的引用
* std::reference_wrapper<Callable>&
* void (&)()（函数引用）

这些类型不适合直接作为类成员存储，所以 std::decay_t<F> 会：
| 原始类型 | decay后的类型 |
| --- | --- |
| const T& | T |
| T&  | T |
| T&&  | T |
|void(&)()（函数引用） | `void(*)()（函数指针）` |

>✅ 所以 std::decay_t<F> 让我们在 SpecificFunctorBridge 中更安全地存储 f，不管它是 lambda、函数引用、成员函数指针，还是包装类。


## 22.5 Optional Bridging
本节为FunctionPtr增加一个功能：测试两个FunctionPtr 是否调用了同一个function。
```cpp
// 为FunctorBridge增加虚函数
virtual bool equals(FunctorBridge const* fb) const = 0;

// SpecificFunctorBridge实现equals功能
virtual bool equals(FunctorBridge<R, Args...> const* fb) const override {
    if (auto specFb = dynamic_cast<SpecificFunctorBridge const*>(fb)) {
        return functor == specFb->functor;
    }
    // functors with different types are never equal:
    return false;
}

// FunctionPtr实现operator==和operator!=接口
friend bool operator==(FunctionPtr const& f1, FunctionPtr const& f2) {
    if (!f1 || !f2) {
        return !f1 && !f2;
    }
    return f1.bridge->equals(f2.bridge);
}
friend bool operator!=(FunctionPtr const& f1, FunctionPtr const& f2) {
    return !(f1 == f2);
}

```

**存在问题**：如果 FunctionPtr 被赋值或初始化为一个没有合适运算符 == 的函数对象（例如，包括 lambda 表达式），程序将编译失败。为什么呢？
> 运算符 == 的问题源于类型擦除：因为一旦 FunctionPtr 被赋值或初始化，我们实际上就丢失了函数对象的类型，所以我们需要在赋值或初始化完成之前捕获所有需要了解的类型信息。这些信息包括对函数对象的运算符 == 的调用，因为我们无法确定何时会需要它


**通过SFINAE来判断operator==是否是有效的**
```cpp
#include <utility> // for declval()
#include <type_traits> // for true_type and false_type
template<typename T>
class IsEqualityComparable
{
private:
    // test convertibility of == and ! == to bool:
    static void* conv(bool); // to check convertibility to bool
    template<typename U>
    static std::true_type test(decltype(conv(std::declval<U const&>() ==  
                 std::declval<U const&>())), 
            decltype(conv(!(std::declval<U const&>() ==
                std::declval<U const&>())))
);
    // fallback:
    template<typename U>
    static std::false_type test(...);
public:
    static constexpr bool value = decltype(test<T>(nullptr,nullptr))::value;
};
```

**将IsEqualityComparable应用于FunctionPtr**
```cpp
#include <exception>
#include "isequalitycomparable.hpp"
template<typename T,
            bool EqComparable = IsEqualityComparable<T>::value>
struct TryEquals
{
    static bool equals(T const& x1, T const& x2) {
        return x1 == x2;
    }
};
class NotEqualityComparable : public std::exception
{
};
template<typename T>
struct TryEquals<T, false>
{
    static bool equals(T const& x1, T const& x2) {
        throw NotEqualityComparable();
    }
};

// SpecificFunctionBridge的equals接口
virtual bool equals(FunctorBridge<R, Args...> const* fb) const override {
    if (auto specFb = dynamic_cast<SpecificFunctorBridge const*>(fb)) {
        return TryEquals<Functor>::equals(functor, specFb->functor);
    }
    // functors with different types are never equal:
    return false;
}
```

## 22.6 Performance Considerations
> 类型擦除兼具静态多态性和动态多态性的部分优势，但并非全部。具体而言，使用类型擦除生成的代码的性能与动态多态性的性能更为接近，因为两者都通过虚函数进行动态调度。
因此，静态多态性的一些传统优势（例如编译器内联调用的能力）可能会丧失。
这种性能损失是否会被察觉取决于具体应用，但通常可以通过比较被调用函数的执行工作量与虚函数调用成本的比率来判断：如果两者接近（例如，使用 FunctionPtr 简单地将两个整数相加），则类型擦除的执行速度可能远慢于静态多态版本。另一方面，如果函数调用执行了大量工作（例如查询数据库、对容器进行排序或更新用户界面），则类型擦除的开销不太可能被衡量。
