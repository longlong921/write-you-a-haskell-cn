# 第 18 章　A-正规形与汇合点

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch18-anf.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch18_anf)

> **新增文件**
>
> - `src/erase.rs`
> - `src/anf.rs`
> - `src/anf_simplify.rs`
> - `src/anf_machine.rs`
> - `src/anf_lint.rs`
>
> **改动文件**
>
> - `src/lib.rs`
> - `src/main.rs`

## 怎样把求值顺序写在程序里？

看一眼 `f (g x) (h y)`。算它的时候，函数和参数有先后之分。`g x` 的结果要有人接着，等轮到 `h y` 时，前面算好的东西也不能丢。从前我们把整棵表达式交给解释器，它自然会替我们记住这些事。现在要生成机器码，后端得能看见每一步。

**A-正规形**（A-normal form，简称 ANF）做的事很朴素。把嵌套的计算拆开，给中间结果起局部名字，下一步就能直接用这些名字。这样，轮到函数调用或其他操作时，它拿到的每个操作数都已经是现成的值了。

第 5 章的 CEK 机也可以在这里派上用场。我们把“拿到结果后还要做什么”记在一摞显式的**续延**（continuation）里，用循环来执行 ANF。这次机器要处理 Fun 里的数据、类型类字典、IO 和汇合点。后端每向机器码走近一步，我们都还能把运行结果和 Core 对照。

## 这一章的语言长什么样

下面这段程序在 `case` 里选出一个值，外面的表达式接着用它。

```fun
main = print (3 + (case Just 4 of { Just x -> x; Nothing -> 0 }))
```

[源码 · case-continuation.fun 第 1 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch18_anf/tests/programs/anf/case-continuation.fun#L1)

走 `Just x` 分支会交出 `x`，走 `Nothing` 分支会交出 `0`。接下来做的事相同，加上 3，再打印一次。ANF 给这段后续工作设一个**汇合点**，让两个分支带着各自的值跳过去，免得把加法和打印各抄两份。这一章默认使用循环执行的 ANF 机，也可以显式指定 `--engine=anf`。换成 `--engine=core`，就能用 Core 解释器跑一遍作对照。

加上 `--no-simplify`，看到的是 ANF 刚转换出来、还没做可选简化时的样子。简化前后的程序得给出相同的标准输出、相同的标准错误首行和相同的进程退出状态。尤其要记得 Fun 是严格求值的。即使一个局部名字后面没人用，绑定它时右侧的表达式也要执行。万一右侧是个会除以零的算式，悄悄删掉它就等于把程序报错的行为一起删掉了。

## 理论

`f (g x) (h y)` 里真正要做的工作是那几次调用。读一个已经绑定的名字，只需取出已有的值。ANF 把这个区别写进了语法。**原子**（atom）可以是字面量，也可以是对已有值的引用。绑定的右侧一次只做一个操作，操作数都用原子表示。`Let` 给结果起个名字。原本层层嵌套的求值过程，就这样摊成了有先有后的步骤[^flanagan1993]。

假设编译器还不知道 `f` 是哪个函数，我们可以把 `f (g x) (h y)` 写成下面的样子。这里把每一次柯里化调用都单独写出来，稍后再看哪些调用可以安全地合在一起。

```text
let gx = g x
let next = f gx
let hy = h y
let result = next hy
return result
```

留意 `f gx` 在哪一行。如果它会报错，程序根本不该算到 `h y`。要是图方便，先把两个括号里的参数都算完，再去调用 `f`，Fun 程序遇到的第一个错误就可能变了。

有分支时还要再想一步。在 `(if b then 10 else 20) + 3` 里，无论选中谁，都欠着一次加法。我们可以只给这段剩余的工作起一次名字。

```text
join finish(value) =
  let result = value + 3
  return result
case b of
  True  -> jump finish(10)
  False -> jump finish(20)
```

这段伪代码展示的是 ANF 的思路。两个分支把选好的值交给 `finish`，于是只用写一份加法。汇合点只在当前函数的局部控制流里有效，不能像普通函数那样把 `finish` 当作结果返回，也不能把它装进列表。

转换器手里有一份**上下文**，说明某个表达式给出结果后要拿它做什么。到了函数末尾，就直接返回那个原子。在表达式内部，就先把结果绑好，再继续往下走。到了 `case` 分支末尾，则可以带着结果跳去执行共享的续延。Maurer 等人讨论的汇合点也有这种特点，它局限在自己的作用域里，不能像普通值那样四处传递[^maurer2017]。因此，我们只需要在合适的位置给后续工作起名，无须把每个函数都改写成显式接收续延的函数。

为什么可以擦除类型？回想 System F 的类型抽象和类型应用，它们描述计算、为计算选定类型，本身没有要在运行时执行的操作[^pierce2002]。字典参数就不同了。字典里装着真的会被调用的方法实现，擦掉它，程序就找不到该执行的代码。因此字典得以普通值的身份留下。到了擦除这一步，IO 动作也已经表示成了一个接收单位值 `()` 的函数。

一个调用是不是在**尾位置**，要看调用回来后还有没有事情要做。如果被调函数算出的结果就是当前函数的结果，这次调用就不用额外留一帧等着接手。非尾调用则必须把没做完的事记下来。因而，即使 Rust 自身的调用栈始终一样深，某些递归程序的内存占用仍会增长，欠下的工作会堆在显式的续延栈上。把 CEK 机的续延摆在明处，正好能看清这些工作到底积在了哪里[^felleisen1986]。

## 实现

`erase::erase` 会去掉可执行表达式里的类型结构，同时保留全局绑定的初始化次序。它用工作列表做后序遍历，去掉 `TyLam` 和 `TyApp`，留下字典项。若上一步漏掉了某个还没编译的嵌套 `Match`，这里就报错。原语、值的打印程序和报错信息仍然需要构造器资料及源码位置，所以这些只读元数据也会保留。

```rust
/// Erase executable types only; read-only metadata supports diagnostics and printers.
pub fn erase(mut program: core::Program) -> Program {
    let globals = program
        .groups
        .iter()
        .flat_map(|g| &g.binds)
        .map(|b| (b.name.name.clone(), expr(&b.rhs)))
        .collect();
    // Consuming Core must not recursively destroy its nested expressions.
    let mut todo = program
        .groups
        .drain(..)
        .flat_map(|g| g.binds.into_iter().map(|b| *b.rhs))
        .collect::<Vec<_>>();
    while let Some(e) = todo.pop() {
        use core::Expr as C;
        match e {
            C::Lam(_, b) | C::TyLam(_, b) | C::TyApp(b, _) => todo.push(*b),
            C::App(a, b) | C::Join(_, _, a, b) => {
                todo.push(*a);
                todo.push(*b);
            }
            C::Let(g, b) => {
                todo.push(*b);
                todo.extend(g.binds.into_iter().map(|b| *b.rhs));
            }
            C::Con(_, _, es) | C::Prim(_, _, es, _) | C::Jump(_, es, _) => {
                todo.extend(es)
            }
            C::Case(e, _, _, alts) => {
                todo.push(*e);
                todo.extend(alts.into_iter().map(|a| a.rhs));
            }
            C::Match(_) => {
                unreachable!("match compilation precedes erasure")
            }
            _ => {}
        }
    }
    Program {
        globals,
        metadata: program,
    }
}
```

[源码 · erase.rs 第 117～160 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch18_anf/src/erase.rs#L117-L160)

### ANF 的语法

下面用 $a$ 表示原子，$r$ 表示绑定右侧，$e$ 表示一个完整的 ANF 主体。每一种写法在下面的 Rust 枚举里都有对应的变体。

$$
\begin{aligned}
e & = \text{let}\;x = r\;\text{in}\;e & \quad & \text{严格绑定} \\
  & \mid \text{letrec}\;\{\overline{x = \lambda\overline{y}.\;e_1}\}\;\text{in}\;e_2 & \quad & \text{定义一组递归函数} \\
  & \mid \text{join}\;j(\overline{x}) = e_1\;\text{in}\;e_2 & \quad & \text{定义局部续延} \\
  & \mid \text{jump}\;j(\overline{a}) & \quad & \text{跳到汇合点} \\
  & \mid \text{case}\;a\;\text{of}\;\{\overline{q\;\overline{x}\to e}\} & \quad & \text{按原子的值选择分支} \\
  & \mid \text{return}\;a & \quad & \text{返回一个值} \\
  & \mid \text{tailcall}\;a(\overline{a}) & \quad & \text{用这次调用替换当前调用} \\
  & \mid \text{fail}_{s} & \quad & \text{在源码位置 }s\text{ 报模式匹配失败}
\end{aligned}
$$

上方加横线表示这里可以放一串东西。$j$ 是汇合点的名字，$q$ 可以是分支要匹配的构造器标签、字面量，或者指示默认分支的标记。构造器分支会给取出的各个字段绑定名字 $\overline{x}$。字面量分支和默认分支都没有要绑定的字段。

再往下拆，就能看出“正规形”规整在哪里。每个操作数一定都是原子，没有把别的调用塞进操作数里、留待现场求值的情况。

$$
\begin{aligned}
a & = x & \quad & \text{局部变量} \\
  & \mid g & \quad & \text{全局引用} \\
  & \mid \ell & \quad & \text{字面量} \\
r & = a & \quad & \text{原子} \\
  & \mid a(\overline{a}) & \quad & \text{函数调用} \\
  & \mid K(\overline{a}) & \quad & \text{构造器} \\
  & \mid p(\overline{a}) & \quad & \text{原语操作} \\
  & \mid \lambda\overline{x}.\;e & \quad & \text{函数值} \\
  & \mid \text{field}\;a\;i & \quad & \text{按下标 }i\text{ 取字段}
\end{aligned}
$$

这里 $K$ 是构造器，$p$ 是原语。为方便阅读，式子里省略了原语的源码位置。Core 里的 lambda 每层接收一个参数，这里的 lambda 可以一次接收多个。类型抽象和类型应用则已经在上一步擦掉了。

普通调用和尾调用在语法上就能分清。$\mathsf{let}\;x = f(a)\;\mathsf{in}\;e$ 表示从 $f(a)$ 回来以后，还得把值交给 $x$，接着执行 $e$。$\mathsf{tailcall}\;f(a)$ 后面就没有这样一段主体了。跳去汇合点的 `jump` 后面也没有要接着执行的主体，不过它的目的地仍在当前函数内。

Rust 枚举区分了普通绑定、递归闭包、分支、汇合点，以及直接交出控制权的几种写法。`Fail` 记下了模式匹配失败的源码位置。

```rust
#[derive(Clone, Debug)]
/// Jumps transfer within a function; tail calls replace its activation.
pub enum Anf {
    Let(Var, Rhs, Body),
    LetRec(Vec<(Var, Lambda)>, Body),
    Join(Id, Vec<Var>, Body, Body),
    Jump(Id, Vec<Atom>),
    Case(Atom, Vec<(AltCon, Vec<Var>, Body)>),
    Ret(Atom),
    TailCall(Atom, Vec<Atom>),
    Fail(Span),
}
```

[源码 · anf.rs 第 45～56 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch18_anf/src/anf.rs#L45-L56)

转换器里的上下文只有三种。算出一个原子以后，`finish` 负责按照上下文的要求处理它。尚未处理的转换任务放在一个向量里，因此一个特别长的表达式不会变成一串特别深的 Rust 函数调用。

```rust
#[derive(Clone)]
enum Context {
    Return,
    Jump(Id),
    Bind(Var, Body),
}

fn finish(&mut self, a: Atom, k: Context) -> Body {
    Rc::new(match k {
        Context::Return => Anf::Ret(a),
        Context::Jump(j) => Anf::Jump(j, vec![a]),
        Context::Bind(v, b) => Anf::Let(v, Rhs::Atom(a), b),
    })
}
```

[源码 · anf.rs 第 65～70 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch18_anf/src/anf.rs#L65-L70) · [源码 · anf.rs 第 122～128 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch18_anf/src/anf.rs#L122-L128)

遇到不在尾位置的 `case`，转换器会给剩余的上下文建一个汇合点，用一个新参数接收分支的结果，再让每个分支都按“跳到这里”的上下文去转换。Core 里已经有的汇合点会保留下来。挨在一起的几层 lambda 则会合成一个多参数 lambda。所以这里说的**元数**（arity）是从函数定义的语法里直接数出来的。

转换函数应用时，柯里化调用的先后顺序不能乱。在 `f a b` 里，`f a` 可能还没轮到求 `b` 就已经报错。要是把两个参数一口气全算完再发起调用，第一个报错就可能换了。只有知道函数的语法元数，转换器才敢把参数凑成一批，而且最多凑到这个数。未知函数一次只吃一个参数。如果实参比已知元数还多，就先完成已知的那一截调用，再处理余下的参数。`known-prefix-error` 这个回归用例专门盯着这个顺序。

`anf_simplify` 会传播原子值，在构造器已知时直接选中对应分支，删掉结果没用而且保证能正常算完的右侧表达式，以及化简那种刚定义就立即给足参数调用的 lambda。函数调用、带效果的操作和可能失败的原语都得留下。做 β 化简时还要检查函数值是否**逃逸**，也就是它是否被拿到原本那次直接调用以外的地方使用。比如之前的 `id id`，第二个 `id` 是作为参数传进去的，定义 `id` 的绑定就不能想删就删。执行之前还有一遍独立的 ANF 结构检查，核对名称、作用域、汇合点需要几个参数，以及新引入的名字是否确实是新名字。

ANF 机在循环中切换状态，执行一段代码、施用一个函数，或者带着值返回。它的栈帧用来记两种事。一种是非尾调用返回后，要把值交给谁、接着干什么。另一种是给的参数太多，第一次调用之后还剩哪些参数要用。

```rust
enum State {
    Code(Body),
    Apply(Value, Vec<Value>),
    Return(Value),
}

enum Frame {
    Bind(Var, Body, Env, Joins),
    Apply(Vec<Value>),
}
```

[源码 · anf_machine.rs 第 86～90 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch18_anf/src/anf_machine.rs#L86-L90) · [源码 · anf_machine.rs 第 81～84 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch18_anf/src/anf_machine.rs#L81-L84)

每个闭包带着一个 lambda 和一份共享的词法环境。遇到一组递归函数，机器在自己管理的表里为整组建一次记录，相关闭包各自记住这组的下标。这样既能避免引用计数形成环，也不用每次调用都重建一组闭包。给的参数太少，就返回一个**部分应用**值。给多了，就把多出来的参数继续用到返回的函数上。尾调用直接替换当前机器状态。跳去汇合点时，使用的是定义汇合点时捕获的词法环境。这两种转移都无须递归调用 Rust 解释器。执行原语时会通过一个只转换数据的适配层，复用已有的原语实现。控制流仍由 ANF 机自己执行。

## 跑起来看看

```text
$ cd crates/ch18_anf
$ cargo run -q -- run tests/programs/anf/case-continuation.fun --lint
7
$ cargo run -q -- dump --pass=types tests/programs/anf/case-continuation.fun
main : IO ()
```

试试 `--pass=erase`，可以看见擦除了类型的 Core。改成 `--pass=anf`，就能看到哪些操作的结果被起了名字。再给后者加上 `--no-simplify` 跑一次，对照一下哪些是转换本身做的、哪些是后来简化出来的。运行示例时也可以加 `--engine=core`，看看参考解释器是否给出同一个答案。完整的转储里连 Prelude 也有。如果只想琢磨一个变化，比如非尾位置的 `case`，或者守卫失败时跳去的汇合点，可以直接看 `tests/anf.rs` 里的小快照。

除了手写的例子，`testkit/src/gen.rs` 还会生成小而且类型正确的程序，里面有算术、局部绑定、分支、递归和列表。测试会独立用溢出回绕的整数运算算出期望值，Core 和 ANF 两个引擎都得跟它对上。失败时保存出错的源码和随机种子，这样一个偶尔出现的错误答案，就变成了一段可以重现、可以调试的程序。

用递归求和跑上一百万次调用，很适合检验机器怎么转移控制。光看它跑完还不够，还要数一数同时有多少个续延在等结果。尾调用应该直接替换当前计算。非尾调用则可能往堆上分配的续延栈里添一件未完成的工作。两种情况都不用再添一层 Rust 调用栈帧。Core 也有自己的显式机器，两个引擎对着跑，还能检查转换有没有改变答案。

## 练习

1. **基础。** 看本章开头示例未经简化的 ANF 转储。`case` 选出结果以后要执行的续延在哪里？
2. **进阶。** 让两个实参各自报出不同的错误。分别构造一次目标已知和一次目标未知的调用，判断哪个实参应该先求值，再运行核对。
3. **进阶。** 整数除法没有 IO 效果。为什么即使后面用不到它的结果，删掉这次除法仍可能出错？
4. **挑战。** 给机器加上计数，分别运行尾递归和非尾递归的求和程序，比较它们的最大续延深度。
5. **挑战。** 给确定性的程序生成器加一种新的、类型正确的语言构造，再加一套独立计算预期结果的办法。

## 延伸阅读

- 用直接风格表示程序，以及基于续延的编译[^flanagan1993]。
- 汇合点与共享续延[^maurer2017]。
- CEK 机的三个要素，控制、环境和续延[^felleisen1986]。
- 用续延构建编译器[^appel1992]。
- 多态与类型擦除[^pierce2002]。

[^flanagan1993]: Cormac Flanagan、Amr Sabry、Bruce F. Duba、Matthias Felleisen（1993）。[The essence of compiling with continuations](https://doi.org/10.1145/155090.155113)。
[^maurer2017]: Luke Maurer、Paul Downen、Zena M. Ariola、Simon Peyton Jones（2017）。[Compiling without continuations](https://doi.org/10.1145/3062341.3062380)。
[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。
[^felleisen1986]: Matthias Felleisen、Daniel P. Friedman（1986）。[Control Operators, the SECD-Machine, and the Lambda-Calculus](https://hdl.handle.net/2022/33996)。
[^appel1992]: Andrew W. Appel（1992）。[Compiling with Continuations](https://doi.org/10.1017/CBO9780511609619)。

---

[← 第 17 章　Prelude 与自动派生](../part-2/ch17-prelude.md) · [中文目录](../README.md) · [第 19 章　闭包转换 →](ch19-closures.md)
