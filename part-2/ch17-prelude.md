# 第 17 章　Prelude 与自动派生

[中文目录](../README.md) · [英文原章](https://sdiehl.github.io/write-you-a-haskell/ch17-prelude.html) · [配套源码](https://github.com/sdiehl/write-you-a-haskell/tree/51c70f804173c9d013187c543a353686b413ad50/crates/ch17_prelude)

> **新增文件**
>
> - `src/derive.rs`
>
> **改动文件**
>
> - `src/primitives.rs`
> - `src/desugar.rs`
> - `src/ast.rs`
> - `src/core_lint.rs`
> - `src/eval.rs`
> - `src/lexer.rs`
> - `src/lib.rs`
> - `src/main.rs`
> - `src/rename.rs`

## 终于可以用 Fun 给自己写标准库了

想把列表里的每个元素变换一下，直接用 `map`。想从右往左把它们合成一个结果，拿起 `foldr`。想分别处理 `Nothing` 和 `Just x`，有现成的 `maybe`。不用每写一个程序都把这些小函数重写一遍，写起 Fun 来才真有了用一门语言做事的感觉。更有意思的是，走到这一步，我们已经可以用 Fun 自己写出这套随程序一起加载的标准库，也就是 Prelude。

还有一类重复劳动可以交给编译器。定义好数据类型，在后面写上 `deriving`，编译器就能根据类型长什么样，生成比较相等、比较大小或者转成可读文本的实例。生成出来的也是普通 Fun 代码，跟手写实例过同一个检查器。比如比较两棵树用到的每一次相等判断，都得在语言里找到依据。

这是进入后端前的最后一章。可以先停下来玩玩这个解释执行版的 Fun。拿标准库里的小函数做编译器实验尤其方便，常常只要看几行类型和定义，你就能自己说出预期结果，再让编译器跑给你看。

## 这时的 Fun 能写什么

声明数据类型时，可以写 `deriving Eq`，也可以用一对圆括号把想派生的类列出来。列表至少有一项，目前可选 `Eq`、`Ord` 和 `Show`。看下面这段程序，声明 `P` 时先写 `x` 后写 `y`，构造值时却故意先填 `y`。猜猜打印时谁在前面？

```fun
data P a = P {x :: a, y :: Int} deriving (Eq, Ord, Show)

main = print (Just (P {y=2,x=Left 1} :: P (Either Int Bool)))
```

[源码 · derive-show-record.fun 第 1～2 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch17_prelude/tests/programs/prelude/derive-show-record.fun#L1-L2)

派生的 `Eq` 先看两边用的是哪个构造器，再比较对应的字段。`Ord` 把声明里排在前面的构造器算作较小。两边用同一个构造器时，才像查字典那样从第一个字段起逐一比较，第一个不相等的字段就能决定结果。派生的 `Show` 按**声明顺序**打印记录字段，所以这里先输出 `x`。构造器应用嵌在别的应用里时，会按需加括号。列表用方括号，元素之间用逗号。Fun 的 `[Char]` 也按列表显示，它和基本类型 `String` 各有自己的表示。函数值本身无法派生比较或显示实例。

现在的 Prelude 已经能折叠、遍历列表，生成有限序列，切分和拼接字符串，转换数值，处理字符，还提供了一组控制单子动作的函数。`[a..b]` 会生成两端都包含在内的整数区间，`iterateN` 让你指定最多生成多少项。Fun 构造列表节点时就要把字段算出，因此没法靠不断推迟尾部的计算造出无限列表。`foldl'` 和 `foldl` 在这里都严格求值。

拼列表用 `++`，拼字符串用 `<>`。`sum` 和 `product` 从整数字面量 `0` 和 `1` 起步，参数都是 `Int` 列表。在 Fun 里，字面量的类型是固定的，`Num` 也没提供把整数字面量转成任意数值类型的操作。

### 把几个小函数接起来

```fun
total = sum . map (\x -> x * x) . filter even

main = print (total [1..10])
```

这段代码会打印 `220`。顺着数据走，要从右往左读。先由 `filter even` 从 1 到 10 里留下 `2, 4, 6, 8, 10`。接着 `map` 把它们各自平方，得到 `4, 16, 36, 64, 100`。最后 `sum` 求和。中间的 `(.)` 把一个函数的输出接到下一个函数的输入。这样写不需要给循环下标起名字，也不用操心每一步怎么走到下一个元素。

读代码时，我们能分别看清“筛选”“平方”“求和”各自做了什么。执行时，这些操作依然有成本。Fun 严格求值，这里筛完和平方后都会建出中间列表。以后可以研究怎样优化掉它们，现在已经可以从这行定义读懂它要算什么。

## 原理

假设要给树手写相等判断。两棵空树相等。两个节点要逐个比较对应的字段。两边用了不同的构造器，立刻就知道不等。这些分支都能从数据类型声明里读出来，编译器完全可以替我们写。

这就是**自动派生**（deriving）。编译器生成实例，然后按手写实例的规则检查它。《Haskell 2010 语言报告》规定了这种按数据结构比较和显示的行为[^haskell2010]。Fun 根据自己的记录、内建字符串和精简过的 `Show` 接口做了适配。

比如有个类型 `data Choice a = Empty | One a`。手写相等判断，最要紧的三种情况如下。

```fun
instance Eq a => Eq (Choice a) where
  (==) Empty Empty = True
  (==) (One x) (One y) = x == y
  (==) _ _ = False
```

为什么实例头里要写 `Eq a =>`？看中间一行就明白了。要比较两个 `One`，得知道存进去的 `x` 和 `y` 怎么比。另两行只看构造器标签就能决定。自动派生从声明里构造出同样的判断，再送进平常用的实例检查器。

类型参数实际出现在字段里，通常就要给它加上相应的类约束。反过来，看 `data Token a = Token Int`。声明头上虽然有 `a`，值里却只装了一个 `Int`。这种只挂名的参数叫**幻影参数**（phantom parameter）。比较两个 `Token` 时用不上 `Eq a` 字典。编译器得先展开类型别名，再判断哪些参数确实用到了。最后，实例求解器还要检查，凭生成的这些约束，是否真能做完每个字段上的操作[^jones1999]。

再看打印。单独一个 `Left 3` 可以直接显示，放进 `Just` 却必须写成 `Just (Left 3)`。单靠类型为 `a -> String` 的函数，怎么知道自己正给外层构造器打印一个参数？还得把所在位置要求的优先级传进来。

Fun 的 `Show` 字典为此留了一个内部方法 `_showsPrec`。手写实例可以用默认实现，派生实例则按各个构造器的结构生成它。最外层的 `show` 从优先级 0 开始。按位置放入的构造器参数使用优先级 11，里面若又是带参数的构造器应用，就会给自己加上括号。记录的字段名和大括号已经标明了边界，字段值直接用普通 `show` 即可。

严格求值也改变了一个熟悉的定义。如果在顶层直接写 `undefined = error "undefined"`，程序一启动就会报错，哪怕没人用它。Fun 在作用域内处理这个名字。凡是引用 Prelude 的 `undefined`，而且名字没被用户自己的定义遮住，就把这处引用展开成一次函数调用，给辅助函数传入单位值 `()`。这样只有执行到使用处才失败，既不用分配 thunk（延迟计算），也不必给绑定加上惰性求值。[第 5 章](../part-1/ch05-evaluation.md)讨论的求值策略在这里直接决定了程序的行为[^pierce2002]。

## 实现

`derive::derive` 紧挨在收集类型类和解析词法名称之前运行。生成的语法树节点都带着原数据类型声明的源码区间，以后某个字段上的操作被拒绝，诊断就能指回写下声明的地方。下面这段代码先展开别名、收集字段里出现的参数，再按类名选择三个生成器之一，把造好的普通实例声明追加到程序中。

```rust
/// Generate only Eq, Ord and Show. Generated terms retain the declaration span.
pub fn derive(
    sess: &mut Session,
    program: &mut Program,
) -> Result<(), Stop> {
    for d in &program.data {
        if d.deriving.is_empty() {
            continue;
        }
        let build = Builder { span: d.span };
        let mut used = BTreeSet::new();
        for c in &d.constructors {
            for ty in &c.fields {
                let expanded =
                    crate::kinds::expand(ty, &program.aliases, d.span)
                        .map_err(|e| {
                            sess.emit(e);
                            Stop
                        })?;
                variables(&expanded, &mut used);
            }
        }
        for class in &d.deriving {
            let methods = match class.as_str() {
                "Eq" => vec![build.eq(d)],
                "Ord" => vec![build.ord(d)],
                "Show" => build.show(d),
                _ => {
                    sess.emit(Diagnostic::error(
                        ErrorCode::Instance,
                        "only Eq, Ord and Show can be derived",
                        d.span,
                    ));
                    return Err(Stop);
                }
            };
            let context = d
                .parameters
                .iter()
                .filter(|p| used.contains(*p))
                .map(|p| PredicateSyntax {
                    class: class.clone(),
                    variable: p.clone(),
                })
                .collect();
            let head = d.parameters.iter().fold(
                TypeSyntax::Name(d.name.clone()),
                |a, p| {
                    TypeSyntax::App(
                        Box::new(a),
                        Box::new(TypeSyntax::Name(p.clone())),
                    )
                },
            );
            program.instances.push(InstanceDecl {
                class: class.clone(),
                head,
                context,
                methods,
                span: d.span,
            });
        }
    }
    Ok(())
}
```

[源码 · derive.rs 第 221～285 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch17_prelude/src/derive.rs#L221-L285)

生成器直接拼出已有的表层表达式和模式节点。以 `Eq` 为例，每一对构造器都有一条分支。两个构造器不同就返回 `False`。构造器相同就从左到右比较字段，第一个不相等的字段就能结束比较。留意下面的 `.rev()`。生成器从最后一个字段开始，把每个新的判断包在已有表达式外层，所以程序真正运行时，第一个字段反而最先比较。

```rust
fn eq(&self, d: &DataDecl) -> Binding {
    let mut rows = vec![];
    for (i, a) in d.constructors.iter().enumerate() {
        for (j, b) in d.constructors.iter().enumerate() {
            let mut rhs =
                self.var(if i == j { "True" } else { "False" });
            if i == j {
                for k in (0..a.fields.len()).rev() {
                    let test = self.call(
                        "==",
                        vec![
                            self.var(&format!("x{k}")),
                            self.var(&format!("y{k}")),
                        ],
                    );
                    rhs = self.expr(Kind::If(
                        Box::new(test),
                        Box::new(rhs),
                        Box::new(self.var("False")),
                    ));
                }
            }
            rows.push(self.row(
                vec![
                    self.constructor(a, "x"),
                    self.constructor(b, "y"),
                ],
                rhs,
            ));
        }
    }
    self.binding("==", rows)
}
```

[源码 · derive.rs 第 71～103 行](https://github.com/sdiehl/write-you-a-haskell/blob/51c70f804173c9d013187c543a353686b413ad50/crates/ch17_prelude/src/derive.rs#L71-L103)

生成 `Ord` 时也枚举每一对构造器。构造器相同就逐字段比较，只有上一个比较得到 `EQ` 才继续。`Show` 则用构造器名、字段名、标点和递归的 `_showsPrec` 调用拼出字符串表达式。这些表达式照常经过名称解析、类型推断、字典翻译、模式匹配编译和 Core Lint 检查。请求派生不支持的类会得到 `E0211`。如果要派生 `Ord`，却无法为所需的 `Eq` 超类提供实例，也过不了已有的实例检查。

库代码在 `prelude.fun` 里，编译器把它放在用户程序前面。库内部调用的私有辅助函数在库自己的作用域里解析，即使用户也定义了同名函数，Prelude 的实现仍然引用原来的那个。闭区间的实现藏着个容易漏掉的细节。序列到达上界时要**先停下来，再考虑要不要加一**。否则上界恰好是 63 位有符号整数的最大值时，加一就会绕回最小值，序列也就停不在该停的位置了。

字符分类和简单的大小写转换用的是随源码提交的 Unicode 17.0.0 数据表，存放在 `unicode.fun` 里。`tools/unicode.py` 记录了上游数据的来源和内容摘要，仓库也保留了数据的许可证。生成出来的表就是紧凑的 Fun 字符串。因此，判断字符属于哪个 Unicode 大类、查一个字符对应的大小写字符，这些操作都可以在语言里跑，省去了一大批新的原语。另有一个刻意的约定，`isDigit` 只认 ASCII 的 `0` 到 `9`，Unicode 里归为数字的字符要多得多。

写标准库也能暴露实现里白花的力气。沿着列表往下走一步，没理由把剩下的整条尾巴复制一遍。现在通过 Rust 的 `Rc` 共享构造器那组不可变的字段。Core Lint 的环境也做了类似的精简，离开作用域就移除相应的局部绑定，省掉整张环境表的拷贝。分析一段 Fun 程序为什么慢，除了看它自己的算法，也值得看看 Rust 拿什么数据结构来表示它。

## 跑起来看看

```text
$ cd crates/ch17_prelude
$ cargo run -q -- run tests/programs/prelude/derive-show-record.fun --lint
Just (P {x = Left 1, y = 2})
$ cargo run -q -- dump --pass=types tests/programs/prelude/derive-show-record.fun
main : IO ()
```

回到开头的问题，传值时先写了 `y`，打印出来还是声明中的 `x, y` 顺序。整条记录作为 `Just` 的参数时，外面多了一对括号。如果想看是谁决定了这些细节，用 `--pass=core` 可以查看生成的实例字典，用 `--pass=match` 可以看到模式编译后具体会检查哪些结构。另外两个测试样例 `derive-show-tree` 和 `derive-show-literals` 分别演示了嵌套的构造器应用和字面量的转义。

拿边界情况来试试这个库。两端都取最大 `Int` 的区间应该只有一个元素。给折叠函数传空列表会怎样？类型声明里加一个幻影参数，派生出来的实例会多要一个约束吗？这样的短例子往往比长长一串普通输入更容易看出定义里的门道。

## 练习

1. **基础。** 定义一个有两个构造器的树类型。自动派生 `Eq`、`Ord` 和 `Show`，再和手写的实例比较。
2. **进阶。** 给类型加一个字段里用不到的参数。查看派生实例的约束部分，里面是否要求为它提供字典？
3. **进阶。** 手动跟一遍打印 `Just (Left (Just 3))` 的过程，记下每一层收到的优先级。
4. **进阶。** 让区间的起点和终点都取为最大的 `Int`，测试结果，再解释为什么检查边界和给当前值加一的先后顺序很重要。
5. **挑战。** 选择一次 Unicode 查表操作，保持 Fun 程序相同，分别测量共享和不共享不可变字段向量时，遍历列表复制了多少尾部数据。

## 延伸阅读

- 标准的派生实例和 Prelude 都可以在《Haskell 2010 语言报告》里找到[^haskell2010]。
- Jones 解释了实例上下文，以及怎样由已有的约束推出所需约束（entailment）[^jones1999]。
- 若想对照类型类声明、实例和派生的语言规则，可再查报告中讲类型类和派生实例的章节[^haskell2010]。
- Pierce 的书从操作语义讲起，可以接着读它对求值策略的讨论[^pierce2002]。

[^haskell2010]: Simon Marlow 主编（2010）。[Haskell 2010 Language Report](https://www.haskell.org/onlinereport/haskell2010/)。
[^jones1999]: Mark P. Jones（1999）。[Typing Haskell in Haskell](https://web.cecs.pdx.edu/~mpj/thih/thih-sep1-1999/index.html)。
[^pierce2002]: Benjamin C. Pierce（2002）。[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)。

---

[← 第 16 章　编译模式匹配](ch16-pattern-compilation.md) · [中文目录](../README.md) · [进入第三部分 · 第 18 章（英文） →](https://sdiehl.github.io/write-you-a-haskell/ch18-anf.html)
