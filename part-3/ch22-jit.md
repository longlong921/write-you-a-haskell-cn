# 第 22 章　JIT 与交互式 REPL

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch22-jit.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch22_jit)

> **新增文件**
>
> - `src/jit.rs`
> - `src/repl.rs`
> - `src/bin/fun.rs`
>
> **改动文件**
>
> - `src/llvm/mod.rs`
> - `src/llvm/control.rs`
> - `rt/src/heap.rs`
> - `src/session.rs`
> - `src/rename.rs`
> - `build.rs`
> - `src/core.rs`
> - `src/core_type.rs`
> - `src/elaborate.rs`
> - `src/erase.rs`
> - `src/lib.rs`
> - `src/main.rs`

## 上一行编出来的函数，下一行还能调用吗？

有了独立的可执行文件，编译器退出以后，程序照样能跑。现在换个玩法，让编译器一直开着，刚生成的机器码就在编译器自己的进程里运行。这就是即时编译（JIT）。拿它做一个交互式环境（REPL），我们就可以敲下一条定义，马上编译，再用下一行试一试。

LLVM 已经会生成机器指令。真正费心的是，这些指令要留多久？先输入 `saved = make 40`，下一行再调用 `saved 2`。这个闭包可能同时存着捕获的整数和指向先前那段机器码的地址。要让这次调用成功，两样东西都得留住。

第 21 章的编译流程和调用约定可以原样沿用。我们要在外面加一层会话，记住已经接受的定义。这样，下一行哪怕拼错一个名字，也可以改一改接着试，无须把前面定义好的东西再敲一遍。

## 这时的 Fun 能写什么

Fun 的语法没变。在 REPL 里可以输入表达式，也可以输入声明。要在同一次输入里放进好几条声明，可以把它们写在一对大括号里。如果用 `:load` 读文件，就可以和平时一样用换行、缩进组织代码。

```fun
make x = \y -> x + y

saved = make 40
```

依次输入上面的两条定义，再输入 `saved 2`，屏幕上就会出现 `42`。你输入纯表达式时，REPL 会替你在外面包一层 Prelude 的 `print`，因此结果的类型要有 `Show` 实例。如果输入的是 `IO` 表达式，REPL 就执行这个动作，然后丢弃它的返回值。想看一个函数，可以输入 `:t` 查它的类型。

| 命令 | 用途 |
| :--- | :--- |
| `:t EXPR` | 推断表达式最一般的类型，并列出所需的类型类约束。也可以写 `:type`。 |
| `:k TYPE` | 推断类型构造器或类型应用的种类（kind）。 |
| `:core EXPR`、`:anf EXPR`、`:llvm EXPR` | 查看相应阶段生成的中间程序，到这一步就停。 |
| `:load FILE` | 加载文件。等检查和初始化都成功，再换掉当前的定义。 |
| `:reset` | 清空定义，释放会话保留的机器码和垃圾收集器的根。 |
| `:engine NAME` | 切换到 `jit`、`core`、`anf` 或 `flat` 引擎，保留已有定义。 |
| `:quit` | 退出会话。也可以写 `:q`。 |

加载文件后，想运行里面的 `main`，还要亲自输入一行 `main`。和写源文件一样，同名的用户定义会报错。想换掉它，可以重置会话，也可以加载改好的文件。编译出错后，先前成功提交的定义仍然可用。

如果要用自己的定义盖住 Prelude 里的同名定义，请把它放在第一批声明里，或写进要加载的文件。后来输入的定义不能回过头改变早先定义里某个名字的含义。给运算符指定结合方向和优先级（fixity）时，也要把这条声明和运算符定义放在同一次输入里。

## 原理

可以把闭包看成一对东西，一边是要运行的代码，另一边是这段代码用得上的环境。环境里完全可以装好几个捕获值[^marlow2004]。Boehm 收集器知道怎样沿着引用找到还活着的堆对象，却不管理 LLVM 生成的机器码，也无从判断某个执行引擎何时可以销毁[^boehm1988]。

顺着刚才的三次输入看一遍。接受 `make` 以后，会话先留住它的机器码。接受 `saved = make 40` 以后，还要留住返回的函数值和它引用的东西。等到第三次输入 `saved 2`，两个条件缺一不可。假如只留下装着 `40` 的闭包，却已经释放了早先那段机器码，闭包里的函数入口就指向了一个不能再调用的地址。

这时输入 `broken = missingName` 呢？名称解析找不到 `missingName`，会话丢掉这份候选定义。你照样可以接着输入 `saved 2`。这里能恢复的是编译错误。机器码执行期间遇到运行时错误，仍会结束整个会话。

我们让会话持有全部执行引擎，每个引擎持有一次输入产生的模块。另有一张表，把全局值的机器字放进已向收集器登记的根槽位。老模块可以一直保持原样，新模块只要声明外部全局变量，就能通过这些槽位取得先前的值。重置时先销毁引擎，再注销槽位。包在外面的 LLVM `Context` 比两者都活得久。

为了让输错一行的代价足够小，会话把新输入单独当成候选，检查和初始化都成功了才接纳它。只查询类型或中间表示时，流程会在初始化之前停下。因此你可以放心问 `error "x"` 的类型，要是直接运行这个表达式，才会触发错误。读代码和跑代码走到不同的终点，探索程序时才不会意外执行它[^pierce2002]。

## 实现

### 让入口执行完能返回

独立程序从 `main` 启动，初始化全局值，执行 IO 动作，然后退出。REPL 还要接收下一行，因此 JIT 后端生成的是可以从 C 调用的 `void fun_entry(void)`。这个入口初始化新加入的全局值，把单位值 `()` 传给本次查询的动作，做完就返回。

快速入口继续用 `tailcc` 调用约定，通过闭包发起的调用继续走 C 侧的 eval/apply。跨越这条 ABI 边界的调用要明确标为 `notail`。我们在一个参数很多的调用测试里就遇到了这个问题。LLVM 自行把从 `tailcc` 函数到 C apply 的调用变成了尾调用。在 AArch64 上，被调用方按约定清理栈时，返回用的栈帧被破坏了。LLVM 验证器接受了这段 IR，在两个后端实际运行同一程序才暴露问题。这个修复也适用于第 21 章。

`Backend::for_jit` 选择这个能返回的入口，并让全局值存放在模块外部。第 21 章里按照待办列表生成代码的流程、验证器和优化过程管理器都能复用。交互式编译固定选 O1，生成独立程序的 `build` 仍然可以在 O0～O3 之间选择。

```rust
/// A returning entry and externally rooted globals replace the native process startup.
pub fn for_jit(
    context: &'ctx Context,
    program: &'program flat::Program,
    inherited: BTreeSet<Id>,
) -> Result<Self> {
    Self::create(context, program, 1, true, inherited)
}
```

[源码 · mod.rs 第 86～93 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch22_jit/src/llvm/mod.rs#L86-L93)

### 开始执行之前，把外部符号的地址找齐

生成独立的可执行文件时，目标文件要调用哪些运行时函数，由系统链接器找到它们的地址。换成 MCJIT，在编译器进程里运行代码，同样得把这些地址交给它。`runtime_symbols` 直接取已链接运行时导出的 C 函数地址，这些函数也就留在了编译器的二进制文件里。`map_symbols` 逐一检查外部函数，再用 `add_global_mapping` 把符号映射到相应地址。以前的快速入口和慢速入口也在这一步映射回已有引擎里的地址。LLVM 内建函数（intrinsics）仍由 LLVM 自己处理。

```rust
/// Resolve declarations before MCJIT finalizes the module. Intrinsics belong to LLVM.
fn map_symbols(
    module: &inkwell::module::Module<'_>,
    engine: &ExecutionEngine<'_>,
    imports: &[(String, usize)],
) -> Result<(), String> {
    let symbols = runtime_symbols();
    for function in module
        .get_functions()
        .filter(|f| f.count_basic_blocks() == 0)
    {
        let name =
            function.get_name().to_str().map_err(|e| e.to_string())?;
        if name.starts_with("llvm.") {
            continue;
        }
        let address = symbols
            .iter()
            .find(|(symbol, _)| *symbol == name)
            .map(|(_, address)| *address)
            .or_else(|| {
                imports
                    .iter()
                    .find(|(symbol, _)| symbol == name)
                    .map(|(_, address)| *address)
            })
            .ok_or_else(|| {
                format!("unmapped JIT runtime symbol: {name}")
            })?;
        engine.add_global_mapping(&function, address);
    }
    Ok(())
}
```

[源码 · jit.rs 第 229～261 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch22_jit/src/jit.rs#L229-L261)

查到不认识的导入项，当场报告错误，机器码还没开始跑。这套映射无须靠宿主程序按名字导出符号。字符串操作、格式化、函数应用、错误处理和 IO，最后都调用第 22 章以 Rust 库（rlib）形式链接进来的运行时。它的 ABI 与独立程序用的运行时静态库一致。

### 谁保管机器码，谁保管 GC 根

看会话的数据结构，就能找到两者各自的主人。

```rust
/// Keep every engine alive until no REPL binding can reference its closures.
/// Fields drop in declaration order: release code before unregistering the slots.
pub struct Jit<'ctx> {
    engines: Vec<ExecutionEngine<'ctx>>,
    roots: BTreeMap<GlobalKey, GlobalRoot>,
    queries: Vec<GlobalRoot>,
    code: BTreeMap<GlobalKey, (usize, usize, usize)>,
    types: BTreeMap<GlobalKey, Vec<String>>,
    context: &'ctx Context,
}
```

[源码 · jit.rs 第 13～22 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch22_jit/src/jit.rs#L13-L22)

一个全局槽位由运行时的 `GlobalRoot` 持有。槽位放在 `Box` 里，地址固定，并登记给 Boehm。`UnsafeCell` 允许生成的机器码写入槽位，同时符合 Rust 的引用规则。LLVM 全局变量映射到这个地址之后，生成的取值指令从这里读，垃圾收集器也从这里找根，两边看到的就是同一个机器字。只把它塞进普通 Rust 容器还差一步，因为 Boehm 不会沿着 Rust 的普通堆对象一路追踪里面的引用。

跨多次输入保留这些槽位，得给每个槽位一个能认得出来的键。用户定义可能和 Prelude 定义同名，编译器会按源码中各组定义出现的次序区分两者。名称解析阶段分配的唯一编号能在**本次编译**中标识 LLVM 声明，加入新声明再编一次，编号却可能会变。所以持久化用的是名字和该名字出现的次序。

先前接受的槽位直接复用，初始化也只做头一次。导入旧值前，我们先核对类型指纹。计算指纹时把绑定的类型变量统一编号（α 规范化），因此像 `a -> a` 和 `b -> b` 这样的改名不会造成误判。再用保存的形参数量核对快速入口实际接受几个参数。这些元数据会穿过类型擦除阶段，一直传给 Flat。遍历类型生成指纹时显式维护待办列表。至于每次查询自己的槽位，总是分配新的，随引擎一起保留，这样已经生成的代码里根据重定位写好的地址就不会悬空。

`Jit::run` 先完成所有映射，然后才取出带有明确函数类型的入口。这里的调用需要 `unsafe`，能安全执行靠的是几个已经核对过的条件。后端生成的 C 签名必须与我们向 Inkwell 请求的签名完全相同。运行时导入的符号必须全部解析完。调用完成之前，引擎和根槽位必须始终有效。拿到入口立刻使用，之后也不再往这个引擎里加代码。Inkwell 文档说明，继续添加代码有可能让之前取到的函数指针失效。

```rust
/// Enter a finalized module without exposing a callable to the rest of the compiler.
fn call_entry(engine: &ExecutionEngine<'_>) -> Result<(), String> {
    // SAFETY: the verified backend emits exactly void fun_entry(void), C ABI.
    // All imports are explicitly mapped; code, constants, and root slots remain live.
    // This engine is never modified after lookup, and no pointer escapes this call.
    unsafe {
        let entry = engine
            .get_function::<unsafe extern "C" fn()>("fun_entry")
            .map_err(|e| e.to_string())?;
        entry.call();
    }
    Ok(())
}
```

[源码 · jit.rs 第 263～275 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch22_jit/src/jit.rs#L263-L275)

### 新定义成功以后再交给会话保管

`Repl::line` 在循环里分派命令。碰到声明，先把它加入一份候选源码列表，再用私有入口 `$repl` 检查、初始化，最后才用这份列表替换会话已经接受的列表。Fun 源码写不出这个私有名字，加载文件时也就不会跟用户的 `main` 撞名。`:load` 同理，先把候选会话整个建好，再换进去。

执行表达式时，用的仍是 Fun 自己的语言操作。

```rust
/// Both display and IO sequencing are ordinary, type-checked Prelude operations.
fn action_query(expression: &str, ty: &crate::types::Type) -> String {
    if matches!(ty, crate::types::Type::Con(id, _) if id.name == "IO") {
        format!("({expression}) >> pure ()")
    } else {
        format!("print ({expression})")
    }
}
```

[源码 · repl.rs 第 237～244 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch22_jit/src/repl.rs#L237-L244)

包好后的查询表达式还要再走一遍类型推断和精化，所以 `print` 需要的类型类实例照常求解，`IO` 的抽象边界也照常生效。查类型、种类或 IR 的命令都在执行前停下。本章新加的遍历和命令控制流程都用循环实现，不用扩大 Rust 栈，也没有引入自动增长栈的库。前面章节留下的递归式前端过程仍有原来的深度限制。

每输入一行，我们都会重新检查已经接受的源码，再把候选程序编译成模块。已有的全局值，以及已有函数的快速、慢速入口，都直接导入，这样既不会重做初始化，也不会把老代码悄悄换成新地址。重新生成但没人用到的局部函数体会被优化掉，不过会话越长，每次要编译的东西和积存的机器码总量也会随之增长。现在这么做，谁持有什么一目了然。要进一步做到增量检查和回收旧代码，就得跟踪更细的依赖关系和生命周期。

> **运行时出错会怎样？** 在 JIT REPL 里，运行时错误和 `exitWith` 都沿用独立程序的行为，直接结束进程。我们不会跨过生成的栈帧展开栈或跳转。编译错误则可以恢复。如果创建执行引擎失败，REPL 会改用 Flat。如果是验证器发现 IR 有错，就把错误报出来。显式指定 `run --engine=jit` 时，引擎启动失败也会直接报错。

## 动手试一试

先按第 21 章装好 LLVM 22 和 Boehm，再从原书仓库根目录运行。

```text
$ python3 scripts/build_runtime.py 22
$ cargo build -p ch22_jit
$ target/debug/fun repl --lint
Fun: JIT ready on first input. Type :help for commands.
fun> make x = \y -> x + y
fun> saved = make 40
fun> saved 2
42
fun> :t saved
saved : Int -> Int
fun> :k Maybe
Maybe : * -> *
fun> :reset
Reset.
fun> :quit
$ target/debug/fun run crates/ch22_jit/tests/programs/native/acceptance.fun --engine=jit --lint
5.0
3.0
6.0
[3.0,10.0]
True
50000005000000
$ target/debug/fun dump --pass=llvm crates/ch22_jit/tests/programs/jit/returned-function.fun > build/jit-example.ll
```

`fun` 和 `fun-ch22` 共用一套命令行逻辑。`run` 默认用 Flat，REPL 默认用 JIT。`--fuel` 和仅供测试的 `--print-value` 都是解释器的功能。把输入重定向到 REPL 时，它会省掉提示符和启动横幅，方便逐字节比对会话输出。

同一个程序交给 JIT、解释器和独立的可执行文件，答案应该一致。还要试两种压力场景：对目标已知的函数做很深的递归，以及限制堆大小后反复分配。已知目标的尾调用仍享有第 21 章的尾调用保证。绕经 C apply 的调用仍有那一章讨论过的限制。

可以自己先定义一个闭包，再零零散散加几条新定义，最后回过头来调用老闭包。生命周期测试比这再多走一步，特意在执行两个模块之间触发垃圾收集。最后那次调用要能成功，除了名字还在作用域里，闭包捕获的数据和它指向的机器码也得真的还在。

## 练习

1. **热身。** 顺着两次输入追踪 `saved = make 40`，指出谁在保管闭包环境，谁保证它的代码地址仍然有效。
2. **进阶。** 去掉一个运行时符号的映射，看看报什么错。为什么必须在调用生成的入口之前发现这个错误？
3. **进阶。** 加一个回归测试，让用户定义遮住 Prelude 里的同名定义。用它说明，为什么只看名字拼写找不准应当复用的槽位。
4. **挑战。** 设计一套独立工作进程的通信协议。运行时出错后要能恢复，还要能接着用已接受的定义。请说明恢复时要重放哪些内容。
5. **挑战。** 设计一套回收旧模块的办法。记得检查藏在构造器里的闭包、部分应用、全局值和捕获的环境。它们都可能让原本打算回收的模块继续被引用。

## 延伸阅读

- 符号怎样映射到地址？调用 `get_function` 前必须满足哪些安全条件？可查阅 [Inkwell 的 ExecutionEngine 文档](https://thedan64.github.io/inkwell/inkwell/execution_engine/struct.ExecutionEngine.html)。
- 如何通过 Rust 绑定构建 LLVM IR，可读[《Compiler Crates》的 Inkwell 一章](https://sdiehl.github.io/compiler-crates/inkwell.html)。
- 想了解闭包、部分应用和慢速入口，见[^marlow2004]。
- 想了解保守式垃圾收集怎样找根、怎样与运行时配合，见[^boehm1988]。
- 想进一步区分类型检查与求值，见[^pierce2002]。

[^marlow2004]: Simon Marlow、Simon Peyton Jones（2004）。[Making a fast curry: push/enter vs. eval/apply for higher-order languages](https://doi.org/10.1145/1016850.1016856)。
[^boehm1988]: Hans-Juergen Boehm、Mark Weiser（1988）。[Garbage collection in an uncooperative environment](https://doi.org/10.1002/spe.4380180902)。
[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。

---

[← 第 21 章　用 LLVM 生成机器码](ch21-llvm.md) · [中文目录](../README.md) · [第 23 章　优化 →](ch23-optimization.md)
