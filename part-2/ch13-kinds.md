# 第 13 章　种类与类型构造器

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch13-kinds.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch13_kinds)

> **新增文件**
>
> - `src/kinds.rs`
>
> **改动文件**
>
> - `src/types.rs`
> - `src/typecheck.rs`
> - `src/core_lint.rs`
> - `src/ast.rs`
> - `src/core.rs`
> - `src/core_type.rs`
> - `src/data.rs`
> - `src/elaborate.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/parse.rs`
> - `src/rename.rs`

## `Int` 是类型，单独一个 `Maybe` 又是什么？

`Int` 描述一类值，`Maybe Int` 也一样。但单独一个 `Maybe` 还少点东西，得先给它一个类型，它才能描述值。之前的编译器会数每种数据类型要几个参数，这就足够应付当时的程序了。可如果某个参数本身也是像 `Maybe` 这样还要吃参数的东西呢？

假设构造器有一个类型为 `f a` 的字段。在这里，`f` 必须能接收类型参数，而 `a` 正好填到那个位置。怎样把这两份不同的工作写清楚？用**种类**（kind）。看起来很眼熟，写法里有箭头，也有 `*`，形式比普通类型还简单。

推断种类的办法也很眼熟。之前推断类型靠的是列方程、解方程，现在可以再用一遍。

## 现在能写什么

```fun
data Wrap f a = Wrap (f a)

unwrap :: Wrap Maybe Int -> Maybe Int
unwrap (Wrap x) = x

main = unwrap (Wrap (Just 42))
```

`Wrap` 的种类是 `(* -> *) -> * -> *`。先看个类型别名的例子，下一节我们再一步步推出这些箭头。

```fun
type Pair a = (a,a)

type Count = Int

addPair :: Pair Count -> Count
addPair (x,y) = x + y

main = addPair (20,22)
```

类型别名（type synonym）只给已有类型起个新名字，不会引入数据构造器，也不会增加运行时表示。在 Fun 中，使用别名时必须一次给齐参数。哪怕在同样的位置，某个数据类型构造器少给几个参数仍然具有正确的种类，别名也不能这样用。别名定义不能成环，即使整组别名从未被使用，也会被拒绝。种类不匹配报 `E0301`，别名参数没给全报 `E0302`，别名循环报 `E0303`。如果所在位置的种类允许，`[]`、`(,)`、`(,,)` 和 `(->)` 这些类型构造器都可以不带参数单独出现。

## 背后的原理

读 `Maybe :: * -> *`，可以借用读函数类型的习惯。箭头左边的 `*` 表示它接收一个能描述值的类型，右边的 `*` 表示它又产出一个能描述值的类型。把 `Int :: *` 传给它，就得到 `Maybe Int :: *`。下面的种类规则记的正是这个步骤。如果 `f` 等待种类为 $\kappa$ 的实参，而 `a` 正好具有这个种类，那么应用 `f a` 的种类就是箭头右边的 $\sigma$。

$$
\frac{\Gamma \vdash f : \kappa \to \sigma \quad \Gamma \vdash a : \kappa}
{\Gamma \vdash f\ a : \sigma}\;\text{（应用）}
$$

### 顺着一个字段推到底

看 `data Wrap f a = Wrap (f a)`。构造器的字段里要装一个值，所以字段的类型 `f a` 必须具有种类 `*`。先把 `a` 的未知种类记作 `k`，那么想让应用 `f a` 的结果具有种类 `*`，`f` 就必须具有种类 `k -> *`。光看这个字段，到这里就推不动了。等整组声明检查完毕，如果 `k` 仍然未知，再把它默认成 `*`。

回头从左到右数这条声明的参数。`Wrap` 先接收种类为 `* -> *` 的 `f`，再接收种类为 `*` 的 `a`，最后产生一个可以描述值的类型。于是得到 `(* -> *) -> * -> *`。`Wrap Maybe Int` 的两个位置都填得恰到好处。`Wrap Int Int` 却在第一个位置就出错了，因为 `Int` 本身不能再接收一个类型参数。

大部分工作都可以交给熟悉的合一算法，这次箭头两头连的是种类。遇到互相递归的声明，先为各个类型名分配临时种类，把依赖分量里的声明全部检查完，再给剩下的未知数设默认值。太早选定默认值，就可能错过这个分量里另一个字段提供的约束。

函数的参数类型和结果类型都必须具有种类 `*`，构造器的字段类型也是一样。一个值可以具有 `Maybe Int` 类型，却不可能直接具有还没传类型参数的 `Maybe` 类型。这里对类型与种类的区分，遵循 Pierce 讲解高阶类型算子时的思路[^pierce2002]。Fun 能推断种类，但没有引入类型层的 lambda，也没有添加让用户显式标注种类的语法。

展开类型别名时，遵守《Haskell 语言报告》要求传足实参的约定[^haskell2010]。展开顺序也有讲究。先展开实参，再进入别名的定义，因此 `Id (Id Int)` 只是合法地嵌套用了两次同一个别名。为了查出真正的循环，检查器用栈记录的是**正在展开哪些定义**。

## 动手实现

类型给项分类，种类则给类型构造器分类。实现时，把种类的合一、出现检查（occurs check）和按依赖排序结合起来，就能推断这些种类。

```rust
pub enum Kind {
    Star,
    Arrow(Box<Kind>, Box<Kind>),
    Meta(u32),
}
```

[源码 · kinds.rs 第 13～17 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch13_kinds/src/kinds.rs#L13-L17)

`Store::unify` 会顺着已解出的变量找到它的解，逐层比较箭头两边的结构，还会拒绝让一个元变量出现在它自己的解里。等一个声明分量的所有字段都检查过，`finish` 才给仍未解出的单元填上默认值。

```rust
pub fn unify(
    &mut self,
    a: &Kind,
    b: &Kind,
    span: Span,
) -> Result<(), Diagnostic> {
    let (a, b) = (self.zonk(a), self.zonk(b));
    match (&a, &b) {
        _ if a == b => Ok(()),
        (Kind::Meta(n), k) | (k, Kind::Meta(n))
            if !self.occurs(*n, k) =>
        {
            self.solutions[*n as usize] = Some(k.clone());
            Ok(())
        }
        (Kind::Arrow(a, b), Kind::Arrow(c, d)) => {
            self.unify(a, c, span)?;
            self.unify(b, d, span)
        }
        _ => Err(Diagnostic::error(
            ErrorCode::Kind,
            format!("cannot unify kind {a} with {b}"),
            span,
        )),
    }
}
```

[源码 · kinds.rs 第 86～111 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch13_kinds/src/kinds.rs#L86-L111)

处理声明时，先登记所有类型的名字，再复用第 9 章的 Tarjan 算法划分强连通分量。检查类型签名时，面对的是这张已经完整的表，签名里的隐式类型变量也能据此推断出种类。接下来，只读的数据类型环境负责检查别名的用法，并将展开后的类型语法转成推断器内部的类型表示。

现在，存储类型的结构会为每个元变量记下种类。实例化类型方案时，既给量化变量换上新标识，也一并复制它们的种类。比如要把 `f a` 与 `Maybe Int` 合一，就把两边最外层的应用各拆成两块，分别解出 `f = Maybe` 和 `a = Int`。两个位置各按自己的种类检查，无须一律限定为值类型。这里尤其要注意 `f` 所在的位置，它完全可以填入种类为 `* -> *` 的 `Maybe`。

Core 按各个类型绑定的唯一标识保存一张种类表。Lint 会独立对照这张表，检查应用、构造器实参和函数的操作数。如果一个内部类型变量没有出现在任何外层的类型方案里，就给它选一个种类正确、没有自由变量的类型作见证。这些编译器专用的类型常量在源语言中没有写法，也没有任何值，到类型擦除时就消失了。第 10 章还能统一拿 `Int` 作见证，现在所需的种类可能是 `* -> *`，光靠 `Int` 就填不上这个位置了。

## 跑起来

```text
$ cd crates/ch13_kinds
$ cargo run -q -- run tests/programs/kinds/wrapper.fun --lint
Just 42
$ cargo run -q -- dump --pass=types tests/programs/kinds/wrapper.fun
main : Maybe Int
$ cargo run -q -- check tests/programs/kinds/synonym-self.fun
E0303
```

最后一条命令会因为别名定义成环而被拒绝。还可以依次试两种写法，先给一个数据类型构造器只传一部分类型参数，再对一个类型别名做同样的事。前者可以恰好用在要求更高种类的位置，后者则会因别名参数没有给齐而被拒绝。表面上都“少了一个参数”，两者遵循的规则却各有不同。

## 练习

1. **热身。** 推断 `data Compose f g a = Compose (f (g a))` 中 `Compose` 的种类。
2. **进阶。** 为什么 `data Bad f = Bad (f f)` 过不了出现检查？
3. **进阶。** 选一组互相依赖的声明，逐步跟踪其中的种类推断。
4. **挑战。** 找一个项，让编译器在把它翻译成 Core 时需要种类为 `* -> *` 的内部见证。导出 Core，看看对应的类型应用长什么样。

## 延伸阅读

- 想进一步了解类型算子，可以读 Pierce 对这个主题的讲解[^pierce2002]。
- 数据类型和类型别名的规则见《Haskell 语言报告》[^haskell2010]。
- 关于高阶类型算子与 System F-omega（System Fω）的关系，也可以参看 Pierce[^pierce2002]。

[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。
[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。

---

[← 第 12 章　记录](ch12-records.md) · [中文目录](../README.md) · [第 14 章　类型类 →](ch14-type-classes.md)
