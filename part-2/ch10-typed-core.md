# 第 10 章　带类型的 Core 与 Core Lint

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch10-typed-core.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch10_core)

> **新增文件**
>
> - `src/core_lint.rs`
> - `src/core_type.rs`
>
> **改动文件**
>
> - `src/core.rs`
> - `src/elaborate.rs`
> - `src/typecheck.rs`
> - `src/eval.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/rename.rs`
> - `src/renamed.rs`

## 源程序通过了检查，编译器就一定翻译对了吗？

类型检查器已经接受了我们的程序。可是接下来，编译器要动手改写它。编译器自己也可能出错，比如把实参传错位置，漏掉一次类型实例化，或者让两个本该不同的变量共用一个身份。源程序再正确，也经不起这样的翻译。

这一章给 Core 加上显式类型，再为它写一个独立的检查器，叫作 **Core Lint**。它的工作比类型推断简单得多。推断器要找答案，Lint 拿到的树上已经写好了答案，它只要逐项核对，看这些答案能不能对上。这样，哪一步改写弄坏了程序，我们就有机会在那一步之后把问题揪出来。

## 现在能写什么

写多态函数时，我们依旧可以省略类型实参。

```fun
id x = x

number = id 42

main = id True
```

到了 Core 里，`id` 的定义明确地引入一个类型参数。第一次使用 `id`，给它传入类型 `Int`。第二次使用，传入类型 `Bool`。两次传类型的动作，都有对应的中间表示（IR）节点。求值器运行时会跳过这些类型信息，所以这段程序仍然打印 `True`。

## 背后的原理

源代码 `id 42` 省略了一个选择。这里用的是恒等函数面向 `Int` 的那一次实例化。System F 可以把这个选择直接写进程序。**类型抽象**用来引入类型参数，**类型应用**用来为这个参数选定具体类型。Girard 和 Reynolds 分别发展了这种显式表达多态的做法[^girard1972][^reynolds1974]。

拿普通 lambda 和普通函数调用作个对照就好懂了。普通 lambda 写下一个形参，等你给它一个值。类型抽象也写下一个形参，只不过等的是一个类型。这个类比能帮我们读懂下面的规则。等程序真正运行时，类型实参就可以擦掉了。

$$
\frac{\Gamma, \alpha \vdash e : \tau \quad \alpha \notin \mathsf{ftv}(\Gamma)}{\Gamma \vdash \Lambda\alpha.e : \forall\alpha.\tau}\;\text{（类型抽象）}
$$

$$
\frac{\Gamma \vdash e : \forall\alpha.\tau \quad \Gamma \vdash \sigma : *}{\Gamma \vdash e[\sigma] : \tau[\sigma/\alpha]}\;\text{（类型应用）}
$$

第一条规则说，如果允许在表达式里使用新类型变量 $\alpha$，就可以检查表达式 $e$，然后用 $\Lambda$ 把这个类型参数绑定起来。这里的 $\mathsf{ftv}(\Gamma)$ 指环境中自由出现的类型变量。第二条规则说，给一个具有 $\forall\alpha.\tau$ 类型的表达式传入类型 $\sigma$，所得类型就是把 $\tau$ 中自由出现的 $\alpha$ 换成 $\sigma$ 的结果。

本章所有被绑定的类型变量，其种类（kind）都是 `*`。等到支持高阶类型构造器时，我们再扩充种类判断。普通函数应用要求函数位置具有箭头类型。如果函数类型外面还有全称量词，要先传入类型实参，才能给它传普通值。

Core 能表示完整的 System F，Fun 源语言目前只开放 Hindley–Milner（HM）多态。这条界线可以通过手工构造 Core 项来测试，源语言无须增加语法[^pierce2002]。

## 动手实现

### Core 里有哪些数据类型

先不看 Rust，用一小段文法列出 Core 表达式的所有形态。每条竖线都读作“或者”。左边写表达式 $e$ 的形式，右边说明它的用途。

$$
\begin{aligned}
e & = x & \quad & \text{局部变量} \\
  & \mid g & \quad & \text{全局引用} \\
  & \mid \ell & \quad & \text{字面量} \\
  & \mid e_1\;e_2 & \quad & \text{应用} \\
  & \mid \lambda x : \tau.\;e & \quad & \text{项抽象} \\
  & \mid \Lambda\alpha.\;e & \quad & \text{类型抽象} \\
  & \mid e[\tau] & \quad & \text{类型应用} \\
  & \mid \text{let}\;b\;\text{in}\;e & \quad & \text{绑定组} \\
  & \mid \text{if}\;e_1\;\text{then}\;e_2\;\text{else}\;e_3 & \quad & \text{条件表达式} \\
  & \mid p[\overline{\tau}](\overline{e}) & \quad & \text{原语操作}
\end{aligned}
$$

这里的 $x$ 指局部值，$g$ 指全局值，$\ell$ 是字面量，$p$ 是原语（内建操作）。字母上面加横线表示一串东西，例如 $\overline{e}$ 表示按求值顺序排好的一串操作数。为了让记号简洁，文法略去了源码区间，Rust 实现仍然保留它们，好让报错能指回源码。

类型和绑定组也各有各的写法。

$$
\begin{aligned}
\tau & = \text{Int}\mid\text{Bool}\mid\text{Char}\mid\text{String} & \quad & \text{基础类型} \\
     & \mid \alpha & \quad & \text{类型变量} \\
     & \mid \tau_1 \to \tau_2 & \quad & \text{函数类型} \\
     & \mid \forall\alpha.\;\tau & \quad & \text{全称类型} \\
b & = x : \tau = e & \quad & \text{非递归绑定} \\
  & \mid \text{rec}\;\{\overline{x : \tau = e}\} & \quad & \text{递归组}
\end{aligned}
$$

在递归组里，任意一个绑定的右侧都能引用这组里的所有名字。Lint 还会检查每个右侧确实是函数，外面可以包若干层类型抽象。在非递归绑定里，新名字只在 `let` 的主体中可用，定义自己的右侧还看不到它。

留意一大一小两个 lambda。$\Lambda\alpha.\;\lambda x : \alpha.\;x$ 中，大写的 $\Lambda$ 引入类型参数，普通的 $\lambda$ 随后用这个参数标明 $x$ 的类型。先给这个恒等函数传类型，再传值，就得到 $(\Lambda\alpha.\;\lambda x : \alpha.\;x)[\mathsf{Int}]\;42$。这就是章首那句 `id 42` 在 Core 里的全貌。

用 Rust 实现时，Core 表达式是一个枚举。每个变量引用，以及每个引入项变量的绑定位置，都带着一个 `Id`，里面记着类型。因此，Lint 无须重跑推断就能检查这棵树。

```rust
/// A Core identifier carries its independently checkable type.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Id {
    pub name: crate::names::Id,
    pub ty: Type,
}
```

[源码 · core.rs 第 5～10 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch10_core/src/core.rs#L5-L10)

表达式构造器明确区分两层操作。在值这一层，`Lam` 定义函数，`App` 应用函数。在类型这一层，`TyLam` 做类型抽象，`TyApp` 做类型实例化。`Let` 保留重命名阶段划分好的递归分量。`Prim` 带着显式类型实参、求值时要计算的操作数，以及失败时用来定位错误的源码区间。

```rust
#[derive(Clone, Debug)]
/// System F Core makes type abstraction and instantiation explicit.
pub enum Expr {
    Var(Id),
    Global(Id),
    TyLam(u32, Box<Expr>),
    TyApp(Box<Expr>, Type),
    Lit(Lit),
    Lam(Id, Box<Expr>),
    App(Box<Expr>, Box<Expr>),
    Let(BindGroup, Box<Expr>),
    If(Box<Expr>, Box<Expr>, Box<Expr>),
    Prim(Op, Vec<Type>, Vec<Expr>, Span),
}
```

[源码 · core.rs 第 28～41 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch10_core/src/core.rs#L28-L41)

一个绑定组把带类型的名字和对应的右侧表达式放在一起。整个程序则由按顺序排列的绑定组，以及一个入口点组成。

```rust
#[derive(Clone, Debug)]
/// A recursive binding is guaranteed to have a lambda right-hand side.
pub struct Bind {
    pub name: Id,
    pub rhs: Box<Expr>,
}
#[derive(Clone, Debug)]
/// An ordered component; recursive members are in scope in every right-hand side.
pub struct BindGroup {
    pub recursive: bool,
    pub binds: Vec<Bind>,
}
#[derive(Clone, Debug)]
/// Globals are initialized in dependency order, with explicit recursive components.
pub struct Program {
    pub groups: Vec<BindGroup>,
    pub main: Id,
}
```

[源码 · core.rs 第 43～48 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch10_core/src/core.rs#L43-L48) · [源码 · core.rs 第 50～55 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch10_core/src/core.rs#L50-L55) · [源码 · core.rs 第 57～62 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch10_core/src/core.rs#L57-L62)

这个版本的 Core 仍然有 `If`。第 11 章才引入数据构造器和模式匹配，第 16 章再引入扁平的 case 分支和汇合点（join point）。先把枚举控制在这个规模，就能专心把带类型的翻译和 Lint 做好。

### 记下类型推断做过的选择

经过重命名的表达式现在都有稳定的节点编号。每次出现变量时，类型检查器会记下为这次使用新建的类型实参。它还会记下每个绑定的类型方案，以及每个 lambda 参数的单态类型。推断结束前，把这些类型都沿着元变量已求出的解整理一遍，这一步称作 **zonk**。这样，后续生成 Core 时就能直接使用结果，不必再访问可变的元变量存储。

递归组里有一点特殊。推断过程中，组内引用最初只能看到一个临时的单态类型。等整个分量泛化完，再生成 Core，就用相应的泛化变量对这些递归引用做实例化。普通引用则直接使用推断时记下的新类型实参。

### 把隐含的多态写进 Core

每个被泛化的元变量都会得到一个新的 Core 类型标识。生成绑定的类型时，先转换单态类型，再在外面包上 `Forall`。生成右侧表达式时，给项包上与量词一一对应的 `TyLam`。局部绑定的类型方案仍然可能引用外层函数的类型抽象所绑定的变量。

```rust
fn group(
    &mut self,
    group: &renamed::BindGroup,
    outer: &BTreeMap<u32, u32>,
    global: bool,
) -> BindGroup {
    for binding in &group.binds {
        self.binder(
            binding.id.clone(),
            &self.typed.schemes[&binding.id],
            outer,
            global,
        );
    }
    let binds = group
        .binds
        .iter()
        .map(|binding| {
            let binder = self.binders[&binding.id].clone();
            let mut variables = outer.clone();
            variables.extend(binder.quantifiers.iter().copied());
            let mut rhs = self.expression(&binding.rhs, &variables);
            for (_, variable) in binder.quantifiers.iter().rev() {
                rhs = Expr::TyLam(*variable, Box::new(rhs));
            }
            Bind {
                name: binder.id,
                rhs: Box::new(rhs),
            }
        })
        .collect();
    BindGroup {
        recursive: group.recursive,
        binds,
    }
}
```

[源码 · elaborate.rs 第 78～113 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch10_core/src/elaborate.rs#L78-L113)

每处需要实例化的引用会加上 `TyApp`，每个 lambda 参数也会带上转换后的类型。Core 区分全局引用和局部变量。Lint 会拿类型标注去和环境里的声明核对。在错误的变量引用上贴一个看似合理的类型，也蒙混不过去。

有些内部类型变量不出现在任何一层外围的类型方案里，这时候要给它选一个没有自由变量的具体类型作见证。例如 `(\f -> 42) (\x -> x)`，返回值始终是整数，整个表达式没有交代右边恒等函数的参数类型。我们用 `Int` 填上这个最终会擦掉的内部标注。源表达式的类型推断规则照旧，有歧义的类型类约束也不会靠这一步解决。

### 让 Lint 独立把关

检查器分别维护项变量和类型变量的作用域，还会记住已经用过的绑定标识。它检查函数应用两侧的类型、原语是否收齐参数、条件表达式的分支、递归函数组，以及绑定的类型标注。下面是它处理显式类型抽象和类型应用的代码。

```rust
Expr::TyLam(variable, body) => {
    if !self.seen_types.insert(*variable) {
        return Err(format!(
            "duplicate Core type binder t{variable}"
        ));
    }
    self.types.insert(*variable);
    let body = self.expression(body)?;
    self.types.remove(variable);
    Ok(Type::Forall(*variable, Box::new(body)))
}
Expr::TyApp(fun, argument) => {
    self.well_formed(argument)?;
    let function = self.expression(fun)?;
    let Type::Forall(variable, body) = function else {
        return Err(format!(
            "Core type application of non-polymorphic type {function}"
        ));
    };
    Ok(body.substitute(variable, argument))
}
```

[源码 · core_lint.rs 第 85～105 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch10_core/src/core_lint.rs#L85-L105)

类型应用要在量词的主体里做替换。类型实参中的自由变量不能因为替换而被里面的量词误绑定。即使 Core 项是别人手工造出来的，替换也要能处理它。遇到会冲突的绑定标识，就先换一个全新的。

```rust
pub fn substitute(&self, variable: u32, replacement: &Type) -> Type {
    match self {
        Self::Var(v) if *v == variable => replacement.clone(),
        Self::Arrow(a, b) => Self::arrow(
            a.substitute(variable, replacement),
            b.substitute(variable, replacement),
        ),
        Self::Forall(v, _) if *v == variable => self.clone(),
        Self::Forall(v, body) => {
            let (v, body) = if replacement.free().contains(v) {
                let fresh = self
                    .largest()
                    .max(replacement.largest())
                    .max(variable)
                    .checked_add(1)
                    .expect("type variable supply exhausted");
                (fresh, body.substitute(*v, &Self::Var(fresh)))
            } else {
                (*v, *body.clone())
            };
            Self::Forall(
                v,
                Box::new(body.substitute(variable, replacement)),
            )
        }
        _ => self.clone(),
    }
}
```

[源码 · core_type.rs 第 47～74 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch10_core/src/core_type.rs#L47-L74)

判断两个类型是否相等，用的是 **α 等价**。例如两个 `forall` 类型即使给绑定变量取了不同的名字，也可以表示同一个类型。自由类型变量就不同了，它们得指向同一个外层绑定才算相等。

```rust
pub fn lint(program: &Program) -> Result<(), String> {
    let mut checker = Checker::default();
    for group in &program.groups {
        checker.group(group, true)?;
    }
    checker.expression(&Expr::Global(program.main.clone()))?;
    Ok(())
}
```

[源码 · core_lint.rs 第 182～189 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch10_core/src/core_lint.rs#L182-L189)

> **试想一个编译器错误。** 假如生成 Core 时漏掉了多态恒等函数上的 `TyApp`，Lint 会报 `Core application of non-function type forall t0. t0 -> t0`。源程序完全可能是对的。出错的地方在生成的 Core 里，它试图把值直接传给一个还在等类型参数的表达式。测试可以直接造出这种有错的 Core，检查 Lint 会不会拦住它，无须再跑一遍源语言的类型推断。

### 擦掉类型之后，仍然严格求值

`TyLam` 和 `TyApp` 都不会分配运行时对象。解释器走到它们时直接进入里面的表达式，不代入任何运行时值。这里做的是**类型擦除**，执行程序沿用原来的严格求值策略。

尤其要注意，类型抽象不会推迟严格绑定右侧的求值。下面的多态绑定虽然名叫 `unused`，也确实无人使用，程序仍然会在选中 `main` 之前因除零而失败。

```fun
unused = (\x -> \y -> y) (div 1 0)

main = 42
```

第 18 章会专门定义一种已经擦除类型的 IR。目前运行整套样例程序时，就已经启用了 `--lint`。

## 跑起来

```text
$ cd crates/ch10_core
$ cargo run -q -- run tests/programs/core/self-instantiation.fun --lint
42
$ cargo run -q -- dump --pass=core --no-prelude tests/programs/layout/signature.fun
nonrec { id#0 : forall t0. t0 -> t0 = (type-lambda t0 (lambda x#2 : t0 x#2)); }
nonrec { main#1 : Int = (app (type-app @id#0 Int) 42); }
$ cargo test -p ch10_core --test system_f
```

可以挑一个很小的 Core 项，故意删掉一次类型应用，再跑 Lint。这次实验不经过源语言类型检查器。我们专门给第二道检查送去一个编译器才会制造的错误，看看它能不能在求值之前拒绝这个项。

## 练习

1. **基础。** 导出 `id id 42` 的 Core，指出其中每一次类型抽象和类型应用。
2. **进阶。** 局部绑定捕获了外层的一个类型变量。为什么这个变量在局部绑定的类型方案中是自由的，放回完整的 Core 程序中却是被绑定的？
3. **进阶。** 构造一个错误的 Core 原语调用，让参数个数正确，某个操作数的类型却不对。
4. **挑战。** 去掉类型替换中给绑定变量换新标识的步骤，举例演示一次变量捕获。再解释为什么单靠 α 等价补救不了它。

## 延伸阅读

Girard 的博士论文和 Reynolds 的论文是显式多态的原始文献[^girard1972][^reynolds1974]。关于 System F 的类型规则和替换规则，可以读 Pierce 的著作[^pierce2002]。

[^girard1972]: Jean-Yves Girard（1972）。[Interprétation fonctionnelle et élimination des coupures de l’arithmétique d’ordre supérieur](https://girard.perso.math.cnrs.fr/These.pdf)。
[^reynolds1974]: John C. Reynolds（1974）。[Towards a Theory of Type Structure](https://doi.org/10.1007/3-540-06859-7_148)。
[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。

---

[← 第 9 章　名称解析](ch09-renamer.md) · [中文目录](../README.md) · [第 11 章　数据类型与模式匹配 →](ch11-datatypes.md)
