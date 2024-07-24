---
layout: post
title: Assignment PA3
tags: [CS143]
---

## 1 前言
这次assignment要求我们实现compiler的parse阶段。首先我们先确定整个代码的框架，然后阅读《A Tour of the Cool Support Code》整个handout，来确定我们所使用的用来构建AST树的接口，最后阅读cool-manual来确定parser rule，根据每一条rule，我们确定其action，并创建其节点，最终我们生成一棵AST树。

参考资料：
[Stanford Compiler PA3](https://doraemonzzz.com/2021/04/24/2021-4-24-Stanford-Compiler-PA3)
[Using the Error Token in Bison](http://marvin.cs.uidaho.edu/Teaching/CS445/bisonErrorToken.html)

## 2 PA3的整个框架
1. assignments/PA3下Makefile
* make parser
 --- 编译生成可执行文件parser。
* make dotest --- 编译parser，然后调用脚本`./myparser good.cl`和`./myparser bad.cl`来测试assignment中的两个测试文件good.cl和bad.cl。

2. cool.y
这是我们的parser的主要部分，目前已经有部分的production作为我们的示例。如`program : class_list`作为整个程序的入口，生成我们的AST的根节点，`ast_root`。

3. good.cl和bad.cl
这是我们的两个测试文件，但是仅仅只是测试一些feature，对于整体的语法测试并不完全。但是bad.cl中包含了一些错误的语法，需要我们对所有的错误都能够及时发现并报错。

4. cool-tree.aps
cool-tree.aps文件通过专门的工具自动生成相应的C++代码文件，即cool-tree.h和cool-tree.cc，这些代码文件构成了编译器的抽象语法树部分，为后续编译器实现提供了基础。其中比较重要的时`cool-tree.cc`，构造AST的接口都在里面，我们只需要调用其中的接口来构建AST即可。

5. tokens-lex.cc 
这是一个lexer，能够从控制台读取由词法分析阶段生成的格式化的标记流。

6. parser-phase.cc 包含一个用于测试解析器的驱动程序。
7. dumptype.cc 这部分代码将AST打印到stream中，打印的部分是可阅读。
8. handle_flags.cc 用来解析命令行的flag。


## 3 cool.y
当我们遇到一个比较大的问题的时候，最好还是将其划分为一个个小问题，通过解决一个个的小问题，我们最终能够不断逼近我们的最终的答案。

对于构建AST这样一个复杂而繁琐的任务的时候，我们先按照3部分进行，首先是cool-manual的阅读，从中我们可以得到我们所需要的BNF；其次，我们通过阅读cool-tree.c和cool-tree.h文件，得到我们构建AST所需要的接口；之后，我们先按照BNF完整的构建我们的AST树，暂时不考虑error handling；最后我们再处理error handling的问题。

### 3.1 BNF
通过阅读cool-manual，发现其BNF如下所示：
```
program ::= [[class; ]]+ 
class ::= class TYPE [inherits TYPE] { [[feature; ]]∗ } 
feature ::= ID( [ formal [[, formal]]∗ ] ) : TYPE { expr } 
            | ID : TYPE [ <- expr ] 
formal ::= ID : TYPE 
expr ::= ID <- expr 
            | expr[@TYPE].ID( [ expr [[, expr]]∗ ] ) 
            | ID( [ expr [[, expr]]∗ ] ) 
            | if expr then expr else expr fi 
            | while expr loop expr pool 
            | { [[expr; ]]+} 
            | let ID : TYPE [ <- expr ] [[,ID : TYPE [ <- expr ]]]∗ in expr 
            | case expr of [[ID : TYPE => expr; ]]+ esac 
            | new TYPE 
            | isvoid expr 
            | expr + expr 
            | expr − expr 
            | expr ∗ expr 
            | expr / expr 
            | ˜expr 
            | expr < expr 
            | expr <= expr 
            | expr = expr 
            | not expr 
            | (expr) 
            | ID 
            | integer 
            | string 
            | true 
            | false
```

### 3.2 构建每条production所对应的action
完整的代码请查看下面的链接：`https://github.com/firstmoonlight/CS143/blob/main/usr/class/assignments/PA3/cool.y`
#### 3.2.1 attr对应的action
```
    /* feature ::= ID : TYPE [ <- expr ]  */
    attr_feature : OBJECTID ':' TYPEID
            {
                $$ = attr($1, $3, no_expr());
            }
        | OBJECTID ':' TYPEID ASSIGN expression
            {
                $$ = attr($1, $3, $5);
            }
    ;
```
#### 3.2.2 method对应的action
```
    /*  feature ::= ID( [ formal [[, formal]]∗ ] ) : TYPE { expr }  */
    feature : attr_feature
        | OBJECTID '(' ')' ':' TYPEID '{' expression '}'
            {
                $$ = method($1, nil_Formals(), $5, $7);
            }
        | OBJECTID '(' formal_list ')' ':' TYPEID '{' expression '}'
            {
                $$ = method($1, $3, $6, $8);
            }
    ;
    
    formal_list : formal
        {
            $$ = single_Formals($1);
        }
        | formal_list ',' formal
        {
            $$ = append_Formals($1, single_Formals($3));
        }
    ;

    /* formal ::= ID : TYPE */
    formal : OBJECTID ':' TYPEID
        {
            $$ = formal($1, $3);
        }
    ;
```
#### 3.2.3 expression对应的action
let这个expression有点难度，因为我们在cool-tree.c中只有这样一个接口，
`Expression let(Symbol identifier, Symbol type_decl, Expression init, Expression body)`，但是let其实是可以接受多个identifier, type_decl, init的。后来查看了标准答案，发现除了第一个identifier, type_decl, init，其他部分作为新的expression，加入到body中。

```
expression 
        // ID <- expr
        : OBJECTID ASSIGN expression
            {
                $$ = assign($1, $3);
            }
        // expr[@TYPE].ID( [ expr [[, expr]]∗ ] ) 
        | expression '.' OBJECTID '(' ')'
            {
                $$ = dispatch($1, $3, nil_Expressions());
            }
        | expression '.' OBJECTID '(' expression_list ')'
            {
                $$ = dispatch($1, $3, $5);
            }
        | expression '@' TYPEID '.' OBJECTID '(' ')'
            {
                $$ = static_dispatch($1, $3, $5, nil_Expressions());
            }
        | expression '@' TYPEID '.' OBJECTID '(' expression_list ')'
            {
                $$ = static_dispatch($1, $3, $5, $7);
            }
        // ID( [ expr [[, expr]]∗ ] ) 
        | OBJECTID '(' ')'
            {
                $$ = dispatch(object(idtable.add_string("self")), $1, nil_Expressions());
            }
        | OBJECTID '(' expression_list ')'
            {
                $$ = dispatch(object(idtable.add_string("self")), $1, $3);
            }
        //  if expr then expr else expr fi 
        | IF expression THEN expression ELSE expression FI
            {
                $$ = cond($2, $4, $6);
            }
        // while expr loop expr pool 
        | WHILE expression LOOP expression POOL
            {
                $$ = loop($2, $4);
            }
        // { [[expr; ]]+} 
        | '{' expressions '}'
            {
                $$ = block($2);
            }
        // let ID : TYPE [ <- expr ] [[,ID : TYPE [ <- expr ]]]∗ in expr 
        | LET let_
            {
                $$ = $2;
            }
        // case expr of [[ID : TYPE => expr; ]]+ esac 
        | CASE expression OF branch_list ESAC 
            {
                $$ = typcase($2, $4);
            }
        // new TYPE 
        | NEW TYPEID
            {
                $$ = new_($2);
            }
        // isvoid expr 
        | ISVOID expression
            {
                $$ = isvoid($2);
            }
        // expr + expr 
        | expression '+' expression
            {
                $$ = plus($1, $3);
            }
        // expr - expr 
        | expression '-' expression
            {
                $$ = sub($1, $3);
            }
        // expr * expr 
        | expression '*' expression
            {
                $$ = mul($1, $3);
            }
        // expr / expr 
        | expression '/' expression
            {
                $$ = divide($1, $3);
            }
        // ~expr 
        | '~' expression
            {
                $$ = neg($2);
            }
        // expr < expr 
        | expression '<' expression
            {
                $$ = lt($1, $3);
            }
        // expr <= expr 
        | expression LE expression
            {
                $$ = leq($1, $3);
            }
        // expr = expr 
        | expression '=' expression
            {
                $$ = eq($1, $3);
            }
        // not expr
        | NOT expression
            {
                $$ = comp($2);
            }
        // (expr)
        | '(' expression ')'
            {
                $$ = $2;
            }
        // ID
        | OBJECTID
            {
               $$ = object($1);
            }
        // integer
        | INT_CONST
            {
                $$ = int_const($1);
            }
        // string
        | STR_CONST
            {
                $$ = string_const($1);
            }
        // true, false
        | BOOL_CONST
            {
                $$ = bool_const($1);
            }
    ;

    expression_list : expression
            {
                $$ = single_Expressions($1);
            }
        | expression_list ',' expression
            {
                $$ = append_Expressions($1, single_Expressions($3));
            }
    ;

    expressions : expression ';'
            {
                $$ = single_Expressions($1);
            }
        | expressions expression ';'
            {
                $$ = append_Expressions($1, single_Expressions($2));
            }
        | error ';'
            {}
    ;

    let_ : OBJECTID ':' TYPEID IN expression
            {
            $$ = let($1, $3, no_expr(), $5);
            }
        | OBJECTID ':' TYPEID ASSIGN expression IN expression
            {
            $$ = let($1, $3, $5, $7);
            }
        | OBJECTID ':' TYPEID ',' let_
            {
            $$ = let($1, $3, no_expr(), $5);
            }
        | OBJECTID ':' TYPEID ASSIGN expression ',' let_
            {
            $$ = let($1, $3, $5, $7);
            }
        | error ',' let_
            {
                $$ = $3;
            }
    ;
 
    branch_list : OBJECTID ':' TYPEID DARROW expression ';'
            {
                $$ = single_Cases(branch($1, $3, $5));
            }
        | OBJECTID ':' TYPEID DARROW expression ';' branch_list
            {
                $$ = append_Cases(single_Cases(branch($1, $3, $5)), $7);
            }
    ;

```


### 3.3 error handling
在PA3的`5 Error Handling`中，要求我们完成以下两个目标：
* 如果一个类定义中有错误，但该类正确地结束了，并且下一个类在语法上是正确的，那么解析器应该能够从下一个类定义重新开始。
* 同样地，解析器应该能够从feature中的错误恢复（继续解析下一个特征）、从let binding中的错误恢复（继续解析下一个变量）以及从`{...}`的block中的表达式错误中恢复。
 

这两部分对应如下三种含义：
 * 如果处理class时报错，应该忽略该错误，然后处理后续的class。
 * 如果处理feature时报错，则忽略该错误，然后处理后续的feature。
 * 如果处理let时报错，则忽略该错误，然后处理后续的let。
 * 如果处理expression时报错，则忽略该错误，然后继续处理后续的expression
 
 代码如下：
 ```
     /* class ::= class TYPE [inherits TYPE] { [[feature; ]]∗ } */
    class	: CLASS TYPEID '{' dummy_feature_list '}' ';'
            {
                $$ = class_($2,idtable.add_string("Object"),$4, stringtable.add_string(curr_filename)); 
            }
        | CLASS TYPEID INHERITS TYPEID '{' dummy_feature_list '}' ';'
            {
                $$ = class_($2,$4,$6,stringtable.add_string(curr_filename)); 
            }
        | CLASS error ';' class
            {
                $$ = $4;
            }
    ;
    
    feature_list : feature ';' /* single feature */
            {
                $$ = single_Features($1);
            }
        | feature_list feature ';'
            {
                $$ = append_Features($1, single_Features($2));
            }
        | error ';'
            {
                $$ = $1;
            }
    ;

    let_ : OBJECTID ':' TYPEID IN expression
            {
            $$ = let($1, $3, no_expr(), $5);
            }
        | OBJECTID ':' TYPEID ASSIGN expression IN expression
            {
            $$ = let($1, $3, $5, $7);
            }
        | OBJECTID ':' TYPEID ',' let_
            {
            $$ = let($1, $3, no_expr(), $5);
            }
        | OBJECTID ':' TYPEID ASSIGN expression ',' let_
            {
            $$ = let($1, $3, $5, $7);
            }
        | error ',' let_
            {
                $$ = $3;
            }
    ;
   
   
    expressions : expression ';'
            {
                $$ = single_Expressions($1);
            }
        | expressions expression ';'
            {
                $$ = append_Expressions($1, single_Expressions($2));
            }
        | error ';'
            {}
    ;
```
 

### 3.4 shift/reduce conflict
在parse的过程中，发现了shift reduce conflict。
如下图所示，cool.output中显示，如果expression后面出现了`LE`等符号的话，bison无法判断是应该应用rule 31来进行reduce还是应该应用rule 38来进行。

这是因为我们没有定义isvoid和LE等符号的优先级以及结合性。
```
   19 expression: expression . '.' OBJECTID '(' ')'
   20           | expression . '.' OBJECTID '(' expression_list ')'
   21           | expression . '@' TYPEID OBJECTID '(' ')'
   22           | expression . '@' TYPEID OBJECTID '(' expression_list ')'
   31           | ISVOID expression .
   32           | expression . '+' expression
   33           | expression . '-' expression
   34           | expression . '*' expression
   35           | expression . '/' expression
   37           | expression . '<' expression
   38           | expression . LE expression
   39           | expression . '=' expression
   
    LE   shift, and go to state 69
    '.'  shift, and go to state 70
    '@'  shift, and go to state 71
    '+'  shift, and go to state 72
    '-'  shift, and go to state 73
    '*'  shift, and go to state 74
    '/'  shift, and go to state 75
    '<'  shift, and go to state 76
    '='  shift, and go to state 77

    LE        [reduce using rule 31 (expression)]
    '.'       [reduce using rule 31 (expression)]
    '@'       [reduce using rule 31 (expression)]
    '+'       [reduce using rule 31 (expression)]
    '-'       [reduce using rule 31 (expression)]
    '*'       [reduce using rule 31 (expression)]
    '/'       [reduce using rule 31 (expression)]
    '<'       [reduce using rule 31 (expression)]
    '='       [reduce using rule 31 (expression)]
    $default  reduce using rule 31 (expression)
```

### 3.5 优先级
cool-manul的11.1 Precedence给出了各个运算符的优先级以及结合性。
从最高到最低的中缀二元运算和前缀一元运算的优先级：
```
.
@
~
isvoid
* /
+ -
<= < =
not
<-
```

所有二元运算都是左结合的，除了赋值是右结合的，还有三个比较运算是没有结合性的。

据此给出代码中的运算符优先级，注意bison中优先级是从低到高排列。
值得注意的是IN操作符，虽然在manual中没有给出其优先级以及结合性，但是为了处理bison的shift/reduce conflict，我们必须确定其优先级，目前我暂时设置其为最低的优先级。
```
    /* Precedence declarations go here. */
    %nonassoc IN
    %right ASSIGN
    %left NOT
    %nonassoc LE '<' '='
    %left '+' '-'
    %left '*' '/'
    %left ISVOID
    %left '~'
    %left '@'
    %left '.'
```

## 4 提交以及测试
将[官方评测脚本](https://courses.edx.org/asset-v1:StanfordOnline+SOE.YCSCS1+1T2020+type@asset+block@pa2-grading.pl)复制到当前的目录`assignment/pa3`下，然后按如下的步骤进行测评：
* make clean
* make lexer
* make parser
* perl pa2-grading.pl

需要注意的是测评脚本需要安装perl以及c shell。
测试是否已经安装c shell和perl
```
// csh
csh --version

// perl
perl -v

```

在ubuntu下的安装命令如下:
```
// c shell
sudo apt update 
sudo apt install csh

// perl
sudo apt update 
sudo apt install perl
```

测试最终通过的结果如下：
```
$ perl pa2-grading.pl
Grading .....
make: 进入目录“/home/test/githubProgram/my_git_hub/CS143/usr/class/assignments/PA3”
make: “parser”已是最新。
make: 离开目录“/home/test/githubProgram/my_git_hub/CS143/usr/class/assignments/PA3”
=====================================================================
submission: ..

=====================================================================
You got a score of 70 out of 70.

Submit code:
70:7cbbc409ec990f19c78c75bd1e06f215


```

## 5 总结
目前我们已经将输入解析为AST树，这颗树中保存了所有的原始信息。AST树的建立表明输入是符合Syntax的，但是我们还需要进一步进行检查，确定其满足Semantic之后，才能进行gen code。
