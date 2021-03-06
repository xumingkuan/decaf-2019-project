# PA1-B

基于 LL(1) 的语法分析与错误恢复

**注意：本阶段仅 Java 和 Rust 版必做。**

## 任务概述

在 PA1-A 中，我们借助语法分析器自动生成工具完成了新语言特性的词法和语法分析。
在这一部分，我们的任务与 PA1-A 相同，但采用自顶向下的语法分析方法，并要求支持一定程度的错误恢复。
本阶段实验的重点是训练自顶向下语法分析/翻译的算法实现。对于词法分析程序，同学们沿用利用 PA1-A 的实现即可。
对于语法分析程序，我们提供了工具从文法描述文件自动生成 LL(1) 分析表，但是错误恢复算法需要大家手工编码实现。
其中，Java 版使用 [ll1pg](https://github.com/paulzfm/ll1pg/tree/course)，
Rust 版使用 [lalr1](https://github.com/MashPlant/lalr1) 的 `#[ll1]` 模式。

本阶段的测试方法与 PA1-A 一致：编译器格式化输出 AST 或者错误信息，测试程序检查其与标准输出是否完全一致。
我们保留了一些测例没有公开，所以请自己再编写一些测例，以更加充分地测试你的实现是否符合新特性的规范。

> 注意：使用分阶段框架进行开发的同学，请先将你上一阶段的工作（词法分析和抽象语法树相关代码）合并到本阶段框架中。

## 实验内容

本阶段实验需要完成两部分内容：LL(1) 语法分析、错误恢复。

### 任务一：LL(1) 语法分析

本阶段实验框架已经给出了原版 Decaf 的语法描述文件（类似于 PA1-A 中的语法描述文件）。
你需要修改这个文件，将 PA1-A 中所述的所有新特性对应的文法改写为 LL(1)。
只要你的改写方式是正确的，那么框架中已经给出的 LL(1) 分析算法就能正确处理所有语法正确的程序。
请留意 ll1pg/lalr1 工具给出的警告，因为这意味着你的文法不是严格 LL(1)，工具会按照某种方法强制解决冲突。
如果它的解决方法不是你想要的，那么生成出来的语法分析器极有可能包含潜在的错误。

为了不为难大家，在本阶段，对于 lambda 表达式

```text
expr ::= ...
       | 'fun' '(' paramList ')' '=>' expr
       | 'fun' '(' paramList ')' block
```

我们不考虑它出现在运算符操作数位置的情形，如在测例 `S1/lambdabad1.decaf` 中：

```decaf
2 + fun () => 0
```

lambda 表达式出现在加法的右操作数位置。简而言之，我们认为 lambda 表达式具有最低的优先级：

```text
expr ::= 'fun' '(' paramList ')' '=>' expr
       | 'fun' '(' paramList ')' block
       | expr1
```

其中 `expr1` 为其他表达式。注意 '=>' 右侧的表达式仍有可能是 lambda 表达式，并按照 '=>' 右结合来处理。
这样在改写为 LL(1) 文法时比较容易。
本阶段测例均遵守这一约定（即 lambda 表达式都不会出现在操作数的位置）。
原有测例 `S1/lambdabad1.decaf` 将在本阶段被忽略。

除此之外，其他所有新特性的文法同 PA1-A，这里不再赘述。

该任务完成后，所有语法正确的测例都能生成正确的 AST（同 PA1-A）。

### 任务二：错误恢复

框架中已经给出的 LL(1) 分析程序没有考虑任何错误恢复，因此对于有语法错误的程序，默认实现甚至有可能抛出运行时错误。
所谓**错误恢复**，是指语法分析器在遇到语法错误时不终止分析过程，而是尝试恢复该错误并继续分析，直至读到 EOF。
带有错误恢复的分析程序可以报告多条语法错误。

在课程讲义 Lecture04 中，我们介绍了应急恢复和短语层恢复的方法。这里，我们提出一种介于二者之间的错误恢复方法：

与应急恢复的方法类似，当分析非终结符$$A$$时，若当前输入符号$$a \notin Begin(A)$$，则先报错，
然后跳过输入符号串中的一些符号，直至遇到$$Begin(A) \cup End(A)$$中的符号：

- 若遇到的是$$Begin(A)$$中的符号，可恢复分析$$A$$；
- 若遇到的是$$End(A)$$中的符号，则$$A$$分析失败，继续分析$$A$$后面的符号。

这个处理方法与应急恢复方法的不同之处在于：

- 我们用集合$$Begin(A)=\{s \mid M[A,s] 非空\}$$（其中，$$M$$为预测分析表）来代替$$First(A)$$。由于$$First(A) \subseteq Begin(A)$$，我们能少跳过一些符号。
- 我们用集合$$End(A)=Follow(A) \cup F$$ 来代替$$Follow(A)$$。其中，$$F$$集合包含了$$A$$各父节点的$$Follow$$集合。因此，我们既能少跳过一些符号，同时由于结束符 EOF 必然属于文法开始符号的$$Follow$$集合，本算法还无需额外考虑因读到 EOF 而陷入死循环的问题。这个处理方法借鉴了短语层恢复中$$EndSym$$的设计。
- 另外，当匹配终结符失败时，只报错，但不消耗此匹配失败的终结符，而是将它保留在剩余输入串中。这部分的处理已经在框架代码中实现了。

一般来说，错误恢复是一个非常难的问题。
本阶段实验只要求大家的实现能够对于给出的测例（即 `S1-LL/` 目录下的所有测例）既不漏报，也不误报。对于其他情况不做要求。

## 实验提示

本次实验分为两个子任务，且具有一定难度，建议你按照以下步骤依次完成：

### 步骤一：阅读文档和代码

请先查阅 [Decaf book](https://decaf-lang.gitbook.io/decaf-book) PA1-B 章节的实验指导，了解：

1. ll1pg/lalr1 工具的基本使用方法，尤其是如何书写 LL(1) 文法及嵌入语义动作
2. 框架中已给出（不含错误恢复）的 LL(1) 分析算法流程，重点关注它如何使用工具生成的 LL(1) 分析表，以及如何调用语义动作

并结合框架代码，理解具体的实现方法。

### 步骤二：增加新特性对应的 LL(1) 文法

这部分工作与 PA1-A 类似，只不过你需要书写的文法比 PA1-A 中的 LR 文法更加复杂。
对于 LL(1) 文法不熟悉的同学，请先复习课程讲义中的相关章节，掌握如何把 CFG 改写为 LL(1) 的基本方法，尤其要掌握：

- 消除左递归
- 提取左公因子
- 显式表达二元运算符优先级和结合性

的方法。接下来，你需要完成：

1. 在文法描述文件中增加新特性用到的终结符 (token)
2. 依次把所有新特性对应的文法改写为 LL(1)，并添加合适的语义动作以构建 AST

> 注意：除了本来就有的 else 语句处的警告外，添加新的产生式不应该再引入其他的警告。
> 如果引入了，那么根据我们的经验，最终得到的 parser 很可能有 bug。

这个步骤完成后，所有语法正确的测例都应该能通过。

### 步骤三：增加错误恢复功能

强烈建议你按照“实验内容”一节中推荐的算法来做，因为正确这个算法后，所有给出的测例都可以通过。
如果你要自己设计其他错误恢复算法，那么也许会花较多时间来调整 (tweak) 处理细节，以通过所有给出的测例。
若你选择后者，请在实验报告中清晰地介绍你采用的错误恢复算法。

不正确的实现极有可能引发运行时错误（例如在 Java 版中也许你会遇到 `NullPointerException`），这时你可以采用“打印调试法”：
在算法中插入一些打印 log 的语句，输出算法的运行过程，以便更好地定位和排除 bug。

至此，本阶段实验的所有任务就完成了。

### 特别说明：文法简化

针对 Decaf 语言规范中的文法

```text
simple      ::= var ('=' expr)? | lValue '=' expr | expr | ε
```

由于将其改写为等价的 LL(1) 文法十分复杂。作为*妥协*，本阶段我们将上述文法扩展为

```text
simple      ::= var ('=' expr)? | expr ('=' expr)? | ε
```

即先在文法层面允许 `lValue` 为任意 `expr`，然后在语义动作中检查等号左边的表达式是否是 `lValue`（以 Java 版为例，Rust 版类似）：

```java
// SimpleStmt : Expr Initializer
if ($1.expr instanceof LValue) {
    var lv = (LValue) $1.expr;
    $$ = svStmt(new Assign(lv, $2.expr, $2.pos));
} /* ... */
```

你在进行实验时**无需**修正此文法。且测例会保证所有赋值语句中，等号左侧都是 `lValue`。

> 但在之后的各阶段，我们依然以语言规范（PA1-A 的实现符合规范）为准。

## 公开测例

公开的测例更新在 [这个 Repo](https://github.com/decaf-lang/decaf-2019-TestCases)。其中：

- `S1/` 同 PA1-A，这里面的测例要么语法正确，要么最多含有一处语法错误。
- `S1-LL/` 是 PA1-B 阶段专用的测例，这里面的测例均含有多处语法错误。

在完成错误恢复后，请确保这些测例均能被正确处理——**既不漏报，也不误报**。

> 注意：`S1/` 中有三个测例需要被忽略：`abstract1.decaf`, `abstract3.decaf`, `lambdabad1.decaf`。
> 最新的 `testAll.py` 脚本已自动忽略。脚本运行命令为：`./testAll.py PA1-B`。
> 使用 Rust 版的同学请手工忽略它们。

## 实验评分和实验报告

实验评分分两部分：

- 评测结果：80%。这部分是机器检查，要求你的输出和标准输出**一模一样**。我们会有**未公开**的测例。但是，含有多处语法错误的未公开测例均与公开的极其相似。一般情况下（你的实现不在公开测试集上“过拟合”），只要你的实现能通过公开测例，它也能通过未公开测例。
- 实验报告（根目录下 `report-PA1-B.pdf` 文件）：20%。要求用中文或英文简要叙述你的工作内容。若你采用了不同于推荐算法的其他错误恢复算法，请介绍你的算法思路。

此外，请在实验报告中回答以下问题：

Q1. 本阶段框架是如何解决空悬 else (dangling-else) 问题的？

Q2. 使用 LL(1) 文法如何描述二元运算符的优先级与结合性？请结合框架中的文法，**举例**说明。

Q3. 无论何种错误恢复方法，都无法完全避免误报的问题。
请举出一个**具体的** Decaf 程序（显然它要有语法错误），用你实现的错误恢复算法进行语法分析时**会带来误报**。
并说明该算法为什么**无法避免**这种误报。

## 备注

在今后各个阶段的实验中，框架总是会使用 PA1-A 阶段实现的语法分析器来解析 Decaf 程序源码。
所以即使本阶段实现有 bug，也不会影响到后面的阶段。
