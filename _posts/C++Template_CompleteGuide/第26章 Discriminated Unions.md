在本章中，我们将开发一个类模板 Variant，它动态存储一组给定的可能值类型中的一种值，类似于 C++ 17 标准库的 std::variant<>。Variant 是一个可区分联合体，这意味着 Variant 知道其可能的值类型中哪些当前处于活动状态，从而提供比等效的 C++ 联合体更好的类型安全性。
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

// TODO



