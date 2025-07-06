> Every expression has a type, which describes the static type of the value that its computation produces.
> Each expression also has a value category, which describes something about how the value was formed and affects how the expression behaves.

## 1、Traditional Lvalues And Rvalues

> Lvalues are expressions that refer to actual values stored in memory or in a machine register, such as the expression x where x is the name of a variable.These expressions may be modifiable, allowing one to update the stored value.

* 传统意义上，左值有存储空间，且其值可以被改变。对于assignment 表达式而言，其只能存在表达式左边，因此称为lvalue，同理rvalue只能存在于表达式右边。
* C/C++改变了上述关于lvalue和rvalue的传统表述
	* lvaue的值不一定能够被改变。例如`int const x; x = 7;`
	* rvalue也可以存在于assignment expression的左侧。(通过将operator=进行重载来实现)
* 在C/C++中，lvalue更准确地表述应该是`localized value`，即我们可以通过`&`来取得其地址或者将其绑定到一个引用上`T&`或者`const T&`。
* `variable, defererencing a pointer, Member access，Function calls returning lvalue references`这些都是左值。注意，`string literals are also (nonmodifiable) lvalues`。

| Expression Type  | Value Category | Explanation                    |
| ---------------- | -------------- | ------------------------------ |
| `x`              | lvalue         | Named variable                 |
| `*p`             | lvalue         | Dereference yields a location  |
| `obj.member`     | lvalue         | Member access of an lvalue     |
| `ptr->member`    | lvalue         | Same as above                  |
| `f()` (`T& f()`) | lvalue         | Returns reference to object    |
| `f()` (`T f()`)  | prvalue        | Returns a value — temporary    |
| `std::move(x)`   | xvalue         | Expiring object — has identity |
| `"abc"`          | lvaue          | String literals(字符串常量)         |
* rvalue 是没有identity的值——它们没有内存地址，即无法取得它们的存储空间。它们是计算的临时结果。

| Expression                | Value Category | Notes                        |
| ------------------------- | -------------- | ---------------------------- |
| `42`, `'a'`, `nullptr`    | rvalue         | literal constants            |
| `x + y`                   | rvalue         | arithmetic result            |
| `std::string("abc")`      | rvalue         | temporary object             |
| `make_str()`              | rvalue         | function returning by value  |
| `std::move(x)`            | xvalue         | expiring object              |
| `r` where `int&& r = ...` | lvalue         | named reference to an rvalue |

### 1.1、Lvalue-to-Rvalue Conversions
* rvalue必须严格存在于assignment expression的右边，`7 = 2`是不可行的。
* lvaue没有这个限制，`x = y`是可行的。`y`被转化为了右值 ： 从y的地址取值，并生成一个右值。

## 2、Value Categories Since C++11

由于c++11中引入了右值引用，C++11中expression不能简单的分为lvalue和rvalue。

![[Pasted image 20250702110116.png]]

* **glavue** ： an expression whose evaluation determines the identity of an object, bit-field, or
function (i.e., an entity that has storage).   --- 即有identity和storage的
* **prvalue** ： an expression whose evaluation initializes an object or a bit-field, or computes the
value of the operand of an operator  --- - 无identiity和storage的，常用于初始化或计算中，是表达式的“最终结果”，不可取地址
* **xvalue** ： a glvalue designating an object or bit-field **whose resources can be reused** (usually
because it is about to “expire”—the “x” in xvalue originally came from “eXpiring value”). --- 是一种 glvalue（有身份），但**可以移走资源**
* **lvalue** ： An lvalue is a glvalue that is not an xvalue. --- 即glavlue中不可移动的部分
* **rvalue** ：An rvalue is an expression that is either a prvalue or an xvalue.  --- - rvalue = prvalue + xvalue, 可用于初始化、函数参数传递, **移动语义**核心就在这两者

### 2.1、Example of lvalues
#### Expressions that designate variables or functions
```cpp
// 表示变量或者表达式

int x = 42;
x = 10;    // x 是 lvalue，可以赋值
int* p = &x; // 可以取地址

void f();
f;         // f 是 lvalue（函数名也是有地址的对象）
```


#### Applications of the built-in unary * operator (“pointer indirection”)
```cpp
// 解引用操作的结果
int y = 5;
int* p = &y;
*p = 20;   // *p 是 lvalue，可以赋值
int* q = &(*p); // 也可以取地址
```


#### An expression that is just a string literal
```cpp
// 纯字符串字面量表达式
const char* str = "hello";
char ch = "hello"[1];  // "hello" 是 lvalue

```
####  A call to a function with a return type that is an lvalue reference
```cpp
// 返回lvalue引用的函数调用
int x = 100;

int& ref() {
    return x;
}

ref() = 200;   // ref() 是 lvalue，可以赋值
int* p = &ref(); // 可以取地址
```

### 2.2、Example of prvalue
#### Expressions that consist of a literal that is not a string literal or a user-defined literal
```cpp
// 不是字符串字面量或用户自定义字面量的字面值
42;        // int
3.14;      // double
'a';       // char
true;      // bool
nullptr;   // std::nullptr_t

```
> **字符串字面量 `"abc"` 是 lvalue**，因为它是数组。


#### Applications of the built-in unary & operator (i.e., taking the address of an expression)
```cpp
// 取地址操作 `&expr` 的结果
int x = 5;
int* p = &x;

```
- `&x` 是 **prvalue**，产生一个右值（指针值），可以赋值或传参。
- 你不能对 `&x` 再取地址（即 `&&x` 是非法的），因为它没有身份

####  Applications of built-in arithmetic operators
```cpp
int a = 2, b = 3;
int c = a + b;  // a + b 是 prvalue
double d = a * 3.14; // a * 3.14 是 prvalue

```
* 算术表达式的结果是**新计算出来的值**，没有身份，不能取地址 ⇒ prvalue。
#### A call to a function with a return type that is not a reference type
```cpp
int make_int() {
    return 42;
}

make_int(); // prvalue

```
- `make_int()` 返回的是 `int`，不是引用 ⇒ 返回结果是 **prvalue**
- 这是 C++ 中最典型的 prvalue：**临时对象/计算结果**

#### Lambda expressions
```cpp
auto f = [] (int x) { return x * 2; };
```
* Lambda 表达式本身是 prvalue
* [](...) { ... } 是一个 prvalue，创建一个匿名闭包对象
* 它没有名字，没有身份 ⇒ 是 prvalue
* 赋值给 f 后，f 是有身份的 lvalue

### 2.3、Example of xvalues

> xvalue 它是一个**有身份的值，但其资源即将被“搬走”或“回收”**。
####  A call to a function with a return type that is an rvalue reference to an object type (e.g., std::move())

```cpp
std::string make_string() {
    std::string s = "hello";
    return std::move(s);  // std::move 返回的是 std::string&& → xvalue
}

int main() {
    std::string s2 = make_string();  // 移动构造（xvalue）
}

```
- `std::move(s)` 返回的是一个右值引用表达式（`std::string&&`）
- 表达式本身是一个 **xvalue**
- 它有地址（指向 `s`），但是会被**当作可搬走资源**使用

#### A cast to an rvalue reference to an object type
```cpp
std::string s = "abc";
std::string&& r = static_cast<std::string&&>(s); // xvalue

std::string t = std::move(r); // 移动构造

```
- `static_cast<std::string&&>(s)` 是一个右值引用表达式 ⇒ **xvalue**
- 代表 `s` 这个变量，但表明它的资源可被“移动”
- 所以构造 `t` 时使用了 `std::string` 的移动构造函数

### 2.4、Note that rvalue references to function types produce lvalues, not xvalues.
```cpp
void f() { }

using FuncType = void();
FuncType&& rf = f;   // rf 是对函数 f 的 rvalue reference

rf();               // ✅ 合法
auto p = &rf;       // ✅ 合法：可以取地址

```
- `rf`的类型是 `FuncType&&`（即 `void(&&)()`）
- 你可能以为 `rf` 是一个 **xvalue**（因为是 `T&&`）
- 但实际上，**在 C++ 的值类别规则中，函数类型的表达式永远是 lvalue**

## 3、Temporary Materialization

**Temporary Materialization**指代的是<u>prvalue-to-xvalue</u> 的转化。

```cpp
int f(int const&);
int r = f(3);
```

* 3是prvalue，而f接收的是glvalue
* `Temporary Materialization`机制会将3作为初始化条件，转化为一个临时xvalue对象

### 3.1、Prvalue Temporary Materialization Situations
#### A prvalue is bound to a reference (e.g., that call f(3) above).
```cpp
void f(const int& r) { }

f(3);  // ✅ 3 是 prvalue，必须 materialize 成临时对象，才能绑定到引用 r
```
#### A member of a class prvalue is accessed.
```cpp
struct S { int x; };

S().x;  // ✅ S() 是 prvalue ⇒ 必须先 materialize ⇒ 才能访问成员 x
```
- `S()` 本身没有对象 ⇒ 要访问 `.x` ⇒ 编译器创建临时对象
- 结果是 `S` 的成员访问表达式
#### An array prvalue is subscripted.
```cpp
struct ArrayHolder { int a[3]; };
ArrayHolder get_arr() { return {1, 2, 3}; }

int x = get_arr().a[1];  // ✅ get_arr() 是 prvalue ⇒ 必须 materialize ⇒ 才能 [1]
```
- prvalue 本身是一个“初始化值”
- 但你要下标 `[1]` ⇒ 需要真实的数组对象 ⇒ 编译器创建临时对象
#### An array prvalue is converted to a pointer to its first element (i.e., array decay).

#### A prvalue appears in a braced initializer list that, for some type X, initializes an object of type `std::initializer_list<X>.`
```cpp
auto list = { std::string{"A"}, std::string{"B"} }; // 每个字符串纯右值物化
```
#### The sizeof or typeid operator is applied to a prvalue.
```cpp
size_t s1 = sizeof(std::string{"temp"}); // 物化临时对象（但不构造）
size_t s2 = sizeof(42);                 // 基本类型不物化（直接计算大小）

const std::type_info& ti = typeid(std::vector<int>{}); // 物化临时 vector
```
#### A prvalue is the top-level expression in a statement of the form “expr;” or an expression is cast to void.
```cpp
std::string{"discarded"}; // 语句 "expr;" 物化临时对象
static_cast<void>(42);    // 转型为 void 导致物化
```


### 3.2、总结
```cpp
class X {
};
X v;
X const c;
void f(X const&); // accepts an expression of any value category
void f(X&&); // accepts prvalues and xvalues only but is a better match
			 // for those than the previous declaration
f(v); // passes a modifiable lvalue to the first f()
f(c); // passes a nonmodifiable lvalue to the first f()
f(X()); // passes a prvalue (since C++17 materialized as xvalue) to the 2nd f()
f(std::move(v)); // passes an xvalue to the second f()
```



## 4、Checking Value Categories with `decltype`

* 通过`decltype`可以检测任意expression的`value category`.
* `decltype((x))` yields 
	* `type` if x is a pvalue
	* `type&` if x is an lvalue
	* `type&&` if x is an xvalue
* `decltype((x))`中`(x)`确保decltype中的是一个表达式，而不是一个variable。

```cpp
if constexpr (std::is_lvalue_reference<decltype((e))>::value) {
	std::cout << "expression is lvalue\n";
}
else if constexpr (std::is_rvalue_reference<decltype((e))>::value) {
	std::cout << "expression is xvalue\n";
}
else {
	std::cout << "expression is prvalue\n";
}
```


## 5、Reference Types

 **C++ 中引用类型（如 `int&`, `int&&`）与值类别（lvalue、xvalue、prvalue）之间的关系**
### 5.1、引用的初始化受到值类别的限制

 引用不能随便绑定任何值，它必须匹配特定的值类别。

|引用类型|可以绑定的表达式值类别|
|---|---|
|`int&`|**只能绑定 lvalue**（有名字、可取地址的变量）|
|`const int&`|可以绑定 **任何值类别**（包括 prvalue、xvalue）|
|`int&&`|**只能绑定 rvalue**（prvalue 或 xvalue）|
```cpp
int x = 10;
int& r1 = x;         // ✅ OK，x 是 lvalue
int&& r2 = 10;       // ✅ OK，10 是 prvalue
// int& r3 = 10;     // ❌ 错，不能用 prvalue 初始化 int&
```

### 5.2、函数返回引用时影响表达式的值类别

函数返回类型是否为引用，**直接决定了调用表达式的值类别**：

|函数返回类型|函数调用表达式的值类别|
|---|---|
|`int&`|lvalue|
|`int&&`|xvalue|
|`int`（非引用）|prvalue|
|`void(&&)()`（函数右值引用）|lvalue（⚠️ 特例）|
```cpp
int& lvalue();
int&& xvalue();
int prvalue();

std::is_same_v<decltype(lvalue()), int&> // yields true because result is lvalue
std::is_same_v<decltype(xvalue()), int&&> // yields true because result is xvalue
std::is_same_v<decltype(prvalue()), int> // yields true because result is prvalue

int& lref1 = lvalue(); // OK: lvalue reference can bind to an lvalue
int& lref3 = prvalue(); // ERROR: lvalue reference cannot bind to a prvalue
int& lref2 = xvalue(); // ERROR: lvalue reference cannot bind to an xvalue


int&& rref1 = lvalue(); // ERROR: rvalue reference cannot bind to an lvalue
int&& rref2 = prvalue(); // OK: rvalue reference can bind to a prvalue
int&& rref3 = xvalue(); // OK: rvalue reference can bind to an xrvalue
```
