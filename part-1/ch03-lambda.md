# 第 3 章　λ 演算

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch03-lambda.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch03_lambda)

> **新增文件**
>
> - `src/names.rs`
> - `src/normal.rs`
>
> **改动文件**
>
> - `src/eval.rs`
> - `src/ast.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`

## 从给数起名，到给计算起名

上一章的计算器已经能给一个数起名字。如果也能给一段计算起名字，每次换个输入再算一遍呢？这就有了函数。光是加入函数，语言能做的事就多得出人意料。只用两种写法，一种定义匿名函数（lambda），一种把函数用在实参上（函数应用），就能写出纯无类型 λ 演算里的所有项。连数据都能用函数表示，后面会看到 Church 编码是怎么做的[^pierce2002]。

不过，写出函数只是开始。如果里外两个参数都叫 `x`，函数体里那个 `x` 究竟指谁？调用函数时，又该不该先把实参算出来？这两件事，我们亲手实现一遍替换和闭包就能看清楚。如果直接把它们交给 Rust，看到的就只是 Rust 替我们作出的选择。

## 现在这门语言能写什么

一个程序依然是一个表达式。每个 lambda 只接收一个参数，连续的函数应用向左结合。`if` 根据布尔值选择分支，比较运算拿两个整数作比较。现在还不用写类型标注。

```fun
let twice = \f -> \x -> f (f x) in twice (\x -> x + 1) 0
```

这里的 `twice` 接收函数 `f`，返回一个把 `f` 连用两次的函数。因此，给它“加一”和初值 `0`，就会算出 `2`。

闭包会记住创建它时，周围的名字各自指向什么。看 `let x = 1 in let f = \y -> x + y in let x = 100 in f 2`。定义 `f` 时，它看到的 `x` 是 `1`，所以 `f 2` 得 `3`。这叫**词法作用域**。倘若调用时才去调用者那里找 `x`，就会取到 `100`，这叫**动态作用域**。

## 原理

### 无类型 λ 演算的语法

纯无类型 λ 演算只有三种表达式，也叫三种“项”。可以写一个变量，把一个函数应用到一个实参上，或者用一个参数和一段函数体构造函数。最后一种叫**抽象**。下面的竖线都读成“或者”。

$$
\begin{aligned}
e & = x & \quad & \text{变量} \\
  & \mid e_1\;e_2 & \quad & \text{函数应用} \\
  & \mid \lambda x.\;e & \quad & \text{函数抽象}
\end{aligned}
$$

纯演算的语法到这里就写完了！为了做实验时不用把每个数字和布尔值都编码成函数，我们的小语言还提供了几种现成的写法。

$$
\begin{aligned}
e & = \cdots & \quad & \text{以上三种形式} \\
  & \mid n & \quad & \text{整数字面量} \\
  & \mid \text{True}\mid\text{False} & \quad & \text{布尔字面量} \\
  & \mid \text{let}\;x = e_1\;\text{in}\;e_2 & \quad & \text{局部绑定} \\
  & \mid \text{if}\;e_1\;\text{then}\;e_2\;\text{else}\;e_3 & \quad & \text{条件表达式} \\
  & \mid -e & \quad & \text{整数取负} \\
  & \mid e_1\;\mathit{op}\;e_2 & \quad & \text{二元运算}
\end{aligned}
$$

这里的 $n$ 是整数，$\mathit{op}$ 可以是 `+`、`-`、`*`、`/`、`==`、`<` 或 `<=`。括号只管分组，语法树上不单独为括号建节点。解析器为了在报错后继续往下读，还会放入一个 `Error` 占位符。成功通过解析的程序里不会有它。本章说“无类型”，意思是目前没有静态类型检查器。即使一段代码在语法上拼得起来，运行时也可能没有意义。

### 归约与求值

调用一个 lambda，可以理解成把函数体里的形参换成传进来的实参。这一步叫 **β 归约**（beta reduction）。把 `\x -> x + 1` 应用到 `41` 上，用 `41` 换掉 `x`，就得到 `41 + 1`。一旦有重名，事情就没这么简单了。我们只替换形参在函数体中**自由出现**的位置，也就是没有被更内层的同名绑定接管的位置。如果内层绑定会把实参里的自由变量也绑住，就得先给内层参数换个名字。下一节就来看一次这样的失误[^pierce2002]。

一个表达式里可能有好几处都能归约。**求值顺序**决定先动哪一处。**正规序**（normal order）先选最外层中最靠左的可归约表达式。本章的完整归约器还会继续深入 lambda 的函数体。**按值调用**（call-by-value）先求出函数和实参的值，之后才进入函数体。遇到 lambda 本身，就已经有了一个值。

如果一个函数根本用不到它的实参，而这个实参一算起来就没完，两种策略的差别马上就显出来了。正规序可以跳过这个实参，按值调用却会一直忙着算它。用 Y 组合子算阶乘就是个例子。正规序能算出答案，本章的严格求值器则会耗尽“燃料”。燃料就是给实验设定的步数预算。用光预算只说明这次实验走到了上限，还不足以证明程序永远不会停。

### 为什么要用 de Bruijn 索引？

名字方便人读代码，但决定绑定关系的是它出现在哪一层。`\x -> x` 和 `\answer -> answer` 表示同一个函数。这样的项叫 **α 等价**（alpha-equivalent）。给形参换个名，并把指向它的地方一起改名，含义就保持不变。可如果直接比较语法树，会看到两组不同的字符串。编译器改写带名字的项时，必须能分清一次改名有没有改变某个变量所指的绑定。

拿下面的表达式做一次归约。它含有自由变量，因此叫**开放项**。自由的是末尾当实参用的那个 `y`，它的值要从表达式外部找。

```text
(\x -> \y -> x) y
```

如果只做文本替换，把 `x` 统统换成 `y`，就会得到 `\y -> y`。这已经变成了恒等函数。原来的意思却是，无论内层函数收到什么，它都该返回从外面传来的那个 `y`。先给内层形参改名，再替换，就能保住这层意思。

```text
(\x -> \z -> x) y
→ \z -> y
```

这里发生的错误叫**变量捕获**。替换函数得设法避开它。另一种记录绑定的方法是用 **de Bruijn 索引**。每遇到一个受绑定的变量，就记下“从这里到绑定它的那一层，要往外跳过几个绑定”。离它最近的绑定编号为 `0`，再往外一层是 `1`。只数沿途的绑定，函数应用节点和代码行数都不影响编号。

下面用 `λ. 函数体` 表示不写形参名的 lambda。里面的数字表示变量索引，不是整数字面量。我们借这套记号讲原理，本章的解析器接受的仍是带名字的源代码。

| 带名字的项 | 换成索引 | 怎么读 |
| --- | --- | --- |
| `\x -> x` | `λ. 0` | 返回最近那一层的形参。 |
| `\x -> \y -> x` | `λ. λ. 1` | 跳过 `y` 这一层，取外面的 `x`。 |
| `\x -> \y -> y` | `λ. λ. 0` | 返回 `y`。 |
| `\f -> \x -> f x` | `λ. λ. 1 0` | 用外层形参当函数，内层形参当实参。 |
| `\x -> \x -> x` | `λ. λ. 0` | 里层的 `x` 遮住了外层的 `x`。 |

再把 `\f -> \x -> f (f x)` 从外向内走一遍。进入第一个 lambda，把 `f` 压入作用域栈。进入第二个，再压入 `x`。这时从最近的绑定往外看，先是 `x`，再是 `f`。

```text
scope, nearest first:  [x, f]
index:                  0  1

f (f x)                 1 (1 0)
\f -> \x -> f (f x)     λ. λ. 1 (1 0)
```

两个 `f` 的索引都是 `1`。进入 `(f x)` 并没有引入新的绑定，所以索引不用调整。就算把两个形参改名为 `g` 和 `seed`，也会得到完全一样的索引树。这样一来，判断两个项是否 α 等价，就成了普通的树结构比较[^pierce2002]。这尤其方便测试打印器。把代码解析、打印、再解析，即使打印器换了变量名，也应该保留同样的绑定关系。

开放项里的自由变量没有绑定可数。我们的 `Term` 类型仍用 `Free(name)` 记下它的名字，另用 `Bound(index)` 表示受绑定的变量。因此，`\x -> x y` 会变成 `Lam(App(Bound(0), Free("y")))`。非递归 `let` 的绑定只在 `in` 后面的主体里生效。看 `let x = x in x`，等号右侧的 `x` 是自由的，最后那个 `x` 才受这个 `let` 绑定。

```text
Let(Free("x"), Bound(0))
```

索引省掉了改名的麻烦，但把一个项搬进或搬出一层绑定时，仍要当心。在 `\a -> (\x -> \b -> x) a` 中，实参 `a` 在原地的索引是 `0`。归约之后得到 `\a -> \b -> a`。还是同一个 `a`，却隔了一层 `b`，索引得改成 `1`。如果照搬原来的 `0`，取到的就成了 `b`。因此，直接在索引树上做替换的求值器需要实现**移位**（shifting）。项跨越作用域时，要调整其中指向外部绑定的引用。在项自身内部就已绑定的引用保持原样。

本章用索引来转换和比较项。正规序归约器仍然在带名字的树上做替换，严格求值器则使用闭包，这样每个实现只需讲清楚一个问题。后面 Fun 的编译阶段还会给绑定分配唯一名字，让打印出来的中间结果更好读。无名项能帮我们看懂绑定关系，编译器的中间表示（IR）也可以根据需要选择其他表示法。

### 用 S、K、I 把绑定改写成函数应用

一个 lambda 项如果没有自由变量，就叫**组合子**（combinator）。把它单独拿出来也能用，因为它用不到周围的绑定。只要三个很小的组合子，就能表达纯无类型 λ 演算中的任何函数。

```text
I = \x -> x
K = \x -> \y -> x
S = \f -> \g -> \x -> f x (g x)
```

| 归约规则 | 作用 |
| --- | --- |
| `I a → a` | 把实参原样返回。 |
| `K a b → a` | 收下两个实参，保留第一个。 |
| `S f g a → f a (g a)` | 把同一个 `a` 分给左右两边，再把左边的结果应用到右边的结果上。 |

大写字母是数学记号。在可执行的测试里，我们用普通局部绑定 `s`、`k`、`i` 来定义它们。函数应用向左结合，所以 `S f g a` 是 `((S f) g) a`。用表里的某条规则归约时，要凑齐该规则列出的所有实参。

`I` 和 `K` 一眼就能看懂。`S` 解决的是这样一个问题。假设一次应用的左右两边都需要同一个输入 `a`，怎么把它同时送到两边？给定 `f`、`g` 和 `a`，分别形成 `f a` 和 `g a`，再用前者作函数、后者作实参。正因如此，`f a (g a)` 里的括号不能随便挪。

试试看，`f` 和 `g` 都取 `K` 会怎样？

```text
S K K a
→ K a (K a)        apply the S rule
→ a                apply the K rule
```

原来 `S K K` 的行为和 `I` 一样！只用两个看起来跟恒等函数不沾边的操作，就能搭出恒等函数。因此只留 S 和 K 也够用，保留 I 可以让式子好读些。可以跑一下本章的 `s k k (i 42)`，跟着看它怎样一步步归约。

为什么偏偏这几条规则能消去 lambda？设函数体是 `t`，用 `[x]t` 代表一个组合子表达式，意思是“接收 `x`，产生 `t`”。假设更内层的 lambda 都已翻译完，这一步只需考虑三种情况。

```text
[x]x       = I
[x]t       = K t                  when x is not free in t
[x](p q)   = S ([x]p) ([x]q)       otherwise
```

如果函数体就是形参本身，用 `I` 原样返回。如果 `t` 里没有自由出现的 `x`，说明这个实参用不到，用 `K t` 忽略它。剩下的情况是一次应用 `p q`，用 `S` 把实参传给翻译好的左右两个子项。这个过程叫**括号抽象**（bracket abstraction）。从最内层的 lambda 开始，一层层往外做，一个原本没有自由变量的纯 lambda 项，最终只会剩下组合子和应用。

例如，把 `f` 当成自由变量，试着消掉 `\x -> f x` 对 `x` 的绑定。

```text
[x](f x)
= S ([x]f) ([x]x)
= S (K f) I
```

给翻译结果传一个实参，就能验算。

```text
S (K f) I a
→ (K f a) (I a)
→ f (I a)
→ f a
```

`x` 消失了，自由变量 `f` 还留着，这正是我们想要的。de Bruijn 索引保留绑定，把变量到绑定的距离编码为数字。括号抽象则消掉绑定，把原本由绑定完成的工作交给函数应用。给同一段计算换一种表示，刚才的过程就算是一次小小的编译。

不过，本章开头问过的求值顺序问题，在这里仍然绕不开。假设 `omega` 会永远算下去，正规序照样能把 `K 42 omega` 归约到 `42`。严格求值器在进入 `K` 的函数体前就会去算 `omega`，永远等不到丢掉那个参数的机会。这几条组合子等式描述的是纯演算中的改写规则。严格求值的编译器如果据此随意删掉计算，就可能改变程序的行为。本章借 SKI 来理解绑定和归约，后面 Fun 的后端会用闭包。这样也避开了直接翻译成组合子时可能膨胀得很大的应用树。

## 实现

`ast.rs` 增加了 lambda、函数应用、布尔值和条件表达式。打印器会看当前位置要求多高的优先级，只在省略括号会改变语法树时才补括号。解析器把函数应用的优先级设得比算术运算高。

```text
Application: Expr = {
    <a:Application> <b:Atom> => {
        let (start, end) = (a.span.start, b.span.end);
        Expr::new(Kind::App(Box::new(a), Box::new(b)), start, end)
    },
    Atom,
};
```

[源码 · parser.lalrpop 第 34～40 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch03_lambda/src/parser.lalrpop#L34-L40)

`names.rs` 负责找出自由变量，以及在替换时避开变量捕获。下面的辅助函数专门处理引入绑定的地方，递归替换再分别处理各种形式的项。挑选新名字时，原函数体和替换项里用过的名字都得避开，无论那些名字是自由的还是受绑定的。这样新名字才不会跟里层 lambda 的参数撞名。

```rust
fn binder(
    name: &str,
    body: &Expr,
    target: &str,
    value: &Expr,
) -> (String, Expr) {
    if name == target {
        return (name.to_owned(), body.clone());
    }
    let mut name = name.to_owned();
    let mut body = body.clone();
    if free(value).contains(&name) && free(&body).contains(target) {
        let mut used = BTreeSet::from([target.to_owned()]);
        all_names(&body, &mut used);
        all_names(value, &mut used);
        let mut fresh = format!("{name}'");
        while used.contains(&fresh) {
            fresh.push('\'');
        }
        let variable = Expr::new(Kind::Var(fresh.clone()), 0, 0);
        body = substitute(&body, &name, &variable);
        name = fresh;
    }
    (name, substitute(&body, target, value))
}
```

[源码 · names.rs 第 65～89 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch03_lambda/src/names.rs#L65-L89)

同一个模块还负责把带名字的项转成索引。栈从底到顶按从外到内的顺序存放作用域里的名字。处理 `let` 时，先转换等号右边，再把新绑定压栈，因为这个名字只在 `in` 后面的主体中可见。

```rust
/// Bound names are distances to binders; free names remain explicit.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Term {
    Bound(usize),
    Free(String),
    Int(i64),
    Bool(bool),
    Lam(Box<Term>),
    App(Box<Term>, Box<Term>),
    Let(Box<Term>, Box<Term>),
    If(Box<Term>, Box<Term>, Box<Term>),
    Neg(Box<Term>),
    Binary(Op, Box<Term>, Box<Term>),
    Error,
}

/// Convert without substitution; alpha-equivalent expressions have equal terms.
pub fn debruijn(expr: &Expr, bound: &mut Vec<String>) -> Term {
    let mut go = |e: &Expr| Box::new(debruijn(e, bound));
    match &expr.kind {
        Kind::Var(n) => bound
            .iter()
            .rev()
            .position(|x| x == n)
            .map_or_else(|| Term::Free(n.clone()), Term::Bound),
        Kind::Lam(n, a) => {
            bound.push(n.clone());
            let body = debruijn(a, bound);
            bound.pop();
            Term::Lam(Box::new(body))
        }
        Kind::Let(n, a, b) => {
            let value = go(a);
            bound.push(n.clone());
            let body = debruijn(b, bound);
            bound.pop();
            Term::Let(value, Box::new(body))
        }
        Kind::App(a, b) => Term::App(go(a), go(b)),
        Kind::Binary(op, a, b) => Term::Binary(*op, go(a), go(b)),
        Kind::If(c, a, b) => Term::If(go(c), go(a), go(b)),
        Kind::Neg(a) => Term::Neg(go(a)),
        Kind::Int(n) => Term::Int(*n),
        Kind::Bool(b) => Term::Bool(*b),
        Kind::Error => Term::Error,
    }
}
```

[源码 · names.rs 第 115～161 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch03_lambda/src/names.rs#L115-L161)

`normal.rs` 实现单步归约。遇到函数应用时，如果左边已经是 lambda，就直接把实参代进函数体，无需先求实参的值。如果左边还不是 lambda，就先尝试把左边归约一步。只有更靠前的地方都无法归约时，才轮到右边的实参。

```rust
Kind::App(function, argument) => {
    if let Kind::Lam(name, body) = &function.kind {
        return Ok(Some(substitute(body, name, argument)));
    }
    if let Some(next) = step(function)? {
        Kind::App(Box::new(next), argument.clone())
    } else if let Some(next) = step(argument)? {
        Kind::App(function.clone(), Box::new(next))
    } else {
        return Ok(None);
    }
}
```

[源码 · normal.rs 第 16～27 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch03_lambda/src/normal.rs#L16-L27)

`reduce` 反复调用 `step`，限制步数，并通过回调把每一步得到的项显示出来。它也能检查开放项，甚至能显示已经无法再归约的应用。这里我们要观察的是项怎样改写，后续 Fun 真正执行程序时会使用另一套引擎。

`eval.rs` 走的是闭包这条路。它把 lambda 创建时的环境存进闭包。

```rust
/// Lexical bindings retained by a closure.
pub type Env = Vec<(String, Value)>;

#[derive(Clone, Debug)]
/// Runtime integers, booleans, and code paired with its lexical environment.
pub enum Value {
    Int(i64),
    Bool(bool),
    Closure(String, Rc<Expr>, Rc<Env>),
}
```

[源码 · eval.rs 第 9～18 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch03_lambda/src/eval.rs#L9-L18)

看看按值调用怎样处理函数应用，代码本身就写明了先后顺序。

```rust
Kind::App(function, argument) => {
    let function = evaluate(function, env, fuel)?;
    let argument = evaluate(argument, env, fuel)?;
    if let Value::Closure(name, body, captured) = function {
        let mut local = (*captured).clone();
        local.push((name, argument));
        evaluate(&body, &local, fuel)
    } else {
        Err("expected a function".into())
    }
}
```

[源码 · eval.rs 第 81～91 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch03_lambda/src/eval.rs#L81-L91)

执行函数体时，以闭包保存的环境为起点，再添上形参到这次实参值的绑定。求值器每递归访问一次都会消耗燃料。命令行工具给这些递归遍历语法树的实现准备了较大的宿主调用栈。第 5 章我们会把下一步做什么显式存进一台抽象机，免去对宿主递归的依赖。增加燃料或调大栈，都不能起到这个作用。

## 动手运行

```text
$ cd crates/ch03_lambda
$ cargo run -q -- run tests/programs/church.fun
2
$ cargo run -q -- dump --pass=debruijn tests/programs/identity.fun
App(Lam(Bound(0)), Int(42))
$ cargo run -q -- repl
Lambda: :quit, :parse EXPR, :trace EXPR.
lambda> :trace (\x -> x) 42
(\x -> x) 42
42
lambda> :quit
```

用 `run --engine=normal` 选择正规序。尝试可能算不完的项时，用 `--fuel=N` 给实验定个上限。两台引擎数的操作不同，即使燃料设成一样，也不一定在同一步停下来。可以写一个忽略实参的函数，给它传一段一执行就会报错的表达式，看看两个引擎为什么会给出不同的结果。

## 练习

1. **基础。** 手算 `\x -> \y -> y x` 的 de Bruijn 索引表示。
2. **进阶。** 在 `\y -> \y' -> x y` 中，把 `x` 替换成自由变量 `y`。检查替换结果里有哪些自由变量。
3. **进阶。** 用括号抽象翻译 `\x -> x x`，再把翻译结果应用到 `I` 上。对照一下，原 lambda 项应用到 `I` 的归约过程是什么样？
4. **进阶。** 逐步归约 `K I a b`，解释哪个实参被丢掉了。
5. **挑战。** 构造一个适用于按值调用的不动点组合子，把它的归约过程和 Y 组合子作比较。

## 延伸阅读

Pierce 讲解了替换、归约和无名项[^pierce2002]。Felleisen 和 Friedman 介绍了抽象机的研究背景，第 5 章我们还会回到这个话题[^felleisen1986]。

[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。
[^felleisen1986]: Matthias Felleisen、Daniel P. Friedman（1986）。[Control Operators, the SECD-Machine, and the Lambda-Calculus](https://hdl.handle.net/2022/33996)。

---

[← 第 2 章　语法分析](ch02-parsing.md) · [中文目录](../README.md) · [第 4 章　简单类型 λ 演算 →](ch04-stlc.md)
