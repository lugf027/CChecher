### 2019 IEEE Symposium on Security and Privacy

### SoK: Sanitizing for Security

<p align="center">
    Dokyung Song, Julian Lettner, Prabhu Rajasekaran,<br/>
    Yeoul Na, Stijn Volckaert, Per Larsen, Michael Franz<br/>
    University of California, Irvine<br/>
    {dokyungs,jlettner,rajasekp,yeouln,stijnv,perl,franz}@uci.edu
</p>
***摘要***  众所周知，C和C ++编程语言是不安全的，但仍然是必不可少的。因此，开发人员会采取多管齐下的方法来解决对手面前的安全问题。这些包括手动，静态和动态程序分析。动态错误查找工具（以下称为 “消毒程序”）可以查找其他类型的分析错误，因为它们会观察程序的实际执行情况，因此可以在发生错误时直接观察错误的程序行为。

大量的消毒程序已由学术界原型化，并由从业人员完善。我们提供了消毒程序的系统概述，重点是消毒剂在发现安全问题中的作用。具体来说，我们对可用工具及其涵盖的安全漏洞进行分类，描述其性能和兼容性，并强调各种折衷方案。

#### 1、INTRODUCTION

C和C ++仍然是底层系统软件（例如，操作系统内核，运行时库和浏览器）的选择语言。关键原因在于它们高效且使程序员可以完全控制底层硬件。但另一方面，程序员必须确保每个内存访问都是有效的，没有计算会导致未定义的行为，等等。在实践中，程序员通常没有履行这些职责，并引入了使代码容易受到利用(容易发生漏洞并被利用这些漏洞)的错误。

同时，绕过(错开了)诸如地址空间布局随机化（ASLR）和数据执行保护（DEP）之类广泛采用的缓解措施，内存破坏漏洞的利用也越来越复杂[1] – [4]。代码重用攻击（例如，面向返回的编程（ROP））破坏了控制方面的数据（如函数指针或返回地址），从而劫持了程序的控制流 [1]。纯数据攻击(例如面向数据的编程（DOP）)利用 可以在合法控制流路径上调用的 指令，并通过仅破坏程序的非控制数据 来破坏程序[4]。

作为针对错误的第一道防线，程序员在将软件部署到生产环境之前，会使用分析工具来识别安全问题。这些工具依赖于静态程序分析、动态程序分析或静态与动态程序分析相组合。静态工具分析程序源代码，并**产生对于所有可能的代码执行都保守地正确的结果**[5] – [9]。相反，动态错误查找工具（通常称为“消毒程序”）分析单个程序执行并**输出仅对单个运行有效的精确分析结果**。

消毒程序现已得到广泛使用，并负责发现许多漏洞。但是，尽管它们在发现漏洞中无处不在且起着关键作用，但消毒程序通常并没有被很好地理解，这阻碍了它们的进一步发展和采用。实际上，尽管该领域有大量研究工作，但只有少数研究得到采用，从而使许多类型的漏洞都没有得到解决。本文提供了消毒程序的系统概述，重点是消毒程序在发现安全漏洞中的作用。我们对可用工具及其涵盖的安全漏洞进行分类，描述其性能和兼容性，并强调各种折衷方案。根据我们的发现，我们对开发人员的发展或研究方向提出以下三个目标（i）找到现有工具所检查不到的漏洞，（ii）改善实际中程序的兼容性（iii）更高效地找到漏洞的方法。

本文的其余部分安排如下：我们从对消毒程序和漏洞缓解措施两者间高层次表现比较开始（第2节）。接下来，我们描述C / C ++中的低层次漏洞（第3节）和分类技术以检测它们（第4节）。然后，我们继续介绍消毒程序的两种关键实现方法：程序检测技术（第5节）和元数据管理（第6节）。然后，我们简要讨论如何用消毒程序方式去驱动一个程序，以及如何最大化地发挥消毒程序的功效（第七节）。接下来，我们将介绍正在积极维护种 或在学术会议上发布过的消毒程序的一个汇总，重点是其精度，兼容性和性能/内存成本（第八节）。我们还将调查这些工具的部署情况（第IX节）。我们以研究方向的未来展望作为论文的结尾（第十节）。

#### 2、EXPLOIT MITIGATIONS VS. SANITIZERS

消毒程序在许多方面类似于许多众所周知的漏洞利用缓解措施，它们可以以类似的方式来检测程序，例如，通过插入嵌入式参考监视器（IRM）。尽管有这些相似之处，但缓解利用漏洞措施和消毒程序的目标和用例却大不相同。我们在表I中总结了主要差异。

<p align="center">表1 Exploit mitigations VS Sanitizers</p>

|                    | 漏洞利用缓解措施 | 消毒程序 |
| ------------------ | ---------------- | -------- |
| 目标               | 缓解攻击         | 查找漏洞 |
| 应用场景           | 生产             | 预发行   |
| 绩效预算           | 非常有限         | 高得多   |
| 违规结果           | 程序终止         | 问题诊断 |
| 在错误位置触发违规 | 有时             | 总是     |
| FPs公差            | 零               | 更高一点 |
| 残差良性错误的结果 | 期望的           | 不想要的 |

两种工具之间的最大区别在于它们执行的安全策略的类型。利用缓解措施部署旨在检测或预防攻击的策略，而消毒程序则旨在查明错误的程序语句的确切位置。控制流完整性（CFI）[10]，[11]，数据流完整性（DFI）[12]和写完整性测试（WIT）[13]是缓解漏洞的示例，因为它们检测到与合规的控制流或数据流路径之间的偏差，这通常是由于漏洞的利用而发生的，但不一定非得发生在易受攻击的程序语句的确切位置。相反，可以将边界检查工具视为消毒程序，因为违反其策略的行为直接在易受攻击的语句位置触发。

一些工具有选择地应用消毒技术，可能还结合了漏洞利用缓解措施的技术。例如，代码指针完整性（CPI）仅在程序直接或间接访问敏感代码指针时才执行边界检查（许多消毒程序中使用的一种消毒程序技术）[14]。因此，我们将CPI视为一种漏洞利用缓解措施的方法，而不是一种消毒程序，因为CPI仅检测可以使用边界检查检测到的所有错误的一小部分。

漏洞利用缓解措施的目的是在生产中部署，因此对各个设计方面提出了严格的要求。首先，漏洞利用缓解措施如果导致不可忽略的运行时间开销，则很少能被实际采用[15]。消毒程序对性能的要求不那么严格，因为它们仅用于测试。第二，漏洞利用缓解措施中的误报检测是不可接受的（也就是对容错性要求较高），因为它们会终止程序。而如果开发人员本意是查看错误报告（报错日志），则消毒程序可以容忍错误的警报。最后，出于可靠性和可用性的原因，在生产系统中允许存在残差的良性错误（例如，写入填充），并且经常会希望这种良性错误；而消毒程序的目的是精确地检测这些错误，因为它们的可利用性（就是程序到底会不会执行到这一段代码）未知。

#### III. LOW - LEVEL VULNERABILITIES



#### IV. BUG FINDING TECHNIQUES



#### V. PROGRAM INSTRUMENTATION



#### VI. META DATA MANAGEMENT



#### VII.  RIVING A SANITIZER



#### VIII. ANALYSIS



#### IX. DEPLOYMENT



#### X. FUTURE RESEARCH AND DEVELOPMENT DIRECTIONS
