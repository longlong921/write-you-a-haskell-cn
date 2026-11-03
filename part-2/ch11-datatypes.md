# 第 11 章　数据类型与模式匹配

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch11-datatypes.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch11_data)

> **新增文件**
>
> - `src/data.rs`
> - `src/syn.rs`
> - `src/desugar.rs`
>
> **改动文件**
>
> - `src/typecheck.rs`
> - `src/core.rs`
> - `src/elaborate.rs`
> - `src/ast.rs`
> - `src/core_lint.rs`
> - `src/core_type.rs`
> - `src/eval.rs`
> - `src/fixity.rs`
> - `src/layout.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/parse.rs`
> - `src/rename.rs`
> - `src/renamed.rs`
> - `src/session.rs`
> - `src/types.rs`

## 怎样写出一棵树的两种形态

靠整数和函数，我们已经写出了不少程序。现在试着描述一棵二叉树。它要么是空树，要么有一个节点，里面放一个值和两棵子树。再比如，查找可能一无所获，也可能找到一个结果。**代数数据类型**让我们直接写下这些可能性，用不着找一个特殊数字来代表“这里什么也没有”。

有了这种数据，怎样拆开它？用**模式匹配**。数据有几种形态，程序就可以照着这几种形态逐个处理。接下来整本书都会反复用到这个写法。

这一章先实现最容易理解的匹配办法，按源码里的顺序一行一行试。它把“一次匹配到底是什么意思”交代清楚。到了第 16 章，我们再把同样的匹配编译成适合生成原生代码的决策树。

## 现在能写什么

一条 `data` 声明引入一个由名字区分身份的类型，以及这个类型的构造器。构造器对字段严格求值，也可以像柯里化函数那样作为值传来传去、分次接收参数。

```fun
data Tree a = Empty | Node (Tree a) a (Tree a)

size Empty = 0
size (Node left _ right) = 1 + size left + size right

main = size (Node Empty 42 Empty)
```

把 `size` 和 `Tree` 的定义对着读。遇到 `Empty`，节点数为零。遇到 `Node`，先把左右子树的节点数加起来，再把眼前这个节点算进去。程序几乎就是照着数据结构写出来的。树再复杂一些，这个思路也一样适用。

把一棵树左右翻转，也可以直截了当地写出来。

```fun
data Tree a = Empty | Node (Tree a) a (Tree a)

mirror Empty = Empty
mirror (Node left x right) = Node (mirror right) x (mirror left)

main = mirror (Node Empty 1 (Node Empty 2 Empty))
```

注意，`mirror` 从来不查看 `x` 的内容。它的类型是 `Tree a -> Tree a`，因此树里装什么都行。递归只改变树的形状，节点上原有的值会原样保留。

一个函数可以用多条等式定义。形参位置支持嵌套的构造器模式、字面量、通配符，以及给整个匹配结果再起个名字的 **as 模式**。列表、二元组、三元组和单元值都有专门的写法。竖线后面的守卫（guard）必须具有 `Bool` 类型。一条等式的守卫全都不成立，就接着试下一条。

```fun
select all@(x : xs) | x < 0 = xs
select all = all

main = select [-1,2,3]
```

第一条等式里的 `all@(x : xs)` 同时给整个非空列表取名 `all`，并把表头、表尾分别绑到 `x` 和 `xs`。负数打头才走这条等式，返回表尾。其他情况交给第二条，原样返回。

一条等式的 `where` 块在这整条等式里都可见，连守卫也能用它。`case` 的一个候选分支同样可以有守卫和自己的 `where` 块。lambda 参数也支持模式。还可以写严格、非递归的模式绑定，比如 `let (x,y) = (20,22) in x + y`。这一章的类型仍然是一阶的。数据类型的参数一律具有种类 `*`，因此 `Tree Int` 合法。想把类型参数本身拿来应用，比如写成 `f Int`，就要等第 13 章。

这个版本初步配备了一个 Prelude（预先加载的基础定义）。编译器嵌入的就是下面这份 Fun 源码。

```fun
data Bool = False | True

data Ordering = LT | EQ | GT

data Maybe a = Nothing | Just a

data Either a b = Left a | Right b

data [] a = [] | (:) a [a]

data () = ()

data (,) a b = (,) a b

data (,,) a b c = (,,) a b c
```

[源码 · prelude.fun 第 1～8 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch11_data/prelude.fun#L1-L8)

## 背后的原理

数据类型声明给每个构造器配一个多态类型方案。以 `Node` 为例，它的类型是 `Tree a -> a -> Tree a -> Tree a`，其中 `a` 受全称量词约束。每使用一次构造器，就用全新的元变量实例化它的类型方案，和使用多态函数是同一个道理[^haskell2010][^pierce2002]。

来看看 `Just 42` 和模式 `Just x` 是怎么对上的。待匹配值的类型是 `Maybe Int`。先把构造器 `Just` 实例化为 `a -> Maybe a`，这里的 `a` 是新建的未知数。把构造器的结果 `Maybe a` 与待匹配值的类型 `Maybe Int` 对齐，就知道 `a` 只能是 `Int`。于是这个分支里的 `x` 也有了类型 `Int`。另一个模式 `Nothing` 不引入任何绑定。不过，不管走哪个分支，返回值的类型必须一样。

下面的规则把这个过程记下来。假定构造器的字段类型已经实例化，就按各个字段的类型去检查对应的子模式。判断右端的 $\Delta$ 是子模式带进分支主体的新绑定。

$$
\frac{K : \tau_1 \to \tau_2 \to T \quad \Gamma \vdash p_1 : \tau_1 \Rightarrow \Delta_1 \quad \Gamma \vdash p_2 : \tau_2 \Rightarrow \Delta_2}{\Gamma \vdash K\;p_1\;p_2 : T \Rightarrow \Delta_1,\Delta_2}\;\text{（构造器模式）}
$$

合并起来的模式环境里，同一个名字不能绑定两次。每一行都按同一组待匹配值的类型来检查，每条成功分支右侧的结果类型也要一致。额外的一条要求是守卫的类型必须能和 `Bool` 合一。

Fun 会先求出字段的值，再构造数据。它没有惰性模式，构造器的字段里也不会藏着一段推迟执行的计算。所有分支都匹配失败时，程序报告运行时错误。想提前提醒用户“有一种情况漏写了”或者“这一行根本走不到”，需要另外做分析。第 16 章编译模式匹配时再来处理[^maranget2007]。

## 动手实现

### 先把类型名字登记好，再检查字段

`data.rs` 先为所有数据类型登记名字，再把各个构造器的字段转成内部类型。因此两个数据类型可以互相递归引用。最终生成的只读表里，记着每个类型接收几个参数、每个构造器的类型方案，以及按声明顺序分配的标签。名字重复、字段类型里用了没绑定的变量、类型参数个数不对，这些错误都能赶在表达式推断之前报出来。

重命名器会解析表达式和模式里出现的构造器名，和普通变量分开处理。每一行模式中的变量分别得到全新的标识。接着处理这行的守卫和等式右侧时，作用域里既有这些模式变量，也有这行自己的局部声明。

### 把表层写法翻译到 Syn

`desugar.rs` 在重命名和推断之间插入一个明确的去语法糖阶段，输出的表示叫 Syn。列表和元组都改写成构造器应用。`if` 改写成对 `True` 和 `False` 的两行匹配。中缀的 `&&`、`||` 先改写成条件表达式。注意这与把运算符当值传递时的行为不同。后一种用法会通过 η 展开生成普通函数，调用它们时，参数按值求值。

如果一个函数只有一条等式，形参也全是变量模式，用普通的 lambda 和 let 就够了。其他情况则为每个参数各生成一层 lambda，最里面放一处模式匹配。

```rust
let params: Vec<_> = (0..rows[0].patterns.len())
    .map(|_| self.sess.fresh_id("$match"))
    .collect();
let scruts = params
    .iter()
    .map(|id| self.node(Kind::Var(id.clone()), span))
    .collect();
let mut body = self.node(Kind::Match(scruts, rows), span);
for id in params.into_iter().rev() {
    body = self.node(Kind::Lam(id, Box::new(body)), span);
}
body
```

[源码 · desugar.rs 第 149～160 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch11_data/src/desugar.rs#L149-L160)

严格模式绑定先把右侧求值一次，检查模式是否匹配，把匹配到的各变量值用嵌套的二元组打包起来，再从包里分别取值。重命名右侧时，要暂时隐藏这次新绑定的名字。这样，外层若有同名变量，右侧引用的仍然是外层那个。即使后面的程序完全没用到某个顶层模式绑定，它依然会执行，也仍然可能匹配失败。

### 给构造器模式推断类型

检查器先实例化构造器的类型方案，把箭头链拆成各个字段类型和最终结果类型，再把结果类型与待匹配值的类型合一。接着拿每个字段实例化后的类型去检查相应的子模式。

```rust
PatKind::Con(id, patterns) => {
    let info = self.data.con(id).clone();
    if patterns.len() != info.fields.len() {
        return Err(Diagnostic::error(
            ErrorCode::Mismatch,
            format!(
                "{} expects {} pattern fields",
                id.name,
                info.fields.len()
            ),
            pattern.span,
        ));
    }
    let mut ty = self.instantiate(&info.scheme);
    let mut fields = Vec::new();
    for _ in patterns {
        let Type::Arrow(a, b) = ty else {
            unreachable!("constructor arity")
        };
        fields.push(*a);
        ty = *b;
    }
    self.unify(expected, &ty, &site)?;
    for (pattern, ty) in patterns.iter().zip(fields) {
        self.pattern(env, pattern, &ty)?;
    }
}
```

[源码 · typecheck.rs 第 334～360 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch11_data/src/typecheck.rs#L334-L360)

模式变量以单态类型进入这一行的环境，普通的局部绑定则照旧支持 let 多态。为了之后生成 System F，检查器除了记下实例化信息，还要记录各个表达式的类型，以及模式中新绑定变量的类型。

### 扩展 Core 和 Lint

如果把构造器当值使用，生成 Core 时就通过 η 展开把它包成 lambda，函数体是已收齐全部参数的 `Con` 节点。遇到匹配，用严格的 let 按源码顺序把各个待匹配表达式各求值一次。在第 16 章引入决策树编译器之前，Core 会原样保留模式的嵌套结构。

```rust
/// Nested patterns remain until the decision-tree compiler in Chapter 16.
#[derive(Clone, Debug)]
pub enum Pat {
    Var(Id),
    Wild,
    As(Id, Box<Pat>),
    Con(crate::data::ConId, Vec<Pat>),
    Lit(Lit),
}

/// A guarded row falls through when all its Boolean tests fail.
#[derive(Clone, Debug)]
pub enum RowBody {
    Plain(Box<Expr>),
    Guarded(Vec<(Expr, Expr)>),
}

/// Row-local declarations run after patterns match and are shared by the guards.
#[derive(Clone, Debug)]
pub struct Row {
    pub pats: Vec<Pat>,
    pub locals: Vec<BindGroup>,
    pub body: RowBody,
    pub span: Span,
}

/// Scrutinees name strict preceding lets; the match records its result type and failure site.
#[derive(Clone, Debug)]
pub struct Match {
    pub scruts: Vec<Id>,
    pub rows: Vec<Row>,
    pub result: Type,
    pub span: Span,
}
```

[源码 · core.rs 第 156～189 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch11_data/src/core.rs#L156-L189)

Lint 独立检查构造器的类型实参、字段个数和每个字段的类型。遇到构造器模式时，它使用待匹配值所属类型的实参来实例化构造器声明，再检查各个绑定。守卫必须返回布尔值，每一行也必须产生这次匹配所声明的结果类型。相关测试直接构造有问题的 Core，不靠表层的类型推断器预先过滤输入。

### 解释器怎样一行一行地匹配

构造器的字段从左到右求值。随后匹配器面对的都是已经算好的值。它递归检查构造器和字面量，遇到变量或 as 模式，就把绑定加入一个只属于当前行的环境。

```rust
fn matches_pattern(p: &core::Pat, v: &Value, env: &mut Env) -> bool {
    match p {
        core::Pat::Wild => true,
        core::Pat::Var(id) => {
            env.push((id.clone(), v.clone()));
            true
        }
        core::Pat::As(id, p) => {
            env.push((id.clone(), v.clone()));
            matches_pattern(p, v, env)
        }
        core::Pat::Lit(l) => matches!(v,Value::Lit(actual) if l==actual),
        core::Pat::Con(id, ps) => match v {
            Value::Con(actual, vs)
                if id == actual && ps.len() == vs.len() =>
            {
                ps.iter().zip(vs).all(|(p, v)| matches_pattern(p, v, env))
            }
            _ => false,
        },
    }
}
```

[源码 · eval.rs 第 326～347 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch11_data/src/eval.rs#L326-L347)

模式失败，扔掉这一行的环境，换下一行重来。模式成功，就将这一行的 `where` 绑定组求值一次，再依次试守卫。守卫全部失败，也转到下一行。如果所有行都用完了，解释器会报告 `fun: pattern match failure at FILE:LINE:COL`，并以状态码 1 退出。

打印结果时，解释器考虑构造器应用的优先级，因此嵌套应用显示成 `Just (Left 3)`。列表和元组使用源码里的习惯写法。字符列表打印成 `['a','b']`，原语字符串则保留双引号。

## 跑起来

```text
$ cd crates/ch11_data
$ cargo run -q -- run tests/programs/data/tree.fun --lint
(True,False)
$ cargo run -q -- dump --pass=syn tests/programs/data/bool-data.fun
nonrec { not#9 = (lambda $match#11 (match $match#11 [ (True) -> False ] [ (False) -> True ])); }
nonrec { main#10 = (app (app (,) (app not#9 True)) (app not#9 False)); }
$ cargo test --test data
```

试试让第一行的模式成功、守卫失败，检查程序会不会接着执行下一条等式。还可以写一个未被引用的顶层模式绑定，并故意让它匹配失败。严格求值意味着即使它的名字没人用，匹配也照样发生。一个只会顺顺利利遍历完二叉树的测试，很容易漏掉这两种行为。

## 练习

1. **基础。** 定义一个翻转二叉树的函数，用一棵左右不对称的树测试它。
2. **进阶。** 一个构造器模式少写了字段，另一个函数则在两条等式里返回了不同类型的结果。分别解释这两种类型错误。
3. **进阶。** 写两个守卫，让它们都失败，从而落到后面一条等式。把守卫共用的计算移进 `where`，再导出 Syn 看看。
4. **挑战。** 比较 Fun 的严格模式绑定与 Haskell 的普通模式绑定。构造一个程序，让它在两种语义下的终止行为或报错行为有所不同。
5. **挑战。** 为 Lint 设计一个测试，交换构造器中两个类型不同的字段，同时保持字段总数不变。

## 延伸阅读

《Haskell 语言报告》规定了数据类型声明、模式、守卫和局部作用域的语义[^haskell2010]。构造值与拆解值，在类型理论中分别对应数据的引入与消去。Pierce 讲解了两者之间的联系[^pierce2002]。Maranget 的论文则讨论接下来的两步，怎样发现模式匹配中的遗漏和冗余，以及怎样把匹配编译成决策树[^maranget2007][^maranget2008]。

[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。
[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。
[^maranget2007]: Luc Maranget（2007）。[Warnings for Pattern Matching](https://doi.org/10.1017/S0956796807006223)。
[^maranget2008]: Luc Maranget（2008）。[Compiling Pattern Matching to Good Decision Trees](https://doi.org/10.1145/1411304.1411311)。

---

[← 第 10 章　带类型的 Core 与 Core Lint](ch10-typed-core.md) · [中文目录](../README.md) · [第 12 章　记录 →](ch12-records.md)
