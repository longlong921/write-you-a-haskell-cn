# 第 23 章　优化

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch23-optimization.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch23_optimize)

> **新增文件**
>
> - `src/rep.rs`
> - `src/reach.rs`
> - `src/dump.rs`
>
> **改动文件**
>
> - `src/erase.rs`
> - `src/llvm/values.rs`
> - `src/llvm/primitives.rs`
> - `src/llvm/control.rs`
> - `src/llvm/mod.rs`
> - `src/anf.rs`
> - `src/anf_simplify.rs`
> - `src/anf_lint.rs`
> - `src/closure.rs`
> - `src/flat.rs`
> - `src/flat_lint.rs`
> - `src/primitives.rs`
> - `rt/src/primitives.rs`
> - `rt/src/abi.rs`
> - `src/main.rs`
> - `src/repl.rs`
> - `src/jit.rs`
> - `src/lib.rs`
> - `rt/src/lib.rs`

## 优化究竟省下了什么？

优化后的程序当然要算出正确答案。可如果同一笔账因此算了两遍，或者每做完一步算术都要分配一个盒子装起结果，留给下一条指令再拆开，代码即便看着更利索，跑起来也可能更费劲。

这一章只动几个地方，每动一处，都想办法看清它究竟省了什么。让程序打印执行踪迹，再让抽象机数一数走过多少步，可以发现计算被重复或被调换了顺序。沿着依赖图走一遍，可以找出哪些全局定义还用得上。生成机器码时留意类型信息，可以让连续几步算术直接在寄存器里完成，省掉中间来回装箱的成本。

哪怕是更大的优化器，也得回答同样的问题。编译到这里，我们对这个表达式究竟知道多少？这样改能省下什么？改完以后，程序还按原来的顺序求值吗？

## 这时的 Fun 能写什么

源语言只添了一个函数，`trace :: String -> a -> a`。它把第一个参数连同一个换行写到标准错误（stderr），再原样返回第二个参数。参数仍然严格求值，所以如果算第二个参数时就出了错，`trace` 根本来不及打印。这个函数是观察程序执行过程的诊断工具，平常程序要做输入输出，依然用 `IO`。

```fun
-- stderr: exact
f x y = x + y
main = print (f (trace "left" 20) (trace "right" 22))
```

[源码 · trace-arguments.fun 第 1～3 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch23_optimize/tests/programs/optimize/trace-arguments.fun#L1-L3)

运行时，stderr 上先出现 `left`，再出现 `right`，标准输出（stdout）上出现 `42`。假如优化器交换了两边的求值顺序，加法的答案还是一样，两个词的顺序却会立刻揭穿它。运行时每次写 trace 消息前都会先刷新 stdout，让此前的普通输出先送出去。

## 原理

### 别把算一次的东西改成算两次

在严格求值的 Fun 里，`let x = f y in x + x` 只计算一次 `f y`。要是把两处 `x` 都换回 `f y`，看起来少了个变量，实际上同一个函数跑了两次。如果里面有一条 `trace`，读者还会看见它打印两次。出错也属于程序可观察的行为。哪怕没人用除法的结果，本该发生的除零错误也不能被优化掉。

简化器可以放心替换原子表达式，因为读取一个已经准备好的值无需再做计算。一个没人使用的计算，如果保证会正常结束，又没有可观察的效果，也可以删。还有一种情形是某个 lambda 定义后马上被调用，而且只用这一次。这时可以把函数体搬到调用处，传给它的参数仍是已经算好的值。要是有两处调用，这条规则就不适用。规定好这些边界，每做一种改写都能在局部说明理由，用不着先分析整个程序的成本[^appel1992]。

把简化打开、关闭各跑一遍，让 ANF 抽象机记录各自走了多少步。两次的结果、错误和 trace 应当一致，简化后走的步数还不能更多。假如某条规则无意间重复了一段计算，这个对照就可能把它揪出来。步数少了多少可以在不同 CPU 上用同一把尺子量，实际耗时则还要另测。

### 顺着引用找出还得留下哪些全局定义

如果定义 `a` 的代码里引用了全局定义 `b`，依赖图里就有一条从 `a` 到 `b` 的边。从 `main` 出发，把下一步该查看的定义放进待办列表。一旦沿某条边首次到达一个定义，就记下来是从谁走过来的。这个做法看起来像第 9 章的依赖分析，这次想知道的则是，这趟执行到底可能用到哪些定义。

只从 `main` 开始还会漏东西。Fun 会严格初始化全局绑定，一个结果从未被读取的初始化表达式，也可能在进入 `main` 前打印一条 trace、报错，甚至永远算不完。只要局部分析还不能断定它删掉也无妨，就得把它也当作搜索起点（根）。例如，含函数调用或控制流的初始化表达式都要留下。相比之下，为一个函数分配闭包时并不会执行函数体，所以没被用到的函数可以删掉。本书的源语言成本模型不考虑分配内存失败。

留下的定义照原有的依赖顺序初始化。在 REPL 里，还得把当前可见的所有绑定都作为根，因为谁也不知道用户下一行会引用哪个名字。想知道某个绑定为什么逃过了裁剪，可以看可达性转储。根会标出来，其余定义各有一条记录，告诉你第一次是沿哪条边找到它的。

### 类型先记下来，等生成代码时再决定值放在哪儿

擦除类型以后，表达式身上不再挂着完整的 System F 类型。不过，类型已经告诉了编译器一些很便宜就能记住的事实。用下面这个枚举就能记下本章需要的部分。

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Rep {
    Int,
    Double,
    Word,
}
```

[源码 · rep.rs 第 2～7 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch23_optimize/src/rep.rs#L2-L7)

如果知道是整数或双精度浮点数，就记为 `Int` 或 `Double`。其余类型一律记为 `Word`，包括还不知道会取什么具体类型的类型变量。这里记录的是**已知的类型信息**。哪怕明确知道结果是 `Int`，函数返回时拿到的仍然是带标签的机器字。什么时候把它解码为寄存器里的普通整数，要等生成代码时决定。函数调用和返回、构造器字段、闭包捕获值、全局值以及汇合点（join point）的参数，都继续使用第 20 章统一的机器字 ABI[^leroy1990]。

在寄存器里做整数运算，要小心一位之差。LLVM 的 `i64` 有 64 位，Fun 的有符号整数只有 63 位，范围是 $-2^{62}$ 到 $2^{62}-1$。把 $2^{61}$ 加上自身，数学结果是 $2^{62}$，刚好越过 Fun 的上限，应该回绕到 $-2^{62}$。可这个正数在 `i64` 里还装得下。放着不管，下一步和零比较就会判成正数，再除以二也会得到错误的符号。

所以每做完一步整数算术，先把结果左移一位，再**算术右移**一位。把最低位算作第 0 位，这两步就是丢掉原来的第 63 位，用第 62 位当符号位复制到最高位。下一条指令看到的值便已经回到了 Fun 的范围。再看一个容易混淆的情况，如果把两个 $2^{62}$ 相加，再按 Fun 的宽度只取低 63 位，结果是零。这里的 $2^{62}$ 指运算中的位模式，它已经超出了 Fun 正整数的范围。

做 `Double` 运算时，中间结果可以直接留在浮点寄存器里。要把它传给另一个函数，调用约定要求参数是机器字，这时才必须给浮点值装箱。比如一个直接用浮点原语累加的循环，每轮末尾都有一次尾调用，累加器也就每轮要装箱一次。省下来的是同一轮内每两步算术之间的那些盒子。

我们会用分配次数检验这一区别。Fun 源码里重载的运算符仍可能通过类型类字典间接调用，所以普通源码写出的循环，生成的未必是上面这种直接使用原语的循环。若还想消掉字典调用，需要另外做专门化。

## 实现

`trace` 的行为只在共享的运行时里实现一次。解释器和原生 ABI 都用它，所以第二个参数即使是个闭包，也能原封不动地返回。

```rust
/// The trace event happens after both strict operands have been evaluated.
pub fn trace(message: &str) -> Result<(), Failure> {
    init_io();
    use std::io::Write;
    std::io::stdout().flush().map_err(|e| e.to_string())?;
    writeln!(std::io::stderr().lock(), "{message}")
        .map_err(|e| e.to_string())?;
    Ok(())
}
```

[源码 · primitives.rs 第 283～291 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch23_optimize/rt/src/primitives.rs#L283-L291)

简化器把改写规则必须遵守的约定写在实现旁边。可以对照前面的例子看其中的每条限制。

```rust
//! A rewrite may delete an unused total RHS, replace an atom by an equal atom,
//! or move a lambda applied exactly once. It may never copy a call, a failing
//! primitive, or an effect, nor move one past another. Allocation failure is
//! outside the source cost model. The oracle checks transitions and trace order.
```

[源码 · anf_simplify.rs 第 2～5 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch23_optimize/src/anf_simplify.rs#L2-L5)

`reach::analyze` 按稳定的顺序遍历依赖图。根里包括那些可能产生可观察行为的初始化表达式。第一次访问某个非根全局定义时，就把引我们到这里的那条边记下来，作为保留它的依据。

```rust
/// Follow dependencies from explicit roots and observable initializers in stable order.
pub fn analyze(program: &Program, roots: &[Id]) -> Reach {
    let bodies = program
        .globals
        .iter()
        .map(|(id, body)| (id.clone(), body))
        .collect::<BTreeMap<_, _>>();
    let mut witness = BTreeMap::new();
    let mut todo = VecDeque::new();
    for id in roots.iter().cloned().chain(
        program
            .globals
            .iter()
            .filter(|(_, b)| !discardable(b))
            .map(|(v, _)| v.clone()),
    ) {
        if bodies.contains_key(&id) && !witness.contains_key(&id) {
            witness.insert(id.clone(), None);
            todo.push_back(id);
        }
    }
    while let Some(id) = todo.pop_front() {
        for next in references(bodies[&id]) {
            if bodies.contains_key(&next) && !witness.contains_key(&next) {
                witness.insert(next.clone(), Some(id.clone()));
                todo.push_back(next);
            }
        }
    }
    Reach {
        kept: program
            .globals
            .iter()
            .filter(|(v, _)| witness.contains_key(v))
            .map(|(v, _)| v.clone())
            .collect(),
        witness,
    }
}
```

[源码 · reach.rs 第 97～135 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch23_optimize/src/reach.rs#L97-L135)

趁完整的类型还在，类型擦除阶段就先把它归到对应的类别里。

```rust
/// This is the single classification point. A type variable proves no numeric representation.
pub(crate) fn representation(ty: &core::Type) -> Rep {
    match ty {
        core::Type::Int => Rep::Int,
        core::Type::Double => Rep::Double,
        _ => Rep::Word,
    }
}
```

[源码 · erase.rs 第 38～45 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch23_optimize/src/erase.rs#L38-L45)

编译器显式维护一份待办列表，按后序遍历，先看子表达式，再记录绑定变量和表达式结果的类型信息。转到 ANF 时，新引入的临时变量也要带上这份信息。一次调用如果给的实参太多，会拆成接连的几次函数应用，其中每一步的结果都要记。闭包转换继续传递这些记录。该有的数值类型信息没了，或者两份记录相互矛盾，内部的 lint 检查就会报错。解释器用不到这张表。转储 IR 时则会显示数值类型标注，读者可以自己追踪这些信息。

等到生成 LLVM IR，另一个枚举才描述值**此刻实际采用的存放形式**。

```rust
/// Storage is local to codegen. All ABI boundaries still carry tagged words.
#[derive(Clone, Copy)]
pub(super) enum Loc<'ctx> {
    Word(IntValue<'ctx>),
    Int(IntValue<'ctx>),
    Double(inkwell::values::FloatValue<'ctx>),
}
```

[源码 · values.rs 第 13～19 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch23_optimize/src/llvm/values.rs#L13-L19)

遇到 ABI 要求机器字的场合，就用 `atom_word` 取值。整数、浮点原语需要的则是原生数值。手头是带标签的值，就为这次使用解码一次。若上一个原语已经把结果放在原生寄存器里，就直接交给下一个原语。整数运算还要调用前面说的规范化函数，然后才把结果作为原生整数留下。

```rust
fn normalize(&self, value: IntValue<'ctx>) -> Result<IntValue<'ctx>> {
    let bits =
        self.builder
            .build_left_shift(value, self.n(1), "wrap.bits")?;
    Ok(self.builder.build_right_shift(
        bits,
        self.n(1),
        true,
        "wrap.int",
    )?)
}
```

[源码 · primitives.rs 第 121～131 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch23_optimize/src/llvm/primitives.rs#L121-L131)

要单独测这项代码生成优化，我们直接用 Flat 构造调用原语的循环，再数一数优化后的 LLVM IR 里还剩几处运行时调用。整数循环一次都不需要。浮点循环在把值送过调用边界时，还要做一次不含指针的内存分配（atomic allocation），给那个 `Double` 装箱。普通 Fun 源码里用重载运算符写的程序，则另外拿去和解释器比结果。分开测，才知道寄存器优化本身带来了什么，以及走字典调用的程序实际上表现如何。

## 动手试一试

从原书仓库根目录出发，运行下面几条命令。

```text
$ cd crates/ch23_optimize
$ cargo run -q -- run tests/programs/optimize/trace-sharing.fun --engine=anf --print-steps
once
42
steps: 92
$ cargo run -q -- run tests/programs/optimize/trace-sharing.fun --engine=anf --print-steps --no-simplify
once
42
steps: 171
$ cargo run -q -- dump --pass=reach tests/programs/optimize/trace-sharing.fun
```

两次都只打印一次 `once`，答案都是 `42`。打开简化以后，ANF 机器从 171 步减到了 92 步。最后一条命令会列出哪些定义留了下来、各自是怎么找到的，末尾还会汇总保留和丢弃的数量。用 `dump --pass=anf` 或 `dump --pass=flat` 可以看绑定变量的数值类型标注。想看看从实际源码里的循环得到什么优化结果，运行下面的命令。

```text
$ cargo run -q -- dump --pass=llvm-opt tests/programs/optimize/double-loop.fun
```

看输出时，除了找本地的浮点指令，还要跟进去看看字典方法调用了谁。只数当前函数里的指令，容易漏算被调用函数干的活。`cargo test --test optimization` 会运行计算步数的对照检查和整组 trace 用例。`cargo test --test native` 还会检查生成的代码是否满足约定的分配次数。

## 练习

1. **热身。** 如果 `trace` 的第二个参数本身又含一次 `trace`，哪条消息先出现？先预测，再用各个引擎验证。
2. **进阶。** 扩展可达性转储。目前每个定义只列出上一个节点，试着显示从根一直到它的完整路径。
3. **进阶。** 给类型信息增加 `Bool` 这一类。比较运算得出的布尔值先保持无标签的形式，等到需要按机器字传递时才打标签。
4. **挑战。** 让汇合点参数直接传递原生整数。先写清楚新的存储约定，再扩展 lint 检查，最后才动代码生成器。
5. **挑战。** 给计算步数的对照检查再加一个计数器，专门统计写入后再也没被读取的存储操作。构造一种改写，让抽象机的状态转移次数保持不变，却多做了一次这样的无用写入。

## 延伸阅读

- 编译器怎样改写程序，以及怎样推理这些改写的执行成本，见[^appel1992]。
- 严格求值的函数式语言在实现中如何表示值，见[^leroy1990]。

[^appel1992]: Andrew W. Appel（1992）。[Compiling with Continuations](https://doi.org/10.1017/CBO9780511609619)。
[^leroy1990]: Xavier Leroy（1990）。[The ZINC experiment: an economical implementation of the ML language](https://caml.inria.fr/pub/papers/xleroy-zinc.pdf)。

---

[← 第 22 章　JIT 与交互式 REPL](ch22-jit.md) · [中文目录](../README.md) · [第 24 章　精确垃圾回收（选读） →](ch24-garbage-collection.md)
