
其实工作了这么久，自己是什么水平，心里是再清楚不过了。非科班出生，工作的前两年做的又是偏向嵌入式应用方面的工作，所以很多时候，面对着浩瀚的计算机知识海洋，总是感到力所不逮。从MIT6.828到CS143，这两个课程中，更是看出了自己的真实水平，其实我自己心里也清楚，在未参考答案的情况下，有几个assigment我自己是做不出来的，有时候我总在想，自己要是在大学或者研究生时期久确定了自己以后从事的方向，如果那时候就遇到这些课程，也许会有所不同的，当然也可能还是现在这样。人总是走在命运规定的道路上。

这个笔记记录下自己的心得，供自己后面回忆，如若有缘人刷到这篇笔记，望见谅，不能提供有用的信息，只是一些总结以及心得。

参考资料：
[CS143：编译原理｜PA2：正则表达式和词法分析](https://zhuanlan.zhihu.com/p/258385544)

## 1 前言
我在做这次的assignment之前，首先看了《flex&bison》的第一章和第二章，对flex有了初步的理解之后，理解题意，参考答案，以及做题，都有很大的提升。

## 2 PA2整体架构
1. assignments/PA2下Makefile
    * make  lexer --- 编译生成可执行文件lexer，即词法分析器。这个程序接受.cl文件作为参数，然后对这个.cl文件进行解析。
    * make dotest --- 编译lexer，然后使用lexer对test.cl进行词法分析。

2. cool.flex
flex文件，符合flex的格式，我们将这个文件通过flex生成C代码，即能解析.flex文件中注明的正则表达式的有限状态机代码，我们可以使用这部分代码进行编译，生成词法分析器lexer。

3. lextest.cc
主函数所在的文件。main函数通过fopen来打开文件，然后调用flex生成的`cool_yylex()`函数字符匹配。`cool_yylex()`的返回值我们可以通过.flex文件自己控制。其返回值应该是一个token标记，标记匹配到的字段是operator，keyword还是identifier等等，之后将匹配得到的字段调用`dump_cool_token`函数进行处理。

4. test.cl
cool语言测试代码，用于测试你的flex的代码是否能够全部且正确分析得到所有的字段。

5. list.h
这个文件实现了一个简单的链表。提供三个简单的函数，构造函数List::List，将一个新的元素加入到链表的头部；hd()，返回链表的头部；tl()，返回链表的尾部。list_length(list<T>* l), 返回链表的长度；list_print(S& str, List<T>* l)，打印链表l
的每个元素到str中；list_map(void f(T*), List<T>* l)，对链表的每个元素调用函数f。在string table和symbol table中会用到这个数据结构。

6. String Tables（stringtab.h)
String Table用来存储identifier，numerical constants和string constants。在这个文件中，String Table使用的是类型Entry -- 每个Entry存储了string，length of string，and an integer index unique to the string。
这个文件中我们提供了3种类型的String Table，分别是IdTable（用于存储identifier的信息），StrTable（用于存储String Constant的信息），IntTable（用于存储Numerical Constant的信息）。每个String Table使用不同的element type（IdEntry, StringEntry, IntEntry），这三种类型都是继承于Entry。
String Table通过上述的List数据结构存储元素，其提供了三种方法来将element加入到String Table中。
* add_string(char* s, int m) 将字符串s的最多m个字符，加入到String Table中。
* add_string(char* s) 将字符串s加入到String Table中。
* add_int(int i) 将integer i转化为字符串，并将其加入到String Table中。

7. Symbol Tables （symtab.h）
除了String Table之外，我们还需要一个Symbol Table来string 的 scope。它的key是程序的name，而value则表示关联到这个name的信息，例如类型信息等等。除了添加和删除symbol，Symbol Table同样需要支持进入和退出scope，以及判断一个symbol是否已经在当前的scope定义。
每个scope是一个链表，链表元素为<identifier, data>，data表示我们想要的每个identifier的信息。可能到PA4才会用到Symbol Table。


## 3 cool.flex

1. cool_yylex返回的是token对应的数字，在parse.h中定义的。
2. cool_yylval.symbol可以存储token对应的符号表入口。
3. cool_yylval.boolean存储bool值，cool_yylval.error_msg存储错误信息
4. token, cool_yylval, curr_lineno就是一次匹配得到的所有信息，代表匹配了什么语句、语句包含了什么额外信息、语句在哪一行，匹配的行为由cool.flex中的代码决定。


### 3.1 nested comments
这边参考了《flex&bison》的第2章，"C语言交叉引用"的小case。
```
     /*
      *  Nested comments
      */
"(*"                    { BEGIN(COMMENT); }
<COMMENT>"*)"           { BEGIN(INITIAL); }
<COMMENT>.
<COMMENT>\n             { curr_lineno++; }
<COMMENT><<EOF>>        { BEGIN(INITIAL); 
cool_yylval.error_msg = "EOF in comment"; return ERROR; }
"--".*                 {}
```
第一个规则在看到`(*`的时候激活COMMENT状态，而在第二个规则遇到`*)`时切换回正常的INITIAL状态，第三个规则匹配两者之间的一切字符。如果匹配到换行符，则维护lineno。规则<COMMENT><<EOF>>发现和报告没有终结的注释。

### 3.2 The multiple-character operators
多字符的操作符主要包括三种<=,<-,=>，在cool-parse.h中分别命名为LE,ASSIGN和DARROW。
```
{DARROW} { return (DARROW); }
{ASSIGN} { return (ASSIGN); }
{LE} { return (LE); }
```

### 3.3 key Words
这些type在cool-parse.h中定义，yytokentype。我们返回其token的值，在dump_cool_token中，会解析这个token并进行打印。
```
{CLASS} { return (CLASS); }
{ELSE} { return (ELSE); }
{FI} { return (FI); }
{IF} { return (IF); }
{IN} { return (IN); }
{INHERITS} { return (INHERITS); }
{LET} { return (LET); }
{LOOP} { return (LOOP); }
{POOL} { return (POOL); }
{THEN} { return (THEN); }
{WHILE} { return (WHILE); }
{CASE} { return (CASE); }
{ESAC} { return (ESAC); }
{OF} { return (OF); }
{NEW} { return (NEW); }
{ISVOID} { return (ISVOID); }
{NOT} { return (NOT); }
```

### 3.4 objid, typeid和int
符号信息保存在符号表中，符号表的结构请看文件include/PA2/stringtab.h。已经定义好了3个全局变量，分别代表类名变量名表、整数表、字符串表，定义在stringtab.h末尾。
类名TYPEID以大写字母开头，变量名OBJECTID以小写字母开头，以此区分两者。
```
    /* OBJECTID, TYPEID, INT_CONST */
[0-9]+                      {
                                cool_yylval.symbol = inttable.add_string(yytext, yyleng);
                                return INT_CONST;
                            }
[A-Z_][a-zA-Z0-9_]*         {
                                cool_yylval.symbol = idtable.add_string(yytext, yyleng);
                                return TYPEID;
                            }
[a-z][a-zA-Z0-9_]*          {
                                cool_yylval.symbol = idtable.add_string(yytext, yyleng);
                                return OBJECTID;
                            }
```

### 3.5 BOOLEAN常量true和false
它们是特殊的OBJECTID，携带的信息直接进入全局变量cool_yylval。cool-manual提醒我们，它们的起始字母必须小写，但后面的字母可大写可小写。
```
t[Rr][Uu][Ee] {
  cool_yylval.boolean = true;
  return (BOOL_CONST);}

f[Aa][Ll][Ss][Ee] {
  cool_yylval.boolean = false;
  return (BOOL_CONST);}
```

### 3.6 字符串字面量
* 定义两状态，分别用于开启String匹配，以及String中的转义字符匹配。
```
%x STRING
%x STRING_ESCAPE
```

* 在初始状态下的引号“触发进入STRING状态：
```
\"                          {
                                BEGIN(STRING); 
                                memset(string_buf, 0, sizeof(MAX_STR_CONST));
                                string_buf_ptr = string_buf;
                            }
```

* 当在没有转义符的情况下遇到引号”，字符串读取结束。
```
<STRING>[^\"\\]*\"          {
                                        /* does not include the last character */
                                        memcpy(string_buf_ptr, yytext, yyleng - 1);
                                        string_buf_ptr += yyleng - 1;
                                        // cout << "string_buf is " << string_buf << endl;
                                        // cout << "leng is " <<  string_buf_ptr - string_buf + 1 << endl;
                                        cool_yylval.symbol = stringtable.add_string(string_buf, string_buf_ptr - string_buf + 1);
                                        BEGIN(INITIAL);
                                        return STR_CONST;
                                    }
```

* 若遇见转义符，应进入转义符处理状态
```
<STRING>[^\"\\]*\\                  {
                                        // does not include the last character escape
                                        memcpy(string_buf_ptr, yytext, yyleng - 1);
                                        string_buf_ptr += yyleng - 1;
                                        BEGIN(STRING_ESCAPE);
                                    }
```
* 转义字符后面匹配单个字符，并记录其转义之后的字符
```
<STRING_ESCAPE>n                    {
                                        // cout << "escape \\n !" << endl;
                                        *(string_buf_ptr++) = '\n';
                                        BEGIN(STRING);
                                    }

<STRING_ESCAPE>b                    {
                                        *(string_buf_ptr++) = '\b';
                                        BEGIN(STRING);
                                    }

<STRING_ESCAPE>t                    {
                                        *(string_buf_ptr++) = '\t';
                                        BEGIN(STRING);
                                    }

<STRING_ESCAPE>f                    {
                                        *(string_buf_ptr++) = '\f';
                                        BEGIN(STRING);
                                    }
```
* 其余字符，我们返回其原型，相当于没有转义
```
<STRING_ESCAPE>.                    {
                                        *(string_buf_ptr++) = yytext[0];
                                        BEGIN(STRING);
                                    }      
```

* 若转义符\后是换行符\n，应作换行符处理：
```
<STRING_ESCAPE>\n                   {
                                        *(string_buf_ptr++) = '\n';
                                        ++curr_lineno;
                                        BEGIN(STRING);
                                    }
```

* 字面量中出现终止符\0应作错误处理：
```
<STRING_ESCAPE>0 {
    cool_yylval.error_msg = "String contains null character";
    BEGIN(STRING);
    return (ERROR);
}
```

* 字面量行尾无转义符、无引号”，字符串前后引号不匹配，作错误处理：
```
<STRING>[^\"\\]*$ {
    // push first
    // contains the last character for yytext does not include \n
    //setup error later
    cool_yylval.error_msg = "Unterminated string constant";
    BEGIN(INITIAL);
    ++curr_lineno;
    return (ERROR);
}

```
* 出现EOF应作错误处理：
```
<STRING_ESCAPE><<EOF>> {
    cool_yylval.error_msg = "EOF in string constant";
    BEGIN(INITIAL);
    return (ERROR);
}

<STRING><<EOF>> {
    cool_yylval.error_msg = "EOF in string constant";
    BEGIN(INITIAL);
    return (ERROR);
}
```

### 3.7 剩余的单个合法字符和非法字符
* 非法字符如下：
```
    /* illegal characters */
[\[\]\'>]                   {
                                cool_yylval.error_msg = yytext;
                                return (ERROR);
                            }
```
* 空白字符
```
    /* blank */
[ \t\f\r\v]                 {}

```
* 换行符，更新lineno
```
\n                          { curr_lineno++; }
    /* legal characters*/
```

* 合法字符可以直接返回字符本身的ASCII码：
```
.                           {
                                return yytext[0];
                            }
```                            
