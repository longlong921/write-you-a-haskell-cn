# 附录 C　中间表示

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/appendix-c-ir.html) · [第 24 章源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch24_gc/src)

Fun 最终版的编译器有三种能独立运行的中间表示（IR）。拿到同一个程序，Core 能告诉你「这里的加法用哪个类型类实例」，ANF 能告诉你「先算哪一步，把结果交给谁」，Flat 能告诉你「这次调用可以直接跳到哪个函数，还是得打开闭包才知道」。每换一种表示，前一阶段就替后一阶段做完一部分决定。下面的 Rust 定义都取自第 24 章。

| 中间表示 | 现在能直接看见什么 | 留给后面决定什么 |
| --- | --- | --- |
| System F Core | 类型、类型类字典、数据构造器、控制流 | 中间结果存在哪些临时变量里，闭包要带走什么环境 |
| ANF | 求值顺序、中间结果、汇合点、尾调用 | 函数要捕获哪些变量，调用时怎么找到入口 |
| Flat | 提升到函数表的代码、闭包里的字段、目标已知和未知的调用 | 值在机器上怎么存放，具体用什么指令 |

## 带类型的 Core

每个值标识符 `Id` 都带着自己的类型。读下面的 `Expr` 时，留意两种不同的「传参」。`TyLam` 和 `TyApp` 处理类型参数，比如把多态的 `id` 用在 `Int` 上。`Lam` 和 `App` 处理值参数。类型类字典也走值参数这条路，所以后来擦除类型抽象和类型应用时，程序照样能从字典里取出要调用的方法。

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Id {
    pub name: crate::names::Id,
    pub ty: Type,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Type {
    Double,
    Int,
    Bool,
    Char,
    String,
    Var(u32),
    Con(crate::data::TyConId, Vec<Type>),
    App(Box<Type>, Box<Type>),
    Arrow(Box<Type>, Box<Type>),
    Forall(u32, Box<Type>),
}

#[derive(Clone, Debug)]
pub enum Expr {
    Var(Id),
    Global(Id),
    TyLam(u32, Box<Expr>),
    TyApp(Box<Expr>, Type),
    Lit(Lit),
    Lam(Id, Box<Expr>),
    App(Box<Expr>, Box<Expr>),
    Let(BindGroup, Box<Expr>),
    Con(crate::data::ConId, Vec<Type>, Vec<Expr>),
    Match(Match),
    Case(Box<Expr>, Id, Type, Vec<Alt>),
    Join(crate::names::Id, Vec<Id>, Box<Expr>, Box<Expr>),
    Jump(crate::names::Id, Vec<Expr>, Type),
    PatternFail(Span, Type),
    Prim(Op, Vec<Type>, Vec<Expr>, Span),
}
```

源程序刚翻译成 Core 时，`Match` 里还可以有嵌套模式。模式匹配编译完，`Match` 就消失了。此时的 `Case` 每次只检查一层，`Join` 给一段后续代码起名，`Jump` 把参数送过去。这些节点有几条硬规矩。构造器和原语必须拿齐参数。跳转也要一次传齐参数，而且只能在汇合点的词法作用域内、从尾位置跳过去。Core Lint 会独立核对这些规矩，因此源程序通过类型检查之后，仍然能查出编译器在改写 Core 时引入的错误。

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum AltCon {
    Con(crate::data::ConId),
    Lit(Lit),
    Default,
}

#[derive(Clone, Debug)]
pub struct Alt {
    pub con: AltCon,
    pub binders: Vec<Id>,
    pub rhs: Expr,
}

#[derive(Clone, Debug)]
pub struct Bind {
    pub name: Id,
    pub rhs: Box<Expr>,
}

#[derive(Clone, Debug)]
pub struct BindGroup {
    pub recursive: bool,
    pub binds: Vec<Bind>,
}
```

绑定按组处理。递归组的成员都是函数，可以互相引用。非递归组按依赖顺序计算各个定义右边的表达式。`Case` 也给已经求出值的匹配对象绑定一个名字，需要它时直接引用即可。模式匹配失败和可能失败的原语操作都保留源码位置，这样报错时还能指回原程序。

## A-正规形（ANF）

ANF 把「取一个现成的值」和「做一步计算」分开。读 `Atom` 只会拿到局部变量、全局名字对应的值，或者字面量。只要名字都在作用域里，这一步既没有副作用，也不会失败。遇到 `Let(变量, 右侧, 后续)`，先把右侧这一步算完，给结果起好名字，再进入后续代码。`Body` 是 `Rc<Anf>`，所以执行程序的抽象机和改写程序时用的工作队列都可以共享同一段不可变的语法树。

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Atom {
    Var(Var),
    Global(Id),
    Lit(Lit),
}

#[derive(Clone, Debug)]
pub struct Lambda {
    pub params: Vec<Var>,
    pub body: Body,
}

#[derive(Clone, Debug)]
pub enum Rhs {
    Atom(Atom),
    Call(Atom, Vec<Atom>),
    Con(ConId, Vec<Atom>),
    Prim(Op, Vec<Atom>, Span),
    Lambda(Lambda),
    Field(Atom, u32),
}

#[derive(Clone, Debug)]
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

读控制流时，可以先问「接下来谁来做事」：

- `Ret` 把值交给当前的续延，也就是等着这个结果的那段计算。
- `TailCall` 用被调用函数的执行现场替换当前现场。当前函数已经没有剩下的工作。
- `Jump` 还在同一次函数调用里，只是带着参数转到它里面的某个汇合点。

比如源码写成 `(case … of …) + 3`，选完分支还得加 3。各分支可以把结果跳转到同一个「加 3」的汇合点。解释 ANF 的抽象机用循环处理这些控制转移，Fun 程序即使尾递归很深，也不需要 Rust 跟着递归调用。[第 18 章　A-正规形与汇合点](../part-3/ch18-anf.md)

## Flat IR

闭包转换完成后，每个 lambda 的代码都有了函数表中的独立条目。函数索引指向这张表。比如第 19 章的这个函数：

```haskell
make flag = let { pick x = if flag then x else False; } in pick
```

`pick` 的形参只有 `x`，执行时却还要读外层的 `flag`。提升之后，函数表里只需存一份 `pick` 的代码。每次调用 `make` 时要记住的那个 `flag` 则放进相应的闭包环境，`pick` 通过显式的环境参数取回它。

```rust
#[derive(Clone, Debug)]
pub struct Func {
    pub name: FuncId,
    pub source: Var,
    pub env: Option<Var>,
    pub params: Vec<Var>,
    pub body: Body,
    pub captures: Vec<Var>,
}

#[derive(Clone, Debug)]
pub struct GlobalInit {
    pub name: Var,
    pub function: Option<FuncId>,
    pub body: Body,
}

#[derive(Clone, Debug)]
pub struct Program {
    pub funcs: Vec<Func>,
    pub globals: Vec<GlobalInit>,
    pub metadata: core::Program,
    pub reps: crate::rep::Table,
}

#[derive(Clone, Debug)]
pub enum Rhs {
    Atom(Atom),
    CallKnown(FuncId, Option<Atom>, Vec<Atom>),
    CallUnknown(Atom, Vec<Atom>),
    AllocCon(ConId, Vec<Atom>),
    AllocClosure(FuncId, Vec<Atom>),
    EnvField(Var, u32),
    Field(Atom, u32),
    Prim(Op, Vec<Atom>, Span),
    PatchClosure(Atom, u32, Atom),
}

#[derive(Clone, Debug)]
pub enum Flat {
    Let(Var, Rhs, Body),
    Case(Atom, Vec<(AltCon, Vec<Var>, Body)>),
    Join(Var, Vec<Var>, Body, Body),
    Jump(Var, Vec<Atom>),
    Ret(Atom),
    TailCallKnown(FuncId, Option<Atom>, Vec<Atom>),
    TailCallUnknown(Atom, Vec<Atom>),
    Fail(Span),
}
```

读到 `CallKnown`，就知道目标在函数表里的哪一项，实参数量也刚好。这个编译器对全局函数和没有逃逸的局部函数可以生成这种直接调用。其余情况走 `CallUnknown` 的 eval/apply 机制，例如收到一个函数参数，或知道函数名但给的参数个数不对。参数正好就执行，给少了就返回一个记住已有参数的闭包，给多了就把余下的参数继续传给上一次调用的结果。

同组的递归闭包可能互相捕获。编译器先把整组闭包都分配好，等每个闭包都有地址，再用 `PatchClosure` 填入互相引用的字段。这个操作只在编译器内部使用，Fun 程序员不能用它修改一个已有的值。[第 19 章　闭包转换](../part-3/ch19-closures.md)

## 数值类型信息与存储方式

第 23 章从已检查的 Core 类型里保留了一点信息。即使类型已经擦除，后端还能知道哪些值一定是整数，哪些一定是双精度浮点数。

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Rep {
    Int,
    Double,
    Word,
}
```

`Rep` 回答「编译器知道这个值是什么」，`Loc` 回答「生成的代码此刻把这个值放在哪里、用什么格式保存」。例如 `Rep::Int` 可以指导 LLVM 在局部算术中直接用原生整数，但跨过函数调用边界，参数和返回值仍然统一用一个机器字传递。后端用下面几种位置区分通用机器字、原生数值，以及可重新读取的 GC 根槽。

```rust
#[derive(Clone, Copy)]
pub(super) enum Loc<'ctx> {
    Root(PointerValue<'ctx>),
    Word(IntValue<'ctx>),
    Int(IntValue<'ctx>),
    Double(inkwell::values::FloatValue<'ctx>),
}
```

使用复制式垃圾回收器时，一次分配就可能触发回收，把活对象搬到新地址。凡是可能发生过回收，后端再次使用这个对象时都要从 `Root` 槽重新读取。纯算术的中间结果则可以一直留在寄存器里，直到调用等边界要求加上标签或装箱。具体哪些值要怎么编码，见[附录 D　运行时 ABI](appendix-d-runtime.md)。

---

[← 附录 B　类型规则](appendix-b-typing.md) · [中文目录](../README.md) · [附录 D　运行时 ABI →](appendix-d-runtime.md)
