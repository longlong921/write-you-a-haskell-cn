# 第 21 章　用 LLVM 生成机器码

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch21-llvm.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch21_llvm)

> **新增文件**
>
> - `src/llvm/mod.rs`
> - `src/llvm/control.rs`
> - `src/llvm/values.rs`
> - `src/llvm/primitives.rs`
> - `src/native.rs`
>
> **改动文件**
>
> - `src/flat.rs`
> - `build.rs`
> - `src/flat_lint.rs`
> - `src/lib.rs`
> - `src/main.rs`

## 让 Fun 程序自己跑起来

到现在，Fun 程序已经能算出结果，但每一步还得由解释器代劳。本章要把程序编译成一个可执行文件，在终端里直接启动。最早写那个小计算器时输入的是算术表达式，走到这里，交给系统运行的已经是我们自己编出来的程序了。

之前做的工作，现在都派上了用场。模式匹配已经拆成分支，闭包已经带上明确的环境，Flat IR（中间表示）也指出了哪些地方要调用函数、哪些地方能在尾部把控制权交出去。有了这些信息，就可以把指令选择、寄存器分配和针对机器的优化交给 LLVM。

每个值在内存里怎么放，一个函数怎么把参数和结果交给另一个函数，这两件事仍要由我们定好。只要跟运行时的约定偏差一个槽位，LLVM 也能顺利把程序编出来，只不过编出来的程序会高效地取错值。下面就从 Flat 和上一章定好的内存布局出发，把这道接口接好。

## 这时的 Fun 能写什么

Fun 的语法和语义照旧，变化在命令行。`fun-ch21 build FILE -o OUT` 会生成一个可执行文件。`run` 仍然选择解释器运行。可以用 `-O0` 到 `-O3` 选择优化级别，默认是 `-O2`。`--dump=llvm` 查看优化前的 LLVM 模块，`--dump=llvm-opt` 查看优化后的版本。

下面是一段验收程序，把记录、类型类字典、高阶函数、IO 和一个严格求值的累加循环都用上了。

```fun
-- engines: native
data Shape = Circle Double | Rect Double Double
  deriving (Show)
data Point = Point { x :: Double, y :: Double }
  deriving (Eq, Show)
class Area a where
  area :: a -> Double
instance Area Shape where
  area (Circle r) = 3.0 * r * r
  area (Rect w h) = w * h
norm :: Point -> Double
norm p = sqrt (p.x * p.x + p.y * p.y)
sumTo :: Int -> Int
sumTo n = go 0 n
  where
    go acc 0 = acc
    go acc k = go (acc + k) (k - 1)
main :: IO ()
main = do
  let p = Point { x = 3.0, y = 4.0 }
  print (norm p)
  mapM_ (print . area) [Circle 1.0, Rect 2.0 3.0]
  print (map (.x) [p, p { x = 10.0 }])
  print (p == p { y = 4.0 })
  print (sumTo 10000000)
```

[源码 · acceptance.fun 第 1～25 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch21_llvm/tests/programs/native/acceptance.fun#L1-L25)

最后一次调用要迭代一千万次。除了算对总和，还要看栈。在前几章显式管理控制流的解释器里，尾调用不必一层层留下调用帧。编成机器码之后，这个循环也得用有界的原生栈空间跑完。

## 原理

LLVM 使用**静态单赋值形式**（static single assignment，SSA）。每条指令的结果只有一处定义，读到一个名字时，就知道它始终指向同一个已经算出的值。Flat 里的绑定都有唯一标识，跟这种表示刚好对得上。`Let` 把一个名字和一个 LLVM 值关联起来。`Join` 则变成一个基本块，几条执行路径都可以跳到这里。

这些路径怎样给同一个块传不同的值？LLVM 用 **phi 节点**解决。汇合点的每个参数各有一个 phi。遇到 `Jump`，就把这条路径要传的值记在相应的 phi 上，再跳到目标块。先前把共享的后续代码写成汇合点，到这里恰好可以照着翻译[^maurer2017]。

假设两条路在某个汇合点碰头，左路带来的值是 `10`，右路带来的值是 `20`。phi 只看刚才是从哪个块过来的，就知道该用哪个值，不必再判断一次当初分叉的条件。

可以把这个合流过程写成一小段 LLVM IR。假设前面的条件已经算成一个 `i1`，也就是一位的布尔值。为看清控制流，例子里直接写普通整数，暂时省去 Fun 的整数标记。

```llvm
left:
  br label %merge
right:
  br label %merge
merge:
  %value = phi i64 [ 10, %left ], [ 20, %right ]
  ret i64 %value
```

前面的条件分支选择 `left` 或 `right`。两条路最后都跳到 `merge`。从 `left` 来，`%value` 就是 `10`。从 `right` 来，它就是 `20`。实际翻译时传入的是带标记的 Fun 机器字，对应关系一样。汇合点的参数变成 `%value`，每次跳转为它补上一对“传来的值、出发的块”。

这里很容易记错块名。一条 Flat 操作可能会生成好几个 LLVM 基本块。比如发出除法时还得插入“除数是不是零”的检查。从某个块开始生成表达式，等到检查完毕，真正跳向汇合点时人已经在另一个块了。**先生成各个跳转实参，再读取 builder 此刻所在的块**，把这个块记作 phi 的前驱。

| Flat 里的构造 | 生成的 LLVM 代码 |
| --- | --- |
| `Let` | 发出计算指令，在表里把名字关联到结果值 |
| `Case` | 查看标签，用 `switch` 跳到对应分支 |
| `Join` | 建一个基本块，每个参数对应一个 phi |
| `Jump` | 给目标的 phi 添上入边和值，接一条 `br` |
| 目标已知的尾调用 | `tail call tailcc`，紧接着 `ret` 返回它的结果 |
| 目标未知的函数调用 | 按 C 调用约定进入运行时的 apply |

光看出一个调用处在尾位置，还不足以保证原生栈不会越长越深。LLVM 对调用约定和随后的返回指令都有要求。我们把所有快入口设成 `tailcc`，在已知目标的尾调用上打上尾调用标记，并紧接着返回它的结果。使用内部链接（internal linkage）还满足了 x86-64 位置无关代码对符号可见性的要求。按这些条件生成代码，目标已知、参数恰好给齐的尾调用，包括互相递归的调用，都有有界栈保证。越过运行时的 C eval/apply 接口，就超出了这个保证的范围。[LLVM 尾调用规则](https://llvm.org/docs/CodeGenerator.html#tail-call-optimization)

## 实现

### 先定下代码要在哪种机器上跑

`llvm/mod.rs` 负责模块、指令构建器（builder）、目标机器和符号表。模块里的 LLVM 值借用的 `Context` 由调用者持有。这样谁活得更久一目了然，Rust 结构体也不必一边拥有上下文、一边再保存指向自己的借用。先把所有函数入口和存全局值的槽位声明好，再去生成函数体。这样前向调用和相互递归的函数都可以按同一套逻辑查到目标。

本章固定使用 Inkwell 0.10.0，启用 `llvm22-1-prefer-dynamic`，配套 LLVM 22。目标只初始化一次。建立目标机器时，平台标识（target triple）取自宿主机，CPU 选择通用型号，重定位使用位置无关模式。**发出第一条指令之前**，就要把平台标识和数据布局设到模块上。我们约定每个值占 64 位，所以会拒绝指针宽度为 32 位的目标。想先用小例子熟悉这套 Rust 绑定，可以看 [Compiler Crates 的 Inkwell 教程](https://sdiehl.github.io/compiler-crates/inkwell.html)。

指令构建操作可能报错，调用方必须处理。Flat lint 检查送进后端的输入。即使没开 `--lint`，每个生成完的 LLVM 模块也要通过 LLVM 验证器。优化使用新版 pass manager，完成后再验一遍。

```rust
/// Run the new pass manager only after constructing a complete, valid module.
pub fn optimize(&self, level: u8) -> Result<()> {
    if level > 3 {
        return Err(Error("optimization level must be 0..3".into()));
    }
    self.module
        .run_passes(
            &format!("default<O{level}>"),
            &self.machine,
            PassBuilderOptions::create(),
        )
        .map_err(|e| Error(e.to_string()))?;
    self.verify()
}
```

[源码 · mod.rs 第 171～184 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch21_llvm/src/llvm/mod.rs#L171-L184)

一个基本块漏了终结指令、phi 的入边不合法、调用参数的类型对不上，验证器都能帮我们及早发现。可它不知道 Fun 想表达什么。实参求值顺序颠倒了，或者整数溢出时的处理跟 Fun 约定不同，依然可能得到完全合法的 LLVM IR。因此还要让解释器和原生代码分别执行相同的程序，拿结果来对照。

### 列一张待办表，逐块生成

代码生成器遍历 Flat 树时不用 Rust 递归。把待处理的块放进一个向量作为工作表（worklist），每次取出一项，沿着连续的 `Let` 用循环往下走。走到末端，按情况发出返回或控制转移指令，或者把汇合点与各分支的主体加入工作表。

环境是从唯一名字查到 SSA 定义的一张表。Flat lint 已经检查过词法作用域，生成每个分支时无需把整张作用域表复制一遍。先为汇合点建好 phi，后面每生成一个跳向它的块，就能再补一条入边。每个块完工时恰好有一条终结指令。

这种遍历方式也用在检查、打印和释放数据上。最后一步尤其容易漏。编译看起来已经成功了，一棵很深的 `Rc` 树在离开作用域时，却可能由 Rust 自动生成的析构器一路递归释放，最后把栈撑爆。我们在释放时也把待断开的引用放进工作表，一批批处理。测试让一棵很深的 Flat 树在很小的 Rust 栈上走完整个过程，从检查、打印、生成和验证，一直到释放，才算把这条链路都走通了。

```rust
pub(super) fn body(
    &mut self,
    b: &Body,
    mut env: Env<'ctx>,
    mut joins: Joins<'ctx>,
) -> Result<()> {
    // Unique binders index SSA values; lint already checked scopes.
    // Branches share this table without cloning lexical environments.
    let block = self.builder.get_insert_block().unwrap();
    let mut work = vec![Block {
        body: b.clone(),
        block,
    }];
    while let Some(Block { body, block }) = work.pop() {
        self.builder.position_at_end(block);
        let mut cursor = body.as_ref();
        while let Flat::Let(v, r, rest) = cursor {
            let value = self.rhs(r, &env)?;
            env.insert(v.clone(), value);
            cursor = rest;
        }
        self.terminal(cursor, &mut env, &mut joins, &mut work)?;
    }
    Ok(())
}
```

[源码 · control.rs 第 108～132 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch21_llvm/src/llvm/control.rs#L108-L132)

### 生成的代码也要认第 20 章的内存布局

所有 Fun 值在 LLVM 里都用 `i64` 表示，布局和第 20 章共用同一份 `runtime_layout`。读取堆对象的某个槽位时，先把机器字转成指针，把指向的区域看成按字排列的数组，再按下标计算地址并读取。Inkwell 的指针是不透明的，所以还得明确告诉它这次访问的元素是什么类型。

```rust
pub(super) fn slot(
    &self,
    pointer: PointerValue<'ctx>,
    index: u64,
) -> Result<PointerValue<'ctx>> {
    // SAFETY: GEP indexes i64 words using the checked ABI layout.
    Ok(unsafe {
        self.builder.build_gep(
            self.word,
            pointer,
            &[self.n(index)],
            "slot",
        )?
    })
}
```

[源码 · values.rs 第 91～105 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch21_llvm/src/llvm/values.rs#L91-L105)

这里的 `unsafe` 是在**构造 LLVM 指令**，Rust 代码本身没有在此处解引用指针。将来生成的程序能否安全取到字段，靠的是前面已经检查过构造器的字段个数，也靠运行时严格遵守布局。计算槽位地址时使用普通 GEP（`getelementptr`），不额外承诺 `inbounds`。构造器的第一个字段就在对象头后一个字的位置。要取闭包捕获的第一个值，还得跳过慢入口地址和形参个数这两个字。

无字段的构造器直接编码成一个立即值。有字段的构造器用 `rt_alloc` 分配，再按槽位顺序填入字段。如果一个数据类型两种构造器都有，`case` 要先看最低位，只有遇到指针才能读对象头。如果该类型只有一种表示，就能省一次分支。匹配整数字面量和字符字面量，直接在带标记的字上做 `switch`。字符串相等比较则早在编译模式匹配时就拆成了原语测试。

字符串字面量会生成按要求对齐的只读对象，里面有对象头、字节数，以及末尾按整字补齐的 UTF-8 字节。一张专门保留的数据表保存着指向这些对象的根值。`llvm.used` 让优化器和链接器留下这张表。这些静态对象不会搬迁或释放。堆上分配的字符串和双精度浮点对象都用 atomic 分配，因为载荷没有指针，不需要扫描。生成的全局变量，以及寄存器和栈槽里仍然活着的值，Boehm 仍然能看见[^boehm1988]。

做整数运算时先算术右移，把数解出来，在 `i64` 上计算，再把结果重新打上标记。Fun 明确定义了整数溢出时要回绕，所以不加 `nsw` 或 `nuw` 这两种承诺“不发生有符号或无符号回绕”的标志。做除法前先判断除数，零就跳到 `rt_div_zero`。最小的 Fun 整数除以负一，结果虽然超出 Fun 的 63 位有符号范围，却还放得进 64 位中间值，再次编码就自然得到要求的 63 位回绕结果。底层的整数除法朝零截断，`div` 和 `mod` 则按向下取整约定计算。需要时还得根据符号修正商和余数。

`Double` 运算会拆箱，用 IEEE 浮点指令计算，再装回箱中，不打开 fast-math 标志。比较使用有序比较指令，保留 NaN 应有的行为。超越函数、饱和数值转换、文本处理、格式化和 IO 都调用解释器也在用的那份运行时。LLVM 可以优化生成的代码，但语言已经定下来的行为要保住。

### 知道目标就走快入口，拿到闭包就按约定应用

每个提升后的函数都有两个入口。**快入口**使用 `tailcc`，第一个参数是环境字，后面依次是各个实参，每项一个字。顶层函数用不上环境，这个位置传零。编译目标已知的调用时，除了给被调用函数标上调用约定，还得在**调用指令本身**标上同样的约定。

```rust
pub(super) fn known(
    &self,
    f: FuncId,
    env: Option<IntValue<'ctx>>,
    args: &[IntValue<'ctx>],
    tail: bool,
) -> Result<IntValue<'ctx>> {
    let mut actual: Vec<BasicMetadataValueEnum> =
        vec![env.unwrap_or(self.n(0)).into()];
    actual
        .extend(args.iter().map(|v| BasicMetadataValueEnum::from(*v)));
    let call = self.builder.build_call(
        self.fast[f.0 as usize],
        &actual,
        "known",
    )?;
    call.set_call_convention(TAIL_CC);
    call.set_tail_call(tail);
    if self
        .builder
        .get_insert_block()
        .unwrap()
        .get_parent()
        .unwrap()
        .get_call_conventions()
        != TAIL_CC
    {
        call.set_tail_call_kind(
            inkwell::values::LLVMTailCallKind::LLVMTailCallKindNoTail,
        );
    }
    Ok(call.try_as_basic_value().unwrap_basic().into_int_value())
}
```

[源码 · values.rs 第 275～307 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch21_llvm/src/llvm/values.rs#L275-L307)

**慢入口**遵循 C ABI，接收闭包值和一个指向实参数组的指针。它按函数定义里的形参个数从数组中取出实参，再进入快入口。闭包保存的代码地址指向慢入口。目标未知时，1 到 4 个参数分别有 `rt_apply1` 到 `rt_apply4`，更多参数则交给通用的 `rt_apply`。通用调用的实参数组在函数入口块分配，这样就算调用点处在循环中，也不用每次经过那里都分配一次。实参给少了或者给多了，仍由运行时按上一章的 eval/apply 协议处理[^marlow2004]。

穿过这个 C 适配层时，没有尾转移保证。尾位置上目标未知、需要经过 `rt_apply` 的调用也一样。因此高阶递归仍可能把原生栈越压越深。要把有界栈保证扩展到这种调用，需要用 LLVM 直接生成 apply 的调用路径，并采用兼容的调用约定。优化器在个别情况下碰巧能消掉一层调用，语言本身却不能据此许诺所有情况都能消掉。

### 初始化、生成目标文件，再链接

`fun_init` 先安装所有全局函数闭包，再按依赖顺序计算严格求值的顶层值。生成的 C 入口 `main` 依次初始化运行时、调用 `fun_init`、把单位值传给源程序的 `main`，最后通过 `rt_exit` 退出，让缓冲区里的输出都写出去。这个顺序决定了顶层如果有多处错误，哪一处先被触发，也保证了光把一个 IO 动作构造出来不会立刻执行其中的效果。

`native.rs` 通过目标机器写出目标文件，随后直接调用系统的 `cc`。链接时用上 `libfun_rt_21.a`、Boehm，以及 rustc 报告的本机库链接参数。可以设置 `FUN_RT` 指向一份已经安装的运行时静态库。没设置就到编译器旁边查找。`just build` 会事先准备好运行时和链接参数。链接工作放在输出文件旁的临时目录里，全部成功才把可执行文件放到指定位置。参数直接交给 `cc`，中间不经 shell 展开。链接失败，原有输出文件也不会被破坏。

跨越 `tailcc` 和 C 运行时这条边界的调用都显式标成 `notail`。如果省掉它，LLVM 有时会自行推断两个调用约定之间也能做尾转移。碰上实参很多的调用，返回时就可能清理错栈。这类问题难查在即使 IR 通过了验证，运行时也照样会出错。在这条边界标上 `notail`，已知 Fun 函数之间原先保证的尾调用仍然成立。

## 跑起来看看

按原书仓库 README 的说明安装 LLVM 22 和 Boehm，将 `LLVM_SYS_221_PREFIX` 设为 LLVM 的安装前缀。在原书仓库根目录构建，再运行验收程序。

```text
$ just build
$ target/debug/fun-ch21 build crates/ch21_llvm/tests/programs/native/acceptance.fun -o build/acceptance --lint
$ build/acceptance
5.0
3.0
6.0
[3.0,10.0]
True
50000005000000
$ target/debug/fun-ch21 dump --pass=llvm crates/ch21_llvm/tests/programs/native/acceptance.fun > build/acceptance.ll
$ target/debug/fun-ch21 dump --pass=llvm-opt crates/ch21_llvm/tests/programs/native/acceptance.fun > build/acceptance-opt.ll
```

看输出时可以对着源程序从上往下核算。点 `(3.0, 4.0)` 到原点的距离是 `5.0`，圆和矩形分别算出 `3.0` 与 `6.0`。最后一行是从 1 加到一千万的和。

同一份样例，让原生代码和解释器都跑一遍。除了正确输入产生的答案，还要比失败时的表现。如果一个程序里有两处会出错，最先报出来的是哪一处，往往能暴露求值顺序有没有被改坏。各个优化级别也值得试试。有些调用约定的问题，要等 LLVM 决定采用某种变换，才会露出来。

另有压力测试用 **512 KiB 的栈**运行原生递归程序。还有一个循环在 **64 MiB 的 Boehm 堆上限**下累计分配至少一亿个机器字。前者检查尾转移能否释放旧调用帧，后者检查已经用不到的堆对象能否回收。单凭 IR 通过 LLVM 验证，推不出这两个结论。

## 练习

1. **基础。** 从表示堆对象的 Fun 机器字开始，顺着 `inttoptr`、GEP、`load` 走一遍读取字段的过程。前面做过的哪项检查，能保证这里的字段下标有效？
2. **进阶。** 画出两个调用方跳到同一个汇合点的基本块。在其中一个调用方里加入一次除法，再指出此时哪个块才应该登记为 phi 的前驱。
3. **进阶。** 如果给整数乘法加上 `nsw`，平常的例子大多还能跑对。为什么这已经改变了 Fun 的语义？
4. **挑战。** 比对 `sumTo` 优化前后的 IR。找到递归转移，分别检查调用指令和目标函数上的调用约定。
5. **挑战。** 设计一个针对未知目标调用的压力测试，让 C apply 接口的栈限制显现出来。若要把这次调用变成有保证的尾调用，ABI 得怎样改？

## 延伸阅读

- 想用 Rust 从头搭一个 LLVM 模块，可以从 [Compiler Crates 的 Inkwell 教程](https://sdiehl.github.io/compiler-crates/inkwell.html) 读起。
- 查阅调用指令、phi 节点、整数溢出标志和指针运算的精确定义，可参考 [LLVM 语言参考手册](https://llvm.org/docs/LangRef.html)。
- 想判断什么时候能对栈大小作出保证，可以对照 [LLVM 尾调用的实现条件](https://llvm.org/docs/CodeGenerator.html#tail-call-optimization)。
- 汇合点如何表达可以共享的后续计算[^maurer2017]。
- 快速函数应用和 eval/apply 之间的边界[^marlow2004]。
- 严格求值语言在实现中如何选择数据表示[^leroy1990]。

[^maurer2017]: Luke Maurer、Paul Downen、Zena M. Ariola、Simon Peyton Jones（2017）。[Compiling without continuations](https://doi.org/10.1145/3062341.3062380)。
[^boehm1988]: Hans-Juergen Boehm、Mark Weiser（1988）。[Garbage collection in an uncooperative environment](https://doi.org/10.1002/spe.4380180902)。
[^marlow2004]: Simon Marlow、Simon Peyton Jones（2004）。[Making a fast curry: push/enter vs. eval/apply for higher-order languages](https://doi.org/10.1145/1016850.1016856)。
[^leroy1990]: Xavier Leroy（1990）。[The ZINC experiment: an economical implementation of the ML language](https://caml.inria.fr/pub/papers/xleroy-zinc.pdf)。

---

[← 第 20 章　数据表示与运行时](ch20-runtime.md) · [中文目录](../README.md) · [第 22 章　JIT 与交互式 REPL →](ch22-jit.md)
