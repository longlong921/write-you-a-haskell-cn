# 第 9 章　名称解析

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch09-renamer.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch09_rename)

> **新增文件**
>
> - `src/rename.rs`
> - `src/scc.rs`
> - `src/renamed.rs`
>
> **改动文件**
>
> - `src/names.rs`
> - `src/typecheck.rs`
> - `src/ast.rs`
> - `src/core.rs`
> - `src/elaborate.rs`
> - `src/eval.rs`
> - `src/fixity.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/parse.rs`
> - `src/types.rs`

## 这个 `x` 到底是哪个 `x`？

两个变量可以都叫 `x`，却指完全不同的东西。表达式小的时候，用眼睛就能顺着作用域找到答案。可编译器一旦开始挪动或改写表达式，光看拼写就不那么踏实了。我们希望在原有的词法结构还清清楚楚时，就把每次出现的名字对应到它引用的那个定义，此后一直带着这个答案往下走。

这是名称解析器（renamer）的第一项工作。第二项是放宽声明的书写顺序。为了能在一个函数里调用辅助函数，过去必须把辅助函数写在前面。可要是两个函数彼此调用，怎么排都有一个得写在后面。我们会在依赖图里找出这种需要一起处理的一组定义，一整组交给类型检查器。

## 这一章的 Fun 能写什么

顶层声明和局部声明都可以引用同一块中写在后面的定义。下面两个函数互相调用，因此属于同一个递归组。

```fun
even n = if n == 0 then True else odd (n - 1)

odd n = if n == 0 then False else even (n - 1)

main = even 100
```

每个声明块形成一个词法作用域。局部定义会遮蔽同名的全局定义，同一块里定义两次相同的名字则会报 `E0102`。递归组里每个绑定的右边都必须是函数。所以 `a = b; b = a` 会报 `E0120`，就算谁也没用到 `a`、`b`，也不能放行。

运算符的优先级与结合性声明（fixity 声明）也要检查作用域。如果在顶层声明了某个运算符的 fixity，这个文件里就必须能找到对应的定义，否则报 `E0130`。使用未绑定的变量会报 `E0101`。如果在当前作用域里能找到拼写接近的名字，编辑距离不超过 2，诊断还会提示你可能想用它。

## 为什么要画一张依赖图

编译器用两样东西记住一个绑定的身份，一是驻留表里保存的拼写，二是新分配的唯一整数。前者方便报错，后者能把被遮蔽的外层 `x` 和遮蔽它的内层 `x` 区分开。类型推断器和 Core 都使用这个身份，不用再沿着作用域反复查名字。这正是第 3 章里变量与绑定的区别，在实际编译器中的用法[^pierce2002]。

给同一个声明块里的每个绑定画一个点。如果一个函数用到了块内的另一个绑定，就从这个函数向那个绑定画一支箭。`even` 调用 `odd`，就有 `even → odd` 这条边。`odd` 又调用 `even`，于是沿箭头还走得回来。这两个点理应放在同一组。图论里管这样的组叫**强连通分量**（strongly connected component，SCC）。从组内任何一个点出发，都能沿箭头抵达组内其他点。如果组外还有一个点也能和这些点互相抵达，就该把它也算进来，一直加到不能再加为止。

把每个强连通分量缩成一个点，得到的新图就没有环了。于是可以先处理被依赖的组，再处理引用它们的组，这正是类型检查器需要的顺序。一个辅助函数如果在递归组之外，就可以先泛化自己的类型，等调用方用到它时，多态已经可用了[^haskell2010]。

给刚才的例子加上 `id x = x`，并把入口改成 `main = id (even 4)`。依赖图上就有了 `main → id`、`main → even`、`even → odd`、`odd → even` 四条边。`id` 自成一组，可以和 `even`、`odd` 的那一组各自检查。两边都处理好，才轮到 `main`。`id` 那组一完成，就能泛化。要是干脆把整个声明块塞进同一个组，那么在检查这个块期间，各处调用 `id` 都得共享一个暂定类型，平白失去了本来可以使用的多态。

在递归组内部，先给每个绑定分一个暂定的单型。有了这些临时类型，就可以推断组里所有函数体，再把各自的推断结果和暂定类型合一。等整组检查完才泛化。出了这个组，每次用到绑定时都可以独立实例化得到的类型方案。这一章的签名只用来核对已推断出的类型，由签名指导的多态递归留到第 14 章。

## 实现

### 把名字的每次使用对应到定义

表层语法树把整个局部声明块原样留到名称解析阶段。名称解析器会先给块里的**所有**定义分配身份，再去看任何一个定义的右边。这样，前面的函数引用后面定义的函数时，后者已经有身份可查。每遇到一个 lambda，再开一个更小的作用域，把形参放进去。查找从最内层开始，一层层往外，找到第一个符合的定义就停。

```rust
pub fn rename(
    sess: &mut Session,
    program: ast::Program,
) -> Result<Program, Stop> {
    let program = crate::fixity::resolve(sess, program)?;
    for fixity in &program.fixities {
        for op in &fixity.operators {
            if !program.bindings.iter().any(|b| b.name == *op) {
                sess.emit(Diagnostic::error(
                    ErrorCode::OrphanFixity,
                    format!("fixity without a definition: {op}"),
                    fixity.span,
                ));
            }
        }
    }
    let mut primitives = Vec::new();
    let mut prelude = BTreeMap::new();
    if !sess.options.no_prelude {
        for op in [
            Op::Div,
            Op::Mod,
            Op::Add,
            Op::Sub,
            Op::Mul,
            Op::Append,
            Op::Eq,
            Op::Lt,
            Op::Le,
        ] {
            let id = sess.fresh_id(op.text());
            prelude.insert(op.text().into(), id.clone());
            primitives.push((op, id));
        }
    }
    let mut renamer = Renamer {
        sess,
        scopes: vec![prelude],
    };
    let groups = renamer.bindings(program.bindings);
    let main = renamer.lookup("main", program.span);
    renamer.sess.finish(Program {
        groups,
        primitives,
        main,
        span: program.span,
    })
}
```

[源码 · rename.rs 第 201～248 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch09_rename/src/rename.rs#L201-L248)

解析过名字的树是一套单独的中间表示（IR）。每次名字出现的位置都存一个 `Id`，里面既有 `Name`，也有报错要用的拼写。打印这棵树时会看到 `x#12` 这样的名字。多留一点文字，即使原来的源码会话已经结束，打印出的树仍然好读。程序比较两个名字时，分配给绑定的身份也参与比较。

遍历期间可以收集多个错误。如果某处找不到定义，先给它一个新的占位标识，这样还可以继续检查别的函数体有没有作用域错误。名称解析结束时，只要发现过这种错误，驱动程序就会停下，求值器没有机会见到占位标识。

### 用 Tarjan 算法找出必须一起处理的组

Tarjan 算法遍历图时给每个点一个访问序号，还要记录一个最低序号，表示这次搜索通过相关路径能回到栈中多早的活跃顶点。如果某点的最低序号等于它自己的访问序号，它就是一个完整强连通分量的根。从栈顶一直弹到这个点，弹出来的点就属于同一组。

```rust
fn visit(&mut self, v: usize) {
    self.index[v] = Some(self.next);
    self.low[v] = self.next;
    self.next += 1;
    self.stack.push(v);
    self.active[v] = true;
    for &w in &self.edges[v] {
        if self.index[w].is_none() {
            self.visit(w);
            self.low[v] = self.low[v].min(self.low[w]);
        } else if self.active[w] {
            self.low[v] = self.low[v].min(self.index[w].unwrap());
        }
    }
    if self.low[v] == self.index[v].unwrap() {
        let mut group = Vec::new();
        loop {
            let w = self.stack.pop().unwrap();
            self.active[w] = false;
            group.push(w);
            if w == v {
                break;
            }
        }
        group.sort_unstable();
        self.result.push(group);
    }
}
```

[源码 · scc.rs 第 16～43 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch09_rename/src/scc.rs#L16-L43)

我们画的箭头是从使用者指向它依赖的绑定。因此，分量完成的顺序恰好是类型推断所需的顺序。遍历时按固定的源码下标访问，这样每次打印出的结果都能复现。只含一个绑定的分量也可能是递归的，但前提是它有一条指回自己的边。为了检验 Tarjan 算法的记账有没有出错，测试还会直接检查点与点之间是否相互可达。这种笨办法虽然慢，分出来的组却应该和 Tarjan 算法完全一致。

### 以组为单位推断类型

推断递归组时，环境里放着组内所有成员的暂定类型。等每个函数体都交出了自己的约束，才开始泛化。接着用签名核对推断得到的类型方案。如果某个签名把类型收窄了，这个变化还要传到组内共享同一类型的位置，最后才能定下要向外部提供的类型方案。

```rust
fn group(
    &mut self,
    env: &mut Vec<(Id, Scheme)>,
    group: &BindGroup,
) -> Result<(), Diagnostic> {
    self.level += 1;
    let size = env.len();
    let provisional: Vec<_> = group
        .binds
        .iter()
        .map(|b| (b.id.clone(), self.fresh()))
        .collect();
    if group.recursive {
        env.extend(
            provisional
                .iter()
                .map(|(id, ty)| (id.clone(), Scheme::mono(ty.clone()))),
        );
    }
    let inferred = group
        .binds
        .iter()
        .zip(&provisional)
        .map(|(b, (_, expected))| {
            let ty = self.infer(env, &b.rhs)?;
            if group.recursive {
                self.unify(expected, &ty, &b.rhs)?;
            }
            Ok(ty)
        })
        .collect::<Result<Vec<_>, Diagnostic>>();
    env.truncate(size);
    self.level -= 1;
    let inferred = inferred?;
    for (binding, ty) in group.binds.iter().zip(&inferred) {
        if let Some(annotation) = &binding.annotation {
            let scheme = self.generalize(ty.clone());
            self.check_signature(&scheme, annotation, &binding.rhs)?;
        }
    }
    self.level += 1;
    for (binding, ty) in group.binds.iter().zip(&inferred) {
        if let Some(annotation) = &binding.annotation {
            let declared = self.instantiate(annotation);
            self.unify(ty, &declared, &binding.rhs)?;
        }
    }
    self.level -= 1;
    for (binding, ty) in group.binds.iter().zip(inferred) {
        env.push((binding.id.clone(), self.generalize(ty)));
    }
    Ok(())
}
```

[源码 · typecheck.rs 第 200～252 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch09_rename/src/typecheck.rs#L200-L252)

最后这一步不能漏。例如 `f` 和 `g` 互相调用，并且它们共享一个参数类型。给 `f` 写了 `Int -> Int` 签名，这个限制也得约束 `g` 对应的参数。若编译器还把不受这个限制的多态方案当作 `g` 的对外类型，调用方就可能写出 `g True`，类型系统却抓不住这个错误。为此，回归测试专门检查这样的调用会被拒绝。

### 递归函数怎样拿到彼此的闭包

解释器给每个递归组保存一份不可变的“配方”，里面是组内各个 lambda 的函数体和定义时的外层环境。调用其中任何一个闭包时，先照配方把整组绑定放回环境，再绑定本次调用的实参。这样 `even` 进了函数体就找得到 `odd`，`odd` 也找得到 `even`。闭包无需保存指向自己的可变指针，引用计数也不会因为它们互相持有而形成环。

```rust
fn recursive_values(bindings: Rc<Vec<Bind>>, env: Rc<Env>) -> Env {
    let mut values = Vec::new();
    for binding in bindings.iter() {
        let Expr::Lam(parameter, body) = &*binding.rhs else {
            unreachable!("recursive lambda invariant")
        };
        let value = Value::Closure {
            parameter: parameter.clone(),
            body: Rc::new(*body.clone()),
            env: env.clone(),
            recursive: Some(bindings.clone()),
        };
        values.push((binding.name.clone(), value));
    }
    values
}
```

[源码 · eval.rs 第 39～54 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch09_rename/src/eval.rs#L39-L54)

顶层值的初始化仍然只进行一次，顺序现在由依赖决定。建立函数闭包时，不会提前运行函数体。Core 的 lint 原先会检查作用域、原语的实参是否传齐以及递归绑定右边是否为 lambda，现在还会检查各个绑定的身份是否真的唯一。

## 跑起来看看

```text
$ cd crates/ch09_rename
$ cargo run -q -- run tests/programs/rename/mutual.fun
True
$ cargo run -q -- dump --pass=rename tests/programs/fun-forward.fun
nonrec { later#10 = 1; }
nonrec { main#9 = later#10; }
$ cargo xtask check-inherit 09
chapter 09: 101 inherited programs verified and chapter tests passed
```

找个小程序，把辅助函数挪到调用者下面，程序现在也能工作。再让两个函数互相调用，查看名称解析阶段打印的结果。你会看到它们被列进同一个递归组，告诉类型检查器这两个定义需要一起推断。

## 练习

1. **基础。** 写一个内外多层都有 `x` 的例子，在每处 `x` 旁边标出它指向哪个定义，再和名称解析阶段的输出核对。
2. **进阶。** 画出一个包含三个函数的递归组，再加一个非递归的辅助函数。画好依赖边，预测算法会按什么顺序输出各个分量。
3. **进阶。** 在一条诊断测试里放两个互不相干的拼写错误，检查名称解析阶段能在同一次运行里把两个错误都报出来。
4. **挑战。** 即使组里的一个函数会返回另一个闭包，为什么用不可变配方表示闭包也不会制造引用计数环？
5. **挑战。** 为了支持由签名指导的多态递归，当前划分依赖组、检查各组的办法，哪些地方必须调整？

## 延伸阅读

《Haskell 语言报告》解释了绑定组和依赖分析[^haskell2010]。关于变量、作用域和绑定的理论基础，可以读 Pierce 的著作[^pierce2002]。

[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。
[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。

---

[← 第 8 章　缩进布局和扩展语法](ch08-layout.md) · [中文目录](../README.md) · [第 10 章　带类型的 Core 与 Core Lint →](ch10-typed-core.md)
