# 第 7 章　设计 Fun

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch07-fun.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch07_fun)

> **新增文件**
>
> - `src/session.rs`
> - `src/core.rs`
> - `src/typecheck.rs`
> - `src/elaborate.rs`
> - `src/diagnostics/codes.rs`
> - `src/diagnostics/mod.rs`
> - `src/parse.rs`
> - `src/source.rs`
>
> **改动文件**
>
> - `src/eval.rs`
> - `src/ast.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/names.rs`
> - `src/types.rs`

## 把前几章的零件装到一起

我们已经写过解析器，看过函数如何求值，也让编译器学会了推断类型。从这一章起，这些本领会用到同一个编译器上。我们要做的语言叫 Fun，最后会把 Fun 程序编译成机器能直接运行的原生代码。每一章都留有可以单独运行的版本。以后要是想弄清某个设计从哪里来，可以随时退回到它刚出现的那一章。

把这些零件接在一起，还有些很实际的问题。类型检查器发现错误，该指给用户看源文件里的哪几个字符？某一遍处理做了一半就失败了，谁来叫停，免得下一遍拿着半成品继续往下跑？这一章会给名字、源码位置和错误信息安排一个共用的地方，然后把各个编译阶段（pass）接起来。后面加新功能时，就能沿用这套骨架。

## 这一章的 Fun 能写什么

现在要用一对花括号把整个程序括起来，里面是一条条声明，用分号隔开，最后一条后面的分号可写可不写。一个声明可以引用自己，也可以引用前面已经出现的声明。要想引用后面才定义的名字，或让两个函数互相调用，得等到第 9 章。

```fun
{
  twice f x = f (f x);
  sum n = if n == 0 then 0 else n + sum (n - 1);
  main = twice (\x -> x + 1) (sum 10);
}
```

这里 `sum 10` 算出 `55`，`twice` 再把“加一”用两遍，所以这个 `main` 的值是 `57`。

局部 `let` 也要写花括号，这一版的每个 `let` 里只能放一个绑定。递归用不着另写 `rec`。定义里用到了自己的名字，编译器就能认出它是递归绑定，但这样的绑定右边必须是 lambda。像 `let { x = x; } in x` 这样，右边求值时立刻就要用到一个还没初始化的 `x`，编译器会报 `E0120`。

`Char` 存一个 Unicode 标量值，`String` 是内建的 UTF-8 字符串。`++` 在这一版只有一种用法，就是拼接两个字符串。整数字面量可以写成十六进制，也可以用下划线分隔数字。注释有单行和块两种，块注释还可以嵌套。

```fun
{ {- outer {- nested -} -} main = "Hello, " ++ "λ\n"; }
```

`run` 仍然打印 `main` 的值，遇到字符串会带上引号，把换行之类的字符显示成转义序列。这一版还不会执行 `IO` 动作。内建整数函数 `div` 和 `mod` 都是柯里化的，参数可以分开传。`div` 的商向下取整，也就是朝负无穷方向取整，`mod` 给出与这个商配套的余数。例如 `div 7 (-3)` 是 `-3`，`mod 7 (-3)` 是 `-2`，正好满足 `7 = (-3) × (-3) + (-2)`。之前计算器按“向零截断”编写的除法测试用例也要随之调整。

## 为什么还需要一门 Core 语言

解析得到的语法树要保留程序员写了什么，比如名字的原始拼写、所用的语法，以及这些字符在源文件中的位置。供解释器执行的中间语言 Core 用途不同，只需留下求值所用的操作。把前者转换为后者，这一步叫**细化**（elaboration）。源码里为书写方便而省略的步骤，到 Core 里都要摊开来。以后我们还会给 Core 加上类型，让一个独立的检查器来验收编译器的每次变换[^pierce2002]。

无论程序变成什么样，Fun 的严格求值约定都得保住。函数调用先求出函数，再求出实参。运算符两边从左往右求值，条件表达式只求被选中的那个分支。这一版会先按声明顺序把每个顶层值初始化一次，最后才取 `main` 的值。所以哪怕某个顶层值一直没人用，只要它初始化时出错，程序照样会停下。顶层函数则要等到真正调用时才执行函数体。

类型系统仍然是 Hindley–Milner，沿用第 6 章基于层级的检查器[^damas1982][^kiselyov2013]。递归绑定只接受函数，正是为了避免“值还没初始化完，就要拿它自己来构造它”的困境。建立递归函数时只需创建闭包，进入函数体后才需要引用这个闭包本身。目前处理普通词法作用域，仍然用名字的拼写去表里查。等到第 9 章，才会真正靠唯一标识区分每个绑定。

## 实现

### 源码位置和编译会话

`source.rs` 给每个文件分配独立标识，并记下每一行从哪个位置开始。为了和词法分析器对得上，源码区间（span）用左闭右开的**字节**范围表示。给人看错误信息时，列号则按 Unicode 标量值计数。文件标识也要算在位置里，否则两个文件里相同的字节偏移就会被当成同一处。

```rust
/// Index of a source file in a session's source map.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct FileId(pub u32);

/// Half-open byte offsets within one source file.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct Span {
    pub file: FileId,
    pub start: u32,
    pub end: u32,
}

impl Span {
    pub fn new(file: FileId, start: usize, end: usize) -> Self {
        Self {
            file,
            start: start.try_into().expect("source exceeds 4 GiB"),
            end: end.try_into().expect("source exceeds 4 GiB"),
        }
    }
    pub fn range(self) -> Range<usize> {
        self.start as usize..self.end as usize
    }
    pub fn through(self, other: Self) -> Self {
        assert_eq!(self.file, other.file);
        Self {
            end: other.end,
            ..self
        }
    }
}

/// Original text and line starts; columns count Unicode scalar values.
pub struct SourceFile {
    pub name: String,
    pub text: String,
    lines: Vec<usize>,
}

#[derive(Default)]
/// Owns every source used by this compilation, including future Prelude input.
pub struct SourceMap {
    pub files: Vec<SourceFile>,
}

impl SourceMap {
    pub fn add(&mut self, name: impl Into<String>, text: String) -> FileId {
        assert!(u32::try_from(text.len()).is_ok(), "source exceeds 4 GiB");
        let mut lines = vec![0];
        lines.extend(text.match_indices('\n').map(|(i, _)| i + 1));
        let id = FileId(
            self.files.len().try_into().expect("too many source files"),
        );
        self.files.push(SourceFile {
            name: name.into(),
            text,
            lines,
        });
        id
    }
    pub fn get(&self, file: FileId) -> &SourceFile {
        &self.files[file.0 as usize]
    }
    pub fn line_col(&self, span: Span) -> (usize, usize) {
        let file = self.get(span.file);
        let offset = span.start as usize;
        let line = file.lines.partition_point(|start| *start <= offset) - 1;
        (
            line + 1,
            file.text[file.lines[line]..offset].chars().count() + 1,
        )
    }
}
```

[源码 · source.rs 第 5～77 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch07_fun/src/source.rs#L5-L77)

`names.rs` 手写了一个字符串驻留表（interner）和一个新名字生成器。拼写相同的字符串只登记一次，共用同一个 `Symbol`。每向生成器要一次 `Name`，拿到的身份却各不相同。`Symbol` 回答“这个词怎么拼”，`Name` 则回答“程序里具体是哪一个绑定”。这里先把接口准备好，当前的词法作用域规则照旧，后面的名称解析器再接手。

一个编译会话（`Session`）管理这些设施，还保存选项和收集诊断信息的地方。

```rust
#[derive(Default)]
/// No pass needs a global interner, source map, or diagnostic sink.
pub struct Session {
    pub sources: SourceMap,
    pub interner: Interner,
    pub supply: Supply,
    pub options: Options,
    pub diagnostics: Vec<Diagnostic>,
}

impl Session {
    pub fn emit(&mut self, diagnostic: Diagnostic) {
        self.diagnostics.push(diagnostic);
    }
    pub fn finish<T>(&self, value: T) -> Result<T, Stop> {
        if self
            .diagnostics
            .iter()
            .any(|d| d.severity == Severity::Error)
        {
            Err(Stop)
        } else {
            Ok(value)
        }
    }
}
```

[源码 · session.rs 第 21～46 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch07_fun/src/session.rs#L21-L46)

每一遍处理都接收 `&mut Session`，返回 `Result<Output, Stop>`。错误的细节已经记在会话里，`Stop` 只表示“到此为止”，不必再装一份错误信息。驱动程序统一渲染收集到的结构化错误，并在调用下一遍处理之前退出。词法分析和重复定义检查可以一次收集多个错误。类型推断一旦遇到无法继续得出有意义结果的错误，就会停下来。

`diagnostics/codes.rs` 统一登记错误码和警告码，其中一些已为后续功能预留。每条诊断包含严重程度、消息、指向源代码的标注和补充说明，交给 Ariadne 排版。标准错误输出的前两行固定给出诊断码和主要源码位置，方便测试程序集的运行器稳定地核对结果。

### 检查表层语言，再转成 Core

`lexer.rs` 用 logos 做词法分析，遇到嵌套注释和字面量中的转义，还会调用专门写的处理函数。HTML 和 PDF 用的语法定义也取自同一份保留字表和运算符表，由 `cargo xtask syntax` 生成。构建本书时会检查这些生成物有没有和词法分析器脱节。

`parser.lalrpop` 构建带源码区间的表达式和顶层绑定。像 `twice f x = ...` 这样的函数方程，多个形参会依次变成嵌套的 lambda，同一组形参里不能重名。检查器按源码顺序处理声明。遇上引用自身的绑定，就先给它一个暂定的单型，用这个类型检查 lambda，再把暂定类型与推断出的结果合一，最后像上一章那样泛化。

`elaborate.rs` 把检查过的语法树翻译成独立的 Core 表示。

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
/// Immediate source values; strings are primitive UTF-8 values, not character lists.
pub enum Lit {
    Int(i64),
    Bool(bool),
    Char(char),
    String(String),
}

#[derive(Clone, Debug)]
/// The initial lambda Core; types, data constructors, and compiled cases arrive later.
pub enum Expr {
    Var(String),
    Lit(Lit),
    Lam(String, Box<Expr>),
    App(Box<Expr>, Box<Expr>),
    Let(Bind, Box<Expr>),
    If(Box<Expr>, Box<Expr>, Box<Expr>),
    Prim(Op, Vec<Expr>, Span),
}

#[derive(Clone, Debug)]
/// A recursive binding is guaranteed to have a lambda right-hand side.
pub struct Bind {
    pub name: String,
    pub rhs: Box<Expr>,
    pub recursive: bool,
}

#[derive(Clone, Debug)]
/// Globals retain source order in this chapter; SCC ordering arrives in Chapter 9.
pub struct Program {
    pub bindings: Vec<Bind>,
    pub main: String,
}
```

[源码 · core.rs 第 5～39 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch07_fun/src/core.rs#L5-L39)

运算符会变成参数已经传齐的原语调用，一元负号会变成“用零减去这个数”。初始内建环境给 `div` 和 `mod` 提供柯里化包装，让你可以一次传一个参数。做隔离测试时，`--no-prelude` 可以移除这两个包装。用 Fun 源码写的 Prelude 要到引入数据类型时才登场。`prim_` 前缀已留给那份 Prelude 使用，在其他源码中使用这种名字会报 `E0111`。

目前 `--lint` 会逐项检查 Core 里的名字是否在作用域内、原语的参数是否传齐，以及递归绑定的右边是否仍是 lambda。第 10 章才会换成完整的 System F 类型检查。拿实际输出和预期输出做比对的测试（golden tests）每次运行都会打开 `--lint`。

### 直接执行 Core，给后面的实现做参照

`eval.rs` 直接解释 Core。递归闭包会记住自己的名字，这样就不必让 Rust 的环境和闭包互相引用成环。调用时，先在捕获的环境里把这个名字绑定到闭包自己，然后再绑定形参。

```rust
Expr::App(function, argument) => {
    let function = evaluate(function, env, fuel)?;
    let argument = evaluate(argument, env, fuel)?;
    if let Value::Closure {
        parameter,
        body,
        env,
        recursive,
    } = &function
    {
        let mut local = (**env).clone();
        if let Some(name) = recursive {
            local.push((name.clone(), function.clone()));
        }
        local.push((parameter.clone(), argument));
        evaluate(body, &local, fuel)
    } else {
        Err("internal error: non-function application".into())
    }
}
```

[源码 · eval.rs 第 72～91 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch07_fun/src/eval.rs#L72-L91)

程序运行器依次初始化每个顶层绑定，把得到的值留在环境里。整数每做完一次算术运算，就按 63 位整数规则规整结果。做除法先检查除数是否为零，再算向下取整的商和对应的余数。打印字符和字符串时，沿用写源码字面量时的转义规则。

这个参考解释器靠递归遍历语法树，跑在栈大小为 512 MiB 的线程上。做实验时可以给它设置步数预算（fuel），但它不保证只用常量大小的调用栈。第 5 章的显式抽象机会在第 18 章回来，充当 A-正规形（ANF）的执行参照。眼下把 Core 解释器写得直接一些，后面追踪细化过程时，就更容易看出一次变换究竟改变了什么。

## 跑起来看看

```text
$ cd crates/ch07_fun
$ cargo run -q -- run tests/programs/fun-top.fun
42
$ cargo run -q -- dump --pass=types tests/programs/fun-string.fun
main : String
$ cargo run -q -- dump --pass=core --no-prelude tests/programs/fun-char.fun
val main = 'λ'
$ cargo run -q -- repl
Fun: expressions, :type EXPR, :core EXPR, :quit.
fun> :type \x -> x
main : a -> a
fun> :quit
```

这里跑的文件 `fun-top.fun` 把“加一”用在 `40` 上两次，所以输出 `42`。章首那段程序则从 `sum 10` 的结果 `55` 开始，用两次“加一”得到 `57`。

挑同一个小程序，依次查看 `tokens`、`parse`、`types` 和 `core` 的输出，一路对照源码是怎么变化的。可以用 `dump --pass=...`，也可以使用通用选项 `--dump=...`。当前的 REPL 每次只求一个表达式的值。要把一次输入的定义留给下一次输入使用，要等引入 JIT 的那一章。

## 练习

1. **基础。** 打开 `--dump=tokens`，比较嵌套注释和运算符 `-->` 会被词法分析器读成什么。
2. **进阶。** 增加一个没人使用的顶层定义，让它在初始化时除以零。解释为什么 `main` 最终连一个值都给不出来。
3. **进阶。** 查看一个局部递归函数的 Core，说明函数体里那个对自身的引用究竟是在哪里绑定的。
4. **挑战。** 从一段能通过源码解析的程序出发，设想编译器的某一步翻译出错，构造一个本不该生成的非法 Core 程序。扩展一条 lint 测试，让它能抓住这个错误。

## 延伸阅读

Fun 会逐步采用《Haskell 语言报告》里的词法和作用域规则，可以对照报告阅读[^haskell2010]。要更系统地了解带类型的中间语言，可以读 Pierce 的著作[^pierce2002]。

[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。
[^damas1982]: Luis Damas、Robin Milner（1982）。[Principal Type-Schemes for Functional Programs](https://doi.org/10.1145/582153.582176)。
[^kiselyov2013]: Oleg Kiselyov（2013）。[How OCaml Type Checker Works—or What Polymorphism and Garbage Collection Have in Common](https://okmij.org/ftp/ML/generalization.html)。
[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。

---

[← 第 6 章　Hindley–Milner 类型推断](../part-1/ch06-hindley-milner.md) · [中文目录](../README.md) · [第 8 章　缩进布局和扩展语法 →](ch08-layout.md)
