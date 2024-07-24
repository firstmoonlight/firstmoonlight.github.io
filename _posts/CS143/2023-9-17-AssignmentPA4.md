---
layout: post
title: Assignment PA4
tags: [CS143]
---

## 1 前言
在这个assignment中，我们将实现semantic check。通过AST树来检查程序是否符合Cool语言规范。对于有错误的代码，应该在这个阶段报错；而对于可以正确通过的代码，它必须收集某些信息供code generator使用。senmantic check阶段的输出是一棵annotated AST，通过这个AST，我们最终可以生成汇编代码。
本次Assignment的任务有以下3个
* 查看所有class并构建继承图。
* 检查继承图是否构造正确。
* 对每个类：
(a) 遍历AST，在符号表中收集所有可见的declaration。
(b) 检查每个表达式的类型是否正确。 
(c) 用类型注释AST。

参考资料：
[shootfirst's CS143](https://github.com/shootfirst/CS143)

## 2 PA4的框架
1. `good.cl` 和 `bad.cl`
测试了semantic checker的一些特性。

2. `semant.h`
该文件包含了semantic checker的声明和定义。如果要使用自己定义的类或者结构体的话，可以将它们放在这里。

3. `cool-tree.aps` 
其包含了用于构建抽象语法树（AST）的所有接口。这个文件和PA3是一样的。同样可以自动生成 cool-tree.h 和 cool-tree.cc。

4. `cool-tree.h` 和 `cool-tree.handcode.h`
这两个文件声明了实现了Cool AST的接口以及类。在PA4中，我们需要为这两个文件中的AST类添加函数，以存储、获取和计算AST的信息。cool-tree.h 和 cool-tree.handcode.h 中已经存在的任何定义不应该被删除。

5. `cool-tree.cc`
包含了构建AST的接口。我们不应修改这个文件，而应将在`cool-tree.h`或 `cool-tree.handcode.h` 中添加的所有方法的定义放在 `semant.cc` 中。

6. `semant.cc` 
该文件是semantic checker的主要文件。主程序 main() 调用 ast_root的semant 方法，来执行semantic check。

7. `ast-lex.cc` 和 `ast-parse.cc` 
这两个文件实现了词法分析器和解析器，用于从控制台读取解析器阶段生成的AST的文本表示。

8. `semant-phase.cc` 
该文件包含了语义分析的driver程序。该程序从标准输入读取文本形式的AST，解析它，然后在标准输出上生成一个 annotated AST。

9. `symtab.h` 包含了符号表的实现。

* 综上，我们主要修改的文件是这3个，`cool-tree.h semant.h semant.cc`。
* 实验的要求是：输入没有type的ast，输出有type的ast。即每个表达式都要有type的标记，同时还有检查是否有错误。原文：
> The semantic phaseshould correctly annotate ASTs with types and should work correctly with the coolc code generator.

## 3 semantic check
我们会将语义分析阶段分为两个pass。首先，检查 inheritance graph，判断类的继承是否正确。如果`inheritance graph`有问题，那么我们直接中止编译。其次，我们再检查所有其他的语义条件，即name scope和type check。

> We suggest that you divide your semantic analysis phase into two smaller components. First, check that the inheritance graph is well-defined, meaning that all the restrictions on inheritance are satisfied. If the inheritance graph is not well-defined, it is acceptable to abort compilation (after printing appropriate error messages, of course!). Second, check all the other semantic conditions. It is much easier to implement this second component if one knows the inheritance graph and that it is legal.

### 3.1 inheritance graph
inheritance主要检查下面的错误
* 继承关系指定的类依赖的有向图必须是无环的。
* 继承关系中，base class必须先被定义。即如果A继承自B，但是B确没有被定义，那么需要报错。
* class如果被redefined，那么也需要报错
* 类名不能是SELF_TYPE;
* 不能继承Str、Bool和Int三个基本类
* Main函数未定义的情况需要报错

cycle检测方法：我们通过`unordered_map<Symbol, Symbol> inhert_graph`来构建类依赖的有向图，同时我们也需要建立Symbol到Class的映射。由于所有的class是从Object_class继承的，因此我们建立以Object_class为root的tree，从root开始遍历，这样我们的复杂度为O(n)。

### 3.2 name scope 和 type checking
完整的代码请查看下面的链接：https://github.com/firstmoonlight/CS143

#### 3.2.1 name scope
name scope和 type check我们同步进行。我们进行下面的两个任务
* 确定每个name所引用的declaration
> keep track of which declaration a name refers to.

* 改写AST为typed AST，即确定每个expression的type

在这个过程中，我们要借助SymbolTable这个类。SymbolTable是符号表，定义在symtab.h，存储标识符的名称和类别，这是它的关键接口：`enterscope()`、`exitscope()`、`addid(SYM s, DAT *i)`、`lookup(SYM s)`、`probe(SYM s)`，它一个 ClassTable，以及current_class构成了环境。

#### 3.2.2 type check
我们通过visitor模式来实现type check。在实际的工程中，visitor模式可以使得代码更加灵活和可扩展，例如我们只需要添加新的visitor类，而不需要修改 AST的node，就可以实现AST的遍历以及信息的采集。

##### 3.2.2.1 class__class
我们遍历其feature，然后对每一个feature，通过TypeCheckVisitor进行检测。
```
Symbol TypeCheckVisitor::visit(class__class& cls) {
    Features fs = cls.get_features();
    for (int i = fs->first(); fs->more(i); i = fs->next(i)) {
        fs->nth(i)->accept(*this);
    }
    return NULL;
}
```

##### 3.2.2.2 attr_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image1.png?raw=true" width="70%">

由于在`gather_attribute`的时候，已经将所有的attibute加入到scope中，因此我们只需要处理init的表达式即可。
* 获取init表达式的type
* 如果type是No_type，说明attribute没有init表达式，因此直接跳过。否则判断type是否是attribute的declare type的子类
       
> The two rules for type checking attribute definitions are similar the rules for let. The essential difference is that attributes are visible within their initialization expressions.

```
Symbol TypeCheckVisitor::visit(attr_class& cls) {
    // because we have add all attr to scope by gather_attribute, so we just check for init
    Symbol init_type = cls.get_init()->accept(*this);
    if (init_type != No_type) {
        if (init_type == SELF_TYPE) init_type = curr_class->get_name();
        if(!cls_table.is_sub_class(init_type, cls.get_type(), curr_class)) {
            cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! decl type is not ancestor of init type." << endl;
        }
    }
    return NULL;
}
```
    
##### 3.2.2.3 method_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image2.png?raw=true" width="70%">

* enterscope()，将所有的formal parameter加入到当前的scope中
* 对函数体的expression进行type check。
* 判断函数体expresison的type和return type是否匹配。
* exitscope()

```
Symbol TypeCheckVisitor::visit(method_class& cls) {
    // just check for expression
    sym_table->enterscope();
    Formals fs = cls.get_formals();
    for (int i = fs->first(); fs->more(i); i = fs->next(i)) {
        fs->nth(i)->accept(*this);
        Symbol type_decl = fs->nth(i)->get_typedecl();
        sym_table->addid(fs->nth(i)->get_name(), new Symbol(type_decl));
    }
    Symbol s = cls.get_expression()->accept(*this);
    if (!cls_table.is_sub_class(s, cls.get_returntype(), curr_class)) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! return type is not ancestor of method body." << endl;
    }
    sym_table->exitscope();
    return NULL;
}
```

##### 3.2.2.4 formal_class
由于formal_class中不存在expression，因此主要是对name和type_decl进行判断。
* 判断type_decl是否在class_table中，且其不能为SELF
* 判断name是否是重复，且不能为self

```
Symbol TypeCheckVisitor::visit(formal_class& cls) {
    // check for duplicated formal parameters and formal parameter type
    Symbol formal_name = cls.get_name();
    Symbol formal_type = cls.get_typedecl();
    if (cls_table.get_class(formal_type) == NULL) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! formal parameter type " << formal_type << " can not be found." << endl;
    } else if (sym_table->probe(formal_name) != NULL) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! formal parameter " << formal_name << " is multiply defined." << endl;
    } else if (formal_name == self) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! formal name should not be self." << endl;
    } else {
        sym_table->addid(formal_name, new Symbol(formal_type));
    }
    return NULL;
}
```
##### 3.2.2.5 assign_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image3.png?raw=true" width="70%">

* 获取Id的type T，且Id不能是self
* 计算得到表达式e1的type T'
* 检查T'是否是T的子类
* 整个表达式的类型为T'

```
Symbol TypeCheckVisitor::visit(assign_class& cls) {
    if (cls.get_name() == self) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Cannot assign to 'self'." << endl;
        return cls.set_type(SELF_TYPE)->get_type();
    }
    Symbol* type = sym_table->lookup(cls.get_name());
    if (type == NULL) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! " << cls.get_name() << " is undefined." << endl;
        return cls.set_type(Object)->get_type();
    }
    Symbol expr_type = cls.get_expression()->accept(*this);
    if (!cls_table.is_sub_class(expr_type, *type, curr_class)) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! " << *type << " is not the ancestor of the rhs expression." << endl;
        return cls.set_type(Object)->get_type();
    }
    return cls.set_type(expr_type)->get_type();
}
```

##### 3.2.2.6 dispatch_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image4.png?raw=true" width="70%">

* 对e0进行type check，获取其类型T0。
* 判断在方法空间M中，是否存在着对应的方法签名，没有则报错，并返回Object，不再进行后续的检测。
* 整个表达式的类型设置为Tn的类型。即如果返回值计算出的类型为SELF_TYPE，那么Tn的类型就是T，否则Tn的类型就是返回值计算出的类型。

```
Symbol TypeCheckVisitor::visit(dispatch_class& cls) {
    Symbol expr_type = cls.get_expression()->accept(*this);
    Symbol casted_expr = (expr_type == SELF_TYPE ? curr_class->get_name() : expr_type);
    auto it = method_tables.find(casted_expr);
    if (it == method_tables.end()) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! there is no class named " << casted_expr << "." << endl;
        return cls.set_type(Object)->get_type();
    }
    method_class* method = get_closest_method(cls_table.get_class(casted_expr), cls.get_name());
    if (method == NULL) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! there is no method " << cls.get_name() << " in class "
        << casted_expr << "." << endl;
        return cls.set_type(Object)->get_type();
    }
    // check for actual parameter with formal parameters
    Expressions exprs = cls.get_actual();
    Formals formals = method->get_formals();
    if (exprs->len() != formals->len()) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! the length of actual parameter is not match with formal parameters" << endl;
        return cls.set_type(Object)->get_type();
    }
    for (int i = exprs->first(); exprs->more(i); i = exprs->next(i)) {
        Symbol expr_type = exprs->nth(i)->accept(*this);
        if (!cls_table.is_sub_class(expr_type, formals->nth(i)->get_typedecl(), curr_class)) {
            cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! " << i << "'th actual parameter is not match with formal parameters" << endl;
            return cls.set_type(Object)->get_type();
        }
    }
    Symbol type = method->get_returntype();
    if (type == SELF_TYPE) {
        type = expr_type;
    }
    return cls.set_type(type)->get_type();
}
```

##### 3.2.2.7 static_dispatch_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image5.png?raw=true" width="70%">

static_dispatch 和 dispatch类似，区别在于，计算得到e0的type为T0，需要在T中查看method是否存在，而不是在T0中查看。

##### 3.2.2.8 cond_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image21.png?raw=true" width="70%">

* 对e1进行type check，判断其type是否为Bool
* 对e2和e3分别进行type_check
* 计算T2和T3的公共祖先节点，并作为整个表达式的type

```
Symbol TypeCheckVisitor::visit(cond_class& cls) {
    Symbol pred_type = cls.get_pred()->accept(*this);
    if (pred_type != Bool) {
        cls_table.semant_error(curr_class->get_filename(), &cls) <<
                        "Error! In 'If', the pred type should be Bool '" << endl;
    }
    // no need to check for no_type, because there is always else
    Symbol t1 = cls.get_thenexpr()->accept(*this);
    Symbol t2 = cls.get_elseexpr()->accept(*this);
    if (t1 == SELF_TYPE) {
        t1 = curr_class->get_name();
    }
    if (t2 == SELF_TYPE) {
        t2 = curr_class->get_name();
    }
    std::unordered_set<Symbol> types = {t1, t2};
    Symbol ancestor = cls_table.find_lowest_common_ancestor(Object, types);
    return cls.set_type(ancestor)->get_type();
}
```

##### 3.2.2.9 loop_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image6.png?raw=true" width="70%">

* 对e1进行type check，判断其type是否为Bool
* 对e2进行type check
* 设置loop_class的type为Object

```
Symbol TypeCheckVisitor::visit(loop_class& cls) {
    Symbol pred_type = cls.get_pred()->accept(*this);
    if (pred_type != Bool) {
        cls_table.semant_error(curr_class->get_filename(), &cls) <<
                        "Error! In 'Loop', the pred type should be Bool '" << endl;
    }
    cls.get_body()->accept(*this);
    return cls.set_type(Object)->get_type();
}
```
##### 3.2.2.10 typcase_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image7.png?raw=true" width="70%">

> Each branch of a case is type checked in an environment where variable xi has type Ti . The type of the entire case is the join of the types of its branches. The variables declared on each branch of a case must all have distinct types.

* 检查case，确保每个branch的type是独一无二的
* 对e0进行type check
* 遍历每个branch，做如下操作
    * enterscope()，将当前的x和T加入到scope中
    * 对当前branch的expression进行type check
    * exitscope()
* 计算所有branch的type的最近的公共祖先，作为整个表达式的type。计算最近公共祖先的时候，参考了[236. 二叉树的最近公共祖先](https://leetcode.cn/problems/lowest-common-ancestor-of-a-binary-tree/description/)，只需要遍历一次graph即可得到所有节点的公共祖先。

```
Symbol TypeCheckVisitor::visit(typcase_class& cls) {
    // check expression
    cls.get_expr()->accept(*this);
    std::unordered_set<Symbol> types;
    std::unordered_set<Symbol> calc_types;
    Cases cases = cls.get_cases();
    for (int i = cases->first(); cases->more(i); i = cases->next(i)) {
        Symbol type_decl = cases->nth(i)->get_typedecl();
        if (types.find(type_decl) != types.end()) {
            cls_table.semant_error(curr_class->get_filename(), cases->nth(i)) <<
                                 "Error! type '" << type_decl << "' duplicated." << std::endl;
            return cls.set_type(Object)->get_type();
        }
        Symbol type = cases->nth(i)->accept(*this);
        types.insert(type_decl);
        calc_types.insert(type);
    }
    Symbol ancestorType = cls_table.find_lowest_common_ancestor(Object, calc_types);
    return cls.set_type(ancestorType)->get_type();
}
Symbol TypeCheckVisitor::visit(branch_class& cls) {
    sym_table->enterscope();
    Symbol name = cls.get_name(), typedecl = cls.get_typedecl();
    sym_table->addid(name, new Symbol(typedecl));
    Symbol type = cls.get_expr()->accept(*this);
    sym_table->exitscope();
    return type;
}
```

##### 3.2.2.11 block_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image8.png?raw=true" width="70%">

遍历body中的每一个expression，然后将最后的那个expression的type作为body_class的type。
```
Symbol TypeCheckVisitor::visit(block_class& cls) {
    Expressions exprs = cls.get_body();
    Symbol type = NULL;
    for (int i = exprs->first(); exprs->more(i); i = exprs->next(i)) {
        type = exprs->nth(i)->accept(*this);
    }
    return cls.set_type(type)->get_type();
}
```

##### 3.2.2.12 let_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image9.png?raw=true" width="70%">

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image10.png?raw=true" width="70%">

* 对init进行type check，如果init的类型是No_type，那么说明let中没有初始化语句，我们不许要判断type_decl和init类型的关系。
* enterscope()，并将当前的identifier 和 type_decl加入到scope中
* 对body进行type check

```
Symbol TypeCheckVisitor::visit(let_class& cls) {
    Symbol identifier = cls.get_identifier();
    if (identifier == self) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! self in let binding." << std::endl;
    }
    Symbol init_type = cls.get_init()->accept(*this);
    Symbol type_decl = cls.get_typedecl();
    if (init_type != No_type && !cls_table.is_sub_class(init_type, type_decl, curr_class)) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! init type '" << init_type << "' is not a derived class of type_decl '" << type_decl << "'." << std::endl;
    }
    sym_table->enterscope();
    sym_table->addid(identifier, new Symbol(type_decl));
    Symbol type = cls.get_body()->accept(*this);
    sym_table->exitscope();
    return cls.set_type(type)->get_type();
}
```
##### 3.2.2.13 运算表达式：plus_class, sub_class, mul_class, divide_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image11.png?raw=true" width="70%">

* 计算左子式的type，判断是否为Int类型
* 计算右子式的type，判断是否为Int类型
* 设置当前的表达式为Int的类型

以mul_class为例：
```
Symbol TypeCheckVisitor::visit(mul_class& cls) {
    Symbol lhs_type = cls.get_lhs()->accept(*this);
    Symbol rhs_type = cls.get_rhs()->accept(*this);
    if (lhs_type != Int || rhs_type != Int) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! '*' meets non-Int value." << endl;
    }
    return cls.set_type(Int)->get_type();
}
```

##### 3.2.2.14 neg_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image12.png?raw=true" width="70%">

* 计算e1的type，判断其是否为Int
* 设置当前的表达式的类型为Int

```
Symbol TypeCheckVisitor::visit(neg_class& cls) {
    Symbol lhs_type = cls.get_expr()->accept(*this);
    if (lhs_type != Int) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! '~' meets non-Int value." << endl;
    }
    return cls.set_type(Int)->get_type();
}
```

##### 3.2.2.15 比较表达式：leq_class, lt_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image13.png?raw=true" width="70%">

* 计算左子式的type，判断是否为Int类型
* 计算右子式的type，判断是否为Int类型
* 设置当前的表达式为Bool的类型

以lt_class为例：
```
Symbol TypeCheckVisitor::visit(lt_class& cls) {
    Symbol lhs_type = cls.get_lhs()->accept(*this);
    Symbol rhs_type = cls.get_rhs()->accept(*this);
    if (lhs_type != Int || rhs_type != Int) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! '<' meets non-Int value." << endl;
    }
    return cls.set_type(Bool)->get_type();
}
```

##### 3.2.2.16 eq_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image14.png?raw=true" width="70%">

> The wrinkle in the rule for equality is that any types may be freely compared except Int, String and Bool, which may only be compared with objects of the same type.

对于Int, String, Bool，在equal两端的type必须是一致的。其它类型的type不做要求。

```
Symbol TypeCheckVisitor::visit(eq_class& cls) {
    Symbol lhs_type = cls.get_lhs()->accept(*this);
    Symbol rhs_type = cls.get_rhs()->accept(*this);
    if (lhs_type == Int || lhs_type == Bool || lhs_type == Str || rhs_type == Int || rhs_type == Bool || rhs_type == Str) {
        if (lhs_type != rhs_type) {
            cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! '=' meets different types." << std::endl;
        }
    }
    return cls.set_type(Bool)->get_type();;
}
```

##### 3.2.2.17 comp_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image15.png?raw=true" width="70%">

* 判断comp中的表达式e1是否为bool，如果不是bool则报错。
* 设置当前的表达式为Bool的类型
```
Symbol TypeCheckVisitor::visit(comp_class& cls) {
    Expression e = cls.get_expr();
    if (e->accept(*this) != Bool) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! 'not' meets non-Bool value." << endl;
    }
    return cls.set_type(Bool)->get_type();
}
```

##### 3.2.2.18 string_const_class, int_const_class, bool_const_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image16.png?raw=true" width="70%">

* 设置string_const的类型为`Str`
* 设置int_const的类型为`Int`
* 设置bool_const的类型为`Bool`。

```
Symbol TypeCheckVisitor::visit(string_const_class& cls) {
    return cls.set_type(Str)->get_type();
}
Symbol TypeCheckVisitor::visit(int_const_class& cls) {
    return cls.set_type(Int)->get_type();
}
Symbol TypeCheckVisitor::visit(bool_const_class& cls) {
    return cls.set_type(Bool)->get_type();
}
```

##### 3.2.2.19 new__class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image17.png?raw=true" width="70%">

判断new__class中的type_name是否是在class_table中存在，若不存在则报错，并将整个表达式的类型设置为Object，这样type check可以继续进行。否则表达式的类型就是type_name。

```
Symbol TypeCheckVisitor::visit(new__class& cls) {
    Symbol type_name = cls.get_typename();
    Symbol type = type_name;
    if (type_name != SELF_TYPE && NULL == cls_table.get_class(type_name)) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Cannot find object " << type_name << endl;
        type = Object;
    }
    return cls.set_type(type)->get_type();
}
```

##### 3.2.2.20 isvoid_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image18.png?raw=true" width="70%">

* 对e1进行type check
* 设置整个表达式的类型为`Bool`

```
Symbol TypeCheckVisitor::visit(isvoid_class& cls) {
    cls.get_expr()->accept(*this);
    return cls.set_type(Bool)->get_type();
}
```

##### 3.2.2.21 no_expr_class
即表达式不存在的情况，此时我们直接返回`No_Type`就行了。
```
Symbol TypeCheckVisitor::visit(no_expr_class& cls) {
    return No_type;
}
```

##### 3.2.2.22 object_class


<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image19.png?raw=true" width="70%">

* 获取object_class的name
* 在SymbolTable中查找，判断是否可以获取其引用

```
Symbol TypeCheckVisitor::visit(object_class& cls) {
    Symbol name = cls.get_name();
    Symbol type;
    Symbol* find_type = sym_table->lookup(name);
    if (NULL == find_type) {
        cls_table.semant_error(curr_class->get_filename(), &cls) << "Use undefined identifier " << name << endl;
        type = Object;
    } else {
        type = *find_type;
    }
   
    return cls.set_type(type)->get_type();
}
```

##### 3.2.2.14 eq_class

<img src="https://github.com/firstmoonlight/MarkdownImages/blob/main/CS143/Image20.png?raw=true" width="70%">

> The wrinkle in the rule for equality is that any types may be freely compared except Int, String and Bool, which may only be compared with objects of the same type.

对于Int, String, Bool，在equal两端的type必须是一致的。其它类型的type不做要求。

```
Symbol TypeCheckVisitor::visit(eq_class& cls) {
    Symbol lhs_type = cls.get_lhs()->accept(*this);
    Symbol rhs_type = cls.get_rhs()->accept(*this);
    if (lhs_type == Int || lhs_type == Bool || lhs_type == Str || rhs_type == Int || rhs_type == Bool || rhs_type == Str) {
        if (lhs_type != rhs_type) {
            cls_table.semant_error(curr_class->get_filename(), &cls) << "Error! '=' meets different types." << std::endl;
        }
    }
    return cls.set_type(Bool)->get_type();;
}
```

## 4 总体流程
program_class类的semant方法是语法制导翻译的入口，是语法制导翻译的核心代码，以下所有都围绕此方法展开。
* 首先调用initialize_constants()，将所有关键字加入idtable中，以免出现变量名为关键字情况。
* 其次semant方法中，实验给的代码调用了ClassTable的构造函数，传入的参数则是包括除了所有非基本类的所有类的链表classes。所以，我们必须在构造方法中将所有类加入到table中。
    * 在构造方法中，首先会调用基本类的初始化install_basic_classes()，此方法应该修改，方法最后调用调用add_class_to_classtable()，将5个基本类加入table。
    * 调用add_class_to_classtable()，然后将传入的classes链表中所有类加入table。add_class_to_classtable()中，主要是对加入时的类进行一次检查，此次检查主要是三个判断：类名不能是SELF_TYPE；不能继承Str、Bool和Int三个基本类；类不能重复定义。 PA4.pdf提到过：In addition, Cool has restrictions on inheriting from the basic classes (see the manual). It is also an error if class A inherits from class B but class B is not defined. 在满足上述三个条件后，将此class加入inherit_graph, der_2_base以及class_table中。若检查出错，则不能加入到table，注意我们的一开始写的错误检查原则。使用semant_error记录错误。
* 然后开始遍历inherit_graph，主要是检查是否无环、是否未定义Main类，是否继承的类未被定义，这三个检查必须是所有类都被加入继承图，才能检查。
    * 无环检查主要是对inherit_graph进行dfs遍历，我们以Object为起点，维持一个visited的set，每次遍历到一个节点的时候，我们就判断其是否在这个set中，如果已经遍历过了，则说明存在着环。
* 若第一次遍历出错则不能进行第二次ast遍历。第二次ast遍历更为复杂，主要是scope check和type check，注意除了二者之外还有其他的。需要用到环境，代表omc
* 第一次遍历未出错，进行第二次遍历。我们通过`visitor模式`进行AST的遍历，传入之前class_table作为参数，构造`TypecheckVisitor`。
* 从`TypecheckVisitor`的program_class开始作为入口，遍历其classes里面的所有类
* 首先进入到该类作用域,明确以下什么时候需要enterscope()和exitscope()，PA4.pdf说：Besides the identifier self, which is implicitly bound in every class, there are four ways that an objectname can be introduced in Cool:attribute definitions;formal parameters of methods;let expressions;branches of case statements。也就是说，除了 进入每个类，还要在这四个地方进行enterscope()和exitscope()。
* 对该类的所有属性进行收集gather_attribute()，放入SymbolTable符号表，因为属性在类中是全局的gather_attribute()中，注意，首先应该将所有祖先的attrubite加入，这里使用递归。然后再加入自己类的，若发生重复定义，则报错，不将其加入table，继续处理其他属性。
* 对该类的所有方法进行搜集gather_attribute()，由于方法只能定义在class中，且class中的函数是全局可见的，因此我们通过一个map记录其class和其中定义的method的关系，以便我们在dispatch和static_dispathch的时候能够快速判断所调用的method是否合法。
* 最后就是最核心的type_check部分。我们遍历AST树的每一个节点，调用accept函数来进行type check，返回其计算得到的type。



## 5 提交以及测试
将[官方评测脚本](
https://courses.edx.org/assets/courseware/v1/2aa4dec0c84ec3a8d91e0c1d8814452b/asset-v1:StanfordOnline+SOE.YCSCS1+1T2020+type@asset+block/pa3-grading.pl)复制到当前的目录assignment/pa3下，然后按如下的步骤进行测评：
* make clean
* make semant
* perl pa3-grading.pl


测试最终通过的结果如下：
```
$ perl pa3-grading.pl
Grading .....
make: 进入目录“/home/test/githubProgram/my_git_hub/CS143/usr/class/assignments/PA4”
make: 对“source”无需做任何事。
make: 离开目录“/home/test/githubProgram/my_git_hub/CS143/usr/class/assignments/PA4”
make: 进入目录“/home/test/githubProgram/my_git_hub/CS143/usr/class/assignments/PA4”
make: “semant”已是最新。
make: 离开目录“/home/test/githubProgram/my_git_hub/CS143/usr/class/assignments/PA4”
=====================================================================
submission: ..

=====================================================================
You got a score of 74 out of 74.

Submit code:
PA3-74:a88e7b09ee0cf795e75880f1c74677af
```


## 6 总结
我们可以看到，语义分析侧重语义检查，以语法分析生成的ast作为输入，进行一系列诸如类继承图检查，方法类型检查等等，同时进行表达式类型检查，为原来的ast的表达式节点添加type字段， 以经过检查的ast作为输出。
