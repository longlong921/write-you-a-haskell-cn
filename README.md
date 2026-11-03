# 手把手写一个 Haskell 风格的编译器

**《Write You a Haskell》中文译本 · 第一部分「基础」与第二部分「前端」**

输入 `1 + 2 * 3`，计算机凭什么知道先算乘法？写下 `\x -> x`，它又凭什么知道这个函数可以接收整数，也可以接收布尔值？Stephen Diehl 从一个计算器讲起，用六章带你亲手做出能回答这些问题的解析器、解释器和类型推断器。

本译本对应原书的 **Rust 实现版**。我们用 Rust 实现一门借鉴 Haskell 的小语言 Fun。第 2～6 章先各做一个独立实验。从第 7 章起，这些想法汇入同一个编译器。读到第 17 章，你就可以用 Fun 写出能读写终端的程序，还能让它替自己的数据类型生成比较和打印的代码。

## 第一部分　基础

| 章 | 中文译文 | 带着这个问题读 |
| ---: | --- | --- |
| 1 | [导读](part-1/ch01-introduction.md) | 怎么把环境跑起来？为什么要给同一个程序准备好几个解释器？ |
| 2 | [语法分析](part-1/ch02-parsing.md) | 两个解析器都算出 `7`，就能断定它们把表达式读成了同一棵树吗？ |
| 3 | [λ 演算](part-1/ch03-lambda.md) | 把 `(\x -> \y -> x) y` 里的实参代进去，为什么不能得到 `\y -> y`？ |
| 4 | [简单类型 λ 演算](part-1/ch04-stlc.md) | 怎样在运行之前就发现程序把一个整数当作函数调用了？ |
| 5 | [求值](part-1/ch05-evaluation.md) | 同样是递归，为什么累加器求和能跑一百万次，而阶乘还得记住一串没做的乘法？ |
| 6 | [Hindley–Milner 类型推断](part-1/ch06-hindley-milner.md) | 没写类型签名，编译器怎么知道同一个 `id` 可以先用于整数，再用于布尔值？ |

## 第二部分　前端

从这里开始，每一章都是 Fun 编译器的一份可运行快照。可以从第 7 章一路跟下来，也可以带着具体问题跳到对应章节，直接对照那一版的源码。

| 章 | 中文译文 | 带着这个问题读 |
| ---: | --- | --- |
| 7 | [设计 Fun](part-2/ch07-fun.md) | 报错时怎么指回写错的那几个字符？同一段源码又是怎样变成可运行的 Core 的？ |
| 8 | [缩进布局和扩展语法](part-2/ch08-layout.md) | 少写的花括号、分号怎么补回来？一串运算符里又该先算哪一个？ |
| 9 | [名称解析](part-2/ch09-renamer.md) | 两个都叫 `x` 的变量怎么分清？相互调用的函数应该按什么顺序检查？ |
| 10 | [带类型的 Core 与 Core Lint](part-2/ch10-typed-core.md) | 同一个 `id` 可以用在整数和布尔值上。编译器如何把这两次选择写进 Core，又怎么检查自己有没有翻错？ |
| 11 | [数据类型与模式匹配](part-2/ch11-datatypes.md) | 匹配 `Just (x:xs)` 时要拆开几层？守卫失败后该接着试哪一行？ |
| 12 | [记录](part-2/ch12-records.md) | 不同的记录里都有 `x`，编译器怎么知道 `p.x` 该取哪个字段？ |
| 13 | [种类与类型构造器](part-2/ch13-kinds.md) | 为什么 `Maybe Int` 能当字段类型，单独一个 `Maybe` 却还差一点？ |
| 14 | [类型类](part-2/ch14-type-classes.md) | 给两个列表做 `==`，比较元素的函数从哪里来？ |
| 15 | [Monad、do 记法与 IO](part-2/ch15-io.md) | 同样一段 `do`，用在 `Maybe` 和 `IO` 上各做了什么？打印动作到底什么时候发生？ |
| 16 | [编译模式匹配](part-2/ch16-pattern-compilation.md) | 怎么把一排模式编成决策树，还能举出一个恰好漏掉的输入？ |
| 17 | [Prelude 与自动派生](part-2/ch17-prelude.md) | 写下 `deriving (Eq, Ord, Show)`，编译器替我们写了哪些代码？ |

## 跟着例子动手

在 GitHub 里直接点上面的章节即可阅读。正文中的 Rust 片段已经展开，终端示例里的 `$` 是提示符。要实际运行程序，请按第 1 章准备环境，在**原书代码仓库**里执行命令。下面固定到本次翻译对应的版本，便于你对照书里的命令、输出和行文。

```sh
git clone https://github.com/sdiehl/write-you-a-haskell.git
cd write-you-a-haskell
git checkout --detach 51c70f804173c9d013187c543a353686b413ad50
cargo run -p ch02_calc -- repl
```

进入 `calc>` 后，分别输入 `1 + 2 * 3` 和 `:parse 1 + 2 * 3`。前者给出值，后者显示表达式的树形结构。输入 `:quit` 退出。

想从前端开始动手，可以在同一份原书代码仓库里试第 7 章的例子。第一条命令算出 `42`，第二条让你看到求值器收到的 Core 程序。

```sh
cd crates/ch07_fun
cargo run -q -- run tests/programs/fun-top.fun
cargo run -q -- dump --pass=core tests/programs/fun-top.fun
```

## 原作与许可

原作 *Write You a Haskell* 由 Stephen Diehl 撰写，版权归原作者所有（© 2014–2026）。本仓库翻译了 Part I「基础」和 Part II「前端」（第 1～17 章），并将编译流程图的标注译成中文。为便于边读边试，译文补充了个别例子的中间步骤，代码片段和终端输出取自对应的上游版本。[英文在线版](https://sdiehl.github.io/write-you-a-haskell/) · [翻译所据源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50)

原书正文、原图及其中文翻译按 CC BY-SA 4.0（署名—相同方式共享）发布。代码示例和编译器源码沿用 MIT 许可证。[正文与图示的授权声明](LICENSE-PROSE) · [CC BY-SA 4.0 条款](https://creativecommons.org/licenses/by-sa/4.0/) · [代码许可证](LICENSE)
