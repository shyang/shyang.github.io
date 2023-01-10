
# Micro C Compiler 极简的C编译器

## 一、Lexical Analysis 词法解析

词法解析可以通过一条正则表达式来解析。举个例子：

```python
pattern = '''
    (?P<String>"(?:\\["n\\]|[^\"])*")|
    (?P<Char>'(?:\\['n\\]|[^'])')|
    (?P<Id>[a-zA-Z_][a-zA-Z0-9_]*)|
    (?P<Int>0[xX][0-9a-fA-F]+|[1-9][0-9]*|0[0-7]*)|
    (?P<Comment>//.*?$|/\*.*?\*/|\#.*?$)|
    \+\+|--|<<|>>|<=|>=|==|!=|&&|\|\||\[|\]|\(|\)|\{|\}|\.|&|\*|\+|-|~|!|/|%|<|>|\^|\||\?|:|;|=|,|
    (?P<Space>\s)
'''

import re
for match in re.finditer(pattern, sourceCode, re.DOTALL | re.MULTILINE)
    # match.lastgroup 是匹配到的类别 'String'|'Char'|'Id'|'Comment'|'Space'
    # match.group(0) 是匹配到的内容，如 "abc" | 'c' | x | // 注释 | 空格换行

```

这条正则表达式可以匹配一个精简版的C语言词法（比如只支持 "\n"，没有浮点数等）。

使用正则表达式易于理解，缺点是通用库的实现效率不是最高。



### 二、Parser 语法分析

语法规则通常含有递归结构，如：
```
primary-expression ::= id | string | '(' expression ')'
expression ::= primary-expression | 其它规则
```
这种情况下 regular expression 就无能为力，最简单的手写方式是通过递归的函数来解析，如：
```python
def primary_expr():
    if match(['Id', 'Int', 'Char', 'String']):
        ...
    else:
        expect('(')
        expr()
        expect(')')
```

这种自顶向下手写解析规则的方式无法直接解析某些有歧义的推导规则的：[C .bnf grammar. Version 1](https://gist.github.com/arslancharyev31/c48d18d8f917ffe217a0e23eb3535957)

### 歧义1:

```
cast-expression ::= unary-expression
                    | '(' type-name ')' cast-expression

其中
unary-expression ::= postfix-expression | ...
postfix-expression ::= primary-expression | ...
primary-expression ::= '(' expression ')' | ...

unary-expr 一路推导到 primary-expr 一样可以有 '(' 作为前缀

'(' x ')' 比如当 x 是变量时，走 unary-expr -> primary-expr 的分支
当 x 是一个类型时，代表走 cast-expr 的分支
```



解决方案1：允许 backtrack
```
def cast_expr():
    while peek('('):
        p = get_pos() # 保存当前 token 的位置
        try:
            type_name()
            expect(')')
        except Exception:
            restore_pos(p) # 前一条规则解析失败时，重置 token 位置，尝试走其它规则
            break
    unary_expr()
```
解决方案2: 精简类型系统，确保 look ahead 两个 token 可以区分 x 是否是一个类型
```
def cast_expr():
    if peek('(', ['char', 'int', 'void']):
        type_name()
        expect(')')
    unary_expr()
```

### 歧义2:

```
assignment-expression ::= conditional-expression
                          | unary-expression assignment-operator assignment-expression

```
unary_expr 也是 conditional_expr 的前缀

解决方案：将 conditional_expr 到 unary_expr 的所有规则展开

https://en.wikipedia.org/wiki/Operator-precedence_parser#Precedence_climbing_method

```
def expr(preced='='):
    unary_expr()

    while current_preced() >= preced:
        if match('='):          # assignment_expr
            expr('=')
        elif match('?'):        # conditional_expr
            expr('=')
            expect(':')
            expr('?')
        elif match('||'):       # logic_or_expr
            expr('&&')
        elif match('&&'):       # logic_and_expr
            expr('|')
        ... 其它中间规则
```

除去这若干有歧义的地方需要些技巧外，其它规则与代码基本上是一一对应。

其它规则示例：编译单元

```python
translation-unit ::= {external-declaration}*

对应代码
def translation_unit():
    while not match('EOF'):
        external_declaration()
```

全局声明的规则：
```python
external-declaration ::= function-definition | declaration
function-definition ::= {declaration-specifier}* declarator compound-statement
declaration ::=  {declaration-specifier}+ {declarator}*

对应代码：
# function_definition 与 declaration 有相同前缀，提取公因子，避免回溯
def external_declaration():
    declaration_specifiers()
    if match(';'):
        return
    declarator()
    if peek('{'):   # function_definition
        compound_stmt()
    else:           # declaration
        while match(','):
            declarator(sym)
        expect(';')
```




### 三、代码生成
可参考 https://github.com/rswier/c4 中设计的虚拟机，相对小巧，便于理解。

```
累加寄存器 acc
程序计数器 pc
栈寄存器   sp, bp
```

实际会生成代码的只有各种 stmt 与 expr。各种声明只是建立符号表，保存其地址、常量、类型等。
如 y = x + 1 实际被解析为

```
   stmt
    expr_stmt
     expr
      primary_expr<Id=y>    IMM LI
     expr<=>                PSH ... SI    删除前一个LI，使右值变左值
      expr
       primary_expr<Id=x>   IMM LI
      expr<+>               PSH ... ADD
       expr
        primary_expr<Int=1> IMM 1
    expr_stmt<;>
```

最终的汇编形式：
```
00016     IMM  1000008  若y是全局变量   acc <- 其地址
00032     PSH           stk 保存 y 的地址
00040     IMM  1000000  若x是全局变量   acc <- 其地址
00056     LI            读入 int        acc = x
00064     PSH           stk 保存 x 的内容
00072     IMM  1        acc <- 1
00088     ADD           acc <- acc + stk.pop()
00096     SI            stk.pop() 的地址（即y）存入 acc, y = acc
```


### 四、代码执行
如果代码生成阶段的目标语言是当前 CPU 的架构，这步由当前系统直接运行。
如果是虚拟机代码，至少还需要写一个解释器。

内存布局设计：
```
[ 代码区 text ][ 全局变量区 data ][ 堆栈区 stack ][ malloc 区，可有多个 ]
text, data, malloc 都是从 0 开始向上生长
stack 区按习惯 sp 指向最高地址向下生长
```

### 传递参数 argc, argv，并调用 main 函数
```
argv 指向的多个字符串放入 data 区
argv 指针本身与 argc push 进 stack 区，做为参数
pc <- main 的地址，开始执行
```



### 一般函数的调用栈
```
调用方
    push 参数           <- bp+2, bp+3, ...
JSR addr
    push 返回地址       <- bp+1
    pc <- addr
ENT nlocals
    push bp             <- bp
    bp = sp
    sp -= nlocals 个局部变量  <- bp-1, bp-2, ...

读局部变量使用：
LEA 偏移
    局部变量与参数之间差 2 个 word
    局部变量    LEA -1, -2, -3, ...
    参数        LEA 2, 3, 4, ...
```



### 执行状态机
```
while True:
    instruction = read_int(pc)
    解码并实现该指令

```
使用虚拟机优点是不依赖具体 cpu 与操作系统，方便快速理解学习。
