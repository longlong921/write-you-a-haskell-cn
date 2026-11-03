# 附录 A　表层语法

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/appendix-a-syntax.html) · [原文源码](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/book/src/appendix-a-syntax.md)

想知道一段程序在 Fun 里该怎么写，可以来这里查。下面收录的是最终版 Fun 的语法。写源码时可以用缩进，读这些规则时要想象编译器已经按缩进补好了花括号和分号。一段代码即使符合语法，里面的名字、种类（kind）和类型也得经得起检查。各节在语法之后列出额外限制，类型检查的具体规则见[附录 B](appendix-b-typing.md)。

## 怎么读这些规则

看 `program ::= ...` 时，可以把 `::=` 念成“可以写成”。单引号括起来的是要在源码里写出的符号或关键字。没加引号时，`|` 表示“任选一项”，`[ X ]` 表示“`X` 可以省略”，`{ X }` 表示“`X` 可以出现零次或多次”，圆括号则用来分组。比如 `'{'` 所代表的是源码里的左花括号，而 `{ ';' topdecl }` 说的是“还可以再写几条顶层声明，每条前面带一个分号”。方括号加了引号时，也要原样写进源码。`varid`、`conid` 和字面量都在最后一节解释。

整份源文件即使省略最外层的花括号，布局处理也会替它补上。所以无论外层花括号写没写，都用下面同一条 `program` 规则。

## 声明

```text
program     ::= '{' [ topdecl { ';' topdecl } ] [ ';' ] '}'
topdecl     ::= datadecl | typedecl | classdecl | instdecl
              | fixity | sig | bind

datadecl    ::= 'data' tycon { tyvar } '=' constr
                { '|' constr } [ deriving ]
constr      ::= con { atype }
              | con '{' fielddecl { ',' fielddecl } '}'
fielddecl   ::= varid { ',' varid } '::' type
deriving    ::= 'deriving' conid
              | 'deriving' '(' conid { ',' conid } ')'
typedecl    ::= 'type' conid { tyvar } '=' type

classdecl   ::= 'class' [ context '=>' ] conid tyvar
                [ 'where' '{' [ member { ';' member } ] [ ';' ] '}' ]
member      ::= sig | bind
instdecl    ::= 'instance' [ context '=>' ] conid insthead
                [ 'where' '{' [ bind { ';' bind } ] [ ';' ] '}' ]
insthead    ::= conid | '(' conid { tyvar } ')'
              | '[' tyvar ']' | '(' ')'
              | '(' tyvar ',' tyvar [ ',' tyvar ] ')'
fixity      ::= ('infixl' | 'infixr' | 'infix') digit op { ',' op }
sig         ::= var { ',' var } '::' qualtype

bind        ::= funlhs rhs | pat rhs
funlhs      ::= var { apat } | apat varop apat
rhs         ::= '=' expr [ 'where' decls ]
              | guard '=' expr { guard '=' expr } [ 'where' decls ]
guard       ::= '|' expr
decls       ::= '{' [ localdecl { ';' localdecl } ] [ ';' ] '}'
localdecl   ::= sig | bind
```

读 `datadecl` 时要多留意一条限制。普通数据类型可以有多个构造器，但只要声明的是记录，就只能有一个构造器。同一种记录里，字段名不能重复。不同的记录类型倒是可以各有一个叫 `x` 的字段。

实例头（`instance` 中类名后面的部分）出现的类型变量必须各不相同，`=>` 左边的约束也只能提到这些变量。运算符的优先级和结合性只能在顶层声明，优先级可取 `0` 到 `9`。`deriving` 只支持 `Eq`、`Ord` 和 `Show`。模式绑定不能递归。允许递归的绑定，去掉语法糖后必须是函数。

## 表达式

```text
expr        ::= infixexp [ '::' qualtype ]
infixexp    ::= lexp { op lexp } | '-' infixexp
lexp        ::= '\' apat { apat } '->' expr
              | 'let' decls 'in' expr
              | 'if' expr 'then' expr 'else' expr
              | 'case' expr 'of' '{' alt { ';' alt } [ ';' ] '}'
              | 'do' '{' stmt { ';' stmt } [ ';' ] '}'
              | fexp
fexp        ::= aexp { aexp }
aexp        ::= atom { projection | update }
atom        ::= var | con | literal
              | '(' expr ')'
              | '(' expr ',' expr [ ',' expr ] ')'
              | '[' [ expr { ',' expr } ] ']'
              | '[' expr '..' expr ']'
              | '(' infixexp op ')' | '(' op infixexp ')'
              | '(' PROJ_SECTION varid ')'
              | conid '{' fbind { ',' fbind } '}'
projection  ::= PROJ varid
update      ::= '{' fbind { ',' fbind } '}'
fbind       ::= varid '=' expr
alt         ::= pat ( '->' expr | guard '->' expr
                { guard '->' expr } ) [ 'where' decls ]
stmt        ::= expr | pat '<-' expr | 'let' decls

var         ::= varid | '(' varsym ')'
con         ::= conid | '(' ':' ')' | '[' ']' | '(' ')'
              | '(' ',' ')' | '(' ',' ',' ')'
op          ::= varsym | '`' varid '`' | ':'
varop       ::= varsym | '`' varid '`'
```

`do` 块的最后一条语句必须是表达式。如果花括号直接跟在一个裸构造器名后面，比如 `Point { x = 1, y = 2 }`，它表示新建记录，每个字段都得恰好填一次。更新已有记录可以只写要改的字段，改完之后仍是同一种记录。元组支持二元组和三元组。`(- e)` 对 `e` 取负。想写“给我一个数，再减去 `e`”，得写成 `\x -> x - e`。`(&&)` 和 `(||)` 都可以当作函数值传递，这两个运算符的定义是固定的。

取字段和更新字段都比函数应用结合得紧。比如 `f r.x` 会先读出 `r.x`，再把它传给 `f`。函数应用向左结合，所以 `f x y` 读作 `(f x) y`。类型里的箭头向右结合，`a -> b -> c` 读作 `a -> (b -> c)`。

一串中缀运算符怎么分组，要看当前作用域里的声明。没给某个运算符声明过，就按 `infixl 9` 处理，即优先级为 9、向左结合。一元负号沿用 Haskell 的优先级 6 规则，比如 `-x * y` 读作 `-(x * y)`。如果相邻运算符优先级一样，结合性却不相容，就必须加括号说明怎么算。

Prelude 给常用运算符定好了以下规则。数字越大，结合得越紧。

| 结合性 | 优先级 | 运算符 |
| --- | ---: | --- |
| 右结合 | 9 | `.` |
| 左结合 | 7 | `*`, `/`, `` `div` ``, `` `mod` ``, `` `quot` ``, `` `rem` `` |
| 左结合 | 6 | `+`, `-` |
| 右结合 | 6 | `<>` |
| 右结合 | 5 | `:`, `++` |
| 不可结合 | 4 | `==`, `/=`, `<`, `<=`, `>`, `>=` |
| 左结合 | 4 | `<$>`, `<*>` |
| 右结合 | 3 | `&&` |
| 右结合 | 2 | `\|\|` |
| 左结合 | 1 | `>>=`, `>>` |
| 右结合 | 0 | `$` |

## 模式和类型

```text
pat         ::= lpat [ ':' pat ]
lpat        ::= conid apat { apat } | '-' intlit | apat
apat        ::= varid [ '@' apat ] | '_' | con
              | conid '{' [ fpat { ',' fpat } ] '}'
              | literal | '(' pat ')'
              | '(' pat ',' pat [ ',' pat ] ')'
              | '[' [ pat { ',' pat } ] ']'
fpat        ::= varid '=' pat

qualtype    ::= [ context '=>' ] type
context     ::= pred | '(' pred { ',' pred } ')'
pred        ::= conid tyvar
type        ::= btype [ '->' type ]
btype       ::= atype { atype }
atype       ::= tycon | tyvar | '(' type ')'
              | '(' type ',' type [ ',' type ] ')'
              | '[' type ']'
tycon       ::= conid | '[' ']' | '(' ')'
              | '(' ',' ')' | '(' ',' ',' ')' | '(' '->' ')'
tyvar       ::= varid
```

同一组参数模式里的变量名必须各不相同，比如 `f x x = ...` 不合法。记录模式里省略的字段按通配符 `_` 处理。模式里的构造器必须吃满参数，比如 `Just` 带一个值，要写 `Just x` 或 `Just _` 才算一个完整模式。

类型同义词也得用满参数，而且定义不能绕一圈又引用自己。Fun 没有显式的 `forall`、种类标注、惰性模式、带模块限定的名字，也不能自定义中缀构造器。

## 词法和缩进布局

标识符使用 ASCII 字符。变量一类的名字 `varid` 匹配 `[a-z_][A-Za-z0-9_']*`，构造器一类的名字 `conid` 匹配 `[A-Z][A-Za-z0-9_']*`。单独一个 `_` 是通配符。下面这些是保留字。

```text
case class data deriving do else if in infix infixl infixr
instance let of then type where
```

词法分析器读到下面这些符号字符时，会把连续的一串尽量读完，当作一个运算符。第二行列出的写法有专门用途，不能拿来自定义运算符。自定义运算符也不能以 `:` 开头。

```text
可用符号字符： ! # $ % & * + . / < = > ? @ \ ^ | ~ : -
保留运算符： .. : :: = \ | <- -> @ =>
```

整数字面量可以写成十进制，也可以用 `0x` 开头写十六进制，数字之间可用下划线分隔。`Double` 字面量带小数点或指数，例如 `1.5`、`1e10`、`2.5e-3`。比如写 `-3` 时，整数这个词法单元只是 `3`，前面的负号由语法单独处理。一个字符用单引号括起来，只能包含一个 Unicode 标量值。字符串用双引号括起来，可以包含 Unicode 文本。两者都支持 `\n`、`\t`、`\\`、`\'`、`\"`、`\0` 和 `\u{…}` 这些转义写法。前面规则中的 `literal` 就是整数、双精度浮点数、字符、字符串这四类字面量。

看到 `--` 时，还得看它后面紧跟着什么。如果下一个字符不属于上面那组符号字符，这里就开始了一条行注释。比如 `-- 说明` 是注释，`-->` 则是一个运算符。块注释写在 `{-` 和 `-}` 之间，里面还可以嵌套块注释。

还没处理缩进，词法分析器就已经看点号两边挨着什么，决定它的用途。点号左边紧挨变量、右侧定界符（如 `)` 或 `]`）或字面量，右边紧挨 `varid` 时，记作取字段的 `PROJ`。如果左边紧挨 `(`，右边紧挨 `varid`，就记作 `PROJ_SECTION`。例如 `(.x)` 是个等着接收记录、再从中取出 `x` 字段的函数。其余情况按普通运算符处理。所以 `f.g` 从 `f` 取字段 `g`，`f . g` 才是把两个函数复合起来。

布局处理会在 `let`、`where`、`of`、`do` 后面以及整份文件的外层补上省略的花括号。同一布局块里，新的一行若跟上一条声明或语句对齐，就相当于用分号把它们隔开。缩进退到块的起始列以左，块就结束。手写的花括号优先。`in` 也可以结束隐式 `let` 块，哪怕它们写在同一行。想看编译器实际补出了哪些符号，可以跟着[第 8 章的例子](../part-2/ch08-layout.md#把看不见的括号和分号写出来)走一遍。这套词法和布局约定源于 [《Haskell 2010 语言报告》](https://www.haskell.org/onlinereport/haskell2010/)。

---

[← 第 25 章　接下来可以做什么](../part-3/ch25-future-work.md) · [中文目录](../README.md) · [附录 B　类型规则 →](appendix-b-typing.md)
