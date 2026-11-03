# 第 14 章　类型类

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch14-type-classes.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch14_classes)

> **新增文件**
>
> - `src/classes.rs`
> - `src/evidence.rs`
> - `src/elaborate/dictionaries.rs`
> - `src/primitives.rs`
>
> **改动文件**
>
> - `src/typecheck.rs`
> - `src/elaborate.rs`
> - `src/types.rs`
> - `src/ast.rs`
> - `src/core.rs`
> - `src/core_lint.rs`
> - `src/core_type.rs`
> - `src/data.rs`
> - `src/desugar.rs`
> - `src/eval.rs`
> - `src/fixity.rs`
> - `src/kinds.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/parse.rs`
> - `src/rename.rs`
> - `src/renamed.rs`
> - `src/syn.rs`

## 比较两个列表，谁来比较里面的元素？

要判断两个列表相不相等，就得一对一对地比较里面的元素。整数有整数的比法，字符串有字符串的比法。我们当然可以让列表比较函数多收一个参数，告诉它“用这个函数比较元素”。可一旦很多函数都要比较，光是把同样几个操作从一层函数传到下一层，就得写不少代码。

用**类型类**（type class），这件事可以交给编译器。我们在类里约定有哪些操作，再为不同的类型写**实例**（instance），说明这些操作具体怎么做。每遇到一次调用，编译器就找出这里要用的实现，把它作为参数传过去。

这套机制最后可以落到我们已有的工具上。把一个类的各个操作放进同一条记录，就得到了一本**字典**（dictionary），里面的字段装着函数。只要编译器把原来省略的字典参数补齐，现成的 Core 解释器就能运行翻译好的程序。这一章就来跟踪这个过程。

## 这一章的语言长什么样

先定义一个类，再为整数写一个实例。

```fun
class Value a where
  value :: a -> Int

instance Value Int where
  value x = x + 1

main = value 41
```

[源码 · minimal.fun 第 1～7 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch14_classes/tests/programs/classes/minimal.fun#L1-L7)

类的参数也可以有高阶的种类（kind）。例如，方法 `unwrap :: f a -> a` 里，`f` 要吃进一个类型 `a` 才能组成类型 `f a`，所以类参数 `f` 的种类是 `* -> *`。即使选定了 `f`，方法里的 `a` 仍可以独立地取不同类型。一个类可以要求**超类**（superclass），也可以给方法提供默认实现。写实例方法时，普通函数能用的多条定义方程、守卫和局部 `where` 绑定，在这里也能用。

这一版 Prelude 定义了 `Eq`、`Ord`、`Show`、`Num`、`Fractional`、`Semigroup` 和 `Monoid`。算术运算和比较会被解析为这些类的方法。数字字面量仍各有固定的类型，`1` 是 `Int`，`1.0` 是 `Double`。因此 `1 + 2.0` 会报错，显式写成 `toDouble 1 + 2.0` 才能通过。Fun 没有替你选定默认类型的规则。

```fun
same :: Ord a => a -> a -> Bool
same x y = x == y

main = same [1,2] [1,2]
```

上面的签名只要求 `Ord a`，函数体却能用 `==`，因为 `Eq` 是 `Ord` 的超类。只要能拿出 `Ord a` 的字典，就已经有了 `Eq a` 的依据。找不到实例会报 `E0210`。实例不合法、实例之间有重叠或缺少必要的方法，都会报 `E0211`。还有一种歧义会报 `E0230`。如果某个待泛化的类型变量只出现在约束里，单靠使用这个绑定的方式就没法确定它。检查时要看 `=>` 右侧的整个类型，函数的参数也算在内。因此上例的 `a -> a -> Bool` 已经交代了 `a`。换个例子，`show Nothing` 最后返回字符串，单看结果推不出 `Nothing` 里本来可以装哪种元素，自然选不出元素的 `Show` 实例。给 `Nothing` 标上 `Maybe Int` 就消除了歧义。

`Double` 支持浮点算术、比较、类型转换和数学函数。底层原语的名字只对嵌入的 Prelude 可见。这一章的源语言还没有 IO，运行 `run` 仍会直接打印 `main` 的值。眼下 `++` 只能拼接字符串，等到完整 Prelude 那一章，它才用来拼接列表。此时若想用同一个运算符拼接字符串或列表，已经可以用 `<>`。

## 理论

`Eq a => a -> a -> Bool` 可以这样读，“只要有人告诉我怎么判断两个 `a` 相不相等，我就能接收两个 `a`，给出一个布尔值。” `=>` 左边叫作**约束上下文**。每用一次这种函数，类型推断器就记下一笔“这里需要 `a` 的相等操作”，这笔待满足的要求叫作 **wanted 约束**。

### 亲手传一次相等字典

暂时忘掉自动选择实例的便利，像写普通函数一样把字典传进去。下面用几个示意性的名字来写翻译后的程序。

```text
same d x y = d.eq x y
same intEq 3 3
```

原来的要求 `Eq a` 变成了一个实实在在的参数 `d`。调用处比较的是整数，编译器就填入 `intEq`。程序真正运行到这里时，该用哪个实例早已确定。

换成列表，还得多装配一层。设 `listEq d` 根据元素字典 `d` 造出列表的相等字典。它能直接处理空列表。两个列表都有头元素时，就用 `d.eq` 比较两个头，然后才去比较尾部。于是比较两个整数列表，调用写全就是 `same (listEq intEq) xs ys`。编译器先从待满足的 `Eq [Int]` 找到列表实例，再读出这个实例的前提 `Eq Int`。找到整数实例，就找到了传给 `listEq` 的参数。所谓搜索实例，最后真的会造出一段能运行的程序。

有了这段具体的翻译，再看约束求解器在干什么就容易了。

有些要求可以当场解决，比如为 `Int` 找字典。有些只知道“某个类型变量得满足这个约束”，还不知道它是哪种类型。对绑定做**泛化**时，可以把这类约束写进类型方案（type scheme），留给将来的调用方提供字典。如果涉及外层的类型变量，这项要求就该交给相应的外层作用域处理。Jones 给出了这些操作的一份可执行实现[^jones1999]。

光知道“有这样的实例”还不够，程序运行时得有东西能传。编译器为约束找到的这份依据叫作**证据**（evidence）。例如，要从已知的 `Ord a` 取得 `Eq a`，直接取超类字典字段就好。要从已知的 `Eq a` 取得 `Eq [a]`，就用列表实例构造字典。沿途使用字典变量、选取字段、应用实例构造函数，拼出来的都是普通表达式。运行时只需对它求值。

检查显式签名时，编译器把其中量化的变量换成不能随意合一的刚性类型常量（Skolem 常量），把签名列出的约束当作已知条件（givens）。有签名的函数甚至能在自己的函数体里以不同的类型调用自己。遵照《Haskell 报告》[^haskell2010]，计算**类型推断**的依赖时，可以略去指向有签名绑定的边，因为签名已经给出了类型。但计算**运行时**的依赖必须保留这些边，递归函数仍需要共享递归环境。同一份源码依赖关系，回答的是两个问题。

Fun 仍对绑定做不受限的泛化。比如有约束的值 `comparison = (==)` 会翻译成接收字典的函数，要等字典传进来才对函数体求值。因此源码里看似能共享一次求值的地方，翻译后可能重复求值。Fun 接受字典传递带来的这个代价，表达式的纯性保证了这样做在语义上是可靠的。

## 实现

`classes.rs` 负责查找超类、匹配实例，并判断已有约束能否推出所需约束（entailment）。主推断器仍按层级管理可变的元变量。匹配某个实例头时，代码会临时用一张小的不可变映射表，记下这次匹配得到的对应关系。整台类型检查器仍沿用先前的算法。

```rust
pub enum Evidence {
    Given(usize, Vec<usize>),
    Instance(usize, Vec<Type>, Vec<Evidence>),
}
```

[源码 · classes.rs 第 53～56 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch14_classes/src/classes.rs#L53-L56)

如果证据来自一个已知约束（given），就记下它在当前词法作用域的字典环境里排第几，以及接下来要沿哪些超类字段往里找。如果靠实例得到了证据，就记下选中的实例、传给它的类型实参，以及它的每个前提各自的证据。

```rust
pub fn resolve(
    &self,
    goal: &Predicate,
    givens: &[Predicate],
    span: Span,
) -> Result<Evidence, Diagnostic> {
    for (i, given) in givens.iter().enumerate() {
        if given.ty == goal.ty
            && let Some(path) =
                self.super_path(&given.class, &goal.class)
        {
            return Ok(Evidence::Given(i, path));
        }
    }
    let (i, args, context) =
        self.by_instance(goal).ok_or_else(|| {
            error(
                ErrorCode::NoInstance,
                format!("no instance for {} {}", goal.class, goal.ty),
                span,
            )
        })?;
    let proofs = context
        .iter()
        .map(|p| self.resolve(p, givens, span))
        .collect::<Result<Vec<_>, _>>()?;
    Ok(Evidence::Instance(i, args, proofs))
}
```

[源码 · classes.rs 第 160～187 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch14_classes/src/classes.rs#L160-L187)

检查类型类声明和实例时，要查好几件事。超类关系有没有环？方法是不是属于这个类？方法上有没有额外的约束？以名义类型构造器开头的实例头里，类型变量是否互不重复？实例之间是否重叠？超类要求的字典是否找得到？推断种类时，先处理超类依赖，再给仍无约束的类参数种类取默认值。实例上下文只能提及实例头中出现的变量，这样每次递归求解前提时，面对的类型在结构上都会变小。

泛化时仍要先解决字段约束，再解决类型类约束。把已经解出的元变量替换成最终类型（zonking）以后，`evidence.rs` 会回到变量的每个使用位置，趁该处的词法环境还保有已知字典，把类型类约束的证据记下来。随后翻译成 Core 时，就可以直接使用这些检查过的证据。

```rust
pub struct EvVar {
    pub node: u32,
    pub index: usize,
}
```

[源码 · evidence.rs 第 14～17 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch14_classes/src/evidence.rs#L14-L17)

`elaborate/dictionaries.rs` 为每个类声明一个有独立名字的 Core 字典类型。字段先放超类字典，再放方法。只属于方法的量化变量用 System F 多态表示，量词就放在该方法字段的类型内部。源语言本身仍不支持写任意阶（rank-N）的多态签名。每个实例会翻译成一个全局值，它以该实例的类型变量和上下文字典为参数。要取出某个方法，只需对字典的构造器做普通的模式匹配。

```rust
pub(super) fn dictionary_type(&self, class: &str, ty: Type) -> Type {
    let c = &self.constructors[self.dicts[class]];
    instantiate(&c.result, &c.parameters, &[ty])
}
```

[源码 · dictionaries.rs 第 83～86 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch14_classes/src/elaborate/dictionaries.rs#L83-L86)

每个默认方法都有一个具名的全局定义。某个实例没写这个方法，就复制一份默认实现，按该实例的类型专门化，再检查这份方法体。这样即便字典是严格构造的，也能避免“为了造好字典，先得拿到这个尚未造好的字典”的循环。求值前还要按字典引入的依赖关系排好次序。非函数值若得递归地初始化，编译器会拒绝。求值器先装好所有函数闭包，再求各个值。没有约束的顶层值依然只求值一次。

`primitives.rs` 给出各原语固定的类型签名和实现。整数运算仍按 63 位回绕。显示浮点数时，只保留足以让字符串读回同一浮点值的最少数字，在指定范围内用普通小数表示，其余相应使用带符号的指数，特殊值也有明确写法。测试覆盖负零、表示形式切换的数量级边界、Unicode 字符串，以及转换失败的情形。到介绍共用运行时的章节，较复杂的操作会搬进 `fun-rt`。

## 跑起来

```text
$ cd crates/ch14_classes
$ cargo run -q -- run tests/programs/classes/minimal.fun --lint
42
$ cargo run -q -- dump --pass=types tests/programs/classes/principal-type.fun
main : Num a => a -> a -> a
$ cargo run -q -- check tests/programs/classes/mixed-literals.fun
E0201
```

最后一条命令还会标出源码的出错位置。查看 Core 转储（`--dump=core`）能看到以 `$D` 开头的字典类型、以 `$f` 开头的实例全局值，以及显式传入字典的调用。那几份小型字典快照测试关闭了 Prelude，方便看清示例。普通构建会先编译嵌入的 Prelude，再处理用户的作用域。用户代码即便重新定义了 Prelude 里的同名变量，也不会改变 Prelude 内部先前解析好的引用。

一般给表达式标类型，不加外层括号也行。有一个地方容易读错。在 `case` 分支的守卫里，请给类型标注加上括号，以免类型里的箭头和引出分支主体的箭头混在一起。

## 练习

1. **基础。** 查看比较两个 `[Maybe Int]` 时传入了哪些字典参数。
2. **进阶。** 写一个实例，使它的约束上下文不足以满足超类要求。指出缺了哪一份证据。
3. **进阶。** 为什么 `show Nothing` 有歧义，`show (Nothing :: Maybe Int)` 却没有？
4. **挑战。** 找到多态递归的测试样例，比较同一程序在类型推断和运行时分别会被划进哪些依赖分量。
5. **挑战。** 故意破坏一份超类字典在 Core 中的类型。绕过源语言的类型推断，直接写一个 Core lint 回归测试抓住这个错误。

## 延伸阅读

- Jones 的 *Typing Haskell in Haskell* 把类型推断和约束求解写成可以运行的 Haskell 代码[^jones1999]。
- 构造器类以及种类较高的类参数[^jones1993]。
- 《Haskell 报告》里关于类型类和实例的语言定义[^haskell2010]。
- 同一份报告也解释了怎样对绑定分组，以及实例声明要满足哪些限制[^haskell2010]。

[^jones1999]: Mark P. Jones（1999）。[Typing Haskell in Haskell](https://web.cecs.pdx.edu/~mpj/thih/thih-sep1-1999/index.html)。
[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。
[^jones1993]: Mark P. Jones（1993）。[A System of Constructor Classes: Overloading and Implicit Higher-Order Polymorphism](https://doi.org/10.1145/165180.165190)。

---

[← 第 13 章　种类与类型构造器](ch13-kinds.md) · [中文目录](../README.md) · [第 15 章　Monad、do 记法与 IO →](ch15-io.md)
