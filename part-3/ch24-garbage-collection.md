# 第 24 章　精确垃圾回收（选读）

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch24-garbage-collection.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch24_gc)

> **新增文件**
>
> - `rt/src/copying.rs`
> - `src/liveness.rs`
> - `src/llvm/roots.rs`
>
> **改动文件**
>
> - `src/llvm/control.rs`
> - `src/llvm/values.rs`
> - `rt/src/heap.rs`
> - `rt/src/apply.rs`
> - `src/llvm/mod.rs`
> - `src/llvm/primitives.rs`
> - `src/native.rs`
> - `src/main.rs`
> - `rt/src/lib.rs`
> - `src/lib.rs`

## 列表搬家了，谁还记着旧地址？

前面一直靠 Boehm 回收内存。写编译器时，我们可以先把对象分配出来，不用逐个告诉回收器，哪些地方还存着指向它的指针。现在条件齐了。编译器知道哪些值后面还可能用到，运行时也知道每种对象里的哪些字段装着 Fun 值。把这两份信息接起来，我们就能自己写一个**精确垃圾回收器**。

本章选择**复制式回收**。把还能用到的对象搬到新空间，旧空间就可以整块重用。麻烦出在搬完之后。假设闭包捕获了一张列表，列表搬家时，我们把闭包里的地址改成新的。可是，调用者的寄存器里可能还存着旧地址。下一次它从那里取表头会发生什么？要让每个活着的引用都找到新家，生成的机器代码和运行时得遵守同一套约定[^henderson2002]。你可以继续用 Boehm，跳过这一章。如果想从头到尾看清一个 Fun 值在内存里的去向，这次搬家正好把前面的知识串起来。

## 在构建时选择回收器

Fun 的语法照旧。构建可执行文件时加上 `--gc=copying`，然后给它安排一个刻薄的测试，让它每次分配之前都回收一次。

```text
$ cd crates/ch24_gc
$ cargo run -q -- build tests/programs/gc/partial.fun \
    -o ../../build/partial --gc=copying --lint
$ FUN_GC_STRESS=1 ../../build/partial
100
αβγ
```

```fun
add x y z = x <> y <> z
main = do
  let f = add "α"
  let g = f "β"
  print (length (map show [1..100]))
  putStrLn (g "γ")
```

[源码 · partial.fun 第 1～6 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch24_gc/tests/programs/gc/partial.fun#L1-L6)

`g` 记住了前两段字符串。中间那句 `print` 让程序有很多机会分配对象、触发搬迁。走到最后一行，它还得能找回前两段，再拼上 `"γ"`。

默认仍是 `--gc=boehm`。复制式回收用于提前编译出的原生可执行文件，解释器继续用自己的 Rust 表示，JIT 仍向 Boehm 注册根。Fun 目前没有并发，这个复制堆也只归一个线程使用。程序开始初始化全局值之前，就要选好回收器。

## 沿着根复制，沿着新副本往下找

把堆分成两块。平时从正在使用的那一块（*from-space*）分配，空闲指针往前挪就行。回收时，从程序还握在手里的引用出发，这些起点叫**根**（root）。顺着根找到的对象都搬到另一块（*to-space*）。第一次搬走对象 A，就把旧对象的头改写成转发标记，在它原先的第一个载荷字里留下新地址。下次再碰到指向旧 A 的引用，顺着这个地址就能找到同一个新 A。

Cheney 算法最巧的地方在于，已经搬到新空间的对象，排在那里就构成了一个待办队列。无需另建栈来遍历整张对象图。只用两个往前走的游标，就分得清哪些对象的字段还没查过[^cheney1970]。

| 新空间从左到右 | 这里放着什么 |
| --- | --- |
| 起点到 `scan` 之前 | 已复制，而且每个该追踪的字段都检查过了 |
| `scan` 到 `free` 之前 | 已复制，等着检查字段的对象。这一段就是队列 |
| 从 `free` 开始 | 空闲空间。下一个需要复制的对象写到这里 |

从 `scan` 取出一个对象。它的某个字段如果指向还没搬来的对象，就把后者复制到 `free`，改掉字段里的旧地址，再向前推进 `free`。扫完当前对象才推进 `scan`。等 `scan` 追上 `free`，就算队列空了。

比如，两个根都指向 A，A 仅有的字段指向 B，而 B 自己没有指针字段。用 A′、B′ 表示搬家后的对象，手动走一遍。

| 现在做什么 | 两个根指向哪里 | 新空间里还有谁待查 |
| --- | --- | --- |
| 处理第一个根，复制 A | 第一个指向 A′，第二个仍指向 A | A′ |
| 处理第二个根，读到 A 的转发地址 | 两个都指向 A′ | A′ |
| 扫描 A′，复制 B，并改写 A′ 的字段 | 两个都指向 A′ | B′ |
| 扫描 B′ | 两个都指向 A′ | 队列已空 |

原先两处共享的 A 只搬了一次，搬完仍然共享。如果 B 还有一条边指回 A，读到旧 A 的转发标记就知道该填 A′，不会绕着环一直复制。最后交换两块空间的角色，从新的 `free` 继续分配。无论图有多深，扫描都不会一层一层占用 Rust 调用栈。

检查哪些字，由对象头里的种类决定。

| 对象或值 | 回收器怎样处理 |
| --- | --- |
| 带字段的构造器 | 每个载荷字都是 Fun 值，逐个检查 |
| 闭包 | 跳过原始代码地址和元数（参数个数），往后才是捕获的值 |
| 部分应用对象（PAP） | 检查里面的闭包和已给的实参，跳过记录实参个数的原始机器字 |
| 堆上的 `Double`、字符串 | 载荷里没有要追踪的指针，整段跳过 |
| 零元构造器、整数 | 机器字的最低位是 1，直接跳过 |
| 静态字符串 | 地址落在旧半空间之外，留在原处 |

## 根要能改，用之前还要重新读

编译器在普通机器栈旁边维护一份显式的根登记表，函数进入和退出时按栈的次序登记、撤销，这就是**影子栈**（shadow stack）。每次函数调用都有自己的槽位数组，每个槽位可以保存一个带标签的机器字。原生整数和浮点中间结果不用登记为根。可能装着指针的带标签结果要放进槽位，后面用它时再从槽里读。

假设局部变量 `xs` 指向 A，一次分配触发回收，列表从 A 搬到了 A′。回收器可以把槽里的 A 改成 A′，可寄存器里先前读出的 A 仍是旧值。所以，**分配前把值放进根槽，分配后从槽里重新取值**，这两步缺一不可。

准备函数实参时也有这个问题。第一个实参准备好了，给后面的 `Double` 装箱却可能触发回收，把前一个实参搬走。生成的代码先把每个转换好的实参存进临时根槽，等全部就绪，再统一读出来发起调用。构造器也一样，跨过分配器调用之前，先保护好各个字段，分配完再把更新后的地址写进新对象。Rust 管理的参数缓冲区同样登记为根。它的 API 返回复制出来的机器字或 ABI 约定的裸指针。如果交出一个指向这些单元的 Rust 共享切片，回收器就可能在共享借用期间改写它。

函数返回时弹出自己的影子栈帧。对目标已知的尾调用，要在进入被调函数**之前**弹出当前帧。从最后一次读出实参，到被调函数登记自己的形参，这一小段路径没有内存分配，所以不会在交接的空当发生搬迁。这样尾递归不会堆积影子栈帧。调用目标未知时，仍有第 21 章讲过的 eval/apply 调用限制。

## 只留下以后还可能用到的值

把每个值都挂在根上，直到函数返回，地址确实安全。可一个长时间运行的循环，可能只在第一轮用过某张大列表，余下几百万轮却一直不让它回收。编译器要回答的是，在程序的**这个位置**，哪些绑定以后还可能被读？这就是**活跃性分析**。

沿控制流反着推。一个 Flat 节点自己读的操作数当然是活跃的。还要看后继节点需要什么，但在通往后继的那条边上才定义的变量，应先扣掉，它们尚不需要从当前节点带过去。比如跳到汇合点（join point）时，后继是汇合点的主体，汇合点自己的形参在入口处才绑定，也要扣掉。

所有节点从空集开始算。某个后继的结果变大了，就把受它影响的前驱放回待办队列。集合只增不减，一直算到谁都不再变化，就得到**最小不动点**。分支能这样处理，绕回汇合点的回边也一样。生成某一步运算的代码前，先把已经用完的绑定槽和临时根区清零。控制流分析可以多保留一点可能还活着的值，但它交给回收器追踪的每个非零槽都必须是合法的带标签值。未装箱的数值中间结果不能混进来。

## 对照实现

`liveness.rs` 用显式待办队列建立控制流图，又用一个待办队列解活跃变量集合。源代码里嵌套多少层，都不用让 Rust 函数跟着递归多少层。

```rust
let mut queue = nodes.keys().copied().collect::<VecDeque<_>>();
let mut queued = nodes.keys().copied().collect::<BTreeSet<_>>();
while let Some(id) = queue.pop_front() {
    queued.remove(&id);
    let n = &nodes[&id];
    let mut live = n.uses.clone();
    for (next, defined) in &n.edges {
        live.extend(result.before[next].difference(defined).cloned());
    }
    if result.before[&id] != live {
        result.before.insert(id, live);
        for p in predecessors.get(&id).into_iter().flatten() {
            if queued.insert(*p) {
                queue.push_back(*p);
            }
        }
    }
}
```

[源码 · liveness.rs 第 136～153 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch24_gc/src/liveness.rs#L136-L153)

`llvm/roots.rs` 分配根槽。一个可能含指针的绑定，原先用 SSA 里的机器字就能表示，现在要记住可以重新加载它的槽。一次函数调用期间，槽本身的地址是稳定的，回收器可以更新里面的值。

```rust
pub(super) fn root_binding(
    &self,
    v: &Id,
    loc: Loc<'ctx>,
) -> Result<Loc<'ctx>> {
    let Some(frame) = &self.frame else {
        return Ok(loc);
    };
    if !matches!(loc, Loc::Word(_) | Loc::Root(_)) {
        return Ok(loc);
    }
    let p = self.slot(frame.array, frame.slots[v])?;
    let w = self.loc_word(loc)?;
    self.builder.build_store(p, w)?;
    Ok(Loc::Root(p))
}
```

[源码 · roots.rs 第 51～66 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch24_gc/src/llvm/roots.rs#L51-L66)

运行时的 `evacuate` 只做三种事。值不用搬就原样返回。对象已经搬过就取转发地址。第一次遇见对象才复制它。对象头记录的大小只含载荷，复制时还要另算头本身占的一个字。

```rust
fn evacuate(
    from: &mut [Word],
    used: usize,
    to: &mut [Word],
    free: &mut usize,
    w: Word,
) -> Word {
    let base = from.as_ptr() as usize;
    let address = w as usize;
    // Immediates, null placeholders, and static string/code addresses never move.
    if layout::is_immediate(w)
        || address < base
        || address >= base + used * 8
    {
        return w;
    }
    assert_eq!((address - base) % 8, 0);
    let start = (address - base) / 8;
    let header = from[start];
    if layout::kind(header) == layout::FORWARD {
        return from[start + 1];
    }
    let words = 1 + layout::size(header);
    assert!(words >= 2 && start + words <= used);
    let dest = *free;
    to[dest..dest + words].copy_from_slice(&from[start..start + words]);
    *free += words;
    let pointer = to.as_mut_ptr().wrapping_add(dest) as Word;
    from[start] = layout::header(layout::FORWARD, 0, 1);
    from[start + 1] = pointer;
    pointer
}
```

[源码 · copying.rs 第 83～114 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch24_gc/rt/src/copying.rs#L83-L114)

回收器先扫根，再消费新空间里的队列。需要扩容时，先把目的空间准备好，才开始搬对象。持有旧地址期间调整正在使用的半空间大小，可能让地址在复制前就失效，代码刻意避开了这一点。

```rust
fn collect(&mut self, reserve: usize) {
    let capacity = if self.used + reserve > self.from.len() {
        self.from
            .len()
            .saturating_mul(2)
            .max(self.used.checked_add(reserve).expect("heap overflow"))
    } else {
        self.from.len()
    };
    self.spare.resize(capacity, 0);
    let mut free = 0;
    for (&address, &len) in &self.roots {
        for i in 0..len {
            let slot = (address as *mut Word).wrapping_add(i);
            // SAFETY: registered ranges remain writable throughout collection.
            unsafe {
                slot.write(evacuate(
                    &mut self.from,
                    self.used,
                    &mut self.spare,
                    &mut free,
                    slot.read(),
                ));
            }
        }
    }
    let mut scan = 0;
    while scan < free {
        let header = self.spare[scan];
        let size = layout::size(header);
        let first = match layout::kind(header) {
            layout::CON => 0,
            layout::CLOSURE => 2, // code and arity are raw, not values
            layout::PAP => 0,
            layout::DOUBLE | layout::STRING => size,
            _ => panic!("invalid object in to-space"),
        };
        for field in first..size {
            if layout::kind(header) == layout::PAP && field == 1 {
                continue;
            }
            let slot = scan + 1 + field;
            let value = self.spare[slot];
            self.spare[slot] = evacuate(
                &mut self.from,
                self.used,
                &mut self.spare,
                &mut free,
                value,
            );
        }
        scan += size + 1;
    }
    std::mem::swap(&mut self.from, &mut self.spare);
    self.used = free;
    self.collections += 1;
    self.copied_words += free;
}
```

[源码 · copying.rs 第 119～176 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch24_gc/rt/src/copying.rs#L119-L176)

设 `FUN_GC_STRESS=1`，每次分配前都触发回收，许多平时靠运气躲过去的悬空地址就容易重现了。即便回收恰好卡在构造闭包或 PAP 的中途，对象也要活下来。原先共享的引用依然得指向同一个对象，环还是环，`Double` 的原始位模式也绝不能被误认成待追踪的指针。

## 跑起来看看

```text
$ cd crates/ch24_gc
$ cargo run -q -- dump --pass=llvm tests/programs/gc/partial.fun --gc=copying
; ... root arrays, rt_roots_push, reloads, rt_roots_pop ...
$ cargo test -p fun-rt-24 --test copying
$ cargo test --test copying
```

在 LLVM 优化之前看这份输出，比较容易认出根数组、压栈、重新加载和弹栈。压力模式有意让回收频繁发生。如果一张仍然活着的列表每增加一个节点就整张搬一次，复制总量会按长度的平方增长。用这个模式抓旧地址，测日常回收性能则另跑一遍默认频率。两种回收器都没有承诺能在操作系统分配内存失败后恢复执行。

采用复制回收的原生程序链接专门的 `copying-only` 运行时归档，其中去掉了 Boehm 的导入和链接选项。普通的运行时 `rlib` 仍为 JIT 提供 Boehm 堆。`just build` 准备好两份归档，并分别记录需要传给本机链接器的参数。测试除了核对回收压力下的程序输出，还会检查可执行文件实际依赖了哪些动态库。

### 什么时候该把堆扩大

能把每个对象正确搬到新地址，还不够。如果半空间几乎装满了活对象，大动干戈搬完一轮，只腾出几个字，下次分配马上又要再搬一轮。为避免这种空忙，因容量不足触发回收后，实现会比较两项大小。若满足本次分配之后，余下的空位还少于活对象占用的空间，就扩容。看代码里的条件，若半空间容量为 C、回收后存活 L 个字、本次要分配 W 个字，这项判断就是 `2 × L + W > C`。压力模式会让每次分配都回收，但单单打开压力模式不会强行要求扩容。

有个回归测试故意让一个半空间的八分之七保持存活，然后反复分配短命对象。它给这期间复制的机器字总数设了上限。这比规定「必须在多少毫秒内跑完」更能直接检验扩容策略，换台机器也不用重定标准。

## 练习

1. **基础。** 画两个指向同一对象的根。一步步跟踪复制过程，说清对象为什么只搬一次，以及最后两个根分别指向哪里。
2. **进阶。** 画一张同时含有闭包和 PAP 的对象图，标出回收器必须跳过的每个载荷字。为什么对原始元数据和 `Double` 的位模式只查最低一位会出错？
3. **进阶。** 构造一条回到汇合点的边，让活跃性分析一轮算不完。列出各轮的集合，看它们怎样增长。
4. **挑战。** 让活跃区间互不重叠的值复用根槽。保留并通过现有压力测试，比较修改前后生成的帧有多大。
5. **挑战。** 给回收器设计一个分代版本。先找出回填递归闭包的哪一步需要写屏障，再动手实现。

## 延伸阅读

Cheney 的原文给出了无需递归的复制算法[^cheney1970]。Henderson 讨论如何在编译环境原本不提供精确根信息时显式登记根[^henderson2002]。想把复制式、分代和并发回收放到同一张图景里，可以读《垃圾回收手册》[^gc2023]。也可以回看 Boehm 和 Weiser 的保守式做法，对照两种方案分别要求编译器提供哪些信息[^boehm1988]。

[^henderson2002]: Fergus Henderson（2002）。[Accurate garbage collection in an uncooperative environment](https://doi.org/10.1145/512429.512449)。
[^cheney1970]: C. J. Cheney（1970）。[A nonrecursive list compacting algorithm](https://doi.org/10.1145/362790.362798)。
[^gc2023]: Richard Jones、Antony Hosking、Eliot Moss（2023）。[The Garbage Collection Handbook: The Art of Automatic Memory Management](https://doi.org/10.1201/9781003276142)。
[^boehm1988]: Hans-Juergen Boehm、Mark Weiser（1988）。[Garbage collection in an uncooperative environment](https://doi.org/10.1002/spe.4380180902)。

---

[← 第 23 章　优化](ch23-optimization.md) · [中文目录](../README.md) · [第 25 章　接下来可以做什么 →](ch25-future-work.md)
