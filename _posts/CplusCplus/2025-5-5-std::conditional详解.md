---
layout: post
title: std::conditional详解
tags: [C++]
---


`std::conditional` 是 C++ 标准库 `<type_traits>` 头文件中提供的一个模板工具，用于在编译时根据布尔条件选择两种类型中的一种。其核心功能类似于三元条件运算符（`?:`），但作用于类型层面而非值层面。

---

### **基本语法**
```cpp
#include <type_traits>

template<bool B, typename T, typename F>
struct conditional;

// 特化版本：
template<typename T, typename F>
struct conditional<true, T, F> { using type = T; };

template<typename T, typename F>
struct conditional<false, T, F> { using type = F; };

// 辅助别名模板（C++11 起）：
template<bool B, typename T, typename F>
using conditional_t = typename conditional<B, T, F>::type;
```

- **参数**：
  - `B`：布尔条件（`true` 或 `false`）。
  - `T`：当 `B` 为 `true` 时选择的类型。
  - `F`：当 `B` 为 `false` 时选择的类型。
- **成员 `type`**：根据 `B` 的值，为 `T` 或 `F`。

---

### **核心功能**
根据布尔条件 `B`，`conditional<B, T, F>::type` 会静态地选择 `T` 或 `F`：
- 若 `B` 为 `true`，则 `type` 是 `T`。
- 若 `B` 为 `false`，则 `type` 是 `F`。

---

### **实现原理**
通过模板特化实现条件选择：
```cpp
// 主模板（默认处理 B = false）
template<bool B, typename T, typename F>
struct conditional {
    using type = F;
};

// 特化模板处理 B = true
template<typename T, typename F>
struct conditional<true, T, F> {
    using type = T;
};
```

---

### **使用示例**

#### 1. **基本用法**
```cpp
using Type1 = std::conditional<true, int, double>::type;  // Type1 = int
using Type2 = std::conditional<false, int, double>::type; // Type2 = double
```

#### 2. **结合类型特性（Type Traits）**
根据类型是否为整数选择不同类型：
```cpp
template<typename T>
struct SelectType {
    using Type = std::conditional_t<std::is_integral_v<T>, int, double>;
};

static_assert(std::is_same_v<SelectType<int>::Type, int>);    // 通过
static_assert(std::is_same_v<SelectType<float>::Type, double>); // 通过
```

#### 3. **条件选择容器类型**
```cpp
template<bool UseVector>
struct ContainerSelector {
    using Container = std::conditional_t<UseVector, std::vector<int>, std::list<int>>;
};

ContainerSelector<true>::Container  vec;  // vec 是 std::vector<int>
ContainerSelector<false>::Container list; // list 是 std::list<int>
```

---

### **与运行时条件的区别**
普通 `if-else` 是运行时的控制流，而 `std::conditional` 是编译时的类型选择：
```cpp
// 运行时条件（无法用于类型选择）：
if (condition) {
    using Type = int;
} else {
    using Type = double; // 错误：无法在运行时改变类型
}

// 编译时条件（正确方式）：
using Type = std::conditional_t<condition, int, double>;
```

---

### **常见应用场景**
1. **类型分发（Type Dispatch）**  
   根据类型特性选择不同的实现。
   ```cpp
   template<typename T>
   struct Wrapper {
       using Storage = std::conditional_t<std::is_trivially_copyable_v<T>, T, T*>;
   };
   ```

2. **优化代码路径**  
   根据编译期条件选择高效的类型或算法。
   ```cpp
   template<bool UseSIMD>
   struct Algorithm {
       using DataType = std::conditional_t<UseSIMD, SIMDVector, ScalarVector>;
   };
   ```

3. **元编程中的分支逻辑**  
   在模板元编程中实现复杂的分支逻辑。
   ```cpp
   template<typename T>
   using NestedType = std::conditional_t<
       std::is_class_v<T>,
       typename T::value_type,
       T
   >;
   ```

---

### **与 `std::enable_if` 的区别**
| 特性                | `std::conditional`                     | `std::enable_if`                     |
|---------------------|----------------------------------------|---------------------------------------|
| **目的**            | 根据条件选择两种类型之一                 | 根据条件启用或禁用模板重载              |
| **返回类型**        | 直接返回 `T` 或 `F`                     | 条件为真时返回 `T`，否则无定义          |
| **典型应用场景**    | 类型选择、分支逻辑                       | SFINAE、模板特化控制                   |

---

### **总结**
- `std::conditional` 是编译时类型选择的工具，通过模板特化实现。
- 常用于根据布尔条件静态选择类型，提升代码的灵活性和可维护性。
- 结合其他类型特性（如 `std::is_integral`、`std::is_pointer` 等），可实现复杂的元编程逻辑。
