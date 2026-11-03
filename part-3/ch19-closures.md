# 第 19 章　闭包转换

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch19-closures.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch19_closure)

> **新增文件**
>
> - `src/closure.rs`
> - `src/flat.rs`
> - `src/flat_machine.rs`
> - `src/flat_lint.rs`
>
> **改动文件**
>
> - `src/lib.rs`
> - `src/main.rs`

## 外层函数都返回了，内层函数去哪里找变量？

在 Fun 里，一个函数可以创建另一个函数，然后把它返回。外层调用早就结束了，内层函数却还能接着用。要是它读到了当初那次调用里的某个变量，后端该去哪里找那个值？

之前的解释器随函数保存一份环境，就解决了这个问题。现在我们要把这个做法明确写进编译结果。每个 lambda 都会变成整个程序共用的一张函数表里的一个条目。至于它需要从外层带走哪些值，就把这些值收进它的**闭包**（closure）里。

有了闭包，还要知道怎样调用它。编译时就知道目标，可以直接去执行那段函数代码。如果函数是作为参数传进来的，或者是上一次调用返回的，就得看看手里的闭包到底指向谁。此外，源程序还可以一次只给一部分参数。本章先把这几种情况都放进一个叫 **Flat** 的中间表示和解释器里跑通，下一章再考虑怎样在机器内存里摆放这些对象。

## 这一章的语言长什么样

我们来写一个局部函数，让它从外层带走一个布尔值。

```fun
make flag = let { pick x = if flag then x else False; } in pick

main = print (make True False)
```

`make` 和 `pick` 的参数中间隔着一个局部绑定，两层 lambda 没有挨在一起。`make True` 返回时，它自己的调用结束了，但它交出去的 `pick` 以后还得读 `flag`，也就是这次传进来的 `True`。这个值不能只放在那次已经结束的调用所用的存储空间里，`pick` 得有自己的捕获环境。本章默认用 `--engine=flat` 执行，仍然可以选择 Core 或 ANF 引擎来对照。

闭包环境的存储由编译器创建。在 Fun 里写程序的人不会因此多出可修改的字段或读写引用的操作。本章引入的唯一一种修改，是先为一组递归闭包预留位置，再把它们捕获的值填进去。等到用户代码能接触这些闭包时，初始化早已完成。

## 理论

怎么知道要带走哪些东西？做一次**自由变量分析**。看看 lambda 里用到的名字，排除自己的参数和函数体内部绑定的名字，余下的就得往外找。全局名字可以直接访问，无须占用闭包的捕获槽位。对于其余每个自由变量，转换器选定一个槽位，创建闭包时把值存进去。在提出来单独存放的函数里，原先“按词法作用域找这个名字”的地方改成“从环境里的这个槽位读值”。

用伪代码来画，开头的程序转换后差不多长这样。

```text
pick_code(env, x) =
  let flag = env[0]
  if flag then x else False

make(flag) = closure(pick_code, [flag])
```

调用 `make True`，会把 `True` 放进一个新环境的第 0 号槽位。再把 `False` 传给返回的函数，就是拿着这个环境和实参 `False` 进入 `pick_code`。`env[0]` 读出 `True`，于是选择 `then x`，返回 `x` 的值 `False`。如果再调用一次 `make False`，共用的还是同一段 `pick_code`，但新闭包有自己的槽位，里面装着 `False`。这就解释了为什么光存一个代码地址还不够。

这里有两种容易混淆的做法。**闭包转换**把捕获的值放在一份环境里。**lambda 提升**（lambda lifting）则可以在调用点允许的情况下，把这些值加进函数的形参表，让调用方直接传来[^appel1992]。本章把局部函数的代码都提到函数表里，但传递外部值统一走环境，连不会逃逸的局部函数也一样。是否要进一步做 lambda 提升，留待以后。如果编译器知道要调用哪个局部函数，它仍然可以绕过通用调用逻辑，直接带着环境跳进相应的函数代码。

已有的 `capture-return.fun` 走的恰好是另一条路。里面的 `make x = \y -> x+y` 有两层挨在一起的 lambda，所以 ANF 转换时已经把它们合并了。调用 `make 20` 时，类型类字典和整数 `20` 都给了，还差 `y`。要把已经给出的东西留到下次，这回保存的是**实参**，和 `pick` 从外层捕获自由变量 `flag` 的情形有别。这种“先收下一部分参数”的值就叫 PAP。

从写 Fun 的人看来，调用一个函数可以每次给一点参数。提到函数表里的机器函数却有固定的元数，一次要收固定数量的参数。这两种用法怎么接在一起？

假设函数的语法元数是 `n`。把收到的参数数一数即可。

- 不到 `n` 个，就把已经有的参数存进一个**部分应用**（partial application，简称 PAP），将它返回。
- 正好 `n` 个，可以进入函数体执行。
- 超过 `n` 个，就先用前 `n` 个调用它，等它返回了一个函数，再把余下的实参交给返回的函数。

这就是一个小型的 **eval/apply 调用协议**。无论参数是少给、给齐还是给多，都有同一套办法接住[^marlow2004]。

相互递归还带来一个先有鸡还是先有蛋的问题。假设 `even` 的环境里要存 `odd`，而 `odd` 的环境里又要存 `even`，先造谁？可以先给两边都分配好闭包，捕获槽位暂时放占位值。等两个闭包都有地址了，再把正确的值填进去。在这项准备工作完成之前，源码里没有哪段代码能调用它们。Fun 要求递归绑定的都是函数，所以这里可以先创建闭包，避免在某个递归值还没造好时就试图求它的值。

## 实现

### Flat 的语法

Flat 延续了 ANF 的显式控制流，把函数挪进单独的函数表。可执行主体里再也没有 lambda 表达式。原本需要创建函数值的地方，改为分配一个闭包，写上函数表里要执行的是谁，再存入所需的捕获值。

$$
\begin{aligned}
e & = \text{let}\;x = r\;\text{in}\;e & \quad & \text{严格绑定} \\
  & \mid \text{case}\;a\;\text{of}\;\{\overline{q\;\overline{x}\to e}\} & \quad & \text{按原子的值选择分支} \\
  & \mid \text{join}\;j(\overline{x}) = e_1\;\text{in}\;e_2 & \quad & \text{定义局部续延} \\
  & \mid \text{jump}\;j(\overline{a}) & \quad & \text{跳到汇合点} \\
  & \mid \text{return}\;a & \quad & \text{返回一个值} \\
  & \mid \text{tail}\;f[\rho](\overline{a}) & \quad & \text{目标已知的尾调用} \\
  & \mid \text{tailapply}\;a(\overline{a}) & \quad & \text{目标未知的尾调用} \\
  & \mid \text{fail}_{s} & \quad & \text{在源码位置 }s\text{ 报模式匹配失败}
\end{aligned}
$$

沿用 ANF 的记号，$a$ 是原子，$j$ 是汇合点名字，$q$ 是分支的匹配标记，上方一条横线表示一串东西。$f$ 指函数表里的一个函数。$\rho$ 是它的环境操作数，有环境时放一个原子，没有时可以省去。对于目标已知的调用，指令直接写明要执行哪段代码。目标未知时，则要看看拿到的是哪个闭包或 PAP，才能决定接下来怎么调用。

$$
\begin{aligned}
a & = x \mid g \mid \ell & \quad & \text{局部变量、全局引用或字面量} \\
\rho & = a \mid \text{none} & \quad & \text{可选的环境操作数} \\
r & = a & \quad & \text{原子} \\
  & \mid \text{call}\;f[\rho](\overline{a}) & \quad & \text{目标已知的调用} \\
  & \mid \text{apply}\;a(\overline{a}) & \quad & \text{目标未知的调用} \\
  & \mid \text{con}\;K(\overline{a}) & \quad & \text{分配构造器对象} \\
  & \mid \text{closure}\;f[\overline{a}] & \quad & \text{分配闭包} \\
  & \mid \text{env}\;x[i] & \quad & \text{读一个捕获槽位} \\
  & \mid \text{field}\;a[i] & \quad & \text{读一个构造器字段} \\
  & \mid p(\overline{a}) & \quad & \text{原语操作} \\
  & \mid \text{patch}\;a[i] = a' & \quad & \text{填入闭包的一个捕获值}
\end{aligned}
$$

这里 $K$ 是构造器，$p$ 是原语，$i$ 是槽位下标。式子省略了原语的源码位置。`patch` 是编译器内部把几个递归闭包连起来用的操作，Fun 源程序不能借它修改状态。ANF 里的一个 `LetRec` 到了 Flat，就拆成先分配、再逐个填槽位的一串操作。

### 提到函数表里的函数怎么调用？

Flat IR 的可执行主体里不再含有 lambda 表达式。每个从中提出来的函数都会记下自己的参数、可选的环境参数、函数体，以及按固定顺序排列的捕获变量。

```rust
#[derive(Clone, Debug)]
/// Only lifted local lambdas receive an environment parameter.
pub struct Func {
    pub name: FuncId,
    pub source: Var,
    pub env: Option<Var>,
    pub params: Vec<Var>,
    pub body: Body,
    pub captures: Vec<Var>,
}
```

[源码 · flat.rs 第 17～26 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch19_closure/src/flat.rs#L17-L26)

`closure::convert` 收集 lambda 定义，找出自由变量，并分析各个函数值都被用在了哪里。它按照变量的唯一标识排列捕获槽位，所以同一段程序每次转换，看到的转储和槽位下标都是稳定的。顶层函数没有环境参数。局部函数提到函数表以后，即使没捕获任何值，也仍然带着环境参数。需要读的捕获值，都会在局部函数开头用 `EnvField` 绑定逐个取出来。

要把一次调用判为**目标已知**，得同时满足两个条件。实参数量恰好等于函数的语法元数，而且被调的一方是全局函数，或者是一个不会逃逸的局部 lambda。光靠类型检查得知“这是个函数”还不够。实际判断只需要下面这段代码。

```rust
fn known(&self, a: &Atom, n: usize) -> Option<(FuncId, Option<Atom>)> {
    let (Atom::Var(v) | Atom::Global(v)) = a else {
        return None;
    };
    let info = self.infos.get(v)?;
    (n == info.lambda.params.len() && (info.global || !info.escapes))
        .then(|| (info.id, (!info.global).then(|| a.clone())))
}
```

[源码 · closure.rs 第 141～148 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch19_closure/src/closure.rs#L141-L148)

逃逸分析会区别对待函数的几种用法。在调用位置把参数给齐，是一种用法。把这个函数作为值返回、存起来，或者当成实参传给别人，则是另一种。直接调用一个已知的局部函数时，仍要把它的闭包作为环境操作数一起传过去。其他普通调用走 `CallUnknown`。在尾位置则选用相应的尾调用形式。因此，即使源码上明明写着某个熟悉的函数名，只要这次只是部分应用，照样得走通用的 apply 流程。

看看绑定右侧可以有哪些操作，就能找到分配对象和读取字段的地方。

```rust
#[derive(Clone, Debug)]
/// PatchClosure is a compiler-only operation used after all letrec allocations.
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
```

[源码 · flat.rs 第 43～55 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch19_closure/src/flat.rs#L43-L55)

转换 `LetRec` 时，先发出整组的所有 `AllocClosure`，才发出第一条 `PatchClosure`。IR 先用整数字面量占住每个捕获槽位，后续的回填操作再把它换成真正要捕获的值。回填操作返回单位值，绑到一个后续不会使用的临时名字上。等所有槽位都填好，才开始运行从原主体转换来的代码，所以不会有哪条源码表达式叫到一个只初始化了一半的闭包。

Flat 的结构检查器逐个检查提出来的函数。如果函数体里还藏着一个没显式取出的自由变量，或者代码引用了不存在的函数，检查就通不过。已知调用的参数数目不对、环境多传或漏传、读取环境时下标越界，以及向汇合点作了不合法的跳转，也都会报错。两个函数倒是可以各自用同一个变量标识来接住捕获值。每条环境读取指令都是在自己函数的作用域里重新绑定那个名字，彼此不会干扰。

Flat 机自己管理一块用下标寻址的存储区（arena），里面放着各个闭包的记录。解释器里代表闭包的值其实就是一个下标。环境槽位可以放任意普通的解释器值，其中当然也可以放另一个闭包的下标。于是两个闭包可以互相指向，却不会在 Rust 的引用计数里绕成一个环。PAP 保存一个函数值以及先前给过的实参。这块存储区随一次程序执行而存在，这里用它模拟对象之间的关系，何时回收对象留到后面再讨论。

执行器在一个循环里切换四种状态，分别处理代码、目标已知的调用、通用 apply，以及从调用返回。已知调用会核对函数条目的元数及环境是否匹配。apply 遇到 PAP，会先把里面存过的实参和新实参摊到一起，再看现在是还不够、刚刚好，还是给多了。多出来的实参会放进显式续延里，等拿到结果再接着用。汇合点仍是函数内部、受词法作用域约束的控制转移。原语操作数仍经过一个数据适配层，但 Flat 函数体全由 Flat 机自己执行。正因为 Core、ANF、Flat 的控制流各自独立实现，让三个引擎算同一题、互相核对才有意义。

## 跑起来看看

```text
$ cd crates/ch19_closure
$ cargo run -q -- run tests/programs/closure/capture-return.fun --lint
[21,22,23]
$ cargo run -q -- dump --pass=types tests/programs/closure/capture-return.fun
main : IO ()
```

给这个文件做一次 `--pass=flat` 转储，找出 `make` 的部分应用。`make` 在全局函数表里的条目没有环境参数，apply 会把收到的字典和整数参数存进 PAP。还有几个例子可以并排看。

- `escape-recursive.fun` 返回一个局部递归闭包，它捕获了外层的一个值。
- `known-local.fun` 直接调用目标已知的局部函数。
- `pap-chain.fun` 把实参分几次传进来。
- `over-application.fun` 一口气给出额外的实参，第一段调用返回的结果本身还是个函数，正好可以接着收。

`tests/anf.rs` 里的 Flat 小快照只挑入口可达的代码，所以不会掺进一大堆和例子无关的 Prelude 函数。

你也可以亲手试三种写法。把实参分几次给，返回一个局部函数，或者写两个互相调用的局部函数，让它们都用上外层的同一个值。站在写 Fun 的人这边，这些写法都很自然，编译后却各自用到了闭包实现的不同环节。Core、ANF 和 Flat 应当给出一致的结果。此外，Flat 机会检查很长的尾递归求和，确认同时悬而未决的续延数量始终有界。

## 练习

1. **基础。** 看开头 `make flag` 示例的 Flat 转储，找出 `flag` 的捕获槽位，再和 `capture-return.fun` 里的 PAP 比一比。两者各自在保存什么？
2. **进阶。** 起初只在把参数给齐的调用里使用某个局部函数，再改成把这个函数作为值返回。看看调用的分类怎么变，并解释原因。
3. **进阶。** 手动跟一遍参数传递。先让一个 PAP 再收到一个参数，接着追踪一次实参过多的调用，其中第一段调用返回一个闭包。余下的参数会交给谁？
4. **挑战。** 写出两个相互递归的局部函数，并让它们再捕获一个外层整数。画出回填完成后各个环境之间的指向关系。
5. **挑战。** 给不会逃逸的函数试做一个 lambda 提升的原型。用差分测试确认各引擎给出的结果仍然一致，再比较提升前后分配对象的次数。

## 延伸阅读

- eval/apply、部分应用与函数调用约定[^marlow2004]。
- 完整编译器中的闭包转换[^appel1992]。
- 严格求值的函数式语言实现和值的表示[^leroy1990]。
- 局部汇合点与可以当作普通值传递的一等续延[^maurer2017]。

[^appel1992]: Andrew W. Appel（1992）。[Compiling with Continuations](https://doi.org/10.1017/CBO9780511609619)。
[^marlow2004]: Simon Marlow、Simon Peyton Jones（2004）。[Making a fast curry: push/enter vs. eval/apply for higher-order languages](https://doi.org/10.1145/1016850.1016856)。
[^leroy1990]: Xavier Leroy（1990）。[The ZINC experiment: an economical implementation of the ML language](https://caml.inria.fr/pub/papers/xleroy-zinc.pdf)。
[^maurer2017]: Luke Maurer、Paul Downen、Zena M. Ariola、Simon Peyton Jones（2017）。[Compiling without continuations](https://doi.org/10.1145/3062341.3062380)。

---

[← 第 18 章　A-正规形与汇合点](ch18-anf.md) · [中文目录](../README.md) · [第 20 章　数据表示与运行时 →](ch20-runtime.md)
