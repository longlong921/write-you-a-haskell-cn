# 第 2 章　语法分析

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch02-parsing.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch02_calc)

> **新增文件**
>
> - `src/lexer.rs`
> - `src/pratt.rs`
> - `src/ast.rs`
> - `src/eval.rs`
> - `build.rs`
> - `src/diagnostic.rs`
> - `src/lib.rs`
> - `src/main.rs`
>
> **改动文件**
>
> 无。

## 为什么从计算器开始

读到 `1 + 2 * 3`，你知道该先算乘法。可计算机收到的只是一串字符。要让它动手算，得先让它认出哪些字符该归到一起。这一步叫作**语法分析**（parsing），做这件事的程序叫**解析器**（parser）。

我们先写一个计算器。它足够小，你可以从键盘敲下的源码一直追到最后算出的答案。同一个计算器，我们要给它写两套解析器。用 LALRPOP 时，我们写下文法，由工具生成解析器。用 Pratt 方法时，怎么处理运算符，全写在一小段 Rust 函数里。把两份实现放在一起看，就能明白生成器到底替我们做了什么。

## 这一章的语言长什么样

一个程序就是一个表达式。能写的东西有十进制整数字面量、变量名、四则运算 `+`、`-`、`*`、`/`、一元负号、括号，以及不允许递归引用自身的 `let` 绑定。

```fun
let x = 7 in (let y = x + 1 in y * y) - x
```

拿里层的 `let y = x + 1 in y * y` 来说，新引入的 `y` 从 `in` 后面才开始生效，等号右边的表达式用不到这个新名字。如果内外两层都绑定了同一个名字，那么在内层的作用域里，读到这个名字就取内层的绑定。乘除的优先级高于加减，这四个运算符都向左结合。一元负号的优先级比它们都高。这里的 `/` 做整数除法，结果向零截断，例如 `-7 / 2` 得到 `-3`。到后面完整的 Fun 语言里，这种整数除法叫 `quot`，而 `/` 留给可以产生分数结果的除法。

## 怎样确定表达式的结构

`1 + 2 * 3` 可以画成两棵不同的树。一棵表示先加再乘，得到 9。另一棵表示先乘再加，得到 7。我们想要第二棵，就得把这个选择写进文法里。做法是给加减、乘除和原子表达式各设一层规则。在加减这一层，一个操作数就可以是一整段乘除表达式。遇到左括号时，再从完整的表达式规则重新解析括号里的内容，于是 `(1 + 2) * 3` 也能表达出来。

### 选哪种解析方法

有人希望整门语言的文法摊开来一目了然，也有人更愿意直接用宿主语言写一个小巧的解析器。比较方案之前，先把三个问题分开。**文法**规定怎样写算合法，**算法**决定如何认出合法的输入，**API** 决定写解析器的人要怎样表达这些规则。

| 方法 | 它怎样工作 | 用在本书里有什么取舍 |
| --- | --- | --- |
| LR(1) / LALR(1) | 生成一个状态机。它不断读入词元（移进），或把已经读到的成分按文法规则合成一个更大的成分（归约）。 | 可以用一份声明式文法描述整门语言。规则有冲突，生成解析器时就会报告。 |
| Pratt 解析 | 比较运算符的结合强度（binding power），决定下一个操作数该归谁管。 | 处理运算符的代码短，也好读。声明和语句块还得另写规则。 |
| Packrat 解析 | 按「哪条规则、输入的哪个位置」缓存每次尝试的结果。 | 对符合条件的固定文法，识别耗时容易预测，代价是需要一张缓存表。 |
| 解析器组合子 | 用普通函数把小解析器拼成大解析器。 | 写起来方便。遇到分支要不要回溯、结果会不会缓存、出错后怎么办，都取决于具体的库。 |

**LR(1)** 从左向右扫描输入，按最右推导的相反顺序构造语法结构。当前状态概括了已经读过的前缀，再向前看一个词元（token），就能决定下一步是把它移进来，还是把已经匹配完的规则归约掉。名字里的「1」说的是向前看一个词元。表达式可以很长，也可以任意嵌套，因为解析器会用栈记住前面的语法上下文。像「一个和可以由一个较小的和再加一项组成」这样的左递归规则，正适合这种自底向上的做法。

**LALR(1)** 会把 LR(0) 项相同的 LR(1) 状态合并，连同各自的向前看词元集合一起合并。这样解析表更小，但原先分开时没有的冲突，合并之后有可能出现。LALRPOP 的名字里虽然有 LALR，默认生成的却是 LR(1) 解析器，LALR(1) 需要另行选择。遇到冲突，生成器等于指出了一个需要重新检查文法的地方。[LALRPOP 文档](https://github.com/lalrpop/lalrpop#lalrpop)

**Pratt 解析**在碰到运算符的地方，当场决定怎么分组。例如读到 `1 + 2`，再看见后面的 `*`，就比较它和 `+` 的结合强度。乘法结合得更紧，所以 `2 * 3` 整体成为加号的右操作数，得到 `1 + (2 * 3)`。再看 `8 - 3 - 1`。在读取第一个减号的右操作数时，门槛设得比减号自身的强度高一点，第二个减号进不来，于是分组是 `(8 - 3) - 1`。同一个 `-` 出现在操作数前面是取负，出现在两个操作数之间是相减，可以分别交给不同的代码处理。这套办法很适合表达式。至于类型类声明怎样写、缩进块从哪里开始到哪里结束，还要靠其他规则来规定。

**Packrat 解析**要解决的是回溯时重复劳动的问题。在输入的某个位置试过一条规则，就把成功或失败记下来。之后回到同一位置再试这条规则，可以直接取结果。它常与解析表达式文法（PEG）搭配使用。PEG 的备选项按顺序尝试，第一个成功就算数，因此调换两个备选项的顺序，就可能改变这门语言接受的输入。在文法固定、每个缓存项所需的工作量都有常数上界的前提下，识别时间随输入长度线性增长，缓存空间通常也线性增长。普通左递归规则需要改写，或者借助专门支持左递归的扩展。[Ford 的 Packrat 与 PEG 介绍](https://bford.info/packrat/)

**解析器组合子**是一种组织解析器的写法，底层有多种算法可选。顺序组合子让两个解析器先后运行。选择组合子按库规定的方式尝试多个分支。有的库会在某一步确定分支，有的会回溯，有的会缓存结果。代码即使读起来很顺，也要另外确认它能否在线性时间内完成、报错后能否继续，以及能否处理左递归。这些都取决于库的设计。

### 为什么用 LALRPOP，还要再写一个 Pratt 解析器

LALRPOP 让整门语言的文法集中在一处。读到一条规则，就可以看到它用哪段带类型的 Rust 代码构造抽象语法树（AST）。出错后如何恢复，也能直接写成文法规则。Pratt 则让我们用很少的代码独立实现计算器的优先级规则。两边都写一遍，既能看到每一步怎么分组，还能拿一份结果去核对另一份。后续章节的编译器只运行 LALRPOP 生成的解析器。

要读懂 Haskell 风格的语法，还得在解析之前做点准备。到第 8 章，`logos` 识别词元，`marginalia` 保留注释和空白，供格式化时使用，`offsides` 则把缩进转换为标记语法块边界的分隔符。后两个库各自负责这些排版信息和 Haskell 风格的布局规则。处理好的词元流再交给 LALRPOP。这样，把缩进转换为语法块就是一步可以单独测试的变换，解析器的各个分支也不用再到处判断缩进。本章的计算器暂时用不到布局规则，也不保留注释。

用户自行声明运算符的优先级和结合方向（fixity），又会带来一个问题。解析器是提前生成的，可每份源文件里都可能有不同的声明，没法在生成时就给每份文件配好一张优先级表。第 8 章会先把连续的运算符和操作数读成一条平坦的链，等掌握了声明，再确定怎样分组。无论用 Pratt 循环还是解析器生成器，我们都得先弄清楚，到底在什么时候才能取得这些声明。

### 语法和类型

这么小的语言已经可以写出类型判断了。例如，如果两个操作数都是 `Int`，那么它们相加，结果也是 `Int`。

$$
\frac{\Gamma \vdash e_1 : \mathsf{Int} \quad \Gamma \vdash e_2 : \mathsf{Int}}
{\Gamma \vdash e_1 + e_2 : \mathsf{Int}}\;\text{(Add)}
$$

读这条规则时，可以把 $\Gamma$ 看作「目前知道哪些名字各有什么类型」。横线上方是两个前提，下方是从它们推出的结论。

眼下所有值都是整数，还用不上类型推断器。不过在求值之前，程序已经会检查词法作用域，确认每个名字在它出现的地方都能找到对应的绑定。Pierce 详细区分了语法、类型和求值[^pierce2002]。想对照真正的 Haskell，可以读《Haskell 报告》[^haskell2010]。后面 Fun 里允许自行设定运算符优先级和结合方向的规则，本章刻意留到以后再加。

## 动手实现

`lexer.rs` 借助 logos 从字符中识别词元，并记下每个词元在源码中占据的字节区间（span）。同一段字符既匹配关键字又匹配同样长的标识符时，关键字优先。遇到非法字符或超出表示范围的整数字面量，就报告 `E0001`。`ast.rs` 给每个表达式节点都保存源码区间。这样我们既能对比两种解析器生成的树，也能追溯每个节点在源码里的位置。

```rust
/// Byte offsets into the single source file, including parentheses when present.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Expr {
    pub kind: Kind,
    pub span: Range<usize>,
}

/// An error node exists only during recovery; the driver never evaluates it.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Kind {
    Int(i64),
    Var(String),
    Neg(Box<Expr>),
    Binary(Op, Box<Expr>, Box<Expr>),
    Let(String, Box<Expr>, Box<Expr>),
    Error,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Op {
    Add,
    Sub,
    Mul,
    Div,
}
```

[源码 · ast.rs 第 5～29 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch02_calc/src/ast.rs#L5-L29)

生成式解析器通过分层的非终结符来体现优先级。非终结符就是文法里还可以继续展开的名字，譬如「和」与「积」。下面摘出优先级相关的规则。完整文法还处理变量绑定、括号、一元负号，并包含一条出错后恢复解析的规则。

```text
Sum: Expr = {
    <a:Sum> "+" <b:Product> => Expr::binary(Op::Add, a, b),
    <a:Sum> "-" <b:Product> => Expr::binary(Op::Sub, a, b),
    Product,
};
Product: Expr = {
    <a:Product> "*" <b:Unary> => Expr::binary(Op::Mul, a, b),
    <a:Product> "/" <b:Unary> => Expr::binary(Op::Div, a, b),
    Unary,
};
```

[源码 · parser.lalrpop 第 15～24 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch02_calc/src/parser.lalrpop#L15-L24)

含 `!` 的规则会在树里插入一个错误节点，同时保存本次恢复对应的诊断信息。恢复后即使拼出了一棵树，负责调度的代码仍然会返回错误，因此这些占位节点不会被送进求值器。`diagnostic.rs` 先打印格式固定的错误码行和出错位置，再让 ariadne 根据字节下标展示对应的源码片段。

```rust
/// Parse and recover enough to report several syntax errors, then stop the pass.
pub fn parse(source: &str) -> Result<Expr, Vec<Diagnostic>> {
    let tokens = lexer::lex(source)?;
    let mut recovered = Vec::new();
    let parsed = parser::ExprParser::new()
        .parse(&mut recovered, tokens.into_iter().map(Ok));
    let mut errors: Vec<_> = recovered
        .into_iter()
        .map(|r| parse_error(r.error))
        .collect();
    match parsed {
        Ok(expr) if errors.is_empty() => Ok(expr),
        Ok(_) => Err(errors),
        Err(error) => {
            errors.push(parse_error(error));
            Err(errors)
        }
    }
}
```

[源码 · lib.rs 第 19～37 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch02_calc/src/lib.rs#L19-L37)

Pratt 解析器的循环每次查看下一个运算符。只要它的结合强度够得上当前门槛，就把它读进来。解析右操作数时把门槛调为 `power + 1`，同优先级的运算符就会留给外层循环，于是当前运算符向左结合。名字、字面量、括号和一元负号都由读取表达式开头的代码处理。`let` 另有单独的入口。

```rust
fn infix(&mut self, min_power: u8) -> Result<Expr, Diagnostic> {
    let mut left = self.prefix()?;
    loop {
        let (op, power) = match self.peek() {
            Some(Token::Plus) => (Op::Add, 1),
            Some(Token::Minus) => (Op::Sub, 1),
            Some(Token::Star) => (Op::Mul, 3),
            Some(Token::Slash) => (Op::Div, 3),
            _ => break,
        };
        if power < min_power {
            break;
        }
        self.at += 1;
        let right = self.infix(power + 1)?;
        left = Expr::binary(op, left, right);
    }
    Ok(left)
}
```

[源码 · pratt.rs 第 72～90 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch02_calc/src/pratt.rs#L72-L90)

`eval.rs` 把变量绑定放在一个栈里，查找时从末尾往回找。这样离使用位置最近的同名绑定就会先被找到，自然实现了词法作用域。二元运算先求左操作数的值，再求右操作数的值。进入绑定的作用域时扩充环境，离开时恢复原状，即使绑定的主体求值失败也一样。从一开始，算术就按 63 位整数回绕处理。溢出的行为由这门小语言自己规定，和 Rust 在调试模式下是否检查溢出无关。下面从 Rust 源文件中摘出负责把数值规整到这一范围的函数。

```rust
pub fn normalize(n: i64) -> i64 {
    (n << 1) >> 1
}
```

[源码 · eval.rs 第 7～9 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch02_calc/src/eval.rs#L7-L9)

```rust
/// The environment is a stack: the most recent binding shadows earlier names.
pub fn evaluate(
    expr: &Expr,
    env: &mut Vec<(String, i64)>,
) -> Result<i64, String> {
    match &expr.kind {
        Kind::Int(n) => Ok(normalize(*n)),
        Kind::Var(name) => env
            .iter()
            .rev()
            .find(|(key, _)| key == name)
            .map(|(_, value)| *value)
            .ok_or_else(|| format!("unbound variable `{name}`")),
        Kind::Neg(value) => {
            Ok(normalize(evaluate(value, env)?.wrapping_neg()))
        }
        Kind::Binary(op, left, right) => {
            let a = evaluate(left, env)?;
            let b = evaluate(right, env)?;
            Ok(normalize(match op {
                Op::Add => a.wrapping_add(b),
                Op::Sub => a.wrapping_sub(b),
                Op::Mul => a.wrapping_mul(b),
                Op::Div if b == 0 => return Err("divide by zero".into()),
                Op::Div => a / b,
            }))
        }
        Kind::Let(name, value, body) => {
            let value = evaluate(value, env)?;
            env.push((name.clone(), value));
            let result = evaluate(body, env);
            env.pop();
            result
        }
        Kind::Error => {
            Err("internal error: attempted to evaluate a recovery node"
                .into())
        }
    }
}
```

[源码 · eval.rs 第 50～89 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch02_calc/src/eval.rs#L50-L89)

`main.rs` 把上述部件接到命令行和交互式环境（REPL）上。可以把同一个表达式交给两个解析器，看它们给出的树是否一致。测试也这么做，还会自动生成具有不同分组方式的表达式。两个算式有时碰巧算出同一个数，光比答案会漏掉解析错误，直接比树就能把这种错误找出来。

## 跑起来看看

```text
$ cd crates/ch02_calc
$ cargo run -q -- run tests/programs/precedence.fun
7
$ cargo run -q -- dump --pass=parse tests/programs/precedence.fun
(+ 1 (* 2 3))
$ cargo run -q -- run tests/programs/precedence.fun --parser=pratt
7
$ cargo run -q -- repl
Calc: :quit to exit, :parse EXPR to inspect the AST.
calc> let x = 6 in x * 7
42
calc> :quit
```

`check` 只检查语法以及名字是否有定义，不会执行算术。因此 `1 / 0` 能通过 `check`，交给 `run` 执行才会报 `fun: divide by zero`，并以状态码 1 退出。要看词法分析器给每个词元记下了哪段字节区间，可以用 `dump --pass=tokens`。

## 练习

1. **基础。** 先在纸上写出 `20 - 3 * 4 - 1` 的值和你预计 `dump --pass=parse` 会打印的树，再运行程序核对。
2. **基础。** 比较 `1` 和 `(1)` 两个表达式的源码区间。为什么它们求值的结果相同？
3. **进阶。** 把 Pratt 解析器读右操作数时的门槛改成 `power`。找一个尽可能短的减法表达式，让人一眼看出结合方向发生了变化。
4. **进阶。** 写一个有两处独立语法错误的表达式，让解析器把两处都报出来。解释它读到哪里可以从第一处错误中恢复，继续往下解析。
5. **挑战。** 修改测试用的表达式生成器，让它插入多余的括号，并验证添加这些括号前后的求值结果始终一致。

## 延伸阅读

Pierce 对「程序长什么样」的语法与「程序是什么意思」的语义作了细致区分[^pierce2002]。《Haskell 报告》则给出了比本章完整得多的具体语法，以及根据运算符声明确定表达式分组的步骤[^haskell2010]。

[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。
[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。

---

[← 第 1 章　导读](ch01-introduction.md) · [中文目录](../README.md) · [第 3 章　λ 演算 →](ch03-lambda.md)
