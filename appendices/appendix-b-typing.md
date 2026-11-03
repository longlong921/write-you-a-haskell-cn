# 附录 B　类型规则

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/appendix-b-typing.html) · [原文源码](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/book/src/appendix-b-typing.md)

写下 `id 42`，凭什么说结果是 `Int`？这份附录把 Fun 源语言的判断依据列在一起。从变量和函数，到模式、绑定组、类型类、记录和 IO，都可以用下面的规则一步步推导。实际的编译器靠合一、类型变量的层级和约束求解来找出一份推导。Core 则把多态显式地写进程序，采用另一套类型规则，见[第 10 章](../part-2/ch10-typed-core.md)。

## 怎样读一条规则

把横线下面当成要回答的问题，从下往上读。想证明下面这句话，需要先证明上面的哪些条件？反过来，从上往下读，就是一小段证明。上面的条件都成立，下面的结论就有了依据。横线上方留空，表示这条规则不需要前提，叫作**公理**。旁边括号里写的是规则的名字，方便回头引用。[^pierce2002]

最常用的判断写作 $P;\Gamma\vdash e:\tau$。$\Gamma$ 这张表记着变量各自的类型方案，$P$ 记着已经成立的类型类约束。在这些已知条件下，可以推出表达式 $e$ 的类型是 $\tau$。分号只用来隔开两组已知条件。看一个最小例子。

$$
\frac{x:\mathsf{Int}\in\Gamma}
     {P;\Gamma\vdash x:\mathsf{Int}}\;\text{(Var-Int)}
$$

如果在 $\Gamma$ 里查到 $x$ 的类型是 $\mathsf{Int}$，就可以断定表达式 $x$ 也有这个类型。$\Gamma$ 是一张有限的映射表。往里添加 $x:\tau$，内层的 $x$ 会遮住外层的同名绑定。符号上方的横线表示一串东西，例如 $\overline{\alpha}$ 表示零个或多个类型变量。带下标 $i$ 的条件，对这串东西里的每一项都要检查。公式下面用文字补充的条件也得满足。

| 记号 | 怎么读 |
| --- | --- |
| $\tau$ | 单型，可以是类型变量、类型构造器、类型应用或箭头类型 |
| $\sigma$ | 类型方案，形如 $\forall\overline{\alpha}.\;Q\Rightarrow\tau$。可以为量化变量选择类型，只要相应的约束 $Q$ 成立 |
| $P,Q$ | 类型类谓词的集合，例如 $\mathsf{Eq}\;\alpha$ 表示类型 $\alpha$ 支持相等比较 |
| $\Gamma$ | 项变量及其类型方案。程序里的函数名、形参名都属于这一层 |
| $\Delta$ | 类型变量、类型构造器以及它们各自的种类（kind） |
| $S\tau$ | 对类型 $\tau$ 做替换 $S$，同时避免原本自由的变量被意外绑定 |
| $\mathsf{fv}(X)$ | 在 $X$ 中自由出现的类型变量 |
| $P\vdash Q$ | 从已知约束 $P$ 可以推出 $Q$ 中的每一项 |
| $\Gamma_p$ | 模式带进当前作用域的变量，每个变量在这里都有一个单型 |

比较两个类型时，要先把已经给足参数的类型同义词展开。比较两个类型方案时，量化变量可以一致地改名，例如 `forall a. a -> a` 和 `forall b. b -> b` 是同一个方案。数据构造器、记录字段、类型类和实例都登记在全局签名里。下文默认这些声明已经通过检查。

## 种类与类型的构造

种类只有两种形状，$*$ 和 $\kappa_1\to\kappa_2$。前者表示一个已经可以用来给值标注类型的类型，例如 `Int`。后者描述类型构造器还需要什么类型参数，给足参数后又得到什么种类。例如 `Maybe` 的种类是 `* -> *`，给它一个 `Int`，就得到种类为 `*` 的 `Maybe Int`。这里的箭头出现在类型这一层，普通函数的箭头则说明怎样把值变成另一个值。

$$
\frac{\alpha:\kappa\in\Delta}{\Delta\vdash\alpha:\kappa}\;\text{(K-Var)}
\quad
\frac{T:\kappa\in\Delta}{\Delta\vdash T:\kappa}\;\text{(K-Con)}
$$

$$
\frac{\Delta\vdash\tau_1:\kappa_1\to\kappa_2
      \quad\Delta\vdash\tau_2:\kappa_1}
     {\Delta\vdash\tau_1\;\tau_2:\kappa_2}\;\text{(K-App)}
$$

$$
\frac{\Delta\vdash\tau_1:*\quad\Delta\vdash\tau_2:*}
     {\Delta\vdash\tau_1\to\tau_2:*}\;\text{(K-Arrow)}
$$

项的类型和数据构造器每个字段的类型，都必须有种类 $*$。写约束 $C\;\tau$ 时，$\tau$ 的种类也得对得上类 $C$ 对参数的要求。因此 `Functor` 接收种类为 `* -> *` 的构造器，`Eq` 接收种类为 `*` 的类型。编译器会把相互依赖的一组类型声明放在一起推断种类，最后仍然没有约束的种类变量取 $*$。类型同义词循环引用或使用时缺少参数，都会报错，即使后面的合一过程碰巧能掩盖这个问题。

## 变量、字面量与函数

$$
\frac{x:\forall\overline{\alpha}.\;Q\Rightarrow\tau\in\Gamma
      \quad P\vdash SQ}
     {P;\Gamma\vdash x:S\tau}\;\text{(Var)}
$$

用到变量 $x$ 时，从环境中取出它的类型方案。$S$ 为每个量化变量选一个种类相符的单型，同时替换约束 $Q$ 和结果类型 $\tau$ 中相应的变量。如果现有条件足以推出替换后的 $SQ$，这一处的 $x$ 就可以按 $S\tau$ 使用。这一步叫**实例化**。数据构造器也按这个办法使用各自声明的类型方案。字面量更简单，类型固定，也不会产生类型类约束。

$$
\frac{}{P;\Gamma\vdash n:\mathsf{Int}}\;\text{(Int)}
\quad
\frac{}{P;\Gamma\vdash d:\mathsf{Double}}\;\text{(Double)}
$$

$$
\frac{}{P;\Gamma\vdash c:\mathsf{Char}}\;\text{(Char)}
\quad
\frac{}{P;\Gamma\vdash s:\mathsf{String}}\;\text{(String)}
$$

$$
\frac{P;\Gamma,x:\tau_1\vdash e:\tau_2}
     {P;\Gamma\vdash\lambda x.e:\tau_1\to\tau_2}\;\text{(Abs)}
$$

$$
\frac{P;\Gamma\vdash e_1:\tau_1\to\tau_2
      \quad P;\Gamma\vdash e_2:\tau_1}
     {P;\Gamma\vdash e_1\;e_2:\tau_2}\;\text{(App)}
$$

看 `id 42`。若 `id` 的方案是 `forall a. a -> a`，就能在这一处把 `a` 选成 `Int`。按 (Int)，`42` 是 `Int`。按 (App)，把 `Int -> Int` 应用到 `Int`，结果就是 `Int`。下一处写 `id True`，又可以把 `a` 选成 `Bool`。

(Abs) 则给 lambda 的形参分配一个单型，检查函数体时一路沿用。因此，`let id = \x -> x in (id 42, id True)` 可以给两个 `id` 分别选类型，而 `\f -> (f 42, f True)` 会要求同一个 `f` 的参数同时是 `Int` 和 `Bool`，类型检查过不去。如果形参位置写的是模式，要先用下文的模式规则找出引入的变量，再用 (Abs)。

## let、泛化与类型签名

$$
\frac{P,Q;\Gamma\vdash e_1:\tau_1
      \quad P;\Gamma,x:\sigma\vdash e_2:\tau_2}
     {P;\Gamma\vdash\mathsf{let}\;x=e_1\;\mathsf{in}\;e_2:\tau_2}
     \;\text{(Let)}
$$

先检查等号右边的 $e_1$，再决定名字 $x$ 在 $e_2$ 里能有多通用。这里赋给 $x$ 的方案是 $\sigma=\forall\overline{\alpha}.\;Q\Rightarrow\tau_1$。能泛化的是那些在 $\tau_1$ 或 $Q$ 中自由出现、又没有在外层 $\Gamma$ 或 $P$ 中自由出现的类型变量。比如 `let id = \x -> x` 推出的 `a -> a` 中，若 `a` 与外层环境无关，就能把它量化，得到 `forall a. a -> a`。如果某个类型变量来自外层已有的绑定，内层的 `let` 就不能替它承诺“什么类型都行”。

进入方案之前，求解器会先化简 $Q$，把已经具体到能判定的约束求掉，并把所有字段约束都解出来。$Q$ 提到的每一个类型变量都必须出现在 $\tau_1$ 里，否则光看怎样使用 $x$，没法确定该为约束选择哪个类型。涉及本次泛化范围之外的变量的谓词留在 $P$ 中。如果剩下的谓词既无法泛化，也找不到满足它的实例，就会报错。Fun 不会替这种约束猜一个默认类型。它也没有“值限制”，所以 `let` 右边即使是一次函数调用，也有机会泛化。

一组互相递归、又都没写签名的绑定，要先给每个 $x_i$ 各分配一个新的单型 $\tau_i$。然后带着这同一组假设，逐个检查所有定义的右边。

$$
\frac{P,Q;\Gamma,\overline{x:\tau}\vdash e_i:\tau_i
      \quad P;\Gamma,\overline{x:\sigma}\vdash e:\tau}
     {P;\Gamma\vdash\mathsf{let}\;\overline{x=e_i}\;\mathsf{in}\;e:\tau}
     \;\text{(Let-Rec)}
$$

整组都检查完以后，相对于外层环境一起泛化求得的类型，约束与歧义的处理都和 (Let) 相同。脱糖后，每个递归定义的右边在语法上都必须是函数。组内递归调用期间，同一个名字一直按同一个单型使用。等这一组检查结束，外部调用者才能按泛化后的方案使用它。

手写签名等于作出一个承诺。检查器会用全新的、不能随意改变的类型变量来检验它。

$$
\frac{P,SQ;\Gamma\vdash e:S\tau}
     {P;\Gamma\vdash e:
        \forall\overline{\alpha}.\;Q\Rightarrow\tau}
     \;\text{(Signature)}
$$

这里的 $S$ 把每个量化变量换成同种类、全新的**刚性变量**（Skolem 变量）。比如在 Fun 里写下签名 `id :: a -> a`，其中的 `a` 会被隐式量化。这就承诺了任取一种类型都成立，因此检查 `id` 的定义时，要把 `a` 当成某个任意但固定的类型。若函数体只有在 `a = Int` 时才说得通，检查就得失败。这些临时变量既不能随意和别的类型合一，也不能逃到外层环境里。函数体需要的每个谓词，都得能从 $P$、签名给出的 $SQ$ 和已声明的实例推出。

写了签名的绑定在递归调用自身时也能使用声明的整个方案，因此可以**多态递归**，在不同调用处选不同类型。划分无签名的递归依赖组之前，依赖分析会先移除指向有签名绑定的边。表达式上的类型标注也按 (Signature) 检查，但检查完还要实例化，才供这个表达式使用。这种标注不会把多态值变成可以直接当参数传来传去的一等值。

## 模式、守卫与 case 分支

$\vdash p:\tau\Rightarrow\Gamma_p$ 回答两个问题。模式 $p$ 能用来匹配什么类型的值？如果匹配成功，分支里会多出哪些可以使用的变量？$\tau$ 回答前一个问题，$\Gamma_p$ 回答后一个。

$$
\frac{}{\vdash x:\tau\Rightarrow x:\tau}\;\text{(P-Var)}
\quad
\frac{}{\vdash\_:\tau\Rightarrow\Gamma_0}\;\text{(P-Wild)}
$$

$$
\frac{\vdash p:\tau\Rightarrow\Gamma_p}
     {\vdash x@p:\tau\Rightarrow\Gamma_p,x:\tau}\;\text{(P-As)}
$$

$$
\frac{K:\forall\overline{\alpha}.\;
          \tau_1\to ... \to\tau_n\to T\;\overline{\alpha}
      \quad\vdash p_i:S\tau_i\Rightarrow\Gamma_i}
     {\vdash K\;\overline{p}:T\;S\overline{\alpha}
          \Rightarrow\overline{\Gamma_i}}\;\text{(P-Con)}
$$

变量模式 `x` 把整个值绑定为 `x`。通配符 `_` 什么都能匹配，留下的 $\Gamma_0$ 是空环境。`x@p` 既保留整个值，又保留子模式 `p` 解出的各部分。例如拿 `whole@(Just n)` 去匹配 `Maybe Int`，成功后 `whole` 是整个 `Maybe Int`，`n` 是里面的 `Int`。同一个模式中，各部分引入的变量名不得重复。

字面量模式的类型就是该字面量的固定类型，不引入新变量。记录模式按声明顺序排回构造器模式，没写的字段都放一个 `_`。列表、元组和单元值 `()` 的模式，也都按各自的数据构造器处理。

$$
\frac{P;\Gamma\vdash e:\tau_s
      \quad\vdash p_i:\tau_s\Rightarrow\Gamma_i
      \quad P;\Gamma,\Gamma_i\vdash e_i:\tau}
     {P;\Gamma\vdash\mathsf{case}\;e\;\mathsf{of}\;
          \overline{p_i\to e_i}:\tau}\;\text{(Case)}
$$

$$
\frac{P;\Gamma,\Gamma_p\vdash g_j:\mathsf{Bool}
      \quad P;\Gamma,\Gamma_p\vdash e_j:\tau}
     {P;\Gamma,\Gamma_p\vdash\overline{g_j\to e_j}:\tau}
     \;\text{(Guards)}
$$

`case` 的每一行都匹配同一个待检值，得到各自的局部变量。这些变量只在相应分支里可用。每个守卫 $g_j$ 必须是 `Bool`，守卫成功后得到的 $e_j$ 则必须与整个 `case` 的结果类型一致。一行上的 `where` 定义在该行的模式环境里按绑定组规则检查，守卫和结果表达式都能使用这些局部定义。

用多行方程定义函数时，每一列模式对应一个参数，逐行检查的方式与 `case` 相同。所有方程的参数个数必须一致。如果有的输入没被任何一行覆盖，或者有一行永远轮不到，编译器会发出警告。这些警告不影响上面的类型推导。所以一个表达式无论推得什么结果类型，运行时仍可能因为没有模式匹配成功而报错。

对非递归模式绑定，先检查等号两边，要求右边表达式的类型与左边模式匹配的类型一致。然后相对于外层环境，按 (Let) 的条件泛化模式引入的各个名字。右边的表达式看不到这次绑定在左边引入的新变量。

## 名义记录

记录的身份由它的类型名决定。设记录类型 $T$ 有一组类型参数 $\overline{\alpha}$，记 $\mathsf{field}(T,l)=\tau_l$，意思是到 $T$ 的声明里查字段 $l$，查出的类型是 $\tau_l$。$S$ 把声明中的类型参数换成实际传入的类型。下面三条规则都要求已经确定对应的记录构造器。

$$
\frac{P;\Gamma\vdash e_i:S\tau_{l_i}}
     {P;\Gamma\vdash K\;\{\overline{l_i=e_i}\}:
        T\;S\overline{\alpha}}\;\text{(Record)}
$$

构造记录时，声明中的每个字段都要给出一次，不能多写一个陌生字段。字段表达式按源码中的先后顺序求值，存放结果的槽位则按类型声明中的顺序排列。

$$
\frac{P;\Gamma\vdash e:T\;S\overline{\alpha}
      \quad\mathsf{field}(T,l)=\tau_l}
     {P;\Gamma\vdash e.l:S\tau_l}\;\text{(Project)}
$$

$$
\frac{P;\Gamma\vdash e:T\;S\overline{\alpha}
      \quad P;\Gamma\vdash v_i:S\tau_{l_i}}
     {P;\Gamma\vdash e\;\{\overline{l_i=v_i}\}:
        T\;S\overline{\alpha}}\;\text{(Update)}
$$

取字段 `e.l` 时，到 $T$ 的声明里查出字段类型，再代入实际类型参数。更新记录时，写到的字段都必须属于 $T$，同一个字段不能更新两次。留意 (Update) 中的 $S$，输入记录、待写入的字段值和结果记录用的都是同一份替换。假设 `Box a` 有一个字段 `value :: a`，那么对于一个 `Box Int`，更新 `value` 时就必须给 `Int`，更新后仍是 `Box Int`。

实际推断时，刚看到 `e.l` 可能还不知道 $T$ 是谁。编译器先记下一条待办事项 $\mathsf{Has}(\tau,l,\tau_r)$，意思是“类型 $\tau$ 应该有一个字段 $l$，读取结果应为 $\tau_r$”。如果只有一种记录声明过这个字段名，就选它，并为它的类型参数引入新变量。如果有好几种记录都用了这个名字，就等 $\tau$ 的最外层类型构造器确定后再选。

泛化之前，求解器会反复尝试，直到再也解不出新的字段约束。还没解出的字段一律算歧义，报 `E0245`。即使通读整份程序能猜出一个选项，也不能把这个选择留到后面。类型方案中不会携带字段约束。`(.l)` 按 `\r -> r.l` 检查，因此也受同一限制。

## 类型类约束怎样推导，声明怎样检查

$$
\frac{\pi\in P}{P\vdash\pi}\;\text{(Given)}
\quad
\frac{P\vdash C\;\tau\quad C\text{ has superclass }D}
     {P\vdash D\;\tau}\;\text{(Super)}
$$

$$
\frac{\mathsf{instance}\;Q\Rightarrow C\;\tau
      \quad P\vdash SQ}
     {P\vdash C\;S\tau}\;\text{(Instance)}
$$

手上已经有的谓词 $\pi$ 可以直接用，这就是 (Given)。(Super) 允许从子类推出超类，例如 `Ord a` 要求 `Eq a`，那么手上有 `Ord a` 就能取得 `Eq a` 的证据。(Instance) 则查找一个已声明的实例，为它的类型变量选择种类正确的替换，使实例头与目标谓词相符。如果替换后的实例前提 $SQ$ 都能推出，目标谓词也就成立。例如要推出 `Eq [Int]`，可以取列表的 `Eq` 实例，再证明它所要求的 `Eq Int`。如果目标是一组谓词，就得逐个推出。在把程序翻译成 Core 时，从子类取超类、应用实例这两步也会构造实际传递的字典，让后续代码拿得到方法实现。

Fun 的一个类型类只有一个参数。方法签名可以用这个参数，但方法不能再另加自己的类型类约束。方法中出现的其他类型变量会被隐式量化。检查默认方法时，使用它声明的类型，并假定该类自己的谓词已经成立。超类之间不能形成环。

实例头中的类型形如“类型构造器应用到若干互不相同的变量”。实例的前提只能约束这些变量，两个实例头不能重叠。检查方法时，先把类方法签名中的类参数换成这个实例头里的类型，再把实例前提当成已知条件检查实现。实例所需的超类字典也必须能取得。省略的方法必须在类里有默认实现。`deriving` 自动生成的实例也得经过同样的检查。

声明一个数据类型，也为它的每个数据构造器引入了一个类型方案。类型参数全部用全称量词绑定，字段依次成为箭头类型的参数，最后返回声明的那个数据类型。例如 `Just` 接受一个 `a`，返回 `Maybe a`。Fun 的构造器不支持存在量化变量、GADT 带来的类型等式，也不能为个别构造器单独加上类型类约束。

## 语法糖、IO 与完整程序

有些写法只是在已有结构上换了一层外衣。对照它们展开后的形式，就能知道该怎么检查类型。

| 源码写法 | 怎样确定类型 |
| --- | --- |
| `if b then x else y` | `b` 必须是 `Bool`，两个分支的类型必须一致 |
| `a && b`、`a \|\| b` | 两边都必须是 `Bool`。先求左边，只有它还不能决定结果时才求右边 |
| `a + b` 及其他运算符 | 按运算符自身的类型方案做函数应用 |
| `-e` | 把 `negate` 应用到 `e` |
| `(e op)`、`(op e)` | 为缺的那个操作数引入新形参，再用 lambda 调用运算符，操作数的位置保持原样 |
| `[a..b]` | 展开成 `enumFromTo a b`，两个端点都必须是 `Int` |
| 列表、元组、单元值 | 使用各自的数据构造器 |
| `where` | 按局部绑定组处理 |

设 $m$ 是一个种类为 $*\to *$ 的 Monad 类型构造器。在 `do` 中写 `p <- e`，要求 $e$ 是一个 $m\;\alpha$ 类型的动作，模式 $p$ 能接住一个 $\alpha$。把匹配引入的变量交给后续语句 $s$，后续语句要产生 $m\;\beta$。整个块的结果于是也是 $m\;\beta$。

$$
\frac{P\vdash\mathsf{Monad}\;m
      \quad P;\Gamma\vdash e:m\;\alpha
      \quad\vdash p:\alpha\Rightarrow\Gamma_p
      \quad P;\Gamma,\Gamma_p\vdash s:m\;\beta}
     {P;\Gamma\vdash\mathsf{do}\;\{p\leftarrow e;\;s\}:m\;\beta}
     \;\text{(Do-Bind)}
$$

没有 `<-` 的中间表达式语句用 `>>` 串接。`let` 语句按绑定规则处理，最后一个表达式提供整个块的结果。如果 `<-` 左边的模式有可能匹配失败，脱糖时会生成 `case`，失败分支调用 `error`。Fun 在这里不需要 `MonadFail` 约束。

对检查源码的类型系统来说，`IO` 是个抽象的类型构造器，种类是 $*\to *$，并且有 `Monad` 实例。源码里不能把 `IO a` 当成 `() -> a` 使用。直到把源程序展开成 Core 时，编译器才揭开这层表示。那些能直接产生副作用的原语，也只有在编译 Prelude 时可见。

检查一整个程序时，编译器要先检查声明的种类，验证类型类及实例组成的环境，再按依赖顺序检查各组绑定，最后解掉剩余约束。程序入口必须有类型 `IO ()`。类型类或记录字段上都不能留下悬而未决的问题。运行时先按依赖顺序执行顶层初始化，再执行入口的 IO 动作。类型正确的程序仍然可能无限递归，也可能主动调用 `error` 而在运行时报错。

[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。

---

[← 附录 A　表层语法](appendix-a-syntax.md) · [中文目录](../README.md) · [附录 C　中间表示 →](appendix-c-ir.md)
