# 第 16 章　编译模式匹配

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch16-pattern-compilation.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch16_match)

> **新增文件**
>
> - `src/match_compile.rs`
> - `src/usefulness.rs`
> - `src/env.rs`
>
> **改动文件**
>
> - `src/core.rs`
> - `src/core_lint.rs`
> - `src/eval.rs`
> - `src/io_lower.rs`
> - `src/lib.rs`
> - `src/main.rs`

## 怎样把一排模式变成机器能执行的判断？

解释器可以从上往下逐条试模式，正好和我们读代码的顺序一致。可要编译成机器码，总得落实到具体指令上，比如检查构造器标签、取出字段、跳到相应分支。要是每条模式都各编各的，同一个值的标签很可能要翻来覆去查好几遍。

不妨把所有模式放进一张表，也就是**模式矩阵**。挑一项值得先做的检查，再根据结果，把剩下的工作分给不同的分支。这样就得到了一棵**决策树**。如果几条路径最后要做同一件事，可以让它们跳到同一个**汇合点**（join point）。例如，几处守卫都检查失败后，可能都该从同一条后续子句继续尝试。

同一张表还能回答两个问题。有没有输入会从所有模式的缝里漏过去？有没有某条子句能接住的所有输入，都早被前面的子句接走了？看清一个值可能长成什么样，既能让匹配高效地运行，又能替写程序的人找出疏忽。

## 这一章的语言长什么样

本章没有新增源语言语法。即便换了一种执行模式匹配的方式，原有程序也要算出同样的结果。下面这个守卫就是很好的试金石。

```fun
f (Just x) | x < 0 = 0
f (Just x) = x
f Nothing = 99

main = print (f (Just 42))
```

[源码 · guard-fallback.fun 第 1～4 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch16_match/tests/programs/match/guard-fallback.fun#L1-L4)

把 `Just 42` 交给第一条子句，`Just x` 的确匹配上了，但接下来 `42 < 0` 为假，于是要继续试第二条子句，返回 `42`。此时如果报匹配失败，或者直接跳到 `Nothing` 那条，都改变了程序的含义。每条定义方程自己的局部 `where` 绑定，在这条方程的守卫和右侧表达式里仍然可见。

现在，检查器还能指出匹配漏掉了什么。`W0001` 给出没有被覆盖的输入样例。`W0002` 指出一条永远走不到的子句，因为在它之前已经把可能的输入接完了。两者都是警告，程序仍然可以运行。运行时如果真的遇到了漏掉的输入，就会报 `fun: pattern match failure at FILE:LINE:COL`。

## 理论

在模式矩阵里，每一行是一条子句，每一列对应一个待匹配的值（scrutinee）。假设已经查明某一列的值是由构造器 `C` 构造出来的，就只保留可能匹配 `C` 的行，再把这一层 `C` 拆开，将它的字段变成新的列。这叫针对 `C` 做**特化**（specialization）。还要另备一张**默认矩阵**，留下所选位置上能兜住其余任意值的那些行。这就是 Maranget 描述的决策树做法[^maranget2008]。

要复现 Fun 的选列规则，只看第一行就够了。从这一行开头，从左往右挑出第一个有可能匹配失败的模式所在的列。这条规则给出的顺序固定，好跟踪，但生成的判断次数未必总是最少。通配符和单独的变量都可以直接接住当前的值。`name@p` 给整值命名的这一步本身也无须检查，里面的 `p` 仍照常匹配。沿途绑好的值会记录下来，留给子句主体使用。构造器模式则要先对上标签，才能安全地取它的字段。

对于由有限个构造器定义的数据类型，所有可能的构造器事先就是已知的。整数和字符按字面量匹配，只列出一批具体字面量并不足以覆盖整个值域，因此需要默认分支。匹配字符串会翻译成相等性测试。匹配 `Double` 也用语言自身的浮点相等原语，所以正零和负零在这里同样相等。

检查某个模式向量是否**有用**（useful），就是问它能不能匹配到一组输入，而前面那些算作已覆盖的行都匹配不到这组输入[^maranget2007]。如果一行源码模式没用，它就是多余的。如果一行全写通配符居然还有用，就能从中找出一个漏掉的输入，作为证例（witness）。守卫让事情稍微麻烦一点，因为模式匹配上了，守卫还可能拒绝它。Fun 对带守卫的行只认一种能保证覆盖的情况，即至少有一个守卫在源码里就直接写成 `True`。它不会尝试证明任意布尔表达式恒为真。

### 同一个问题，查出两类警告

“这个模式还接得到前面漏掉的输入吗？”这就是 Maranget 所说的**有用性**问题。把前面的行记为 `P`，候选模式向量记为 `q`，那么问题可写成 `U(P, q)`。要给出肯定回答，我们还可以指出一组输入该长什么样，让它既能绕过 `P`，又能匹配 `q`。这组形状就是证例[^maranget2007]。

检查**可达性**时，把当前子句当作 `q`，它之前真正提供覆盖的行当作 `P`，看这条子句还能不能接到新情况。检查**穷尽性**时，把所有能提供覆盖的行合起来当作 `P`，再用全是通配符的一行当作 `q`，看还有没有输入漏网。前面的行要合在一起看。即使没有一行能独自盖住当前这条子句，几行合在一起也可能让它永远走不到。

来看一个双参数函数。第一个参数是 `Maybe Bool`，第二个是 `Bool`。

```fun
choose Nothing     _     = 0
choose (Just True) _     = 1
choose (Just _)    False = 2
```

先不看等号右边，只留下两列参数模式。

| 行 | 第一个参数 | 第二个参数 |
| --- | --- | --- |
| 1 | `Nothing` | `_` |
| 2 | `Just True` | `_` |
| 3 | `Just _` | `False` |

现在问 `(_, _)` 还有用吗？第一列里出现了 `Maybe` 的全部两种构造器，可以分开看。若第一个参数是 `Nothing`，第 2、3 行就不用看了，剩下第 1 行，第二个参数写的是通配符，所以整个 `Nothing` 分支都已覆盖。若第一个参数是 `Just`，就删掉第 1 行，再把第一列换成 `Just` 里面那个字段。

| 行 | `Just` 里面的值 | 第二个参数 |
| --- | --- | --- |
| 2 | `True` | `_` |
| 3 | `_` | `False` |

`Just` 里面是 `True`，第 2 行全包了。里面是 `False`，就只剩第 3 行管用，可它只接第二个参数是 `False` 的情况。于是找到了一个洞。里面是 `False`，第二个参数是 `True`。把先前拆下来的 `Just` 装回去，就得到原始参数的证例 `(Just False, True)`。加上 `choose (Just False) True = 3`，匹配就全了。再加任何子句都走不到，即使原先没有一条子句能独自覆盖所有输入。

### 如何特化，以及何时走默认矩阵

`usefulness.rs` 的覆盖分析每次从第一列入手。下一步怎么缩小问题，取决于候选模式的样子，以及这一列的构造器有没有出现齐。

1. **候选以构造器开头。** 假设它是 `C(p1, ..., pk)`。留下以 `C` 或通配符开头的行，去掉以其他构造器开头的行。把 `C` 换成它的各个字段，把通配符展开成 `k` 个通配符。对候选向量也做相应展开，再继续比较。
2. **候选以通配符开头，构造器齐全。** 如果这一列所属数据类型的每种构造器都出现了，就逐种特化。只要有一个分支还能接到新输入，就能得到证例。
3. **候选以通配符开头，构造器不全。** 转去搜索默认矩阵。只留下以通配符开头的行，然后删掉它们的第一列。如果剩余列找到一个能漏过去的后缀，就把一个尚未出现的构造器放在它前面。字面量也走这条路，在测试过的首列字面量之外另选一个即可。

列都拆完时，要区分“连一行都没有”和“还有一行，但这一行是空的”。前者说明以前的子句一个都没活到这里，候选留下的空向量没人覆盖，所以它有用。后者说明有条先前的子句一路匹配成功，剩下的输入都已被它接住，候选就没用了。有了这两个终止条件，即使构造器有许多层嵌套，也不用枚举递归数据类型的所有可能值。

递归返回时，把沿途拆下的构造器一层层套回证例的字段上。因此 Fun 的 `useful` 找到证例时会用 `Some` 返回一个向量，里面依次是各列的模式。零列时也可能成功，得到 `Some(vec![])`。只有确实找不到证例时才返回 `None`。调用方因此能知道匹配到底漏了什么。生成警告的代码先显示一个具体的遗漏形状，再把它加入临时的覆盖矩阵，接着搜索别的缺口，最多给出三个例子。

### 守卫让覆盖分析止步于哪里

一条有守卫的子句也可能走不到，比如前面早已把符合其模式的输入接完了。反过来，它通常不能算作已经完全覆盖了自己的模式，因为守卫还有机会失败。Fun 先拿该行跟此前的覆盖范围比较，只有在它没带守卫，或者至少一个守卫从语法上看就是字面量 `True` 时，才把它加入覆盖矩阵。检查器不会求解任意的守卫表达式。因此有些“缺了一个分支”的警告，只是分析做得保守。反过来，即使模式有用，也不能据此断定它的守卫真的会成功。

分析依据类型来找完整的构造器家族，把模式变量当作通配符。as 模式新绑的名字不会改变覆盖范围。分析也不去证明递归字段的类型究竟有没有可构造的值。负责报警和负责生成代码的是两个步骤。无论有没有警告，决策树都得保留源码子句的先后次序，以及守卫失败后继续往下试的行为。

### 让决策树的几条路径共用一段代码

**汇合点**为一个表达式内部的局部续延起了名字。它的参数带入共用代码需要的那些值。每次跳转都要把参数一次给齐，而且只能尾跳转，跳出去后原地没有工作等着继续做。汇合点不能逃出它的作用域，也不能像普通函数那样当作值传给别人。后面讲 A-正规形（ANF）时还会用到这个区别[^maurer2017]。

## 实现

编译器内部的一行，把“还有哪些模式要检查”和“最终执行哪段右侧表达式”分开。`values` 记着沿当前路径已经绑定的值，`target` 和 `params` 标出各路径可以共用的子句主体及其参数。

```rust
#[derive(Clone)]
struct Row {
    pats: Vec<Pat>,
    values: BTreeMap<u32, Id>,
    target: names::Id,
    params: Vec<Id>,
}
```

[源码 · match_compile.rs 第 10～16 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch16_match/src/match_compile.rs#L10-L16)

在生成判断之前，`Compiler::compile` 先给每条子句的右侧表达式分配一个带参数的汇合点。它还把这条子句之后的剩余矩阵编译成另一个汇合点，专供失败时跳转。守卫按书写顺序运行，若都失败，就跳进那张剩余矩阵继续试。这样决策树的多个叶子可以跳往同一个子句主体，多个守卫失败的路径也可以共用同一段后续代码，都不必复制代码。

进入构造器分支，要根据当前待匹配值的**实际类型参数**给字段绑定类型。下面的辅助函数先查构造器声明，再用实参替换声明里的参数。

```rust
fn fields(&self, con: &crate::data::ConId, ty: &Type) -> Vec<Type> {
    let c = &self.constructors[con.index as usize];
    let args = match ty {
        Type::Con(_, args) => args.as_slice(),
        Type::Bool => &[],
        _ => unreachable!("typed data scrutinee"),
    };
    c.fields
        .iter()
        .map(|t| crate::elaborate::instantiate(t, &c.parameters, args))
        .collect()
}
```

[源码 · match_compile.rs 第 31～42 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch16_match/src/match_compile.rs#L31-L42)

例如，对 `Maybe Int` 的 `Just` 分支做特化，绑出来的字段就该是 `Int`，声明里的类型变量在这里已经有了具体取值。取记录字段也一样。原先只含一个构造器的记录匹配，现在变成检查标签的分支，加上普通的字段绑定。Core lint 会独立核对这些实例化后的类型。

`Compiler::tree` 递归处理矩阵。如果一行都不剩，就产生 `PatternFail`。如果第一行已没有可能匹配失败的模式，就带着沿途为各绑定收集到的值，跳往该子句的汇合点。否则就按选中的列生成扁平的 `Case`，对字符串或 `Double` 字面量则生成相等性测试，然后把各特化分支和默认分支的子树接上去。

原始嵌套模式一旦编译成树，原来逐条写的样子就消失了。因此警告分析要赶在这一步之前，用原模式分析。下面的特化函数实现的就是刚才用表格手推的矩阵操作。

```rust
fn specialize(
    matrix: &[Vec<Pat>],
    head: &Pat,
    arity: usize,
) -> Vec<Vec<Pat>> {
    matrix
        .iter()
        .filter_map(|row| {
            let mut fields = if wild(&row[0]) {
                vec![Pat::Wild; arity]
            } else if same(&row[0], head) {
                match peel(&row[0]) {
                    Pat::Con(_, ps) => ps.clone(),
                    _ => vec![],
                }
            } else {
                return None;
            };
            fields.extend_from_slice(&row[1..]);
            Some(fields)
        })
        .collect()
}
```

[源码 · usefulness.rs 第 27～49 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch16_match/src/usefulness.rs#L27-L49)

`useful` 处理矩阵时还带着候选向量和每一列的类型。下面这段代码负责发起查询，反复寻找新的证例，并把诊断信息记下来。

```rust
/// Warnings do not alter the decision tree; unmatched execution still fails at runtime.
pub fn warn(
    sess: &mut Session,
    m: &Match,
    cons: &[ConDecl],
    truths: &std::collections::BTreeSet<u32>,
) {
    let types = m.scruts.iter().map(|i| i.ty.clone()).collect::<Vec<_>>();
    let mut matrix = vec![];
    for (index, row) in m.rows.iter().enumerate() {
        if useful(&matrix, &row.pats, &types, cons).is_none()
            && !(index > 0 && m.rows[index - 1].span == row.span)
        {
            let mut d = Diagnostic::error(
                ErrorCode::Redundant,
                "clause can never match",
                row.span,
            );
            d.severity = Severity::Warning;
            sess.emit(d);
        }
        if covers(&row.body, truths) {
            matrix.push(row.pats.clone())
        }
    }
    let mut missing = vec![];
    for _ in 0..3 {
        let Some(p) =
            useful(&matrix, &vec![Pat::Wild; types.len()], &types, cons)
        else {
            break;
        };
        missing.push(
            p.iter()
                .map(ToString::to_string)
                .collect::<Vec<_>>()
                .join(" "),
        );
        matrix.push(p)
    }
    if !missing.is_empty() {
        let mut d = Diagnostic::error(
            ErrorCode::NonExhaustive,
            "non-exhaustive pattern match",
            m.span,
        );
        d.severity = Severity::Warning;
        d.notes.push(format!("missing: {}", missing.join("; ")));
        sess.emit(d)
    }
}
```

[源码 · usefulness.rs 第 187～237 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch16_match/src/usefulness.rs#L187-L237)

为 `do` 中的模式生成的失败兜底行，会沿用原模式的源码位置。如果相邻的行是编译器为同一处源码自动生成的，诊断阶段就抑制这类多余的警告。用户自己写的子句仍正常检查。是否报警不会用来决定运行时走哪一支，否则即便只是误报，也可能悄悄改变程序的结果。

现在 Core lint 会检查扁平分支的形状、构造器字段的类型、绑定名在全局是否唯一，以及汇合点的实参数量和类型。跳转必须相对于目标汇合点处于尾位置。进入一个非尾位置的子表达式时，要清空此处可用的外层汇合点，子表达式仍可以建立自己的局部汇合点。

Core 解释器用显式的向量存放求值帧和返回帧。源程序调用函数时，解释器切换当前表达式和环境，无须让 Rust 自己递归调用。遇到跳转，就逐帧回退到对应的词法汇合点，再将控制权交给它的主体。闭包通过稳定的索引借用代码，调用函数不必克隆它的语法树。连释放构造器和环境也用待办列表逐步处理，免得光是释放一条长列表就把宿主栈撑爆。测试会在只有 256 KiB 的宿主栈上运行非尾递归，并回收列表。

保存下来的环境共享一棵不可变平衡树。捕获作用域只需复制一个 `Rc` 引用，添加绑定也只需复制搜索路径上的节点。`env.rs` 用一个工作向量来完成更新。这样即使作用域越来越大，解释器每次保存续延时也不必把整个作用域重抄一遍。递归闭包用索引引用抽象机持有的一组递归绑定，每次调用都可以复用同一组对象，同时避免形成引用计数环。

## 跑起来

```text
$ cd crates/ch16_match
$ cargo run -q -- run tests/programs/match/guard-fallback.fun --lint
42
$ cargo run -q -- dump --pass=types tests/programs/match/guard-fallback.fun
main : IO ()
```

想看编译前后到底变了什么，可以对同一个文件分别用 `--pass=core` 和 `--pass=match` 转储。前者保留了嵌套的 `Match`，后者则能看见扁平的 `Case`、执行各子句右侧表达式的汇合点，以及守卫失败后的跳转。转储里也包含 Prelude。如果只想仔细研究一棵小决策树，可以看 `tests/matches.rs` 里关闭了 Prelude 的那些用例。

验证翻译有没有改变含义，办法之一是让同一段匹配在编译前后分别运行，核对结果。覆盖分析则需要自己的检验方法。对于很小的布尔模式，可以把所有输入逐一列出，直接算出真实的覆盖结果，再与有用性算法的答案比较。前一种对照能抓到运行时走错分支的问题，后一种则能独立发现“程序运行正确，警告却报错了”的问题。

## 练习

1. **基础。** 给开头的守卫示例画出决策树，标明守卫失败后要跳到哪里。
2. **进阶。** 在构造器子句之前插入一条通配符子句。先预测哪一条会被报为多余，再运行检查器。
3. **进阶。** 对一张两列的矩阵做特化。第一行的第一个模式是变量，第二个模式是构造器模式。
4. **挑战。** 在实验副本里改一改选列规则。用完整的原有程序集继续对照编译前后的执行结果，并比较生成的树的大小。
5. **挑战。** 构造一段不合法的 Core，从非尾位置的子表达式跳到某个外层汇合点。如果允许它这么跳，原来等着做的哪一段续延会被丢掉？

## 延伸阅读

- 决策树与选列时的启发式规则[^maranget2008]。
- 模式的有用性、匹配是否穷尽，以及如何发现多余的子句[^maranget2007]。
- 用汇合点构造编译器中间表示[^maurer2017]。
- 源语言里的模式和守卫应当怎样匹配[^haskell2010]。

如果想脱离 Fun 编译器，单独研究这些矩阵操作，可以看用 Rust 编写的 [Patternkit](https://github.com/sdiehl/patternkit)。Fun 把自己的实现直接放在本章的代码里，方便读者对照正文查看检查算法。

[^maranget2008]: Luc Maranget（2008）。[Compiling Pattern Matching to Good Decision Trees](https://doi.org/10.1145/1411304.1411311)。
[^maranget2007]: Luc Maranget（2007）。[Warnings for Pattern Matching](https://doi.org/10.1017/S0956796807006223)。
[^maurer2017]: Luke Maurer、Paul Downen、Zena M. Ariola、Simon Peyton Jones（2017）。[Compiling without Continuations](https://doi.org/10.1145/3062341.3062380)。
[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。

---

[← 第 15 章　Monad、do 记法与 IO](ch15-io.md) · [中文目录](../README.md) · [第 17 章　Prelude 与自动派生 →](ch17-prelude.md)
