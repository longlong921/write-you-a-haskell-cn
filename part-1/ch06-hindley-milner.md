# 第 6 章　Hindley–Milner 类型推断

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch06-hindley-milner.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch06_poly)

> **新增文件**
>
> - `src/algorithm_w.rs`
> - `src/union_find.rs`
>
> **改动文件**
>
> - `src/types.rs`
> - `src/ast.rs`
> - `src/cbn.rs`
> - `src/cek.rs`
> - `src/eval.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/names.rs`
> - `src/normal.rs`

## 为什么要学类型推断

看看 `\x -> x`。我们没说 `x` 是什么类型，可光看函数体就知道，传进来什么，它就原样返回什么。因此，它的类型一定形如 `a -> a`。两个 `a` 指的是同一个类型。

一次都不用运行这个函数，我们就知道了它每次调用都满足什么条件。传整数、布尔值，甚至传另一个函数，都行。**类型方案**（type scheme）可以把这层意思写下来，允许每处使用这个函数的代码各自选择 `a`。Hindley–Milner 类型推断要找的，就是程序所允许的最一般的类型方案[^damas1982]。

我们会写两套推断器。第一套叫算法 W（Algorithm W），每确定一条“把这个类型变量替换成那个类型”的信息，就把它记在替换表里[^damas1982]。第二套直接更新可变的存储单元，再用层级（level）判断哪些变量可以泛化。两者解的是同一组方程。把两种做法都走一遍，再决定给 Fun 用哪一种，取舍的理由就更清楚了。

## 这一章的语言长什么样

这一章的小语言叫 Poly。它支持省略参数类型标注的 lambda、函数应用、多态的 `let`、只允许绑定函数的 `let rec`，还有整数、布尔值、条件表达式和算术运算。上一章介绍的显式 `fix` 也保留下来，方便做对照实验。

```fun
let id = \x -> x in
let first = id 1 in
id True
```

第一次用 `id` 时传入 `1`，第二次传入 `True`。每次使用都能独立地把 `id` 的类型方案实例化。lambda 的参数遵守另一条规则。试着写 `\f -> let first = f 1 in f True`，推断器就会拒绝它，因为在这个 lambda 的整个函数体里，所有 `f` 必须共用同一个类型。

现在可以把上一章写在参数上的类型标注去掉了。尤其是恒等函数，以前得在“接收整数”和“接收布尔值”之间选一个，现在一份定义就能兼顾两者。我们沿用已经熟悉的求值器，来单独研究编译器怎么知道这里可以自由选择类型。

## 理论

### 类型规则怎么读

第一次看到满页的类型规则，很容易被符号吓住。其实这些符号只是帮我们省去反复说同一句话的麻烦。每条规则管一种语法。读的时候先看横线下面，弄清要得出什么结论，再看横线上面，找出要对各个组成部分确认什么。Pierce 在介绍类型系统时一直用这种写法[^pierce2002]。

$$
\Gamma \vdash e : \tau
$$

这句话读作“根据 $\Gamma$ 中已有的假设，表达式 $e$ 的类型是 $\tau$。”下面是后面反复用到的记号。

| 记号 | 怎么读 |
| --- | --- |
| $e$ | 程序里的一个表达式，例如 `x`、`42`、`f x`。 |
| $\tau$、$\alpha$、$\beta$ | 表示类型，或者代表某个类型的类型变量。 |
| $\sigma$ | 类型方案，可以用量词绑定一些类型变量。 |
| $\Gamma$ | 类型环境，记录作用域里有哪些名字，以及这些名字已知的类型或类型方案。 |
| $\Gamma, x : \tau$ | 在现有环境里增加一个局部假设，即 `x` 的类型是 $\tau$。 |
| $\vdash$ | “根据左边这些假设，可以得出右边的结论。” |
| $\alpha \to \beta$ | 函数接收一个 $\alpha$ 类型的值，返回一个 $\beta$ 类型的值。 |
| $\forall \alpha.\; \alpha \to \alpha$ | 使用这个方案时可以自行选择用什么类型替换 $\alpha$，但同一次使用里两处都要换成同一个类型。 |

冒号用来断定一个表达式的类型。写下这个判断，无需对表达式求值。$\Gamma$ 记录编译时对名字类型的假设，运行时的值另有存放之处。本章里，`let` 绑定的名字对应类型方案，lambda 参数对应单型（monotype）。为了统一表示，代码把单型存成没有量词的方案。规则里的希腊字母是数学上的占位符。等到实现推断器，我们才需要可以修改的**元变量**（metavariable），用它们一步步求出满足规则的类型。

横线读作“只要上面的条件都成立，下面的结论就成立”，与除法无关。如果横线上面空着，这条规则就不需要前提。例如，光看整数字面量 `42` 本身就知道它的类型是 `Int`，没有子表达式需要先检查。

$$
\frac{}{\Gamma \vdash 42 : \mathsf{Int}}\;\text{(Int)}
$$

函数应用的规则有两个前提。读的时候，留意它们是怎么通过同一个输入类型联系起来的。

$$
\frac{\Gamma \vdash e_1 : \alpha \to \beta \quad
\Gamma \vdash e_2 : \alpha}
{\Gamma \vdash e_1\ e_2 : \beta}\;\text{(App)}
$$

例如，`e1` 接收整数并返回布尔值，那么 `e2` 就必须是整数，整个调用的结果就是布尔值。公式里两次出现的 $\alpha$，正好表达了实参与形参类型必须相同。推断器靠**合一**（unification）找出怎样才能让两者相同。规则本身只描述什么样的类型关系成立，求解两个前提的先后顺序留给实现决定。

### 动手推导一个类型

来推导 `\x -> x + 1` 的类型。先给 `x` 一个待定的类型。加法要求两边都是整数，于是 `x` 的类型也就定成了 `Int`。为让推导树写起来短一点，把带有这条局部假设的环境记作 $\Delta = \Gamma, x : \mathsf{Int}$。

$$
\frac{
  \frac{x : \mathsf{Int} \in \Delta}{\Delta \vdash x : \mathsf{Int}}\;\text{(Var)}
  \quad
  \frac{}{\Delta \vdash 1 : \mathsf{Int}}\;\text{(Int)}
}{\Delta \vdash x + 1 : \mathsf{Int}}\;\text{(Add)}
$$

左边的叶子通过查环境确认 `x` 是整数，右边的叶子根据字面量规则确认 `1` 是整数。这样就满足了加法规则 Add 的两个前提。接下来用 lambda 规则，把刚才对参数的局部假设写进函数类型。

$$
\frac{\Gamma, x : \alpha \vdash e : \beta}
{\Gamma \vdash \lambda x.\;e : \alpha \to \beta}\;\text{(Lam)}
$$

这里的 $\lambda x.\;e$ 就是代码里的 `\x -> e`。应用 Lam 规则，我们得到整个函数的类型 `Int -> Int`。走出函数体以后，外层环境无需保留关于局部参数 `x` 的那条假设。如果把 `1` 换成 `True`，Add 所要求的一个前提就不成立，这棵推导树也就搭不起来了。

### 从单型到类型方案

`let id = \x -> x in ...` 有意思的地方在于，用到 `id` 的每个位置都能选择自己的参数类型。我们用**类型方案**来记录这种自由。**单型**描述一个类型，其中可以有暂时还没解出来的变量。类型方案则进一步标出，哪些变量允许调用方每次重新选择。每用一次方案，就把这些变量换成一组全新的变量，这叫**实例化**（instantiation）。反过来，要从推断出的类型产生方案，得先判断哪些变量可以放心交给调用方选择，这叫**泛化**（generalization）[^damas1982]。

如果外层作用域还依赖某个类型变量，就不能把它泛化。想象一个局部函数，内部用到了外层的值，而这个值的类型已经受到约束。要是还让每个调用者自由指定这个类型，函数就等于许下了兑现不了的承诺。所以，lambda 参数一开始只有一个单型，在函数体里用它多少次，都共享这一个类型。

$$
\frac{\Gamma \vdash e_1 : \tau \quad
\sigma = \mathsf{gen}(\Gamma, \tau) \quad
\Gamma, x : \sigma \vdash e_2 : \beta}
{\Gamma \vdash \mathsf{let}\ x = e_1\ \mathsf{in}\ e_2 : \beta}
\;\text{(Let)}
$$

Let 规则分三步读。先推断等号右边 $e_1$ 的类型，再相对于环境把它泛化，最后把得到的方案绑定给 `x`，在扩充后的环境里检查主体 $e_2$。中间那个关于 $\sigma$ 的等式用来定义方案，是规则的附带条件，不需要执行一段额外的程序。一个类型变量出现在类型或环境里，又没有被方案的量词绑定，我们就说它在其中是**自由的**。右侧类型里的自由变量，只有在环境里没有自由出现的那些，才可以泛化。

回到 `let id = \x -> x in ...`。等号右边的类型是 $\alpha \to \alpha$。只要 $\alpha$ 与外层环境无关，就可以给 `id` 绑定方案 $\forall \alpha.\;\alpha \to \alpha$。以后每次查找 `id`，都重新实例化一次。例如，`id 1` 选 `Int`，`id True` 选 `Bool`。而在 `\f -> ...` 里，Lam 规则给参数 `f` 的是一个单型，所以它的所有用法都得满足同一个类型。给类型变量加上量词，是 `let` 规则提供的本领，单凭一个名字碰巧指向函数，并不会得到这种多态性。

合一要解决的是“两个类型怎样才会相等”。遇到两个箭头类型，先让两者的参数类型合一，把由此得到的替换应用到返回类型上，再让返回类型合一。要用一个复合类型解出某个变量，必须先检查这个变量没有出现在那个类型的内部。这个步骤叫**出现检查**（occurs check）。例如在 `x x` 里，若 `x` 的类型是 `a`，它既当实参又当函数，就要求 `a` 等于 `a -> b`。把右边的 `a` 再展开，永远也写不完。推断器会报 `E0202`。若只是两个基本类型不相容，例如要求 `Int` 等于 `Bool`，报的是 `E0201`。

检查递归函数体时，先给这个函数一个临时的单型，函数调用自己时也只能用它。等整个函数体检查完，外面的 `let` 才能做泛化。实现里把递归改写成 `fix`，函数通过一个普通的单型参数引用自己，这就落实了这条规则。本章不支持多态递归。按环境做泛化的规则，以及后面更高效的按层级做泛化的办法，可参看 Kiselyov 和 Rémy 的工作[^kiselyov2013][^remy1992]。

Poly 没有可变状态，所以不必要求 `let` 的右边在语法上已经是一个值才允许泛化。举个例子，把恒等函数应用到另一个恒等函数上，虽然右边是个函数调用，它的结果仍然可以绑定为多态的名字。后面构建的 Fun 也允许这样的绑定泛化。

### 两次函数调用能告诉我们什么

看看函数组合这短短一行定义里藏了多少类型信息。

```fun
let compose = \f -> \g -> \x -> f (g x) in
compose (\x -> x + 1) (\x -> x * 2) 20
```

运行时先把 `20` 乘 `2`，再加 `1`，结果是 `41`。可单看 `compose` 的定义，里面压根儿没有整数。它的**主类型**（principal type，也就是最一般的类型）是 `(b -> c) -> (a -> b) -> a -> c`。`g` 返回的值马上要作为参数交给 `f`，所以这两个位置必须共用一个 `b`。至于 `a`、`b`、`c` 彼此是否相同，定义里没有任何要求。函数体里只有 `g x` 和 `f (g x)` 两次调用，推断器就能读出这整套关系。

## 实现

两种算法共用一套不可变的数据结构来表示输出的类型和方案。至于怎么遍历表达式、怎么做合一，各写各的。等推断结束，再按类型变量第一次出现的顺序给它们统一编号。比如一套算法得出 `a -> a`，另一套得出 `b -> b`，差别只在变量名。这样整理之后，就能直接比较两个主类型。

```rust
/// Variables are fresh identities; algorithms choose their own representation of solutions.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum Type {
    Int,
    Bool,
    Var(u32),
    Arrow(Box<Type>, Box<Type>),
}

/// Quantified variables are freshened independently at every occurrence.
#[derive(Clone, Debug)]
pub struct Scheme {
    pub variables: Vec<u32>,
    pub ty: Type,
}

impl Scheme {
    pub fn mono(ty: Type) -> Self {
        Self {
            variables: vec![],
            ty,
        }
    }
}

impl Type {
    pub fn arrow(a: Self, b: Self) -> Self {
        Self::Arrow(Box::new(a), Box::new(b))
    }
    /// Rename by first occurrence, making principal types comparable up to alpha-renaming.
    pub fn canonical(&self) -> Self {
        fn go(ty: &Type, names: &mut BTreeMap<u32, u32>) -> Type {
            match ty {
                Type::Var(n) => {
                    let next = names.len() as u32;
                    Type::Var(*names.entry(*n).or_insert(next))
                }
                Type::Arrow(a, b) => {
                    Type::arrow(go(a, names), go(b, names))
                }
                other => other.clone(),
            }
        }
        go(self, &mut BTreeMap::new())
    }
}
```

[源码 · types.rs 第 6～51 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch06_poly/src/types.rs#L6-L51)

### 算法 W

`algorithm_w.rs` 把替换、实例化、泛化和推断分别实现。Poly 的算术、条件表达式和不动点（`fix`）也有相应的推断分支。每个分支都保留源码位置（span），这样报错时能指回引发问题的那段代码。

一次替换就是一张“类型变量 → 类型”的映射表。复合两个替换时，先应用较早的替换，再用较新的替换改写前一步的结果。把替换应用到类型方案时，还要避开方案自己的量词绑定的变量，因为它们只在这个方案里有效。

```rust
impl Subst {
    fn apply(&self, ty: &Type) -> Type {
        match ty {
            Type::Var(n) => {
                self.0.get(n).map_or_else(|| ty.clone(), |t| self.apply(t))
            }
            Type::Arrow(a, b) => Type::arrow(self.apply(a), self.apply(b)),
            other => other.clone(),
        }
    }
    /// self after earlier: every earlier range is rewritten by the newer substitution.
    fn compose(&self, earlier: &Self) -> Self {
        let mut result: BTreeMap<_, _> =
            earlier.0.iter().map(|(n, t)| (*n, self.apply(t))).collect();
        for (n, t) in &self.0 {
            result.entry(*n).or_insert_with(|| t.clone());
        }
        Self(result)
    }
    fn environment(&self, env: &Env) -> Env {
        env.iter()
            .map(|(name, scheme)| {
                let mut filtered = self.clone();
                for n in &scheme.variables {
                    filtered.0.remove(n);
                }
                (
                    name.clone(),
                    Scheme {
                        variables: scheme.variables.clone(),
                        ty: filtered.apply(&scheme.ty),
                    },
                )
            })
            .collect()
    }
}
```

[源码 · algorithm_w.rs 第 16～52 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch06_poly/src/algorithm_w.rs#L16-L52)

处理函数应用时，先推断函数部分。从中得到的新信息要更新到环境里，再用这个环境推断实参。接着给调用结果取一个全新的类型变量，用“实参的类型 → 这个新变量”构造箭头类型，再让它与函数的类型合一。

```rust
Kind::App(f, a) => {
    let (s1, tf) = self.infer(env, f)?;
    let (s2, ta) = self.infer(&s1.environment(env), a)?;
    let result = self.fresh();
    let s3 = unify(
        &s2.apply(&tf),
        &Type::arrow(ta, result.clone()),
        expr,
    )?;
    Ok((s3.compose(&s2.compose(&s1)), s3.apply(&result)))
}
```

[源码 · algorithm_w.rs 第 140～150 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch06_poly/src/algorithm_w.rs#L140-L150)

处理 `let` 时，先用推断等号右侧得到的替换更新环境，然后才在这个环境里做泛化。要是拿旧环境做判断，就可能把一个已经受外层约束的变量误当成可自由选择的变量，放进类型方案里。

```rust
Kind::Let(n, value, body) => {
    let (s1, t) = self.infer(env, value)?;
    let mut local = s1.environment(env);
    local.insert(n.clone(), generalize(&local, s1.apply(&t)));
    let (s2, result) = self.infer(&local, body)?;
    Ok((s2.compose(&s1), result))
}
```

[源码 · algorithm_w.rs 第 153～159 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch06_poly/src/algorithm_w.rs#L153-L159)

### 用可变单元和层级记录推断结果

`union_find.rs` 不再维护显式的替换表，而是为每个元变量分配一个可更新的单元。单元里保存着创建时的层级，还能存放这个变量已经解成的类型（尚未求解时留空）。沿着已经求解的变量去找最终类型时，推断器顺手做**路径压缩**，让以后再来查询的人少走几步。

```rust
#[derive(Clone)]
struct Meta {
    level: u32,
    solution: Option<Type>,
}

#[derive(Default)]
struct Infer {
    level: u32,
    metas: Vec<Meta>,
}

impl Infer {
    fn fresh(&mut self) -> Type {
        let n = self.metas.len() as u32;
        self.metas.push(Meta {
            level: self.level,
            solution: None,
        });
        Type::Var(n)
    }
    fn zonk(&mut self, ty: &Type) -> Type {
        match ty {
            Type::Var(n) => {
                match self.metas[*n as usize].solution.clone() {
                    None => ty.clone(),
                    Some(t) => {
                        let t = self.zonk(&t);
                        self.metas[*n as usize].solution = Some(t.clone());
                        t
                    }
                }
            }
            Type::Arrow(a, b) => Type::arrow(self.zonk(a), self.zonk(b)),
            other => other.clone(),
        }
    }
```

[源码 · union_find.rs 第 10～46 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch06_poly/src/union_find.rs#L10-L46)

开始推断 `let` 等号右边的表达式时，先把层级加深一层。如果接下来要把一个较早创建、层级较浅的元变量解成某个类型，先别急着写入。做出现检查时，要顺着候选类型一路遍历，把沿途遇到的、比这个老变量更深的元变量全都降到它的层级。这些变量即使出生在 `let` 里面，现在也已经与外层连上了关系，必须按外层变量看待。

```rust
/// One traversal rejects cycles and prevents younger variables escaping their scope.
fn occurs_and_lower(
    &mut self,
    n: u32,
    level: u32,
    ty: &Type,
    expr: &Expr,
) -> Result<(), Diagnostic> {
    match self.zonk(ty) {
        Type::Var(m) if m == n => Err(types::occurs(expr)),
        Type::Var(m) => {
            self.metas[m as usize].level =
                self.metas[m as usize].level.min(level);
            Ok(())
        }
        Type::Arrow(a, b) => {
            self.occurs_and_lower(n, level, &a, expr)?;
            self.occurs_and_lower(n, level, &b, expr)
        }
        _ => Ok(()),
    }
}
fn unify(
    &mut self,
    a: &Type,
    b: &Type,
    expr: &Expr,
) -> Result<(), Diagnostic> {
    let (a, b) = (self.zonk(a), self.zonk(b));
    match (&a, &b) {
        _ if a == b => Ok(()),
        (Type::Var(n), t) | (t, Type::Var(n)) => {
            self.occurs_and_lower(
                *n,
                self.metas[*n as usize].level,
                t,
                expr,
            )?;
            self.metas[*n as usize].solution = Some(t.clone());
            Ok(())
        }
        (Type::Arrow(a, b), Type::Arrow(c, d)) => {
            self.unify(a, c, expr)?;
            self.unify(b, d, expr)
        }
        _ => Err(types::mismatch(expr, &a, &b)),
    }
}
```

[源码 · union_find.rs 第 49～96 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch06_poly/src/union_find.rs#L49-L96)

离开右侧表达式回到外层，泛化就很简单了。只挑尚未求解、而且层级仍比当前层级深的变量，不必遍历整个类型环境。以后每用一次类型方案，就为其中被量化的变量各建一个新单元。没被量化的变量继续使用原先的单元，所以对它们已有的约束仍然有效。

```rust
Kind::Let(n, value, body) => {
    self.level += 1;
    let result = self.infer(env, value);
    self.level -= 1;
    let scheme = self.generalize(result?);
    env.push((n.clone(), scheme));
    let result = self.infer(env, body);
    env.pop();
    result
}
```

[源码 · union_find.rs 第 181～190 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch06_poly/src/union_find.rs#L181-L190)

试试 `\x -> let y = \z -> x z in ...`。每当有人调用 `y`，传给 `y` 的实参最终都会传给同一个 `x`。所以 `y` 的类型依赖外层参数 `x`，不能允许两次调用为 `y` 任意选取毫无关系的类型。既用整数又用布尔值调用 `y`，应当通不过。读降低层级的代码时可以把这个例子放在手边，它恰好说明了泛化过头会错在哪里。我们会让两套推断器处理这类例子，对比它们的判断，其中也包括本来就应当被拒绝的程序。

## 跑起来

```text
$ cd crates/ch06_poly
$ cargo run -q -- run tests/programs/poly-id.fun
True
$ cargo run -q -- dump --pass=types tests/programs/missing-annotation.fun
a -> a
$ cargo run -q -- dump --pass=types-w tests/programs/missing-annotation.fun
a -> a
$ cargo run -q -- repl
Poly: :quit, :type EXPR, :parse EXPR, :trace EXPR.
poly> :type \f -> \x -> f x
(a -> b) -> a -> b
poly> :quit
```

`types` 和日常的类型检查走的都是基于层级的推断器。选 `types-w` 可以看到算法 W 独立算出的结果。输出主类型时，里面的自由类型变量默认带有全称量词，例如显示的 `a -> a` 就读作“对任意 `a`，有 `a -> a`”。程序运行时默认仍由严格求值的 CEK 机执行，更换推断算法不会改变程序算出的值。

## 练习

1. **基础。** 先给每个参数各写一个新的类型变量，再推断 `\x -> \y -> x` 的类型。
2. **进阶。** 跟着 `let id = \x -> x in id id` 的推断过程，逐步写下每次得到的替换。
3. **进阶。** 解释为什么 `\f -> let a = f 1 in f True` 会被拒绝，而章首两次调用 `id` 的例子却能通过。
4. **挑战。** 找一个内层函数捕获外层 lambda 参数的例子。画出推断过程中产生的元变量单元，标出各自的层级，并找出具体是哪一次合一导致某个变量降了层级。

## 延伸阅读

Damas 和 Milner 给出了主类型方案的理论结果[^damas1982]。Kiselyov 从 OCaml 的具体实现出发解释了层级的工作原理[^kiselyov2013]，这一做法建立在 Rémy 的研究之上[^remy1992]。要系统了解类型判断和推导，可以读 Pierce 的著作[^pierce2002]。

我还写过一份专门讲类型系统的教程，从另一个角度解释这些想法。[Typechecker Zoo](https://sdiehl.github.io/typechecker-zoo/)

[^damas1982]: Luis Damas、Robin Milner（1982）。[Principal Type-Schemes for Functional Programs](https://doi.org/10.1145/582153.582176)。
[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。
[^kiselyov2013]: Oleg Kiselyov（2013）。[How OCaml Type Checker Works—or What Polymorphism and Garbage Collection Have in Common](https://okmij.org/ftp/ML/generalization.html)。
[^remy1992]: Didier Rémy（1992）。[Extension of ML Type System with a Sorted Equational Theory on Types](https://gallium.inria.fr/~remy/ftp/eq-theory-on-types.pdf)。

---

[← 第 5 章　求值](ch05-evaluation.md) · [中文目录](../README.md)
