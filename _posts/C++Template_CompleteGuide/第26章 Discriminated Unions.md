---
layout: post
title: 第22章 Bridging Static and Dynamic Polymorphism
tags: [c++语法]
---

在本章中，我们将开发一个类模板 Variant，它动态存储一组给定的可能值类型中的一种值，类似于 C++ 17 标准库的 std::variant<>。

Variant 是一个可区分联合体，这意味着 Variant 知道其可能的值类型中哪些当前处于活动状态，从而提供比等效的 C++ 联合体更好的类型安全性。

```c++
#include "variant.hpp"
#include <iostream>
#include <string>

int main() {
    Variant<int, double, std::string> field(17);
    if (field.is<int>()) {
        std::cout << "Field stores the integer "
             << field.get<int>() << '\n';
    }
    field = 42; // assign value of same type
    field = "hello"; // assign value of different type
    std::cout << "Field now stores the string '" << field.get<std::string>() << "'\n";
}

```

## 26.1 Storage
### 1、利用tuple
将所有的类型都存储到tuple中，通过`discriminator`来获取得到对应的value，即当`discriminator`为0时，通过`get<0>(storage)`来获取。

**但是tuple由于需要存储所有的值，导致其空间利用率很低**
```cpp
template<typename... Types>
class Variant {
public:
    Tuple<Types...> storage;
    unsigned char discriminator;
};
```
### 2、利用union
通过union来存储，和tuple的实现类似，只是tuple用的是class而variant用的是union。union的实现可以更好的利用空间。

**但是union不能够被继承**，因此这种方法的实现功能有限。
```c++
template<typename... Types>
union VariantStorage;

template<typename Head, typename... Tail>
union VariantStorage<Head, Tail...> {
    Head head;
    VariantStorage<Tail...> tail;
};

template<>
union VariantStorage<> {

};
```

## 3、使用char数组
> a character array large enough to hold any of the types and with suitable alignment for any of the types, which we use as a buffer to store the active value

```cpp
template<typename... Types>
class VariantStorage {
    using LargestT = LargestType<Typelist<Types...>>;
    alignas(Types...) unsigned char buffer[sizeof(LargestT)];
    unsigned char discriminator = 0;
public:
    unsigned char getDiscriminator() const { return discriminator; }
    void setDiscriminator(unsigned char d) { discriminator = d; }
    void* getRawBuffer() { return buffer; }
    const void* getRawBuffer() const { return buffer; }

    // std::launder() is sufficient for now to know that it returns its argument unmodified
    template<typename T>
    T* getBufferAs() { return std::launder(reinterpret_cast<T*>(buffer)); }
    template<typename T>
    T const* getBufferAs() const {
        return std::launder(reinterpret_cast<T const*>(buffer));
    }
};
```

## 26.2 Design

### 1、VariantChoice
使用继承的机制来操作每个Type。

```cpp
#include "findindexof.hpp"

template<typename T, typename... Types>
class VariantChoice {
    using Derived = Variant<Types...>;
    Derived& getDerived() { return *static_cast<Derived*>(this); }
    Derived const& getDerived() const {
        return *static_cast<Derived const*>(this);
    }

protected:
    // compute the discriminator to be used for this type
    constexpr static unsigned Discriminator =
        FindIndexOfT<Typelist<Types...>, T>::value + 1;
        
public:
    VariantChoice() { }
    VariantChoice(T const& value); // see variantchoiceinit.hpp
    VariantChoice(T&& value); // see variantchoiceinit.hpp
    bool destroy(); // see variantchoicedestroy.hpp
    Derived& operator= (T const& value); // see variantchoiceassign.hpp
    Derived& operator= (T&& value); // see variantchoiceassign.hpp
};
```
* `using Derived = Variant<Types...>;` 因为`Types`包含了所有的类型，因此通过`Types`可以重新构造了子类

* `FindIndexOfT<Typelist<Types...>, T>::value + 1;`可以找到T对应的id。如下所示
```cpp
template<typename List, typename T, unsigned N = 0, bool Empty = IsEmpty<List>::value>
struct FindIndexOfT;

// recursive case:
template<typename List, typename T, unsigned N>
struct FindIndexOfT<List, T, N, false> :
        public IfThenElse<std::is_same<Front<List>, T>::value,
                                std::integral_constant<unsigned, N>,
                                FindIndexOfT<PopFront<List>, T, N+1>
                                >
{
};

// basis case:
template<typename List, typename T, unsigned N>
struct FindIndexOfT<List, T, N, true>
{
};
```

### 2、Variant架构
```cpp
template<typename... Types>
class Variant : 
        private VariantStorage<Types...>,
        private VariantChoice<Types, Types...>...
{
    template<typename T, typename... OtherTypes>
    friend class VariantChoice; // enable CRTP
    ...
};
```
* Variant只有一个`VariantStorage<Types...>`父类
* Variant却有多个`VariantChoice<Types, Types...>...`父类 
```cpp
// Variant<int, double, std::string> 有如下三个base class
VariantChoice<int, int, double, std::string>,
VariantChoice<double, int, double, std::string>,
VariantChoice<std::string, int, double, std::string>
// The discriminator values for these three base classes will be 1, 2, and 3, respectively
// discriminator 0预留用来特殊处理
```

### 3、Variant的完整实现
```cpp
template<typename... Types>
class Variant : private VariantStorage<Types...>,
private VariantChoice<Types, Types...>...
{
	template<typename T, typename... OtherTypes>
	friend class VariantChoice;
    
public:
	template<typename T> bool is() const; // see variantis.hpp
	template<typename T> T& get() &; // see variantget.hpp
	template<typename T> T const& get() const&; // see variantget.hpp
	template<typename T> T&& get() &&; // see variantget.hpp
	
    // see variantvisit.hpp:
	template<typename R = ComputedResultType, typename Visitor>
	    VisitResult<R, Visitor, Types&...> visit(Visitor&& vis) &;
	template<typename R = ComputedResultType, typename Visitor>
	    VisitResult<R, Visitor, Types const&...> visit(Visitor&& vis) const&;
	template<typename R = ComputedResultType, typename Visitor>
	    VisitResult<R, Visitor, Types&&...> visit(Visitor&& vis) &&;
	
    using VariantChoice<Types, Types...>::VariantChoice...;
	Variant(); // see variantdefaultctor.hpp
	Variant(Variant const& source); // see variantcopyctor.hpp
	Variant(Variant&& source); // see variantmovector.hpp
	template<typename... SourceTypes>
	    Variant(Variant<SourceTypes...> const& source); // variantcopyctortmpl.hpp
	template<typename... SourceTypes>
	    Variant(Variant<SourceTypes...>&& source);
        
	using VariantChoice<Types, Types...>::operator=...;
	Variant& operator= (Variant const& source); // see variantcopyassign.hpp
	Variant& operator= (Variant&& source);
	    template<typename... SourceTypes>
	Variant& operator= (Variant<SourceTypes...> const& source);
	    template<typename... SourceTypes>
	Variant& operator= (Variant<SourceTypes...>&& source);
	
    bool empty() const;
	~Variant() { destroy(); }
	void destroy(); // see variantdestroy.hpp
};
```


## 26.3 Value Query and Extraction

### 1、询问其当前活跃值是否为特定类型T
如下定义的is()成员函数用于判断variant当前是否存储了T类型的值。
```cpp
template<typename... Types>
template<typename T>
bool Variant<Types...>::is() const
{
    return this->getDiscriminator() ==
            VariantChoice<T, Types...>::Discriminator;
}

// v.is<int>() 判断v的active value是不是int类型
```
注：如果要查找的类型（T）不在类型列表中，`VariantChoice`将无法实例化，因为`FindIndexOfT`不会包含`value`成员，从而导致`is<T>()`函数编译失败。

### 2、访问active的value
```cpp
#include <exception>
class EmptyVariant : public std::exception {
};

template<typename... Types>
template<typename T>
T& Variant<Types...>::get() & {
	if (empty()) {
		throw EmptyVariant();
	}
	assert(is<T>());
	return *this->template getBufferAs<T>();
}
```
如果get不存在的type，那么会抛出`EmptyVariant exception.`


## 26.4 Element Initialization, Assignment and Destruction

`VariantChoice`的`initialization`, `assignment` 和 `destruction`等操作。

### 26.4.1 Initialization
* 讨论用` variant` 所存储的某个类型的值来初始化它。
例如用一个 `double` 值来初始化 `Variant<int, double, string>`。

```cpp
#include <utility> // for std::move()

template<typename T, typename... Types>
VariantChoice<T, Types...>::VariantChoice(T const& value) {
	// place value in buffer and set type discriminator:
	new(getDerived().getRawBuffer()) T(value);
	getDerived().setDiscriminator(Discriminator);
}

template<typename T, typename... Types>
VariantChoice<T, Types...>::VariantChoice(T&& value) {
	// place moved value in buffer and set type discriminator:
	new(getDerived().getRawBuffer()) T(std::move(value));
	getDerived().setDiscriminator(Discriminator);
}

```

* 通过引入 using 声明，将 VariantChoice 的构造函数继承到 Variant 中：
```cpp
using VariantChoice<Types, Types...>::VariantChoice...;
```

* 这个 using 声明会为 Types 中的每个类型 T 生成拷贝或移动构造函数。以 Variant<int, double, string> 为例，最终生成的构造函数等效于：
```cpp
Variant(int const&);    // int 类型的常量引用构造函数
Variant(int&&);         // int 类型的移动构造函数  
Variant(double const&); // double 类型的常量引用构造函数
Variant(double&&);      // double 类型的移动构造函数
Variant(string const&); // string 类型的常量引用构造函数  
Variant(string&&);      // string 类型的移动构造函数
```

### 26.4.2 Destruction
直接析构对应的buffer。
```cpp
template<typename T, typename... Types>
bool VariantChoice<T, Types...>::destroy() {
	if (getDerived().getDiscriminator() == Discriminator) {
	// if type matches, call placement delete:
		getDerived().template getBufferAs<T>()->~T();
		return true;
	}
	return false;
}
```

* VariantChoice::destroy() 操作仅在Discriminator匹配时有效。
* Variant::destroy() 会调用其所有基类的 VariantChoice::destroy()，因为
希望无条件销毁 variant 中存储的值
* 通过设置Discriminator为0，表明当前的Variant为空
```cpp
template<typename... Types>
void Variant<Types...>::destroy() {
    // call destroy() on each VariantChoice base class; at most one will succeed:
    bool results[] = {
        VariantChoice<Types, Types...>::destroy()...
    };
    // indicate that the variant does not store a value
    this->setDiscriminator(0);
}

// c++17中可以将results优化掉，没必要保留这样的局部变量
// call destroy() on each VariantChoice base class; at most one will succeed:
// (VariantChoice<Types, Types...>::destroy() , ...);
```

### 26.4.3 Assignment
```cpp
// 如果赋值的类型与` variant` 所存储的类型相同，那么我们直接将这个值copy或者move到buffer中
// 否则先destroy buffer中的值，然后通过place new初始化
template<typename T, typename... Types>
auto VariantChoice<T, Types...>::operator= (T const& value) -> Derived& {
	if (getDerived().getDiscriminator() == Discriminator) {
		// assign new value of same type:
		*getDerived().template getBufferAs<T>() = value;
	}
	else {
		// assign new value of different type:
		getDerived().destroy(); // try destroy() for all types
		new(getDerived().getRawBuffer()) T(value); // place new value
		getDerived().setDiscriminator(Discriminator);
	}
	return getDerived();
}

template<typename T, typename... Types>
auto VariantChoice<T, Types...>::operator= (T&& value) -> Derived& {
	if (getDerived().getDiscriminator() == Discriminator) {
		// assign new value of same type:
		*getDerived().template getBufferAs<T>() = std::move(value);
	}
	else {
		// assign new value of different type:
		getDerived().destroy(); // try destroy() for all types
		new(getDerived().getRawBuffer()) T(std::move(value)); // place new value
		getDerived().setDiscriminator(Discriminator);
	}
	return getDerived();
}
```

* 每个 VariantChoice 都提供了一个赋值运算符，用于将其存储值类型的值拷贝（或移动）到 variant 的存储空间中。
* Variant 通过以下 using 声明继承这些赋值运算符：
```cpp
using VariantChoice<Types, Types...>::operator=...;
```
* 处理Self-assignment
`v = v.get<T>() ` 
当发生Self-assignment时，意味着discriminator必然匹配，因此这类情况会直接调用类型T的赋值运算符，而无需走这个两步流程.

* 处理 Exceptions
如果现有值的销毁已完成，但新值的初始化抛出异常，variant 将处于什么状态？在我们的实现中，Variant::destroy() 会将判别器(discriminator)重置为0。在正常情况下，初始化完成后判别器会被正确设置。而当新值初始化过程中发生异常时，判别器将保持为0，表示该variant当前未存储任何值。在我们的设计中，这是产生无值variant的唯一途径。

##### **处理 std::launder()**
C++ 编译器基于静态类型和表达式的语义进行优化。如果你先析构一个对象，再在相同内存地址上构造另一个对象，编译器不一定意识到旧对象已被销毁，新对象是另一个不同类型。这可能导致：
* 编译器认为旧对象还“活着”，并缓存其某些成员值；
* 优化器使用了已失效的值；
* 程序行为未定义 (UB)，即使代码看起来没问题

**示例**
```cpp
struct A { const int x; };
struct B { const int y; };

alignas(std::max(alignof(A), alignof(B))) char buffer[sizeof(B)];
A* a = new (buffer) A{42};   // 使用 placement new 构造 A
a->~A();                     // 手动析构 A
B* b = new (buffer) B{17};   // 在相同位置构造 B
```
如果你之后再通过 reinterpret_cast<A*>(buffer)->x 访问 x，编译器可能不会认为你违反了任何规范，因为它看不到 B 的构造有发生（编译器分析的是“表达式”，不是“运行时地址”）。


**🧼 std::launder() 的作用（C++17 新引入）cpp**
```cpp
T* std::launder(T* p);
```
它的功能简单来说是：
>  让编译器意识到你在 p 所在地址上构造了一个新的对象，请抛弃以前关于这个地址的任何“假设”。

也就是说：
* std::launder(p) 通知编译器：这里有一个新的对象，类型可能与之前不同。
* 使用返回值访问内存是合法且受定义的；
* 不使用 launder() 则可能 UB。


**⚠️ 编译器角度（为什么 std::launder 有用）**
C++ 编译器优化依赖“类型相关性”（Type-based alias analysis），而不是“你怎么分配内存”：
* 它不会“看到”你在 buffer 上构造新对象；
* 它也不会“自动清除”对旧对象的假设；
* 但 它会尊重 std::launder() 的语义。


## 26.5 Visitors
### 1、visitor的作用
is() 和 get() 成员函数只能通过指定特定的type来调用，但是我们可以用visitor来实现类似下面的功能：
```cpp
if (v.is<int>()) {
    std::cout << v.get<int>();
} else if (v.is<double>()) {
    std::cout << v.get<double>();
} else { 
    std::cout << v.get<string>();
}
```
```cpp
v.visit([](auto const& value) {
    std::cout << value;
});
```

### 2、visitorImpl的实现
```cpp
template<typename R, typename V, typename Visitor,
typename Head, typename... Tail>
R variantVisitImpl(V&& variant, Visitor&& vis, Typelist<Head, Tail...>) {
	if (variant.template is<Head>()) {
		return static_cast<R>(
					std::forward<Visitor>(vis)(
						std::forward<V>(variant).template get<Head>()));
	}
	else if constexpr (sizeof...(Tail) > 0) {
		return variantVisitImpl<R>(std::forward<V>(variant),
					std::forward<Visitor>(vis),
					Typelist<Tail...>());
	}
	else {
		throw EmptyVariant();
	}
}
```

### 3、visitor的实现
直接调用visitorImp即可
```cpp
template<typename... Types>
template<typename R, typename Visitor>
VisitResult<R, Visitor, Types&...>
Variant<Types...>::visit(Visitor&& vis)& {
	using Result = VisitResult<R, Visitor, Types&...>;
	return variantVisitImpl<Result>(*this, std::forward<Visitor>(vis),
				Typelist<Types...>());
}

template<typename... Types>
template<typename R, typename Visitor>
VisitResult<R, Visitor, Types const&...>
Variant<Types...>::visit(Visitor&& vis) const& {
	using Result = VisitResult<R, Visitor, Types const &...>;
	return variantVisitImpl<Result>(*this, std::forward<Visitor>(vis),
				Typelist<Types...>());
}

template<typename... Types>
template<typename R, typename Visitor>
VisitResult<R, Visitor, Types&&...>
Variant<Types...>::visit(Visitor&& vis) && {
	using Result = VisitResult<R, Visitor, Types&&...>;
	return variantVisitImpl<Result>(std::move(*this),
				std::forward<Visitor>(vis),
				Typelist<Types...>());
}
```

### 26.5.1 Visit Result Type
Visitor的返回值有时候无法确定，因此有时需要显式指定。例如
```cpp
// 无法知道value的类型
[](auto const& value) {
    return value + 1;
}

// 显式指定
v.visit<Variant<int, double>>([](auto const& value) {
    return value + 1;
});
```


### 26.5.2 Common Result Type

```cpp
using std::declval;

template<typename T, typename U>
class CommonTypeT
{
public:
    using Type = decltype(true? declval<T>() : declval<U>());
};

template<typename T, typename U>
using CommonType = typename CommonTypeT<T, U>::Type;
```



## 26.6 Variant Initialization and Assignment

### 1、Default Initialization

默认构造初始化不会将variant初始化为空，因为空的variant的一些操作可能会抛出异常。
因此默认初始化会构造variant中的第一个type。
```cpp
template<typename... Types>
Variant<Types...>::Variant() {
    *this = Front<Typelist<Types...>>();
}
```

### 2、Copy/Move Initialization

> To copy a source variant, we need to determine which type it is currently storing, copy-construct that value into the buffer, and set that discriminator.

利用visitor以及VariantChoice中的 copy assignment 操作。
```cpp
// copy assignment
template<typename... Types>
Variant<Types...>::Variant(Variant const& source) {
	if (!source.empty()) {
		source.visit([&](auto const& value) {
			*this = value;
		});
	}
}

// move assignment是类似的
template<typename... Types>
template<typename... SourceTypes>
Variant<Types...>::Variant(Variant<SourceTypes...> const& source) {
	if (!source.empty()) {
		source.visit([&](auto const& value) {
			*this = value;
		});
	}
}
```


### Assignment
> The Variant assignment operators are similar to the copy and move constructors above


```cpp
template<typename... Types>
Variant<Types...>& Variant<Types...>::operator= (Variant const& source) {
	if (!source.empty()) {
		source.visit([&](auto const& value) {
			*this = value;
		});
	}
	else {
		destroy();
	}
	return *this;
}
// When the source variant contains no value (indicated by a discriminator 0), we destroy the value of the destination, implicitly setting its discriminator to 0.


```

