# 第 20 章　数据表示与运行时

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch20-runtime.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch20_runtime)

> **新增文件**
>
> - `rt/src/layout.rs`
> - `rt/src/abi.rs`
> - `rt/src/heap.rs`
> - `rt/src/apply.rs`
> - `rt/src/primitives.rs`
> - `rt/build.rs`
> - `rt/src/lib.rs`
>
> **改动文件**
>
> - `src/primitives.rs`
> - `src/lib.rs`
> - `src/main.rs`

## 一个 `id` 怎么接住所有类型的值？

前几章写解释器，想表示哪些值，在 Rust 的枚举里列出来就行了。至于这些值在内存里占多大、放在哪儿，Rust 替我们管了不少事。可接下来要生成的机器码手里没有这套枚举。一个整数、一节链表、一个闭包，到底在内存里长什么样，得由我们自己定。

拿最简单的多态函数 `id` 来说。我们希望只编译它一次，无论传来整数、字符串还是一棵树，都走同一套传参和返回约定。怎么做到？约好每个值在传递时都占**一个机器字**。能直接塞进去的就直接存，装不下的就存一个指针，再借一个比特来区分这两种情况。

这一章先把运行时单独做出来。这样，我们能先弄清楚数据怎么摆、参数怎么传，再去接 LLVM。运行时里的原语也可以给已经写好的解释器共用。比如打印浮点数，同一个数究竟该打印成什么样，只需要在一个地方定好规矩。

## 这时的 Fun 能写什么

Fun 程序的写法和含义都沿用上一章。整数仍然按 63 位有符号数回绕，字符仍是 Unicode 标量值，字符串仍用 UTF-8，构造器的字段仍会先求值。只要从程序里看，换一种内存布局之后，这些行为就应该和以前一样。

拿前几章传下来的这个程序试试。它特意挑了几个容易打印错的浮点值，包括 NaN、无穷大和有正负之分的零。

```fun
_old = (0.0/0.0,1.0/0.0,-0.0)

main = print _old
```

[源码 · float-special.fun 第 1～3 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch20_runtime/tests/programs/classes/float-special.fun#L1-L3)

这一章的运行时叫 `fun-rt-20`。它既能编成编译器使用的 Rust 库（`rlib`），也能编成别的程序可以链接的静态库（`staticlib`）。目前的章节编译器仍然解释执行 Core、ANF 或 Flat。现在已经可以拿一段 C 程序链接运行时，验证两边能否正确传值。要让 `fun build` 把一份 Fun 源码编译成可执行文件，还得等下一章用 LLVM 生成目标文件。

## 原理

在这里，一个值占一个 64 位机器字。先看它最低的一位。最低位为 1，表示值就装在这个字里，称为**立即值**。最低位为 0，就按对齐过的指针处理。编码整数时，先左移一位，腾出最低位，再把那一位设成 1。余下的 63 位存有符号整数。解码要用算术右移，让负数的符号位跟着补回来。字符的 Unicode 标量值和无字段构造器的编号，也可以用这套立即值编码。

```rust
pub const fn integer(n: i64) -> Word {
    ((n as u64) << 1) | 1
}

pub const fn signed(w: Word) -> i64 {
    (w as i64) >> 1
}
```

[源码 · layout.rs 第 58～60 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch20_runtime/rt/src/layout.rs#L58-L60) · [源码 · layout.rs 第 62～64 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch20_runtime/rt/src/layout.rs#L62-L64)

假如拿到的机器字恰好是 `1`，它是什么意思？单位值 `()`、`False`、空列表和 `Nothing` 都可能用这个字表示。程序允许怎样使用这个值，类型检查时就定好了。对列表做模式匹配的代码会按列表去看它，不会临时拿它当布尔值。所以，运行时无需单靠这 64 位猜出完整的源语言类型。

这也是统一表示对多态特别方便的原因。`id` 可以原样把这个字传下去，不必按每一种具体类型另写一套搬运代码。给机器字打标记，再把大值装进堆对象（装箱），就能在严格求值的函数式语言里用同一套办法传值[^leroy1990]。

堆对象的第一个字是**对象头**（header）。最低的 8 位说明对象种类，接下来的 24 位存构造器标签，最高的 32 位记录头部后面还有多少个字。构造器标签只在各自的数据类型内编号，顺序跟声明一致。Prelude 里常用构造器的编号固定下来，运行时创建的列表、可选值才能和编译器创建的同类值互相通用。

本章用 Boehm 垃圾回收器管理堆。它会查看内存中的机器字，把看起来像指针的字当作潜在指针[^boehm1988]。好处是编译器不用在每处代码里说明哪些指针还活着。代价也很具体。如果一个普通数恰巧长得像地址，它可能把原本已经用不着的对象留在堆里。随意搬动对象也行不通，因为要搬就得改掉所有指向它的真指针，可回收器分不清哪些位模式真能改。

第 24 章会换个做法，让编译器把通向活对象的入口，也就是**根**（root），明确告诉回收器。这种配合是实现精确移动式回收的基础。Henderson 的论文和《垃圾回收手册》都讨论了这里的区别[^henderson2002][^gc2023]。

## 实现

编译器和运行时都以 `rt/src/layout.rs` 为 ABI（应用二进制接口）的布局定义。编译器以 `runtime_layout` 这个名字重新导出它，因为 `layout` 先前已经用来处理源码的缩进布局。组装对象头时，先检查构造器标签能否放进为它预留的位数。

```rust
/// Bits 0..8: kind, 8..32: tag, 32..64: payload word count.
pub const fn header(kind: u8, tag: u32, size: u32) -> Word {
    assert!(tag <= 0x00ff_ffff);
    kind as Word | ((tag as Word) << 8) | ((size as Word) << 32)
}
```

[源码 · layout.rs 第 74～78 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch20_runtime/rt/src/layout.rs#L74-L78)

对象头之后的内容称为**载荷**（payload）。不同对象用到的槽位如下。

| 对象种类 | 对象头之后，从前往后存什么 |
| --- | --- |
| `CON` | 构造器的各个字段，每个字段一个字 |
| `CLOSURE` | 慢入口的代码地址、函数形参个数，随后是捕获的值 |
| `PAP` | 闭包值、已给实参的个数，随后是这些实参 |
| `DOUBLE` | 一个字，保存 binary64 浮点数的原始位模式 |
| `STRING` | 字节数，随后是按整字补齐的 UTF-8 字节 |
| `FORWARD` | 留给后面要做的复制式垃圾回收器 |

### 顺着一个机器字看堆

拿到变量里的一个字，先从最低位判断该怎么读。

| 变量里的 64 位机器字 | 最低位 | 怎么读 |
| --- | ---: | --- |
| `85` | 1 | 直接解码，得到整数 `42` |
| 对齐的地址 `p` | 0 | 到堆中地址 `p`，从对象头开始读 |

每个槽位占 8 字节。下表中的偏移量以对象头所在地址 `p` 为起点。

| 字节偏移 | 这里放着什么 |
| ---: | --- |
| +0 | 对象头 |
| +8 | 第 1 个载荷字 `payload[0]` |
| +16 | 第 2 个载荷字 `payload[1]`（如有），此后每 8 字节一个槽位 |

把偏移 +0 的对象头单独拆开，从高位往低位读。

| 位编号（含两端） | 位宽 | 含义 |
| --- | ---: | --- |
| 63～32 | 32 位 | `size`，对象头之后的载荷字数 |
| 31～8 | 24 位 | `tag`，构造器标签 |
| 7～0 | 8 位 | `kind`，对象种类 |

对比 `Just 42` 和 `Double 3.0`。它们都只有一个载荷字。`Just` 的构造器标签为 `1`，字段里装的是 Fun 整数 `42` 编码后的 `85`。而 `Double` 的载荷直接存下 IEEE 浮点数的位模式，这个字不用按 Fun 的值编码再解一遍。

| 字节偏移 | `Just 42` | `Double 3.0` |
| ---: | --- | --- |
| +0 | 对象头 `CON, tag=1, size=1` | 对象头 `DOUBLE, tag=0, size=1` |
| +8 | `85`，带标记的 Fun 整数 `42` | `0x4008000000000000`，浮点数 `3.0` 的原始位模式 |

闭包把代码和代码要用的外部值放在一起。设想局部函数 `f x y = k + x + y`，其中 `k = 10`。它需要两个实参，还捕获了一个值 `k`。如果先写 `f 20`，实参只来了一个，于是会产生一个**部分应用对象**（partial application，简称 PAP）。PAP 指回原来的闭包，同时记住已经收到的 `20`。

| 字节偏移 | 从 PAP 对象的开头读 `f 20` | 从闭包对象的开头读 `f` |
| ---: | --- | --- |
| +0 | 对象头 `PAP, tag=0, size=3` | 对象头 `CLOSURE, tag=0, size=3` |
| +8 | 闭包值，指向**右列对象的 +0 处** | 慢入口的代码地址 |
| +16 | 已给实参个数 `1` | 函数的形参个数 `2` |
| +24 | 已给实参 `20`，编码后为 `41` | 捕获的 `k = 10`，编码后为 `21` |

留意哪些字能当成 Fun 值来读。从 PAP 的 +8 处取出指针，顺着它就到了闭包的对象头。形参个数 `2` 和已给实参个数 `1` 都直接存原始计数。闭包捕获的 `k` 和 PAP 暂存的 `20` 才按 Fun 值编码，所以表中分别是 `21` 和 `41`。如果捕获的是堆对象，对应的字里就会存指针。慢入口存的是代码地址。现在把这个 PAP 应用于 `22`，运行时会取出暂存的 `20`，连同新来的 `22` 一起交给闭包。闭包仍从自己的环境里取 `k = 10`，最后算出 `10 + 20 + 22 = 52`。

字符串先存字节数，再存 UTF-8 字节，末尾按整字补齐。比如 `"hi"` 的载荷占两个字，一个写长度 2，一个装 `68 69` 这两个十六进制字节和余下的填充位。填充不是字符串内容，也不用额外放一个零字节当结束标记。

| 字节偏移 | 字符串对象 `"hi"` |
| ---: | --- |
| +0 | 对象头 `STRING, tag=0, size=2` |
| +8 | 字节数 `2` |
| +16 | `68 69 .. .. .. .. .. ..`（十六进制，`..` 表示一个填充字节） |

分配时别把单位搞混。`rt_alloc` 接收的是**算上对象头的总字数**。`rt_alloc_atomic` 接收的是**字节数**，并告诉 Boehm 这块内存的内容无须扫描。浮点数和字符串的载荷都不含值指针，适合用后者。字符串对象里记的是字节数，而 Fun 里求字符串长度的原语数的是 Unicode 标量值。整个布局都按 8 字节对齐。

还有个危险的时间窗口。我们拿着一组字段，打算分配对象再把它们复制进去，偏偏**分配这一步就可能触发垃圾回收**。一个普通 Rust `Vec` 的缓冲区由 Rust 分配器管理，Boehm 不会自动顺着它去扫描里面的字。于是 `Roots` 专门持有一块地址稳定的堆缓冲区，把中间存放字段的地址区间登记给回收器。缓冲区两端各留一个不登记的保护字，避免相邻登记区间连在一起。`Roots` 析构时，先撤销登记，再释放缓冲区。看看下面的构造顺序，在哪一步保护了输入字段，之后才可以分配新对象？

```rust
/// Allocate a scanned object, protecting fields even when supplied in a Rust Vec.
pub fn object(kind: u8, tag: u32, fields: &[Word]) -> Word {
    let roots = Roots::new(fields);
    let size = u32::try_from(fields.len()).expect("object too large");
    let p = rt_alloc(fields.len() + 1);
    // SAFETY: p has header + fields words; protected input and destination do not overlap.
    unsafe {
        p.write(layout::header(kind, tag, size));
        ptr::copy_nonoverlapping(
            roots.as_slice().as_ptr(),
            p.add(1),
            fields.len(),
        );
    }
    p as Word
}
```

[源码 · heap.rs 第 150～165 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch20_runtime/rt/src/heap.rs#L150-L165)

运行时在主线程上初始化 Boehm。对 Boehm 来说，Rust 创建的其他工作线程是外来线程，它们得登记自己的栈，退出时再撤销登记。直接调用 ABI 的测试会挑两个时机强制回收，一是函数调用进行到一半，二是互相引用的闭包环境仍然活着。如果实现只是碰巧在分配时没遇到回收才跑得对，这些测试就能戳穿它。

闭包里存着一个按 C 调用约定进入的慢入口，以及函数定义中写了几个形参（arity，元数）。慢入口接收闭包本身的值和一个实参数组指针，数组里的参数恰好够这次调用。创建闭包时，把这些信息按前面的布局装起来就行。

```rust
/// Store a code address, syntactic arity, then captured values.
pub fn closure(code: Slow, arity: usize, captures: &[Word]) -> Word {
    assert!(arity > 0);
    let mut fields = vec![code as usize as Word, arity as Word];
    fields.extend_from_slice(captures);
    heap::object(layout::CLOSURE, 0, &fields)
}
```

[源码 · apply.rs 第 10～16 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch20_runtime/rt/src/apply.rs#L10-L16)

`rt_apply` 先把实参缓冲区登记为根，再把 PAP 存的旧参数和这次收到的新参数接起来，按第 19 章的 eval/apply 规则处理。实参不够，就分配一个 PAP 留待下次。恰好够，就进入函数代码。还有多余的，就先算出这次调用返回的闭包，再把剩余参数交给它[^marlow2004]。`rt_apply1` 到 `rt_apply4` 底层共用这套实现。`rt_closure_set` 只在构造递归函数组、闭包尚未对外可见时，检查并回填捕获值的槽位。

`apply` 里虽然用了循环，从它调用 C 慢入口，却不能保证原生栈上的尾调用一定消掉。慢入口里的代码可能又调回 `apply`，于是旧调用帧还没退，新调用帧又来了。第 21 章会为目标已知的快速调用指定 LLVM 的尾调用约定。ANF 和 Flat 解释器能够靠迭代执行避开宿主语言的递归调用，那是解释器本身的设计。把函数放到 C ABI 后面，不会自动获得同样的保证。

`primitives.rs` 提供安全、带类型的值接口，`abi.rs` 才处理一个个机器字。这三个解释器各自保留了控制流引擎和堆模型，只需把自己的数据值转成安全接口要的样子即可。面向原生代码的导出函数则按已经检查过的源语言类型，把机器字解出来，调用同一份原语实现，再把结果编码回去。`unsafe` 内存访问集中在运行时，旁边注明调用者必须保证对象仍然存活、类型也正确。如果 Rust 的栈展开越过 `extern "C"` 入口，进程会中止。Fun 程序自己的错误则走运行时明确提供的报错和退出路径。

以 `show_double` 为例。它先让 Rust 给出足以把浮点数原样读回来的最短数字串，再整理显示格式。非零数的绝对值在 `[1e-4, 1e16)` 内时，用普通小数写法。其他有限非零数用科学计数法，指数带正负号。零保留正负号，特殊值按约定写成 `NaN`、`Infinity` 和 `-Infinity`。测试分别从安全接口和机器字 ABI 检查边界，还会把有限值打印后重新读入，比对前后的位模式。`readInt` 接受首尾空白和十进制数的正负号，超出 63 位有符号范围就读不出有效整数，返回 `Nothing`。浮点 `round` 遇到恰好在两个整数中间的值时取偶数，超出目标范围则停在边界（饱和转换），NaN 转成零。

## 跑起来看看

下面的 `double-specials` 样例比开头的更全，还专门踩中了普通小数和科学计数法之间的切换边界。

```text
$ cd crates/ch20_runtime
$ cargo run -q -- run tests/programs/runtime/double-specials.fun --lint
-0.0
NaN
Infinity
-Infinity
0.0001
1.0e+16
$ cargo run -q -- dump --pass=types tests/programs/runtime/double-specials.fun
main : IO ()
```

回到原书仓库根目录，运行 `cargo test -p fun-rt-20`，可以检查各个原语的行为，也可以跑独立的 ABI 测试。`python3 scripts/check_runtime_abi.py` 会编译静态运行时，向 rustc 查询本机还需要链接哪些原生库，再用系统的 C 编译器把一段冒烟测试程序链接起来，最后看它能否打印 `42`。`just build` 也会先单独构建运行时，再构建整个工作区。Linux 需要安装 `libgc-dev`。macOS 用 Homebrew 的 `bdw-gc`，如果装在其他位置，可以用 `GC_LIB_DIR` 指定库目录。

对象布局画得对，还得在实际运行里经得起考验。回收器的错误常常要等到刚好那一次分配触发回收才显现。测试因此会在构造递归闭包的途中、在函数调用还握着活参数时强制回收，也覆盖部分应用、一次给多了参数的调用、Unicode 字符串，以及程序出错前已经产生的输出。

## 练习

1. **基础。** 分别算出 `-1`、`0`、`True` 和单位值编码后的机器字。解释为什么两个字完全相同，它们表示的源语言值仍可能不同。
2. **进阶。** 画出一个含两个字段的构造器对象，再画一个暂存了一份实参的 PAP。标出每个对象的对象头和载荷字数。
3. **进阶。** 一个指针明明还要用，却只保存在普通 Rust 向量里。为什么 Boehm 可能看不见它？
4. **挑战。** 写一个 ABI 测试，在递归闭包已经分配、捕获槽位还没回填时强制回收一次。确保所有未建完的对象在回收时都能从根找到。
5. **挑战。** 假如回收器会移动对象，为了更新闭包捕获的值、PAP 的实参、全局变量和仍然活着的局部值，还需要哪些额外信息？

## 延伸阅读

- 编译器不提供指针位置时，保守式垃圾回收如何工作[^boehm1988]。
- 让编译器参与进来，怎样找全垃圾回收的根[^henderson2002]。
- 严格求值的函数式语言可以怎样表示运行时数据[^leroy1990]。
- eval/apply 和部分应用如何衔接[^marlow2004]。
- 各类垃圾回收算法和实现中的取舍[^gc2023]。
- Takenobu Tani 的 *GHC (STG, Cmm, asm) Illustrated* 用图展示 GHC 的堆对象、栈布局和调用约定，可以拿来和本章逐项比较。[PDF](https://takenobu-hs.github.io/downloads/haskell_ghc_illustrated.pdf) · [GitHub](https://github.com/takenobu-hs/haskell-ghc-illustrated)

[^leroy1990]: Xavier Leroy（1990）。[The ZINC experiment: an economical implementation of the ML language](https://caml.inria.fr/pub/papers/xleroy-zinc.pdf)。
[^boehm1988]: Hans-Juergen Boehm、Mark Weiser（1988）。[Garbage collection in an uncooperative environment](https://doi.org/10.1002/spe.4380180902)。
[^henderson2002]: Fergus Henderson（2002）。[Accurate garbage collection in an uncooperative environment](https://doi.org/10.1145/512429.512449)。
[^gc2023]: Richard Jones、Antony Hosking、Eliot Moss（2023）。[The Garbage Collection Handbook: The Art of Automatic Memory Management](https://doi.org/10.1201/9781003276142)。
[^marlow2004]: Simon Marlow、Simon Peyton Jones（2004）。[Making a fast curry: push/enter vs. eval/apply for higher-order languages](https://doi.org/10.1145/1016850.1016856)。

---

[← 第 19 章　闭包转换](ch19-closures.md) · [中文目录](../README.md) · [第 21 章　用 LLVM 生成机器码 →](ch21-llvm.md)
