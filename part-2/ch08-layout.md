# 第 8 章　缩进布局和扩展语法

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch08-layout.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch08_layout)

> **新增文件**
>
> - `src/layout.rs`
> - `src/fixity.rs`
> - `src/format.rs`
>
> **改动文件**
>
> - `src/parse.rs`
> - `src/typecheck.rs`
> - `src/ast.rs`
> - `src/elaborate.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/types.rs`

## 让代码的写法顺手些

一开始全写花括号，好处是语法简单，每一块从哪儿开始、到哪儿结束都一目了然。可有了缩进，代码本身已经把层次摆在眼前。能不能让我们少敲些标点，照样把结构交代清楚？

这一章给 Fun 加上缩进布局、用 `where` 定义局部辅助函数或值的写法、类型签名，以及自定义运算符。这些功能，大多可以在运行程序之前就处理完。把缩进换成标点，把只写一侧参数的运算符表达式（section）换成 lambda，再根据运算符的优先级和结合性给长表达式分组。这样，写程序的人有了更多顺手的写法，解释器却不必跟着长出一堆新规则。

## 这一章的 Fun 能写什么

只要能从缩进看出语句边界，就可以省略花括号和分号。`where` 块归它所跟随的那条方程所有，所以在块里照样能用这条方程的形参。一个 `let` 块也可以放多条局部声明了。

```fun
addTwo x = x + y
  where
    y = 2

main = let result = addTwo 40 in result
```

同一块里的类型签名放在定义之前或之后都行。签名中的类型变量默认带全称量词。你可以把推断出的类型限制得更具体，但签名承诺的适用范围不能超出实现的实际能力。

```fun
id :: a -> a
id x = x

main = let { n :: Int; n = id 42; } in n
```

定义运算符既能用前缀写法，也能用中缀写法。运算符的优先级和结合性合称 *fixity*。一条 fixity 声明在整个文件里都生效，写在它之前的运算式也按这条声明分组。没有声明 fixity 的运算符默认优先级为 9，向左结合。普通函数名用反引号括起来，也能当中缀运算符用。

```fun
infixr 5 %%

a %% b = a * 10 + b

main = 1 %% 2 %% 3
```

这个程序打印 `33`。`infixr` 里的 `r` 表示向右结合，所以要算的是 `1 %% (2 %% 3)`。里层先得到 `23`，外层再算 `1 * 10 + 23`。前面提到的 section 是给二元运算符先填好一个参数，留下另一个稍后再传。例如 `(40 +)` 表示“拿 40 加上传入的数”，`(+ 2)` 表示“拿传入的数加 2”。减号比较特殊，`(- 2)` 直接表示负数。取负的优先级为 6，所以 `-2 * 3 + 4` 读成 `(-(2 * 3)) + 4`。如果想在一个优先级不低于 6 的运算符后面紧接取负表达式，要把后者用括号括起来，例如 `3 * (-2)`。

## 原理

既然解析器已经认识花括号和分号，我们只需先把缩进翻译成这两种标点。《Haskell 语言报告》给出了这套翻译规则[^haskell2010]。我们先手工走一遍，再看代码怎样做。

### 把看不见的括号和分号写出来

一个块的第一条声明定下这块的左边界。在同一列又看到一条声明，就开始下一个条目。缩进退回外层的边界，内层的块也就结束了。

```fun
scale x = result
  where
    doubled = x + x
    result = doubled * 3

main = let n = 7 in scale n
```

布局处理在解析之前运行。把上面这段交给它，效果就相当于写了下面的代码。里面每一个花括号和分号，都是布局阶段补上的。

```fun
{ scale x = result
    where { doubled = x + x
          ; result = doubled * 3
          }
; main = let { n = 7 } in scale n
}
```

先给整个文件套上一对花括号。`doubled` 定下了 `where` 块的起始列，下一行的 `result` 与它对齐，于是要在 `result` 前补一个分号。`main` 退到了更左边，所以先补右花括号，结束 `where` 块，再补分号，开始下一条顶层声明。留意同一行的 `let n = 7 in ...`。这里缩进没有变浅，遇到 `in` 仍要结束 `let` 的声明块。读到文件末尾，补最后一个右花括号即可。自己把这些标点写出来，解析出的分组也一样。

后面学模式匹配和 IO 时，`of` 和 `do` 后面也用同一套办法分块。

```fun
main = do
  case flag of
    True -> putStrLn "yes"
    False -> putStrLn "no"
  putStrLn "done"
```

把这段程序里省掉的标点也补回来。

```fun
{ main = do
    { case flag of
        { True -> putStrLn "yes"
        ; False -> putStrLn "no"
        }
    ; putStrLn "done"
    }
}
```

最后一个 `putStrLn` 比 `True` 和 `False` 靠左，所以 `of` 块结束。它又和 `case` 对齐，所以这条动作仍在 `do` 里面。虽然要到后面几章才能运行这段程序，翻译缩进的规则现在就说得清。用 `--dump=layout` 能看到真实的词元流，还能从源码区间分辨哪些标点是程序员写的，哪些是补的。

接下来，编译器要决定一串中缀运算符如何分组。它先比较相邻的运算符，再构造相应的函数应用。优先级高的先结合，优先级相同才看结合方向。两个运算符都向左结合时从左往右收拢，都向右结合时则从右往左收拢。同一优先级遇到不可结合的运算符，或两个运算符的结合方向冲突，就得用括号写清楚[^haskell2010]。

类型签名相当于给调用者的承诺。写 `a -> a` 就是说，无论调用者选什么 `a`，这个函数都接得住，还会返回同类型的值。检查器如果自己悄悄决定 `a = Int`，就把承诺的意思给改了。

对照 `id x = x` 和 `inc x = x + 1`。给二者写 `Int -> Int` 都说得通。要是写 `a -> a`，就只有 `id` 能兑现。假如检查 `inc` 时允许用 `Int` 解出签名里的 `a`，`inc` 就可能蒙混过关。调用者随后有权传给它一个布尔值，可函数体根本算不了。我们需要一个变量来代表“调用者任选的类型”，并且不能让检查器挑个方便的具体类型交差。

做法是把签名里量化的变量替换成各自独立的**刚性变量**，叫作 *skolem*。推断产生的灵活变量可以在合一时求出具体类型，而 skolem 代表任意类型，必须保持其任意性。我们把推断出的类型方案用全新的灵活变量实例化，试着让它和刚性的签名合一。然后还要检查 skolem 有没有“逃逸”，也就是本应只在这次核查中有效的变量，会不会渗进外围作用域。两项检查都通过，推断出的类型才能支持签名许下的承诺[^damas1982][^pierce2002]。

## 实现

### 处理缩进，也把注释留下来

logos 词法分析器把注释连同原始文本一起输出。`TriviaLexer` 把注释记在一张旁表里，只把对语法有意义的词元往下传。接着由 `LayoutLexer` 根据缩进插入标点。这个适配器把显式的花括号和圆括号配成组，再递归处理组内的布局，所以用圆括号括起来的 `let` 也能得到自己的声明块。

```rust
pub fn layout(
    sess: &mut Session,
    file: FileId,
) -> Result<Vec<Spanned>, Stop> {
    let tokens = lexer::lex(sess, file)?;
    let source = &sess.sources.get(file).text;
    let eager = !matches!(tokens.first(), Some((_, Token::Open, _)));
    // offsides measures byte columns with fixed-width tabs. Give it a shadow
    // source whose byte positions are visual columns, then restore original spans.
    let (shadow, forward, backward) = visual_source(source);
    let tokens = tokens
        .into_iter()
        .map(|(lo, t, hi)| (forward[lo], t, forward[hi]))
        .collect::<Vec<_>>();
    match region(&shadow, &tokens, eager)
        .map(|tokens| {
            tokens
                .into_iter()
                .map(|(lo, t, hi)| (backward[lo], t, backward[hi]))
                .collect()
        })
        .map_err(|(lo, hi, message)| (backward[lo], backward[hi], message))
    {
        Ok(tokens) => Ok(tokens),
        Err((lo, hi, message)) => {
            sess.emit(Diagnostic::error(
                ErrorCode::Parse,
                message,
                Span::new(file, lo, hi),
            ));
            Err(Stop)
        }
    }
}
```

[源码 · layout.rs 第 39～72 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch08_layout/src/layout.rs#L39-L72)

底层库没有提供“读到 `in` 就关闭对应的 `let` 块”这个操作。适配器只好自己记下哪些虚拟布局层已经在 `in` 处结束，以免后面又为它们补上分号或右花括号。空块，以及显式分号旁边多余的虚拟分号，也都由它处理。源文件中本来就有的词元保留原始字节偏移。补入的标点放在相应位置，区间宽度为零，不会假装自己占着源码里的一个字符。

本章的词法布局测试已经覆盖 `of` 和 `do`。支持 `case … of` 和 `do` 表达式本身则分别是第 11、15 章的事，现在把它们交给完整编译器，语法仍然不能通过。

### 给一串运算符分组

文法先记住第一个操作数，后面跟着一串“运算符、下一个操作数”。源码里的圆括号也先留着，等分组器确定它们的边界。然后用一个栈按优先级和结合性处理整条链。栈中给一元运算单设一种条目，这样取负也能参与优先级比较。

```rust
fn chain(
    &mut self,
    first: Expr,
    pairs: Vec<(String, Expr)>,
) -> Result<Expr, Diagnostic> {
    let mut values = Vec::new();
    let mut operators: Vec<Option<String>> = Vec::new();
    self.operand(first, &mut values, &mut operators)?;
    for (op, value) in pairs {
        let next = self.fix(&op);
        while let Some(previous) = operators.last() {
            let prior =
                previous.as_ref().map(|p| self.fix(p)).unwrap_or(Fix {
                    precedence: 6,
                    assoc: Assoc::Left,
                });
            if prior.precedence < next.precedence {
                break;
            }
            if prior.precedence == next.precedence && previous.is_some()
            {
                if prior.assoc != next.assoc
                    || prior.assoc == Assoc::None
                {
                    return Err(self.error(value.span,"operators with conflicting fixities need parentheses"));
                }
                if prior.assoc == Assoc::Right {
                    break;
                }
            }
            let prior = operators.pop().unwrap();
            self.reduce(&mut values, prior);
        }
        operators.push(Some(op));
        self.operand(value, &mut values, &mut operators)?;
    }
    while let Some(op) = operators.pop() {
        self.reduce(&mut values, op);
    }
    Ok(values.pop().unwrap())
}
```

[源码 · fixity.rs 第 159～199 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch08_layout/src/fixity.rs#L159-L199)

section 会变成 lambda。自动生成的参数名是程序员用源码标识符拼不出来的，因此不会意外和现有名字撞上。只要内建运算符没被同名定义遮蔽，原有的算术节点就继续用。自定义运算符则变成普通的柯里化函数应用。Core 现有的表达式形式已经够用。

### 核对类型签名

先收集整块声明，再把签名和各自的定义配对。检查器先推断一个绑定的类型并泛化，再复制出一份临时的类型推断状态，在副本上试验签名是否成立。通过之后，把后续代码真正要用的类型方案收窄到签名允许的范围。这一版在推断递归定义时，递归调用仍然共享一个单型。要让递归调用凭签名使用多态，还得等后面更完整的签名检查器。

```rust
fn check_signature(
    &mut self,
    inferred: &Scheme,
    declared: &Scheme,
    expr: &Expr,
) -> Result<Scheme, Diagnostic> {
    fn rigid(ty: &Type) -> Type {
        match ty {
            Type::Var(n) => Type::Skolem(*n),
            Type::Arrow(a, b) => Type::arrow(rigid(a), rigid(b)),
            t => t.clone(),
        }
    }
    fn has_skolem(ty: &Type) -> bool {
        match ty {
            Type::Skolem(_) => true,
            Type::Arrow(a, b) => has_skolem(a) || has_skolem(b),
            _ => false,
        }
    }
    let mut trial = self.clone();
    let actual = trial.instantiate(inferred);
    let expected = rigid(&declared.ty);
    let invalid = trial.unify(&actual, &expected, expr).is_err()
        || (0..self.metas.len()).any(|n| {
            self.metas[n].level <= self.level
                && has_skolem(&trial.zonk(&Type::Var(n as u32)))
        });
    if invalid {
        return Err(Diagnostic::error(
            ErrorCode::Signature,
            format!(
                "signature {} is not an instance of inferred type {}",
                declared.ty,
                inferred.ty.canonical()
            ),
            expr.span,
        ));
    }
    // Specialization can constrain variables captured from an outer scope.
    self.level += 1;
    let actual = self.instantiate(inferred);
    let expected = self.instantiate(declared);
    let result = self.unify(&actual, &expected, expr);
    self.level -= 1;
    result?;
    Ok(self.generalize(expected))
}
```

[源码 · typecheck.rs 第 148～195 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch08_layout/src/typecheck.rs#L148-L195)

### 格式化后，还能读回同一棵树吗？

格式化器使用 marginalia 提供的文档排版组合规则（document algebra），并读取附带的注释映射。它选择的标准写法会显式写出块的花括号，保留源码里的圆括号。注释按原来出现的顺序，在对应的词元边界前面各占一行，文字也原样保留。这样就不会不小心把新补的标点放进单行注释里，结果被一起注释掉。

```rust
pub fn format(sess: &mut Session, file: FileId) -> Result<String, Stop> {
    let parsed = parse::parse(sess, file)?;
    crate::fixity::resolve(sess, parsed)?;
    let (semantic, trivia) = lexer::trivia(sess, file)?;
    let tokens = layout::layout(sess, file)?;
    let source = &sess.sources.get(file).text;
    let mut comments = attach(
        source,
        &trivia,
        semantic.iter().map(|(lo, _, hi)| span(*lo, *hi)),
        AttachOptions::default(),
    );
    // Tokens are smaller anchors than statements. Put their comments on
    // separate leading lines so a trailing line comment cannot swallow a
    // newly inserted separator or a following `where`.
    for (lo, _, hi) in &semantic {
        let entry = comments.entry(span(*lo, *hi));
        entry.leading.append(&mut entry.trailing);
    }
    let mut cursor = 0;
    let doc = document(source, &tokens, &mut cursor, false);
    let rendered = pretty::render(&doc, &comments, RenderOpts::default());
    Ok(format!("{}\n", rendered.trim_end()))
}
```

[源码 · format.rs 第 18～41 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch08_layout/src/format.rs#L18-L41)

测试会把格式化后的文本重新解析，去掉源码位置，再与原来的语法树逐项比较。光看两段代码算出来的值相同还不够，两棵不同的树可能碰巧算出同一个数。测试程序集中，所有语法正确的程序都要能这样来回走一趟。把格式化的输出再格式化一次，还必须逐字节相同。另外有专门的用例检查嵌套块注释和 `where` 旁边的注释能否保留下来。

## 跑起来看看

最后两条命令会用到一个自己准备的文件。进入 `crates/ch08_layout` 后，先用 `cp tests/programs/layout/right-fixity.fun example.fun` 复制一份样例。这样 `fmt` 会直接改写这份副本，最后的 `--check` 再检查它是否已符合格式。

```text
$ cd crates/ch08_layout
$ cargo run -q -- run tests/programs/layout/right-fixity.fun
33
$ cargo run -q -- dump --pass=fixity tests/programs/layout/negation.fun
{
  main = -(2 * 3) + 4;
}
$ cargo run -q -- fmt example.fun
$ cargo run -q -- fmt example.fun --check
```

三个阶段的输出可以并排对照。`--dump=layout` 显示补上的标点，`--dump=parse` 显示尚未分组的运算符链，`--dump=fixity` 显示确定后的分组。试试只改缩进，或者只改一条 fixity 声明，看看最先出现差异的是哪一层。

## 练习

1. **基础。** 挑三个优先级或结合性不同的运算符，先预测表达式会怎样分组，再用 fixity 输出核对。
2. **进阶。** 解释为什么 `(1 + 2 *)` 会被拒绝，再用括号改成编译器接受的写法。
3. **进阶。** 在运算符和操作数中间放一段嵌套注释，给格式化器加一个测试。既要检查前后的语法树相同，也要检查注释都还在。
4. **挑战。** 写一个局部类型签名，让它的 skolem 试图通过捕获的 lambda 参数逃逸到外层。跟踪检查器在临时状态里做了哪些检查，最后在哪里拒绝了它。
5. **挑战。** 对比本章专门处理 `in` 的规则和《Haskell 语言报告》里更一般的 `parse-error(t)` 规则。找出一种语法，使后者能处理前者照顾不到的情况。

## 延伸阅读

《Haskell 语言报告》精确定义了布局和运算符分组的规则[^haskell2010]。签名检查用到的主类型性质，可以追溯到 Damas 和 Milner 的工作[^damas1982]。要从更广的类型理论背景理解多态类型和它的各个实例有什么区别，可以读 Pierce 的著作[^pierce2002]。

[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。
[^damas1982]: Luis Damas、Robin Milner（1982）。[Principal Type-Schemes for Functional Programs](https://doi.org/10.1145/582153.582176)。
[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。

---

[← 第 7 章　设计 Fun](ch07-fun.md) · [中文目录](../README.md) · [第 9 章　名称解析 →](ch09-renamer.md)
