# 第 5 章　求值

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch05-evaluation.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch05_eval)

> **新增文件**
>
> - `src/cek.rs`
> - `src/cbn.rs`
>
> **改动文件**
>
> - `src/eval.rs`
> - `src/types.rs`
> - `src/ast.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/names.rs`
> - `src/normal.rs`

## 为什么要换一种求值方式

想解释一个表达式怎么求值，递归求值器很好用。遇到一个子表达式，就调用一次 Rust 函数进去算，代码的形状和语法树几乎一样。可每多嵌套一层调用，就可能多占一层 Rust 调用栈。我们从没打算规定小语言的程序能递归多深，却无意中把宿主语言的栈大小变成了它的限制。把栈调大，只能让问题晚一点出现。

换个做法。每当求值器暂时放下手头的工作，先写清楚“拿到结果之后，还要做什么”。把这份记录存进我们自己的数据结构，再用一个循环驱动整个求值过程，原来交给 Rust 调用栈保管的信息就能由我们自己掌握了。这就是抽象机的思路。

本章用的抽象机叫 **CEK 机**。C（control）告诉我们当前在算什么，E（environment）记下当前的变量绑定，K（continuation，续延）记下得到结果之后还要做什么[^felleisen1986]。把求值器原先暗中记着的事摊开来，我们就能直接查看它下一步打算做什么。

## 这一章的语言长什么样

写递归绑定时，要在等号前声明类型，等号右边则必须在语法上就是一个 lambda 表达式。比如 `let rec x : Int = x in x`，虽然带了类型标注，右边却只有一个变量，解析器会报 `E0120`。

```fun
let rec sum : Int -> Int -> Int =
  \(acc : Int) -> \(n : Int) ->
    if n == 0 then acc else sum (acc + n) (n - 1)
in sum 0 100
```

递归函数可以在自己的函数体里使用自己的名字，外面 `let` 的 `in` 后面也能用这个名字。类型检查器会核对声明的类型和从等号右侧推断出的类型是否一致。也可以在程序里显式写 `fix`。它比函数应用结合得更紧，所以下面的式子会先用 `fix` 得到递归函数，再把 `6` 传给它。

```fun
fix (\(self : Int -> Int) -> \(n : Int) ->
  if n == 0 then 1 else n * self (n - 1)) 6
```

两个例子都能通过类型检查，但只有求和是尾递归。求阶乘时，每一层都得等递归调用把结果交回来，才能乘上本层的 `n`。

## 理论

如果给函数 `f` 一个值，再从 `f` 得到的还是那个值，它就是 `f` 的一个**不动点**。据此可以把 `fix f` 展开来，按 `f (fix f)` 理解它的行为。对应的类型规则也很好理解，`f` 接收什么类型，就得返回什么类型[^pierce2002]。

$$
\frac{\Gamma \vdash f : \tau \to \tau}
{\Gamma \vdash \mathsf{fix}\ f : \tau}\;\text{(Fix)}
$$

有了 `fix`，类型完全正确的程序也可能永远算不完。试试 `fix (\(x : Int) -> x)`，给它多少“燃料”（步数预算），它就能耗掉多少。类型检查器尽到了自己的职责，类型正确从来没有保证过这里的递归一定会结束。`let rec` 用的也是不动点，只是额外要求右侧写成函数，好让运行时知道怎么把它初始化。后面的 Fun 也会保留这个限制。

**按值调用**（call-by-value）先算出函数和实参的值，再进入函数体。**按名调用**（call-by-name）把尚未求值的实参表达式代入形参出现的位置，函数体每次用到它时再算。遇到 lambda 本身就停下，不继续算它的函数体。**按需调用**（call-by-need）会先把计算挂起，第一次真正需要它时才算，再把结果存下来，让各处共享。本章用来做对照的小解释器实现按名调用，直接做替换，不缓存结果。这一点从替换代码就能看出来。第 3 章的完整正规序归约器会深入 lambda 的函数体，这里的按名求值器遇到 lambda 就停，这也是它被称为“弱”求值的原因。

Fun 选择按值调用，是为了在逐步改写中间表示时，容易看清每一步的计算顺序以及什么时候需要分配内存。构造器也会采用严格求值。不过 lambda 本身就算一个值，所以我们可以把某个动作写进函数，先持有这个函数，等调用时再执行动作。第 15 章实现 `IO` 就会用到这一点。如果要把惰性作为语言的默认行为，就还得为挂起的计算设计运行时表示，并重新分析副作用什么时候发生、空间会被占用多久。这些问题超出了本书编译器的范围。

## 实现

解析器用下面的构造函数检查 `let rec` 等号右侧是否满足限制，再把合法的递归绑定改写成更基础的 `let` 和 `fix`。

```rust
/// Lower function-only recursive bindings to a fixed point with an annotated self parameter.
pub fn recursive(
    name: String,
    ty: Type,
    rhs: Expr,
    body: Expr,
    start: usize,
    end: usize,
) -> Result<Expr, Diagnostic> {
    if !matches!(rhs.kind, Kind::Lam(..)) {
        return Err(Diagnostic::new(
            "E0120",
            "recursive binding must be a function",
            rhs.span,
        ));
    }
    let self_function =
        Expr::new(Kind::Lam(name.clone(), ty, Box::new(rhs)), start, end);
    let fixed = Expr::new(Kind::Fix(Box::new(self_function)), start, end);
    Ok(Expr::new(
        Kind::Let(name, Box::new(fixed), Box::new(body)),
        start,
        end,
    ))
}
```

[源码 · ast.rs 第 151～175 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch05_eval/src/ast.rs#L151-L175)

`Value::Recursive` 保存构造不动点的函数。以后每次读到这个递归变量，就把它自身的递归引用传给保存的函数。对递归函数来说，这一步会在原先的环境里重建闭包，不必让 Rust 对象相互持有而形成所有权环。进入 `fix` 时，定义不动点的函数会立即求值，所以初始化阶段的错误也会当场暴露。我们用这套实现来讲清楚显式递归，它只处理这个场景，不能拿来当作通用的惰性绑定。

### 把还没做完的事记下来

来算 `1 + (2 * 3)`。算出 `1` 后，还欠一次加法，但得先知道右边的值。等到开始算 `2 * 3`，又多欠了一次乘法。把这些待办事项逐条写下来，把轮到先做的那件放在最前面，就得到下表。这里只列出和算术有关的步骤，完整的机器轨迹还会更细。

| 当前在做什么 | 接下来还要做什么 |
| --- | --- |
| 求 `1` 的值 | 求 `2 * 3` 的值，再做加法 |
| 求 `2` 的值 | 求 `3` 的值，先乘，再与 `1` 相加 |
| 求 `3` 的值 | 与 `2` 相乘，再与 `1` 相加 |
| 返回 `6` | 与 `1` 相加 |
| 返回 `7` | 全部完成 |

一旦这些待办事项成了数据，求值过程就可以写成循环。比如，求左操作数时，`Left` 帧替我们记住还没求值的右操作数和它的环境。轮到右边时，`Right` 帧替我们保管刚算出的左侧整数。原先藏在 Rust 调用栈里的信息，现在都有了名字。

`cek.rs` 把当前要做的事（`Control`）和记录后续工作的续延帧（`Frame`）分开。每一种帧都对应递归求值器里一个“调用下去，等结果回来再继续”的位置。

```rust
/// The remaining work after the current expression returns a value.
enum Frame {
    Argument(Rc<Expr>, Rc<Env>),
    Apply(Value),
    Let(String, Rc<Expr>, Rc<Env>),
    If(Rc<Expr>, Rc<Expr>, Rc<Env>),
    Neg,
    Left(Op, Rc<Expr>, Rc<Env>),
    Right(Op, i64),
    Fix,
}

/// Control alternates between evaluating an expression and returning a value.
enum Control {
    Expr(Rc<Expr>, Rc<Env>),
    Value(Value),
    Apply(Value, Value),
}

/// Statistics make tail-space behavior testable without relying on timing.
pub struct ResultValue {
    pub value: Value,
    pub steps: usize,
    pub max_continuations: usize,
}
```

[源码 · cek.rs 第 9～33 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch05_eval/src/cek.rs#L9-L33)

以函数应用为例。机器先压入 `Argument` 帧，去求函数的值。这一步完成后，换成 `Apply` 帧，轮到求实参的值。两个值都拿到了，就取出闭包在定义处捕获的环境，加上“形参绑定到实参值”这一项，然后进入函数体。此时，这次应用用到的帧已经出栈了。

```rust
Control::Apply(
    Value::Closure(name, body, captured),
    argument,
) => {
    let mut env = (*captured).clone();
    env.push((name, argument));
    Control::Expr(body, Rc::new(env))
}
Control::Apply(_, _) => {
    return Err("expected a function".into());
}
```

[源码 · cek.rs 第 117～127 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch05_eval/src/cek.rs#L117-L127)

回头看开头的求和。一次递归调用返回的就是调用方要交出去的答案，调用方没有后续工作需要记住。阶乘却还欠着一次乘法，这件待办事项就得占一个帧。

现在，用 CEK 机求和、求阶乘，都不会因为这些递归调用让 Rust 的调用栈越长越深。但看我们自己维护的续延栈，只有尾递归的求和也能让它保持有界。我们怎么验证这个区别？分别跑一次很短的求和和一次包含一百万次调用的求和，比较两次记录到的最大续延深度。如果只知道程序碰巧在一台栈很大的机器上跑完了，就没法从中看出这么多。

完整的状态转移循环会按表达式的种类往下走。一个值返回时，就弹出一个等待它的帧，接着做帧里记录的事。每次转移前都记录当前深度，用来统计 `max_continuations`。我们还用燃料计数器限制步数，这样也能放心观察不终止的例子。CEK 一次状态转移算一步，递归的树求值器每访问一个表达式算一步，两者的步数不能直接对比。

用作对照的按名求值器很小。它处理函数应用和 `let` 时直接做替换，省去了环境查找。

```rust
Kind::App(f, a) => {
    let function = evaluate(f, fuel)?;
    if let Kind::Lam(name, _, body) = function.kind {
        evaluate(&substitute(&body, &name, a), fuel)
    } else {
        Err("expected a function".into())
    }
}
Kind::Let(n, a, b) => evaluate(&substitute(b, n, a), fuel),
```

[源码 · cbn.rs 第 16～24 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch05_eval/src/cbn.rs#L16-L24)

如果要改成按需调用，就要把这种替换换成一个共享单元，第一次需要实参时才求值，并把结果存在单元里供后续使用。这项实现留待以后。这里的默认执行路径仍然走严格求值的 CEK 机，想使用对照解释器要显式选择。

## 跑起来

```text
$ cd crates/ch05_eval
$ cargo run -q -- run tests/programs/sum-tail.fun
5050
$ cargo run -q -- run --engine=core tests/programs/factorial.fun
720
$ cargo run -q -- dump --pass=types tests/programs/fix-factorial.fun
Int
$ cargo run -q -- run --engine=cbn tests/programs/strict-argument.fun
1
$ cargo test -q a_million_tail_calls_use_bounded_continuations
```

默认引擎是 `cek`。`core` 选择递归的闭包求值器，`cbn` 选择用来对照的按名解释器，`normal` 则保留了前面章节的完整正规序归约器。用 `--fuel=N` 限制执行步数，尤其是递归遍历语法树的解释器，预算宜小一些。只有 CEK 消除了对宿主调用栈深度的依赖。两个严格求值引擎应当对结果和失败给出一致判断。再想想按名解释器。如果某个实参一算就会报错，但函数体压根儿没用到它，会发生什么？先猜猜哪些引擎会报错、报什么错，跑一下，再去看代码。

## 练习

1. **基础。** 求 `1 + 2 * 3` 的值，每当一个操作数返回时，画出当时的续延帧。
2. **进阶。** 把尾递归求和改写成非尾递归求和，分别测量两个版本的最大续延深度。
3. **进阶。** 解释为什么闭包必须捕获定义时的环境。要是换成调用方的环境，会出什么问题？
4. **挑战。** 构造一个表达式，其中一段开销很大的计算在按名调用时会重复执行，按值调用时却只算一次。如果让多次使用共享结果，又会有什么变化？

## 延伸阅读

Felleisen 和 Friedman 展开介绍了用抽象机描述求值的做法[^felleisen1986]。Pierce 讨论了求值策略、类型和不动点之间的联系[^pierce2002]。Appel 把“将续延显式表示出来”的相关技巧用到了编译过程里[^appel1992]。等我们写到 A-正规形（ANF）编译器，还会回头看到这层联系。

[^felleisen1986]: Matthias Felleisen、Daniel P. Friedman（1986）。[Control Operators, the SECD-Machine, and the Lambda-Calculus](https://hdl.handle.net/2022/33996)。
[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。
[^appel1992]: Andrew W. Appel（1992）。[Compiling with Continuations](https://doi.org/10.1017/CBO9780511609619)。

---

[← 第 4 章　简单类型 λ 演算](ch04-stlc.md) · [中文目录](../README.md) · [第 6 章　Hindley–Milner 类型推断 →](ch06-hindley-milner.md)
