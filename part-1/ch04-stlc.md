# 第 4 章　简单类型 λ 演算

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch04-stlc.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch04_stlc)

> **新增文件**
>
> - `src/types.rs`
>
> **改动文件**
>
> - `src/ast.rs`
> - `src/eval.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/names.rs`
> - `src/normal.rs`

## 能不能在运行前抓住错误？

把 `1 True` 交给上一章的求值器，它会真的去执行，直到发现整数 `1` 根本没法当函数调用。可我们光看代码就知道有问题。能不能让编译器也提前看出来？

这就是类型检查器的工作。一个函数的类型告诉我们，它接受什么类型的输入，又会返回什么类型的结果。检查一次调用时，只要看看实参的类型能否和函数要求的输入类型对上。这些规则很短，完全可以一条条写下来。后面的 Rust 实现几乎可以和规则逐条对照着读。

这一章先请写程序的人标出每个 lambda 形参的类型。有了这些起点，我们就能集中精力看懂“怎么检查”。到第 6 章，再让编译器自己把类型推出来，连那种同一份代码能处理多种类型的情况也能涵盖。

## 现在这门语言能写什么

类型有 `Int`、`Bool`，还有用箭头写的函数类型。箭头向右结合，`Int -> Bool -> Int` 就是 `Int -> (Bool -> Int)`。如果箭头左侧的输入本身是个函数，就要给它加括号，如 `(Int -> Bool) -> Int`。

```fun
let choose = \(b : Bool) -> \(x : Int) -> \(y : Int) ->
  if b then x else y
in choose True 10 20
```

每个 lambda 的形参都要标出类型，函数调用、算术运算和作用域仍然和上一章一样。再试一次把整数当函数调用，这次你会看到 `E0201`。这个错误来自类型检查器，求值器还没接手，错误就已经找到了。

## 原理

### 加上类型的语法

新添的东西只有一处，每个 lambda 的形参都带着类型。先看看哪些类型可以写出来。一个函数的输入可以是另一个函数，输出也可以是函数。

$$
\begin{aligned}
\tau & = \text{Int} & \quad & \text{整数} \\
     & \mid \text{Bool} & \quad & \text{布尔值} \\
     & \mid \tau_1 \to \tau_2 & \quad & \text{函数类型}
\end{aligned}
$$

表达式的各种写法在上一章都见过。变化在 $\lambda x : \tau.\;e$ 这里。形参上的标注告诉检查器，检查函数体 $e$ 时，可以认定 $x$ 的类型是 $\tau$。

$$
\begin{aligned}
e & = x & \quad & \text{变量} \\
  & \mid e_1\;e_2 & \quad & \text{函数应用} \\
  & \mid \lambda x : \tau.\;e & \quad & \text{带类型标注的函数抽象} \\
  & \mid n & \quad & \text{整数字面量} \\
  & \mid \text{True}\mid\text{False} & \quad & \text{布尔字面量} \\
  & \mid \text{let}\;x = e_1\;\text{in}\;e_2 & \quad & \text{局部绑定} \\
  & \mid \text{if}\;e_1\;\text{then}\;e_2\;\text{else}\;e_3 & \quad & \text{条件表达式} \\
  & \mid -e & \quad & \text{整数取负} \\
  & \mid e_1\;\mathit{op}\;e_2 & \quad & \text{二元运算}
\end{aligned}
$$

和之前一样，$n$ 表示整数，$\mathit{op}$ 可以是 `+`、`-`、`*`、`/`、`==`、`<` 或 `<=`。为了聚焦语法，这里略去了源码位置区间和解析器出错时用来恢复解析的占位符。这套语法规定哪些表达式能解析成树。要知道树上的操作能不能接得起来，还要用到下面的类型规则。比如 `1 True` 符合函数应用的语法，能解析出一棵树，却推导不出类型。

### 类型规则与安全性

先学会读规则。看到 $\Gamma \vdash e : \tau$，可以念成“在 $\Gamma$ 这些已知条件下，表达式 $e$ 的类型是 $\tau$”。这里的大写希腊字母 $\Gamma$（Gamma）装着我们对变量类型的假设。接下来会看到形似分数的写法。横线下面是想要推出的结论，横线上面是要先对各个小部分证实的前提。横线读作“只要上面都成立，下面就成立”。

**类型环境**记录名字到类型的对应关系。遇到变量，就从里向外找最近的同名绑定。遇到 lambda，就把形参的类型添进环境，再去检查函数体。遇到函数应用，就看实参的类型是否符合函数对输入的要求[^pierce2002]。

$$
\frac{\Gamma, x : \tau \vdash e : \sigma}
{\Gamma \vdash (\mathsf{lambda}\ x : \tau . e) : \tau \to \sigma}
\;\text{(Abs)}
$$

第一条叫抽象规则（Abs）。假设给 `x` 的类型是 $\tau$，在这个假设下能推出函数体 `e` 的类型是 $\sigma$，那么整个函数就具有类型 $\tau \to \sigma$。

$$
\frac{\Gamma \vdash f : \tau \to \sigma \quad \Gamma \vdash a : \tau}
{\Gamma \vdash f\ a : \sigma}\;\text{(App)}
$$

第二条叫应用规则（App）。如果 `f` 的类型是 $\tau \to \sigma$，而实参 `a` 的类型刚好是 $\tau$，那么调用 `f a` 得到的结果类型就是 $\sigma$。

`if` 要求条件的类型是 `Bool`，两个分支的类型必须相同。算术运算接收整数，比较运算返回布尔值。遇到 `let`，先在原环境里检查等号右边的表达式，然后才把新名字添进去，所以 `let` 仍然是非递归的。这一章也不会自动给类型变量加上“对任意类型都成立”的量词。因此一个类型标成 `Int -> Int` 的恒等函数，就只能接收 `Int`，不能再传 `Bool` 给它。

拿 `\(x : Int) -> x + 1` 从头推一次。先把 `x` 的类型 `Int` 记入环境。这下 `+` 两边都是 `Int`，函数体 `x + 1` 的类型自然也是 `Int`。用抽象规则收个尾，整个函数的类型就是 `Int -> Int`。给它传 `41`，就满足应用规则，结果还是 `Int`。可要是把实参换成 `True`，同一条规则会要求 `Bool` 和 `Int` 相等。这条前提无法成立，检查器就该停在这里报错。

有了类型检查，我们具体能保证什么？先看**保持性**（preservation，亦称类型保持）。它说，一个表达式求值一步，类型应当保持不变。原来是 `Int` 的表达式，归约一步不该突然变成函数。要证明函数调用也遵守这条性质，关键是说明，把类型正确的实参代进函数体，函数体的类型仍然不变。这叫**替换引理**（substitution lemma）。

另一个性质是**进展性**（progress）。一个没有自由变量、并且通过了类型检查的表达式，要么已经是值，要么还能再走一步。这样运行时就不会卡在“把 `42` 当作函数调用”这种无从执行的操作上。本章的除法要单独说明，除以零仍然会报告运行时错误。类型系统能排除这里的类型搭配错误，但一次计算能否顺利得到结果，还可能取决于具体输入。Pierce 的书给出了这些性质背后的证明[^pierce2002]。

## 实现

`types.rs` 用递归结构来表示函数类型。箭头两端都可以再放一个类型，打印时会在需要的地方补上括号。这时还用不到代表未知类型的元变量，也用不到记录类型替换的映射表。

```rust
/// Simple types have no variables, schemes, or implicit quantification.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Type {
    Int,
    Bool,
    Arrow(Box<Type>, Box<Type>),
}

impl Type {
    pub fn arrow(domain: Self, codomain: Self) -> Self {
        Self::Arrow(Box::new(domain), Box::new(codomain))
    }
}

impl fmt::Display for Type {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::Int => write!(f, "Int"),
            Self::Bool => write!(f, "Bool"),
            Self::Arrow(a, b) => {
                if matches!(**a, Self::Arrow(..)) {
                    write!(f, "({a})")?;
                } else {
                    write!(f, "{a}")?;
                }
                write!(f, " -> {b}")
            }
        }
    }
}
```

[源码 · types.rs 第 9～38 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch04_stlc/src/types.rs#L9-L38)

语法里，箭头右侧的结果类型通过右递归来解析，于是箭头自然向右结合。抽象语法树（AST）中的 lambda 节点现在除了形参名，还存着一个 `Type`。只要一个 lambda 成功解析，它的参数类型就一定跟着保存在树上。

```text
Type: Type = {
    <a:TypeAtom> "->" <b:Type> => Type::arrow(a, b),
    TypeAtom,
};
TypeAtom: Type = {
    "Int" => Type::Int, "Bool" => Type::Bool, "(" <Type> ")",
};
```

[源码 · parser.lalrpop 第 52～58 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch04_stlc/src/parser.lalrpop#L52-L58)

检查器按上面的类型规则逐条实现。环境用栈保存，找变量时从最里面的绑定开始往外找。检查完函数体，即使中途返回错误，也要把为这一层加入的绑定弹出。遇到函数应用，先找出函数的类型，再检查它的实参。这样一旦实参的类型不对，诊断可以直接标出那个实参，而不必把整段调用都圈起来。

```rust
fn infer(
    expr: &Expr,
    env: &mut Vec<(String, Type)>,
) -> Result<Type, Diagnostic> {
    match &expr.kind {
        Kind::Int(_) => Ok(Type::Int),
        Kind::Bool(_) => Ok(Type::Bool),
        Kind::Var(name) => env
            .iter()
            .rev()
            .find(|(n, _)| n == name)
            .map(|(_, ty)| ty.clone())
            .ok_or_else(|| {
                Diagnostic::new(
                    "E0101",
                    format!("unbound variable {name}"),
                    expr.span.clone(),
                )
            }),
        Kind::Lam(name, parameter, body) => {
            env.push((name.clone(), parameter.clone()));
            let result = infer(body, env);
            env.pop();
            Ok(Type::arrow(parameter.clone(), result?))
        }
        Kind::App(function, argument) => {
            let function_ty = infer(function, env)?;
            let Type::Arrow(domain, codomain) = function_ty else {
                return Err(mismatch(
                    function,
                    "a function type",
                    &function_ty,
                ));
            };
            expect(argument, infer(argument, env)?, &domain)?;
            Ok(*codomain)
        }
        Kind::Let(name, value, body) => {
            let ty = infer(value, env)?;
            env.push((name.clone(), ty));
            let result = infer(body, env);
            env.pop();
            result
        }
        Kind::If(condition, yes, no) => {
            expect(condition, infer(condition, env)?, &Type::Bool)?;
            let ty = infer(yes, env)?;
            expect(no, infer(no, env)?, &ty)?;
            Ok(ty)
        }
        Kind::Neg(value) => {
            expect(value, infer(value, env)?, &Type::Int)?;
            Ok(Type::Int)
        }
        Kind::Binary(op, left, right) => {
            expect(left, infer(left, env)?, &Type::Int)?;
            expect(right, infer(right, env)?, &Type::Int)?;
            Ok(match op {
                Op::Eq | Op::Lt | Op::Le => Type::Bool,
                _ => Type::Int,
            })
        }
        Kind::Error => Err(Diagnostic::new(
            "E0002",
            "unrecovered syntax error",
            expr.span.clone(),
        )),
    }
}
```

[源码 · types.rs 第 71～139 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch04_stlc/src/types.rs#L71-L139)

一个叫 `expect` 的小函数负责按结构比较两个简单类型。如果不匹配，就在出问题的表达式对应的字节区间上报错，同时写明“期望什么类型”和“实际得到什么类型”。Ariadne 负责把那段源码和标记排版出来。这个独立的类型检查器找到第一个类型错误就停下。此前的词法检查和作用域检查仍然各自收集它们能发现的错误。

求值器仍用第 3 章的闭包，执行时可以忽略形参上的类型标注。不过，替换、正规序归约和 de Bruijn 索引转换在处理 lambda 时，都会把标注一起带过去。这样测试保持性时，就能检查归约途中的每一个项。即使打印成了不带变量名的形式，参数类型不同的项也依然分得清。

## 动手运行

```text
$ cd crates/ch04_stlc
$ cargo run -q -- run tests/programs/typed-return.fun
42
$ cargo run -q -- dump --pass=types tests/programs/church.fun
Int
$ cargo run -q -- repl
STLC: :quit, :type EXPR, :parse EXPR, :trace EXPR.
stlc> :type \(f : Int -> Bool) -> \(x : Int) -> f x
(Int -> Bool) -> Int -> Bool
stlc> :quit
```

`check` 命令只做检查。两个执行引擎都是先检查类型，再开始求值。`:trace` 也会先拒绝类型不对的输入，通过之后才显示归约步骤。可以试试 `1 / 0`，它理应能通过类型检查，因为这个表达式的类型确实是 `Int`。除以零的错误要等到执行时才会出现。

## 练习

1. **基础。** 使用抽象规则，一步步推导本章 `choose` 的类型。
2. **基础。** 说明 `(Int -> Bool) -> Int` 里的括号为什么不能去掉。
3. **进阶。** 写一次函数调用，让函数本身类型正确，却收到一个类型不对的实参。看看错误信息标出了源码中的哪一段。
4. **挑战。** 写出本章这套演算的替换引理，明确说明用来替换的项要满足什么条件，并把遇到 lambda 时的证明过程推一遍。

## 延伸阅读

Pierce 给出了进展性与保持性的形式化证明[^pierce2002]。Damas 和 Milner 解释了为什么需要主类型方案和多态 `let`，我们将在简单类型的基础上继续往这个方向走[^damas1982]。如果想借图来理解，Takenobu Tani 的《Type Introduction Illustrated for Casual Haskellers》从简单类型一路画到 GHC 的扩展。[图解 PDF](https://takenobu-hs.github.io/downloads/type_introduction_illustrated.pdf) · [GitHub 仓库](https://github.com/takenobu-hs/type-introduction-illustrated)

[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。
[^damas1982]: Luis Damas、Robin Milner（1982）。[Principal Type-Schemes for Functional Programs](https://doi.org/10.1145/582153.582176)。

---

[← 第 3 章　λ 演算](ch03-lambda.md) · [中文目录](../README.md) · [第 5 章　求值 →](ch05-evaluation.md)
