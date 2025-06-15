---
layout: post
title: 第22章 Templates and Inheritance
tags: [c++语法]
---

本章主要介绍CRTP和mixins的技巧。

## 21.1 The Empty Base Class Optimization(EBCO)

> Even empty classes, however, have nonzero size.

```cpp
#include <iostream>
class EmptyClass {
};
int main()
{
	std::cout << "sizeof(EmptyClass): " << sizeof(EmptyClass) << '\n';
	// 大部分操作系统中，打印的值为1
	// 少部分严格要求对齐的操作系统，其打印值可能是4
}
```

### 21.1.1 Layout Principles

**empty base class optimization (EBCO)** : 尽管 C++ 中没有零大小类型，但 C++ 标准确实规定，
当使用空类作为基类时，只要它不会导致其被分配到与同类型的其他对象或子对象相同的地址，就无需为其分配空间。

#### **1、 EBCO的实例**
```cpp
#include <iostream>
class Empty {
using Int = int; // type alias members don’t make a class nonempty
};
class EmptyToo : public Empty {
};
class EmptyThree : public EmptyToo {
};
int main()
{
	std::cout << "sizeof(Empty): " << sizeof(Empty) << '\n';
	std::cout << "sizeof(EmptyToo): " << sizeof(EmptyToo) << '\n';
	std::cout << "sizeof(EmptyThree): " << sizeof(EmptyThree) << '\n';
	// 打印结果：
	// sizeof(Empty): 1
	// sizeof(EmptyToo): 1
	// sizeof(EmptyThree): 1
}
// 由于编译器实现了EBCO，导致在 EmptyToo 类中，Empty 类没有被赋予任何空间。
```

#### **2、EBCO的限制**
```cpp
#include <iostream>
class Empty {
using Int = int; // type alias members don’t make a class nonempty
};
class EmptyToo : public Empty {
};
class NonEmpty : public Empty, public EmptyToo {
};
int main()
{
	std::cout << "sizeof(Empty): " << sizeof(Empty) << '\n';
	std::cout << "sizeof(EmptyToo): " << sizeof(EmptyToo) << '\n';
	std::cout << "sizeof(NonEmpty): " << sizeof(NonEmpty) << '\n';
	// sizeof(Empty): 1
	// sizeof(EmptyToo): 1
	// sizeof(NonEmpty): 2
}

```

##### 🚫 为什么 EBCO 有限制？

为了保持 C++ 指针模型的语义：

> 两个指针如果值不同，它们就必须指向不同的对象（subobject）。

也就是说：
```cpp
Empty e1, e2; 
assert(&e1 != &e2); // 不同对象的地址必须不同
```
但如果你在同一个对象里放了两个空类：
```cpp
struct A {}; 
struct B {};  
struct C : A, B {};
```
如果对 `A` 和 `B` 都应用了 EBCO，它们就会映射到 `C` 对象的同一个地址，这样：
```cpp
C c; 
A* a = &c; 
B* b = &c; 
assert(a != b); // ⚠️ 如果 a == b，违反了语言规则
```
这种情况下，**指针比较语义就会出错**，因此 **标准禁止** EBCO 优化同一个完整对象中出现的多个空子对象。


### 21.1.2 Members as Base Classes
#### 1、EBCO 对成员变量无效

* ***EBCO（Empty Base Class Optimization）在数据成员上是存在局限性的**，尤其是当模板参数可能是空类型（empty type）时，**如果直接作为成员变量就无法获得 EBCO 优化带来的空间节省**。

* 下面这个例子就 **不能** 利用 EBCO：
```cpp
template<typename T1, typename T2>
class MyClass {
private:
    T1 a;
    T2 b;
};

```

* 如果 `T1` 是一个空类型，那它作为 `a` 这个成员，**仍然会占据至少一个字节**，用于区分不同对象在内存中的地址。

#### 2、 不能简单的直接继承

```cpp
template<typename T1, typename T2>
class MyClass : private T1, private T2 {
};
```

上面的代码存在有问题，虽然T1和T2如果都是空类的时候，可以优化空间。
* T1 / T2 是非类类型或 union : 继承只适用于类类型，不能从基础类型（如 `int`、`double`、指针等）或 union 类型继承
* T1 和 T2 是同一个类型 : 如果你试图两次继承同一个类型，会触发 **二义性错误**
* 参数类型是 final 的类 : 你不能继承 `final` 类：
* 改变了类的接口结构 :  如果你从模板参数继承，**你可能会暴露其公共成员**；或者引入意料之外的行为（如虚函数被覆盖）
```cpp
	struct Base {
	    virtual void foo();
	};

	template<typename T>
	struct Derived : public T {
	    void foo(); // 意外 override！
	};
```

#### 3、更实用的方法啊
   ✅ 使用条件
* ***模板参数 CustomClass 是 class 类型**（不是基础类型如 int、double）
* ***CustomClass 可能是空类（empty class）**
* ***还有另一个要同时存储的成员（比如一个指针）**

```cpp
template<typename CustomClass>
class Optimizable {
private:
	CustomClass info; // might be empty
	void* storage;
	...
};
```
								$\Downarrow$
								$\Downarrow$
								$\Downarrow$
```cpp
template<typename CustomClass>
class Optimizable {
private:
	BaseMemberPair<CustomClass, void*> info_and_storage;
	...
};
```
**BaseMemberPair的实现**
```cpp
#ifndef BASE_MEMBER_PAIR_HPP
#define BASE_MEMBER_PAIR_HPP
template<typename Base, typename Member>
class BaseMemberPair : private Base {
private:
	Member mem;
public:
	// constructor
	BaseMemberPair (Base const & b, Member const & m)
	: Base(b), mem(m) {
	}
	// access base class data via first()
	Base const& base() const {
	return static_cast<Base const&>(*this);
	}
	Base& base() {
		return static_cast<Base&>(*this);
	}
	// access member data via second()
	Member const& member() const {
		return this->mem;
	}
	Member& member() {
		return this->mem;
	}
};
#endif // BASE_MEMBER_PAIR_HPP
```

### 21.2 The Curiously Recurring Template Pattern (CRTP)

#### 1、代码样板
* CRTP代码样板：
```cpp
template<typename Derived>
class CuriousBase {
...
};
class Curious : public CuriousBase<Curious> {
...
};
```

* 子类为模板的CRTP代码样板
```cpp
template<typename Derived>
class CuriousBase {
...
};
template<typename T>
class CuriousTemplate : public CuriousBase<CuriousTemplate<T>> {
...
};
```

> 通过将派生类通过模板参数传递给其基类，基类可以根据派生类定制其自身行为，而无需使用虚函数。这使得 CRTP 能够有效地分解出只能作为成员函数（例如构造函数、析构函数和下标运算符）或依赖于派生类身份的实现。


#### 2、CRTP简单应用：计算被创建的类的个数
 * 在每个构造函数中增加一个整数静态数据成员
 * 在析构函数中减少该成员
 * 然而，在每个类中都提供这样的代码非常繁琐，而且通过单个（非 CRTP）基类实现此功能会混淆不同派生类的对象计数
```cpp
#include <cstddef>
template<typename CountedType>
class ObjectCounter {
private:
	inline static std::size_t count = 0; // number of existing objects
protected:
	// default constructor
	ObjectCounter() {
		++count;
	}
	// copy constructor
	ObjectCounter (ObjectCounter<CountedType> const&) {
		++count;
	}
	// move constructor
	ObjectCounter (ObjectCounter<CountedType> &&) {
		++count;
	}
	// destructor
	~ObjectCounter() {
		--count;
	}
public:
	// return number of existing objects:
	static std::size_t live() {
		return count;
	}
};
```

CRTP的使用 ：1、每个MyString都继承自不同的父类，因此不会导致不同子类的计数混淆。2、子类可以继承父类的count计数，因此不需要在每个子类中加入计数。
```cpp
#include "objectcounter.hpp"
#include <iostream>
template<typename CharT>
class MyString : public ObjectCounter<MyString<CharT>> {
...
};
int main()
{
	MyString<char> s1, s2;
	MyString<wchar_t> ws;
	std::cout << "num of MyString<char>: " << MyString<char>::live() << '\n';
	std::cout << "num of MyString<wchar_t>: " << ws.live() << '\n';
}
```

### 21.2.1 The Barton-Nackman Trick
#### 1、早期C++的`template operator==`局限性
* 作为类的成员函数，会导致`==`两边的类型不对称因此一般情况下都是作为name space scope function
* Barton-Nackman 技巧被提出时（1994 年），**C++ 还不支持函数模板的重载**，该`operator==`会导致所有其它的重载失效

```cpp
template<typename T>
bool operator== (Array<T> const& a, Array<T> const& b)
{
...
}
```
								$\Downarrow$
								$\Downarrow$
								$\Downarrow$
```cpp
template<typename T>
class Array {
	static bool areEqual(Array<T> const& a, Array<T> const& b);
public:
...
	friend bool operator== (Array<T> const& a, Array<T> const& b) {
		return areEqual(a, b);
	}
};
```
1、通过早期的C++的`friend name injection`过程，使得friend函数可以在enclosing scope可见。
2、`friend bool operator== (Array<T> const&, Array<T> const&);`**不是函数模板**，因为它没有模板头 (`template<typename T>`)，只是类模板的每个实例都会生成一个特化的友元函数。
3、注意：现代C++，不再有`friend name injection`来使得函数可见，而是通过**Argument-Dependent Lookup (ADL)** 来找到函数，例如<u>编译器在处理 `a == b` 时，会去查找与 `a` 和 `b` 所属类型相关联的命名空间或类定义中是否存在 `operator==`。</u>

#### 2、现代C++中的`friend function definition`

现代C++过**Argument-Dependent Lookup (ADL)** 来找到函数，如下代码所示，**foo(w)可以被调用而foo(s)不行，是因为s的name sope中没有friend函数**。

```cpp
class S {
};
template<typename T>
class Wrapper {
private:
	T object;
public:
	Wrapper(T obj) : object(obj) { // implicit conversion from T to Wrapper<T>
	}
	friend void foo(Wrapper<T> const&) {
	}
};
int main()
{
	S s;
	Wrapper<S> w(s);
	foo(w); // OK: Wrapper<S> is a class associated with w
	foo(s); // ERROR: Wrapper<S> is not associated with s，although S can implicit convert to Wrapper through Wrapper constructor
}
```

### 21.2.2 Operator Implementations

很多情况下，`(==, !=, >, <=, >=)`这些运算符中只有一个的定义真正重要，而其他运算符可以简单地根据该运算符来定义。` (!=)`可以通过`(==)`实现，`(>, <=, >=)`可以通过`(<)`来实现。

#### 1、模板`operator!=`滥用陷阱问题
```cpp
template<typename T>
bool operator!= (T const& x1, T const& x2) {
	return !(x1 == x2);
}
```
直观地看，它为所有支持 `==` 的类型自动提供了 `!=`，这个通用模板存在一下问题：
* **所有类型“看起来”都能 `!=`**
```cpp
// 	可以通过SFINAE来规避。
template<typename T>
std::enable_if_t<std::is_convertible_v<decltype(std::declval<T>() == std::declval<T>()), bool>, bool>
operator!=(T const& x1, T const& x2) {
    return !(x1 == x2);
}

```
* **会屏蔽用户提供的“更合适”的重载**
```cpp
template<typename T>
bool operator==(T const& x1, T const& x2) {
	std::cout << "global scope operator!=" << std::endl;
	return false;
}

struct Base {
    bool operator==(Base const&) const {
	    std::cout << "base operator==" << std::endl;
    }
};

struct Derived : Base {
    // 没有定义 ==, 但是可以用 Base 的 == 来比较
};

int main() {
	Derived d1, d2;
	bool x = (d1 != d2);
	// 输出 "global scope operator!="
}
```

如果定义了通用的`==,` 编译器会选择这个模板，而不会去用 Base 的 `operator==`，因为模板是“完美匹配”。但这个选择是 **语义错误的** —— 用户本意是让 Derived 隐式向 Base 转换后比较，但模板阻止了这个发生。**模板会优先于隐式转换后选择的非模板重载函数**。

#### 2、CRTP + Barton-Nackman trick
 CRTP 可以为这些通用运算符提供更generic的实现，从而提供增加代码重用的好处，而不会产生过于滥用template的副作用：
```cpp
template<typename Derived>
class EqualityComparable
{
public:
	friend bool operator!= (Derived const& x1, Derived const& x2) {
		return !(x1 == x2);
	}
};

class X : public EqualityComparable<X>
{
public:
	friend bool operator== (X const& x1, X const& x2) {
	// implement logic for comparing two objects of type X
	}
};
int main()
{
	X x1, x2;
	if (x1 != x2) { }
}
```
`EqualityComparable<>` 使用`CRTP `为其派生类提供运算符` !=`，该运算符基于派生类的`operator== `定义。CRTP 在将行为分解到基类中同时保留最终派生类的身份。

### 21.2.3 Facades
外观模式 ： 通过 CRTP 派生类暴露的的接口来定义CRTP 基类的大部分公共接口。

#### 1、CRTP实现IteratorFacade

`IteratorFacade`用来简化自定义迭代器的开发，**核心思想就是只要求开发者提供最小必要的“核心接口”**，然后由 `IteratorFacade` 自动生成 STL 所需的完整接口。其提供接口并不需要那么多：

```cpp
template<typename Derived, typename Value, typename Category,
typename Reference = Value&, typename Distance = std::ptrdiff_t>
class IteratorFacade
{
public:
	using value_type = typename std::remove_const<Value>::type;
	using reference = Reference;
	using pointer = Value*;
	using difference_type = Distance;
	using iterator_category = Category;
	
	// input iterator interface:
	reference operator *() const { return asDerived().dereference(); }
	pointer operator ->() const { ... }
	Derived& operator ++() { 
		asDerived().increment();
		return asDerived(); 
	}
	Derived operator ++(int) { 
		Derived result(asDerived());
		asDerived().increment();
		return result;
	}
	friend bool operator== (IteratorFacade const& lhs,
	IteratorFacade const& rhs) { ... }
	...
	
	// bidirectional iterator interface:
	Derived& operator --() { ... }
	Derived operator --(int) { ... }
	// random access iterator interface:
	reference operator [](difference_type n) const { ... }
	Derived& operator +=(difference_type n) { ... }
	...
	
	friend difference_type operator -(IteratorFacade const& lhs,
	IteratorFacade const& rhs) { ... }
	friend bool operator <(IteratorFacade const& lhs,
	IteratorFacade const& rhs) { ... }
	...

private:
	Derived& asDerived() { return *static_cast<Derived*>(this); }
	Derived const& asDerived() const {
		return *static_cast<Derived const*>(this);
	}
};
```

#### 2、Iterator需要实现的必要接口
* 因为STL 要求一个迭代器提供 `++`, `--`, `*`, `==`, `!=`, `+`, `-`, `[]` 等操作符。但如果每次都手写很容易出错，代码也繁琐。
	- 用 **CRTP + IteratorFacade** 的方式
	- 要求你实现几个核心操作（如 `dereference()`）
	- 然后由 Facade 来自动定义这些操作符

* 这样只实现下面少量逻辑就能得到完整的符合 STL 要求的迭代器。
* 不需要全部实现，例如**如果 `Derived` 没有实现 `decrement()`，那 `operator--()` 会导致编译失败**，但如果你从不使用 `--it`，那么这个成员永远不会被实例化，也不会出错。

| 迭代器类型           | `Derived` 必须实现的方法                    |
| --------------- | ------------------------------------ |
| Input / Forward | `dereference() increment() equals()` |
| Bidirectional   | `上述 + decrement()`                   |
| Random Access   | `上述 + advance(n) measureDistance()`  |

#### 3、Defining a Linked-List Iterator
```cpp
template<typename T>
class ListNode
{
public:
	T value;
	ListNode<T>* next = nullptr;
	~ListNode() { delete next; }
};

template<typename T>
class ListNodeIterator
: public IteratorFacade<ListNodeIterator<T>, T,
std::forward_iterator_tag>
{
	ListNode<T>* current = nullptr;
public:
	T& dereference() const {
		return current->value;
	}
	void increment() {
		current = current->next;
	}
	bool equals(ListNodeIterator const& other) const {
		return current == other.current;
	}
	ListNodeIterator(ListNode<T>* current = nullptr) : current(current) { }
};
// 该Iterator我们仅仅只需要实现少量的接口，而不需要定义所有的operator函数
```

#### 4、Hiding Interface
为了隐藏`ListNodeIterator`中暴露的public接口，我们可以重新设计 IteratorFacade，使其所有操作都通过一个单独的访问类（`IteratorFacadeAccess`）在派生的 CRTP 类上执行。
```cpp
// inherit/iteratorfacadeaccessskel.hpp
// ‘friend’ this class to allow IteratorFacade access to core iterator operations:
class IteratorFacadeAccess
{
// only IteratorFacade can use these definitions
template<typename Derived, typename Value, typename Category, typename Reference, typename Distance>
friend class IteratorFacade;

// required of all iterators:
template<typename Reference, typename Iterator>
static Reference dereference(Iterator const& i) {
	return i.dereference();
}
...

// required of bidirectional iterators:
template<typename Iterator>
static void decrement(Iterator& i) {
	return i.decrement();
}
// required of random-access iterators:
template<typename Iterator, typename Distance>
static void advance(Iterator& i, Distance n) {
	return i.advance(n);
}
...

};
```

* 相应的，IteratorFacade的接口修改如下
```cpp
template<typename Derived, typename Value, typename Category,
typename Reference = Value&, typename Distance = std::ptrdiff_t>
class IteratorFacade {
public:
	...
	// 改为通过IteratorFacadeAccess来访问Derived class
    auto operator*() const {
        return IteratorFacadeAccess::dereference(static_cast<const Derived&>(*this));
    }

    Derived& operator++() {
        IteratorFacadeAccess::increment(static_cast<Derived&>(*this));
        return static_cast<Derived&>(*this);
    }
    ...
};
```

* ListIterator仅仅需要将其所有的接口设置为private，并声明`friend class IteratorFacadeAccess;`

#### 5、Iterator Adapters
C++ 中的 **迭代器适配器（iterator adapter）** 是一种特殊类型的迭代器，它**包装（wrap）已有的迭代器类型，并对其行为进行修改或增强**，使其能够以不同的方式访问底层数据。

|类型|示例|功能|
|---|---|---|
|**插入适配器**|`std::back_inserter`, `std::inserter`|将算法输出插入到容器尾部、中间等|
|**流适配器**|`std::istream_iterator`, `std::ostream_iterator`|将流对象转换为迭代器|
|**反向适配器**|`std::reverse_iterator`|将正向迭代器变为反向|
|**变换/投影适配器**|自定义 `transform_iterator`, `projection_iterator`|对元素应用函数或访问子成员|
|**过滤适配器**|Boost 的 `filter_iterator`|仅访问符合条件的元素|
|**压缩/连接适配器**|zip、join（如 range-v3 提供）|合并多个容器或展平嵌套容器|
##### 提出问题
例子：假设有一个容器保存了`struct Person`，可以认为是`std::vector<Person>`。
```cpp
struct Person {
	std::string firstName;
	std::string lastName;
	friend std::ostream& operator<<(std::ostream& strm, Person const& p) {
		return strm << p.lastName << ", " << p.firstName;
	}
};
```
如果我想要用`std::copy`遍历这个迭代器，但是遍历过程中只想要获取成员`firstName`，那么就可以用到**迭代器适配器（iterator adapter）**。
```cpp
std::vector<Person> authors;
std::copy(...);
```

##### ProjectionIterator设计
```cpp
template<typename Iterator, typename T>
class ProjectionIterator : public IteratorFacade
			<
				ProjectionIterator<Iterator, T>,
				T,
				typename std::iterator_traits<Iterator>::iterator_category,
				T&,
				typename std::iterator_traits<Iterator>::difference_type
			>
{
	using Base = typename std::iterator_traits<Iterator>::value_type;
	using Distance = typename std::iterator_traits<Iterator>::difference_type;
	Iterator iter;
	T Base::* member;
	friend class IteratorFacadeAccess;
	... // implement core iterator operations for IteratorFacade
	T& dereference() const {
		return (*iter).*member;
	}
	void increment() {
		++iter;
	}
	bool equals(ProjectionIterator const& other) const {
		return iter == other.iter;
	}
	void decrement() {
		--it
	}
public:
	ProjectionIterator(Iterator iter, T Base::* member) 
				: iter(iter), member(member) { }
};
```

```cpp
template<typename Iterator, typename Base, typename T>
auto project(Iterator iter, T Base::* member) {
	return ProjectionIterator<Iterator, T>(iter, member);
}
```

##### 迭代器适配器的使用
通过使用迭代器适配器，我们就可以使用标准算法std::copy获取到Person的firstName成员了。
```cpp
#include <vector>
#include <algorithm>
#include <iterator>
int main()
{
std::vector<Person> authors = { {"David", "Vandevoorde"},
								{"Nicolai", "Josuttis"},
								{"Douglas", "Gregor"} };

std::copy(project(authors.begin(), &Person::firstName),
			project(authors.end(), &Person::firstName),
			std::ostream_iterator<std::string>(std::cout, "\n"));
}
```


## 21.3 Mixins

#### 概述
**Mixin（混入类）** 是一种设计技巧，用来“组合功能”，而不是通过传统的“继承一个父类然后重写它”的方式来扩展类的功能。

它和传统继承的区别在于：
- **传统继承**是：子类继承父类，功能垂直叠加。
- **Mixin 是横向组合**：通过**多个“功能模块”作为基类**，将功能拼装在目标类里。
#### ⚠️ 注意事项
- **多重继承的菱形问题**：Mixin 应避免引入共同基类。
- **命名冲突**：多个 Mixin 有相同成员函数时需要解决冲突（比如虚继承、CRTP、别名等）。
- **构造顺序**：Mixin 会按照声明顺序构造。

#### 例子
通过Mixins技巧，当我们需要为Point类增加功能的时候，就不需要再修改Point类当中的接口了，直接在Mixins中加入一个功能类。
```cpp
template<typename P>
class Polygon
{
private:
	std::vector<P> points;
public:
	... // public operations
};
```
								$\Downarrow$
								$\Downarrow$
								$\Downarrow$
```cpp
template<typename... Mixins>
class Polygon
{
private:
	std::vector<Point<Mixins...>> points;
public:
	... // public operations
};
```

### 21.3.1 Curious Mixins
* 把 Mixin 设计为 **CRTP 模板类**，然后在组合目标类（如 `Point`）时，将目标类自身作为模板参数传给 Mixin，实现“功能对类的定制”。
* `Mixins<Point>` 表示将目标类型 `Point` 自己传入到这些 Mixin 模板中。
- 这样，**Mixin 类就能通过模板参数拿到 Point 的完整定义，实现反向依赖和定制行为**。

```cpp
#include <iostream>

template<typename Derived>
struct Printable {
    void print() const {
        auto const& self = static_cast<Derived const&>(*this);
        std::cout << "Printable: x=" << self.x << ", y=" << self.y << std::endl;
    }
};

template<typename Derived>
struct LogOnCreation {
    LogOnCreation() {
        std::cout << "Created an instance of " << typeid(Derived).name() << "\n";
    }
};

template<template<typename>... Mixins>
class Point : public Mixins<Point<Mixins...>>... {
public:
    double x, y;

    Point() : Mixins<Point>()..., x(0.0), y(0.0) {}
    Point(double x, double y) : Mixins<Point>()..., x(x), y(y) {}
};

// main.cpp
int main() {
    Point<Printable, LogOnCreation> p(3.14, 2.71);
    p.print();
}

// 输出
// Created an instance of class Point<Printable,LogOnCreation>
// Printable: x=3.14, y=2.71

```


### 21.3.2 Parameterized Virtuality
#### 1、**virtuality inference**技巧

```cpp
#include <iostream>
class NotVirtual {
};
class Virtual {
public:
	virtual void foo() {
}
};
template<typename... Mixins>
class Base : public Mixins... {
public:
	// the virtuality of foo() depends on its declaration
	// (if any) in the base classes Mixins...
	void foo() {
		std::cout << "Base::foo()" << '\n';
	}
};
template<typename... Mixins>
class Derived : public Base<Mixins...> {
public:
	void foo() {
		std::cout << "Derived::foo()" << '\n';
	}
};
int main()
{
	Base<NotVirtual>* p1 = new Derived<NotVirtual>;
	p1->foo(); // calls Base::foo()
	Base<Virtual>* p2 = new Derived<Virtual>;
	p2->foo(); // calls Derived::foo()
}
```
- `Base` 中的 `foo()` 是 **非虚函数**。
- 但当它继承自某个基类（比如 `Virtual`）时，如果 `Virtual::foo()` 是虚函数，那么即使 `Base` 没有显式写 `virtual`，这个 `foo()` **在编译时会变成 virtual 函数**！

> **函数的 virtual 属性会沿着继承链向下传播（只要签名匹配）**。

#### 2、原因
由于 **C++ 的虚函数机制**：
- 如果一个类中有虚函数 `foo()`，那么派生类中签名一致的 `foo()` 会**自动变为虚函数**，即使没写 `virtual`。
- C++ 编译器在判断是否是虚函数时，会从 **所有基类中查找是否已经声明为 virtual**，这也包括 **多继承和模板展开后的基类**。

#### 3、应用

用户可以传入 `true/false` 来决定是否启用虚函数，**而不需要更改类的结构或写多个类**。
```cpp
template<bool UseVirtual> using MaybeVirtualBase 
		= std::conditional_t<UseVirtual, Base<Virtual>, Base<NotVirtual>>;
```

#### 4、虚函数注意
##### ✅ 虚函数并不是万能的基类设计方案
虽然**给类模板中的成员函数加上 `virtual`**，看起来似乎可以同时支持：
- 创建具体实例（concrete class）；
- 用作继承的基类（base class for specialization）；
但是，**仅仅依赖“加几个虚函数”** 并不足以设计出一个优秀的、可拓展的基类模板体系。
---
##### ❗ 为什么“加 virtual”不够：
1. **虚函数只是行为拓展的一种机制**，不能代替架构层面的设计；
2. 如果一个类既用于“最终实例化使用”，又要用于“派生出新功能”，其职责容易变得不清晰；
3. 虚函数引入运行时多态，也可能带来性能开销和二义性；
4. 很多时候，**模板本身就是静态多态的工具**，和虚函数（动态多态）混合使用会导致概念冲突或复杂化。
---
##### ✅ 更推荐的做法是：
> 设计**两个分离的工具类（或类模板）**，分别用于：
- 直接实例化使用（Concrete）；
- 派生/继承/拓展使用（Base & CRTP等）；
这样架构更清晰，也更便于维护
---
##### 💡 举个例子：

#### ❌ 不推荐方式（混合用）：

```cpp
template<typename T> class Mixed { 
public:     
	void doSomething() { ... }     
	virtual void extendable() { ... } // 试图拓展 
};
```

##### ✅ 推荐方式（职责分离）：
```cpp
// 用于具体使用 
template<typename T> class Worker {
public:     
	void doWork() { 
	... 
} 
};  
// 用于继承拓展 
template<typename Derived> 
class WorkerExtension { 
public:     
	void extend() { 
	static_cast<Derived*>(this)->customBehavior(); 
} 
};
```

