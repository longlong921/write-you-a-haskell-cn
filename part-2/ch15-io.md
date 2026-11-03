# 第 15 章　Monad、do 记法与 IO

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch15-io.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch15_io)

> **新增文件**
>
> - `src/io_lower.rs`
>
> **改动文件**
>
> - `src/desugar.rs`
> - `src/eval.rs`
> - `src/typecheck.rs`
> - `src/ast.rs`
> - `src/data.rs`
> - `src/elaborate.rs`
> - `src/fixity.rs`
> - `src/kinds.rs`
> - `src/layout.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/primitives.rs`
> - `src/rename.rs`
> - `src/session.rs`

## 一个程序怎样和人打交道？

到目前为止，程序和外界打交道的方式很单一。驱动程序求出 `main` 的值，打印出来，然后退出。现在我们想写的程序能先问一句，读到对方的回答，再决定下一步做什么。

我们希望普通表达式仍然是纯的。为此，先把“打算做什么”做成一个值，等到了约定好的入口才真正执行。比如，求出一个表示打印的 IO 值，这一步就可以算完，屏幕上还没有任何输出。有了这条界线，就能用熟悉的函数构造、组合动作。

怎样描述这些组合方式？第 14 章的类型类正好派上用场。`Functor`、`Applicative` 和 `Monad` 既适用于 IO，也能描述 `Maybe`、`Either` 和列表上的一些常用操作。先从每个操作实际干了什么入手，再亲手把简写展开，这几个名字就容易理解了。

## 这一章的语言长什么样

从这一章起，完整程序的入口必须具有类型 `main :: IO ()`。签名可以自己写，也可以让编译器推断。最小的例子只要调用 Prelude 新增的打印函数。

```fun
main = putStrLn "hello"
```

[源码 · hello.fun 第 1～1 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch15_io/tests/programs/io/hello.fun#L1-L1)

`do` 记法也适用于普通数据。下面两次绑定分别取用 `Just` 里的值，最后的 `pure` 通过同一个 `Maybe` 实例把结果重新装进去。

```fun
main = print (do { x <- Just 20; y <- Just 22; pure (x+y) })
```

[源码 · maybe.fun 第 1～1 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch15_io/tests/programs/io/maybe.fun#L1-L1)

`do` 块的最后必须是一个表达式。`x <- action` 引入的 `x` 只在后面的语句里可见。`let` 在这里是一条引入局部声明的语句，后面不用配 `in`。绑定左侧可以写嵌套模式，包括元组、列表、用 `@` 同时命名整体的 as 模式，以及名义记录模式。如果某个模式可能匹配失败，一旦真的失败，Fun 会带上源码位置调用 `error`。Fun 没有 `MonadFail`，所以即使整个块用的是 `Maybe`，这种模式匹配失败也会引发运行时错误。

程序通过 `putStr`、`putStrLn`、`getLine`、`print` 和 `exitWith` 同外界打交道。要按顺序遍历、丢弃动作的结果，或者按条件执行动作，可以用 `mapM`、`mapM_`、`sequence_`、`when` 和 `unless`。缺少入口，或入口的类型不对，会报 `E0250`。如果已经显式写了签名，但函数体对不上，可能在检查签名时就先报 `E0203`。

## 理论

假设第一个动作会产生一个 `a`，而程序的后半段知道怎样用这个 `a` 构造下一个动作。**绑定**（bind，写作 `>>=`）把这两段接起来。它的第二个参数是函数，因为下一步做什么可能要等知道上一步的结果才决定。从左往右读这个类型就能看到这个关系[^jones1993]。

$$
\mathsf{Monad}\ m \Rightarrow m\ \alpha \to (\alpha \to m\ \beta) \to m\ \beta
$$

先不管内部怎么表示，把下面的小程序展开看看。

```fun
main = do
  name <- getLine
  putStrLn name
```

展开后就是 `getLine >>= (\name -> putStrLn name)`。为什么需要这层 lambda？第一步还没读到输入，就没有 `name` 可以拿来构造打印动作。先把“拿到名字以后要做什么”保存为函数，就能把整个组合动作建好，建好时既没读，也没打印。等驱动程序执行它，才会读一行，把读到的字符串传给 lambda，再执行 lambda 返回的动作。

换一个例子，`let unused = putStrLn "hello" in pure ()` 会构造一个打印动作，随即把它丢掉。执行最后得到的动作，屏幕上什么也不会出现。不过，Fun 是严格求值的，构造动作时需要的纯实参依然会先求值。

在源语言的类型检查器眼里，`IO` 是一个只凭名字辨认、内部表示对用户不可见的类型构造器。到了 Core，一个动作则表示成从单位类型到结果的函数。

$$
\mathsf{IO}\ \alpha \quad\Rightarrow\quad () \to \alpha
$$

多出的单位参数 `()` 给了我们一个推迟执行的地方。构造一个函数，函数体并不会立刻运行。`returnIO` 构造的函数会在调用时返回先前存好的值。`bindIO` 构造的函数会在调用时执行第一个动作，再把它的结果交给下一步。真正产生外部效果的原语都放在这些函数体里。

只有受信任的 Prelude 和编译器能使用这个内部表示。用户程序拿到的是抽象的 `IO a` 值，不能随手传个 `()` 就在任意表达式里启动动作。有这道边界，普通 Fun 表达式才能保持纯性。

别忘记严格求值。构造 `putStrLn "hello"` 时，先求出字符串实参，再得到一个动作。真正打印要等到动作执行。若改成 `putStrLn (error "bad")`，构造动作时就要求得字符串，因此可能连动作都没造好便报错。编排动作决定外部效果发生的先后，纯实参何时求值仍服从 Fun 的规则。《Haskell 报告》也给出了类似的 `do` 展开规则[^haskell2010]，但解释 Fun 的行为时，要记得 Haskell 用的是惰性求值。

Monad 定律约定了一个实例应当怎样表现。编译器负责检查类型，这些等式是否成立则由实现实例的人负责。尤其在严格语言里，只要等式的一侧可能永远算不完，推理就要小心。底层仍然用[第 14 章](ch14-type-classes.md)的字典传递，求值器无需新增一套选择类型类方法的机制。

## 实现

解析器会在词法重命名之前调用 `desugar::do_block`。从块的末尾往前处理，已经处理好的后半块自然成了当前绑定的**续延**，也就是“拿到结果后要接着做的事”。左侧只绑定一个变量时，直接生成 lambda。左侧是可能失败的模式时，就生成带失败分支的 `case`。编译器生成的临时名字无法写成合法的源码标识符，因此不会跟用户的变量撞名。

```rust
/// Expand do syntax into ordinary applications, matches, and local bindings.
pub fn do_block(
    statements: Vec<Statement>,
    span: Span,
) -> Result<crate::ast::Expr, Diagnostic> {
    use crate::ast::{
        Expr as E, Kind as K, Pat as P, PatKind as PK, Row as R,
        RowBody as B,
    };
    let node = |kind| E { kind, span };
    let app = |a, b| node(K::App(Box::new(a), Box::new(b)));
    let mut statements = statements;
    let Some(Statement::Expr(mut body)) = statements.pop() else {
        return Err(Diagnostic::error(
            ErrorCode::Parse,
            "do must end with an expression",
            span,
        ));
    };
    for statement in statements.into_iter().rev() {
        body = match statement {
            Statement::Expr(e) => {
                app(app(node(K::Var(">>".into())), e), body)
            }
            Statement::Let(bs) => node(K::LetGroup(bs, Box::new(body))),
            Statement::Bind(p, e) => {
                let continuation = match &p.kind {
                    PK::Var(n) => node(K::Lam(n.clone(), Box::new(body))),
                    PK::Wild => node(K::Lam("$do".into(), Box::new(body))),
                    _ => {
                        let site = p.span;
                        let rows = vec![
                            R {
                                patterns: vec![p],
                                body: B::Plain(Box::new(body)),
                                locals: vec![],
                                span: site,
                            },
                            R {
                                patterns: vec![P {
                                    kind: PK::Wild,
                                    span: site,
                                }],
                                body: B::Plain(Box::new(E {
                                    kind: K::DoFail,
                                    span: site,
                                })),
                                locals: vec![],
                                span: site,
                            },
                        ];
                        node(K::Lam(
                            "$do".into(),
                            Box::new(node(K::Case(
                                Box::new(node(K::Var("$do".into()))),
                                rows,
                            ))),
                        ))
                    }
                };
                app(app(node(K::Var(">>=".into())), e), continuation)
            }
        };
    }
    Ok(body)
}
```

[源码 · desugar.rs 第 343～408 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch15_io/src/desugar.rs#L343-L408)

这样安排以后，编译器生成的绑定会和手写的绑定一起接受作用域检查，不必在重命名之后再单独跑一遍 `do` 脱糖。`do_pattern` 会检查 `<-` 左边写的东西能不能当作模式。遇到失败分支，先生成带源码范围的 `DoFail` 节点，之后再翻译成普通的错误调用，把文件名、行号和列号一并交给它。等到了 Core，所有 `do` 节点都已经展开了。

类型检查器只凭 Prelude 受信任的 `FileId`，才允许把它内部的 IO 实现看作接收单位值的函数。用户能看到的类型方案里，`IO` 仍然是名义类型。翻译成 Core 之后、运行 lint 之前，`io_lower::lower` 会遍历 Core 里所有的类型，包括绑定变量的类型、类型应用、构造器字段、匹配结果类型，以及字典类型。只盯着最外层的 `IO a` 替换，会漏掉 `Monad IO` 字典这种高阶用法。这里的 `IO` 还没有接收类型实参，遍历时要把它换成部分应用的函数类型构造器 `(->) ()`，意即“给我一个结果类型，我就能构成从单位类型到它的函数类型”。

运行器先安装函数闭包，再按依赖顺序对严格的顶层值求值。最后，把单位值传给入口函数，程序的 IO 才正式开始。

```rust
/// Initialize globals, then run the checked IO action exactly once.
pub fn run_io(program: &core::Program, fuel: usize) -> Result<(), Failure> {
    execute(program, fuel, true).map(|_| ())
}
```

[源码 · eval.rs 第 453～456 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch15_io/src/eval.rs#L453-L456)

一个没用到的顶层动作，也要在启动时初始化成一个值。它的函数体却不会被执行。下面的回归用例就能看出其中区别。

```fun
unused = putStrLn "bad"

main = putStrLn "ok"
```

[源码 · deferred.fun 第 1～2 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch15_io/tests/programs/io/deferred.fun#L1-L2)

Fun 程序出了错，和程序自己要求以某个退出码结束，在求值器里由不同的结果表示。正因为做了区分，`exitWith 7` 才会产生退出码 7，而不会被当成错误信息打印出来。

```rust
/// Language termination is separate from an ordinary runtime failure.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Failure {
    Runtime(String),
    Exit(i32),
}
```

[源码 · eval.rs 第 8～13 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch15_io/src/eval.rs#L8-L13)

程序完成时，以及输出运行时诊断前，驱动程序都会刷新标准输出。输入原语在读取之前也会刷新输出，这样即使提示语末尾没有换行，用户也能先看见提示，程序才阻塞着等回答。测试运行器接受 `.in` 文件，把同样的输入发给每个执行引擎。另有一个回归测试用管道跟程序一问一答，无须借助终端缓冲就能验证刷新是否正确。

把已有的例子搬到这一章时，要稍微改一下。驱动程序不会再替我们打印 `main` 的值，打印得写进程序本身。如果某个值没有 `Show` 实例，要么补上实例，要么从中选出能打印的内容。如果只是调试编译器里的一个小片段，还可以用 `--print-value` 调用先前的值打印器。

## 跑起来

在原书仓库根目录，先进入本章的 crate。第一条运行命令能看到程序显式执行的打印动作，接着的转储命令能看到推断出的入口类型。

```text
$ cd crates/ch15_io
$ cargo run -q -- run tests/programs/io/hello.fun --lint
hello
$ cargo run -q -- dump --pass=types tests/programs/io/hello.fun
main : IO ()
$ cargo run -q -- run tests/programs/io/deferred.fun --lint
ok
```

用 `--pass=core` 可以找出传入的字典参数和接收单位值的 lambda。不妨对照两个打印动作，一个构造出来就搁在一旁，另一个被 `main` 排进执行顺序。两个都已经是值，只有后者的函数体会运行。本章使用的执行引擎仍然是 Core 解释器。

## 练习

1. **基础。** 把示例里的第二个 `Just` 换成类型标注合适的 `Nothing`，解释为何它后面的续延不会被调用。
2. **基础。** 构造一个用不到的打印动作，再让它的字符串实参调用 `error`。比较这两种程序的行为。
3. **进阶。** 在同一个 `do` 块里写一个 `let` 语句、一次元组模式绑定，以及一次遮蔽同名变量的绑定。沿着脱糖后的代码，画出每个名字的作用域。
4. **进阶。** 在 Core 转储里找一本 `Monad IO` 字典，指出降低 IO 表示这一步会改写哪些类型。
5. **挑战。** 扩展管道交互测试，让程序连续给出两条提示。不依赖终端的行缓冲，验证提示和应答的顺序。

## 延伸阅读

- 《Haskell 报告》里的 `do` 表达式和 Monad 相关库[^haskell2010]。
- 接受高阶类型构造器的类型类[^jones1993]。
- 字典传递和带约束的类型（qualified types）[^jones1999]。

[^jones1993]: Mark P. Jones（1993）。[A System of Constructor Classes: Overloading and Implicit Higher-Order Polymorphism](https://doi.org/10.1145/165180.165190)。
[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。
[^jones1999]: Mark P. Jones（1999）。[Typing Haskell in Haskell](https://web.cecs.pdx.edu/~mpj/thih/thih-sep1-1999/index.html)。

---

[← 第 14 章　类型类](ch14-type-classes.md) · [中文目录](../README.md) · [第 16 章　编译模式匹配 →](ch16-pattern-compilation.md)
