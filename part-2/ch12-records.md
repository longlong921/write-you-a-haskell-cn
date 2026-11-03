# 第 12 章　记录

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch12-records.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch12_records)

> **新增文件**
>
> 无。
>
> **改动文件**
>
> - `src/typecheck.rs`
> - `src/data.rs`
> - `src/lexer.rs`
> - `src/elaborate.rs`
> - `src/ast.rs`
> - `src/core.rs`
> - `src/desugar.rs`
> - `src/eval.rs`
> - `src/fixity.rs`
> - `src/format.rs`
> - `src/main.rs`
> - `src/parse.rs`
> - `src/rename.rs`
> - `src/renamed.rs`

## 构造器的第三个参数，到底指什么？

一个点只有两个坐标，靠位置区分大概还记得住。换成一个配置对象，一口气塞进去六个整数呢？每回读代码都得把参数表在脑子里过一遍。不如给字段起个名字。

Fun 的**记录**（record）建立在已有的数据类型之上。我们用点号读取字段，并规定一种记录类型只能有一个构造器。这样一来，字段访问不会因为值恰好来自另一个构造器而失败，字段名也不占普通函数的名字。比如两种记录都可以有一个叫 `x` 的字段。这解决了传统 Haskell 字段选择函数带来的两个不便[^haskell2010][^recorddot]。

真正值得琢磨的是推断。看到 `p.x` 时，编译器凭什么知道 `p` 是哪种记录？有时候答案得从表达式的其他地方找。

## 现在能写什么

```fun
data Point = Point { x :: Int, y :: Int }

move p = p { x = p.x + 1 }

main = move (Point { y = 4, x = 3 })
```

构造记录时，每个字段都得给，且只能给一次。字段表达式按**书写顺序**求值，即使这个顺序和声明相反。按位置构造、按位置匹配的老写法也都能用。用字段名写模式时，可以省略不关心的字段，它们会自动补成通配符。例如 `Point { x = n }` 只把第一个坐标绑定到 `n`。

取字段（又叫**投影**）和更新记录的优先级都高于函数应用。因此，`f p.x` 读作 `f (p.x)`，`f p { x = 3 }` 读作 `f (p { x = 3 })`。单独写 `(.x)` 是一个字段投影函数的简写，相当于 `\p -> p.x`。点号也可以连着用，`shape.origin.x` 先取 `origin`，再从结果取 `x`。字段更新保留原记录的完整类型，连类型实参也不会变。

```fun
data Label = Label { x :: String }

coordinate :: Point -> Int
coordinate p = p.x
```

现在 `Point` 和 `Label` 都有 `x`，但 `coordinate` 的签名已经指出这个 `x` 属于 `Point`。直接给表达式加标注，写成 `(p :: Point).x`，也有同样效果。如果某个字段名只在一种记录中声明过，检查器不用标注就能推断出点号左边的类型。

## 背后的原理

哪怕两份声明里的字段一模一样，它们定义的仍然是两种不同的类型。这就叫**名义类型**，类型的身份由声明决定。一个 `Point` 不会因为碰巧也有 `x`、`y` 两个字段，就自动变成另一种记录。结构类型或行多态（row polymorphism）还可以采用另一种规则，允许函数接收任何含有所需字段的记录[^pierce2002]。Fun 采用名义类型，这样在运行之前就能确定一次字段访问对应哪个构造器、哪个字段位置。

投影会产生一个待解约束，把点号左边的**接收者**类型与取出来的结果类型联系起来。

$$
\frac{\Gamma \vdash e : \tau \quad \mathsf{Has}(\tau,x,\alpha)}{\Gamma \vdash e.x : \alpha}\;\text{（字段投影）}
$$

把它读成一句话。如果表达式 $e$ 的类型是 $\tau$，而 $\tau$ 有一个名叫 $x$、类型为 $\alpha$ 的字段，那么 $e.x$ 的类型就是 $\alpha$。

`Has` 是检查器必须在内部解决的任务，不会跟着泛化进入类型方案。如果一个字段名只有一个归属，检查器立刻用新的类型参数实例化相应记录类型，再把它和接收者的类型合一。如果有多个可能，就先挂起这个约束，等到能认出接收者属于哪个命名类型。泛化一个绑定组之前，检查器会反复尝试解约束，直到再也没有进展。如果本轮正要泛化的那一层还留有未解的字段约束，就报 `E0245` 并提示加上类型标注。属于外层的约束可以留给外层继续解。

举个例子，假设 `Point` 和 `Size` 都声明了 `x`。单看 `\p -> p.x`，确实没法决定要用哪一个。但给这个 lambda 加上签名 `Point -> Int`，检查签名时就能得知 `p` 是 `Point`，先前挂起的约束也就能落实到 `Point` 的 `x` 字段上。缺少足够信息时，Fun 会报告歧义。要想推断出一个“任何带 `x` 的记录都能传进来”的函数，需要采用另一套类型系统。

点号语法可以参考 GHC 的 record-dot 提案。GHC 用 `HasField` 机制支持重载的字段访问。Fun 则要求在生成 Core 之前，把每个字段访问都落实到一种具体的命名记录上[^recorddot]。

## 动手实现

### 点号到底是字段访问，还是普通运算符

这里的空白会影响词法单元的含义。`f.g` 表示从 `f` 取字段 `g`，`f . g` 则使用普通的函数组合运算符。词法分析后的一个处理步骤会比较原始词法单元的源码区间，趁注释、空白的归集和布局处理还没有模糊它们的邻接关系，先识别投影点号。

```rust
/// Classify dot adjacency before trivia capture and layout can erase spacing.
fn projection_dots(
    file: FileId,
    tokens: &mut [Spanned],
    errors: &mut Vec<Diagnostic>,
) {
    for i in 1..tokens.len().saturating_sub(1) {
        if !matches!(&tokens[i].1,Token::Operator(op) if op==".")
            || tokens[i].2 != tokens[i + 1].0
            || !matches!(tokens[i + 1].1, Token::Ident(_))
        {
            continue;
        }
        if tokens[i - 1].2 != tokens[i].0 {
            continue;
        }
        if tokens[i - 1].1 == Token::Left {
            tokens[i].1 = Token::ProjSection;
            if !matches!(
                tokens[i + 2..].iter().find(|(_, t, _)| !matches!(
                    t,
                    Token::LineComment(_) | Token::BlockComment(_)
                )),
                Some((_, Token::Right, _))
            ) {
                errors.push(Diagnostic::error(
                    ErrorCode::ProjectionDot,
                    "a projection section must have the form (.field)",
                    Span::new(file, tokens[i].0, tokens[i].2),
                ));
            }
        } else if matches!(
            tokens[i - 1].1,
            Token::Ident(_)
                | Token::Right
                | Token::BracketClose
                | Token::Close
                | Token::Int(_)
                | Token::Char(_)
                | Token::String(_)
        ) {
            tokens[i].1 = Token::Proj;
        }
    }
}
```

[源码 · lexer.rs 第 283～327 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch12_records/src/lexer.rs#L283-L327)

投影点号前面必须是标识符、字面量或右定界符，并且紧挨着点号。如果点号紧接在 `(` 后面，后面又紧跟一个变量标识符，就识别成用来写 `(.x)` 的投影片段。其他点号仍当运算符处理。注意 `x .y` 里的点号和左侧隔了一格，依这条规则，它仍然是普通运算符。

一旦认出了投影片段，发现 `(.x y)` 这样的错误写法，就报 `E0004`。格式化器也得保证点号和字段名紧挨着，把附着在这里的注释挪到这段序列以外。两处行为都要有测试，因为单单挪动空格就可能改变程序含义。

### 让字段信息留在所属的记录里

每个记录构造器在保存字段类型的同时，也保存字段名。验证数据类型时，同一声明里把记录构造器与其他构造器混用，报 `E0240`。同一份记录声明里出现重名字段，报 `E0104`。按字段名构造记录时，还要分别检查少给了字段、同一字段给了两遍、给了不存在的字段，对应 `E0241`、`E0242`、`E0243`。

记录模式会在重命名阶段转成按位置匹配的构造器模式。重命名器查出每个字段名对应哪个位置，再将没写出的位置全补成通配符。这样一来，后面的模式检查器和 Core Lint 直接使用现成的规则就够了，也不用把任何字段名塞进普通值的环境。

### 把待定的字段访问逐个解开

一条待解约束记着接收者类型、字段名、刚刚分配的字段类型变量、源码里的出现位置，以及推断层级。有了位置标记，同一次更新里涉及的多个字段也能一一分清。

```rust
/// A field site's position distinguishes updates of several named slots.
pub type FieldSite = (u32, usize);

/// Field constraints never enter a polymorphic scheme.
#[derive(Clone)]
struct FieldWanted {
    record: Type,
    label: String,
    field: Type,
    site: FieldSite,
    span: crate::source::Span,
    level: u32,
}
```

[源码 · typecheck.rs 第 14～26 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch12_records/src/typecheck.rs#L14-L26)

推断器遇到投影表达式，就创建这样一条约束，先把新类型变量当作投影的结果类型返回。以后约束解开了，再拿记录声明中经实例化的字段类型来和它合一。

```rust
Kind::Project(e, label) => {
    let record = self.infer(env, e)?;
    let field = self.fresh();
    self.field(
        record,
        label.clone(),
        field.clone(),
        (expr.node, 0),
        expr,
    )?;
    Ok(field)
}
```

[源码 · typecheck.rs 第 599～610 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch12_records/src/typecheck.rs#L599-L610)

`solve_field` 有三种结果。能辨认记录，就确定构造器和字段位置。如果已经知道接收者类型，而它无法提供这个字段，可以报 `E0244`。留意只有一个候选记录时的顺序。求解器先按字段名选中它，再让它的类型和接收者合一，因此两种类型冲突时，报的是类型不匹配 `E0201`。候选不唯一、接收者类型也还没确定时，就继续挂起。外面还需要一层循环，因为解开一个字段，有时恰好会确定另一个字段的接收者类型。回想 `shape.origin.x`，知道了 `origin` 的类型，才知道第二个点号要去哪种记录里找 `x`。

```rust
fn solve_fields(
    &mut self,
    complete: Option<u32>,
) -> Result<(), Diagnostic> {
    loop {
        let pending = std::mem::take(&mut self.wanted);
        let count = pending.len();
        for wanted in pending {
            if !self.solve_field(&wanted)? {
                self.wanted.push(wanted);
            }
        }
        if self.wanted.len() == count {
            break;
        }
    }
    if let Some(level) = complete
        && let Some(w) = self.wanted.iter().find(|w| w.level > level)
    {
        let mut error = Diagnostic::error(
            ErrorCode::AmbiguousField,
            format!("ambiguous field access .{}", w.label),
            w.span,
        );
        error.notes.push(
            "add a type annotation to identify the record type".into(),
        );
        return Err(error);
    }
    Ok(())
}
```

[源码 · typecheck.rs 第 144～174 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch12_records/src/typecheck.rs#L144-L174)

根据签名来检查表达式时，要先把 lambda 参数的类型确定下来，再去检查函数体。假如函数体里的某个局部绑定访问了一个几种记录共用的字段，那么在泛化这个局部绑定组之前，它就需要拿到外层签名提供的信息。原有的刚性签名检查依然保留。如果签名声称函数比实际情况更为多态，检查器还是会拒绝。

### 严格地构造、读取和更新记录

源码中字段的书写顺序和内存中字段的排列顺序各管各的。生成 Core 时，先为用户写下的每个字段表达式生成一个严格 let，依次求出值，再将这些变量按**声明顺序**放到构造器的对应位置。

```rust
Kind::Record(id, fields) => {
    let declaration =
        self.constructors[id.index as usize].clone();
    let arguments = self.typed.instantiations[&expr.node]
        .iter()
        .map(|t| convert(t, variables))
        .collect::<Vec<_>>();
    let mut values = vec![Expr::Lit(Lit::Int(0)); fields.len()];
    let mut locals = vec![];
    for (label, e) in fields {
        let index = declaration
            .labels
            .as_ref()
            .unwrap()
            .iter()
            .position(|n| n == label)
            .unwrap();
        let parameter = Id {
            name: self.sess.fresh_id("$field"),
            ty: instantiate(
                &declaration.fields[index],
                &declaration.parameters,
                &arguments,
            ),
        };
        locals.push(BindGroup {
            recursive: false,
            binds: vec![Bind {
                name: parameter.clone(),
                rhs: Box::new(self.expression(e, variables)),
            }],
        });
        values[index] = Expr::Var(parameter);
    }
    let mut body = Expr::Con(id.clone(), arguments, values);
    for local in locals.into_iter().rev() {
        body = Expr::Let(local, Box::new(body));
    }
    body
}
```

[源码 · elaborate.rs 第 303～342 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch12_records/src/elaborate.rs#L303-L342)

读取字段时，把接收者求值一次，匹配它唯一的构造器，返回所需字段绑定的值。更新也使用同一种匹配，按书写顺序计算各个替换表达式，然后重新构造记录。没改动的字段放回原来的位置。因为结果类型与接收者完全相同，更新操作无法把 `Box Int` 变成 `Box Bool`。

因此落到 Core 里，用到的全是[第 11 章](ch11-datatypes.md)已有的构造。独立 Lint 检查它们的类型，解释器执行普通的匹配与构造。打印值时，解释器会查看构造器的字段名，按声明顺序显示 `Point {x = 3, y = 4}`。等到后面生成原生代码时，取字段可以变成一次从相应位置加载数据的操作。

## 跑起来

```text
$ cd crates/ch12_records
$ cargo run -q -- run tests/programs/records/nested.fun --lint
(3,Outer {inner = P {x = 10, y = 4}})
$ cargo run -q -- dump --pass=types tests/programs/records/shared-signature.fun
main : Int
$ cargo test --test records
```

试着再定义一种带 `x` 的记录，拿掉原来用来消歧的类型标注，看看编译器在什么时候发现仅凭字段名已经没法决定类型。另一个实验是让接收者表达式和要更新的字段表达式分别触发不同的错误。最先报哪个错，应当取决于源码规定的求值次序，最终的字段存放位置无权改变这个结果。

## 练习

1. **基础。** 定义一个多态记录 `Box a`，并推断它的字段投影函数 `(.字段名)` 的类型。
2. **进阶。** 再定义一种与它共用字段名的记录，观察歧义错误，再用签名消除歧义。
3. **进阶。** 同一个绑定组中，为什么后面出现的一个独有字段，可能帮先前悬而未决的字段访问消除歧义？
4. **挑战。** 写一条记录构造表达式，让字段书写顺序与声明顺序不同。通过最先触发的错误证明实际的求值顺序。
5. **挑战。** 假如要支持行多态的字段访问，推断和泛化分别要改哪些地方？这次只分析，不必实现这个功能。

## 延伸阅读

《Haskell 语言报告》描述了传统的记录构造、记录更新和字段选择函数[^haskell2010]。GHC 第 282 号提案解释了为什么引入记录点号语法，以及它和重载字段访问的关系[^recorddot]。Pierce 的著作讲解结构化的记录类型，可以和 Fun 的名义类型设计对照着读[^pierce2002]。

[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。
[^recorddot]: Neil Mitchell、Shayne Fletcher（GHC 提案 282）。[Record Dot Syntax](https://ghc-proposals.readthedocs.io/en/latest/proposals/0282-record-dot-syntax.html)。
[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。

---

[← 第 11 章　数据类型与模式匹配](ch11-datatypes.md) · [中文目录](../README.md) · [第 13 章　种类与类型构造器 →](ch13-kinds.md)
