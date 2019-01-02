
# <center> **论文笔记** </center>

**题目** ：K-Miner: Uncovering Memory Corruption in Linux  
**出处**：NDSS 2018  
**作者**：David Gens, Simon Schmitt, Simon Schmitt and Ahmad-Reza Sadeghi  
**单位**：CYSEC/Technische Universitat Darmstadt, Universität of Duisburg-Essen  
**原文**： [http://wp.internetsociety.org/ndss/wp-content/uploads/sites/25/2018/02/ndss2018_05A-1_Gens_paper.pdf](http://wp.internetsociety.org/ndss/wp-content/uploads/sites/25/2018/02/ndss2018_05A-1_Gens_paper.pdf)  
**作者研究方向**：kernel- userland isolation, chip-and firmware security, static analysis and formal verification  
**相关材料**：
[Slides](http://wp.internetsociety.org/ndss/wp-content/uploads/sites/25/2018/03/NDSS2018_05A-1_Gens_Slides.pdf)、
[Video](https://www.youtube.com/watch?v=1uLKA7Ux8hg&index=1&list=PLfUWWM-POgQurVIGZX6g4huGVe4LOIpu3&t=87s)、
[SourceCode](https://github.com/ssl-tud/k-miner)、
[HomePage](http://david.g3ns.de/)、
[Lab](https://www.trust.informatik.tu-darmstadt.de/people/david-gens/)  


# **一、背景**
&ensp;&ensp;&ensp;&ensp;目前，已经有很多种在运行时保护系统的方法（例如：控制流完整性CFI等），运行时保护系统并不能消除造成内存崩溃（memory corruption）的根本原因。然而，与运行时相对应的是静态源码分析，但是在目前的静态分析框架中，存在很多不完善的地方，例如，这种方法只能限制在局部的、过程内的检查，或者是只能基于单个源文件的检查（这些限制都是由于路径爆炸造成的），这对于代码量庞大的工程（例如：Linux内核）来说，是根本行不通的。  
&ensp;&ensp;&ensp;&ensp;为了克服以上的这些不足，作者在这篇文章中提出了一种新的、基于LLVM的静态源码分析框架：K-Miner，它能够分析大规模的内核源码（例如：Linux内核），采用**大规模指针分析**（scalable pointer analysis）、**过程间分析**（inter-procedural analysis）和**全局-上下文敏感分析**（global, context-sensitive analysis）等技术，可以系统的挖掘几种不同类型的内存崩溃漏洞（包括dangling pointers, user-after-free, double-free 和 double-lock）。  


# **二、面临的挑战**
&ensp;&ensp;&ensp;&ensp;静态分析方法必须考虑所有可能的路径和状态，这就导致路径爆炸问题，但是，剪枝或者是忽略某些部分（路径或状态等）都会导致分析结果不可靠。这就限制了静态源码分析只能是基于局部的过程内分析，或者是基于单个文件的分析。  
&ensp;&ensp;&ensp;&ensp;而在本文中，作者为了处理上述问题，采用的方法是：基于系统调用接口来分割内核代码，每个系统调用作为一个执行（分析）路径的入口，这样就有效的解决路径爆炸问题，使得K-Miner可以分析复杂的内核代码，并可以执行过程间的数据流（inter-procedural data-flow analysis）分析。  
&ensp;&ensp;&ensp;&ensp;但是，分割内核代码也不是一件简单的事情，因为在内核中，有很多的全局数据结构被频繁的使用，以及每个系统调用之间的同步问题，全局内存状态（context）问题，复杂的指针关系和别名问题等。  
&ensp;&ensp;&ensp;&ensp;具体来说，K-Miner面临四个挑战：  
1. **处理全局状态问题：**  
&ensp;&ensp;&ensp;&ensp;在静态分析过程中，要想执行过程间分析，就必须考虑全局的数据结构，因此，局部的指针访问（例如函数内部）可能会影响全局的数据结构，因为局部指针可能只是一个别名（alias）而已。本文中使用的是稀疏程序表示（sparse program representation）技术，并结合控制流分析和数值流分析等技术来解决全局状态问题。  
2. **处理代码量太大问题：**  
&ensp;&ensp;&ensp;&ensp;目前的Linux内核代码超过240万行，对于如此巨大的代码量，对它直接进行静态分析是及其困难的，因此，作者通过分割内核代码的方式，按照系统调用接口把它进行划分，这样就可以极大的减少单次分析的代码量，使K-Miner分析Linux内核成为可能。  
3. **处理误报率太高的问题：**  
&ensp;&ensp;&ensp;&ensp;粗粒度的近似分析程序行为，出现高误报率是不可避免的，而开发者没有这么多的时间去手工分析大量的误报结果，因此，减少误报率是提高程序可用性的重要基础。作者利用尽可能多的信息，并精心设计每次的分析过程（individual analysis passes）来减少误报率，并在生成报告之前，对分析结果进行净化（sanitize）、去重（deduplicate）和过滤（filter），最大限度的减少误报率。  
4. **多路分析问题：**  
&ensp;&ensp;&ensp;&ensp;为了尽可能多的检测造成内存崩溃的bug，K-Miner必须能够同步每次的分析结果（individual pass），并且利用已知的结果去分析未知的结果，提高分析的准确性和精确性。为了提高分析效率，还必须重用LLVM产生的中间结果（IR）,因此，IR产生的结果首先被保存到磁盘中，然后在后序的分析中重新导入。  


# **三、关键技术之数据流分析**
&ensp;&ensp;&ensp;&ensp;静态分析的通用方法是把程序和一些预编译的属性（pre-compiled properties）作为输入，对于给定的属性，去查找所有可能的路径。而这里的属性包括：活性分析（liveness analysis）、死码分析（dead-code analysis）、类型状态分析（typestate analysis）和空值分析（nullness analysis）。  
&ensp;&ensp;&ensp;&ensp;在图1中，子图a是一个简单的示例代码，通过分析子图c的指针分配图PAG（Pointer Assignment Graph）我们可以知道，存在一条路径，使得变量b被赋值为空，因此会产生一个对应的空值报告。  
<div align="center">
    <img src="Material\\Figure1.jpg"/>
</div>   
&ensp;&ensp;&ensp;&ensp;此外，在静态分析中，另外一个被使用的数据结构是：过程间控制流图ICFG（Inter-procedural Control-Flow Graph）,如子图b所示，它是一个全局的控制流图，可以被用来执行路径敏感性分析（path-sensitive analysis）。最后子图d是一个数值流图VFG（Value-Flow Graph）,该图作为污点分析技术的辅助图，可以用来跟踪单个的内存对象。  


# **四、前提假设**
&ensp;&ensp;&ensp;&ensp;本文提出的静态分析框架基于以下三个前提假设： 
- 攻击值可以控制用户空间的进程，并且可以调用任何的系统调用。  
- 操作系统独立于用户进程，例如，通过虚拟内存机制或者是划分不同权限级别等，经典的是x86架构和ARM架构。  
- 攻击者不能插入恶意代码到内核中（内核是被签名的）。  


# **五、K-Miner运行机制**
<div align="center">
    <img src="Material\\Figure2.jpg"/>
</div>
&ensp;&ensp;&ensp;&ensp;如图2所示，K-Miner是基于LLVM和过程间静态数值流分析（SVF）之上静态源码分析框架，的主要的工作流程分为三步：  

1. K-Miner首先接收两个输入，一个是内核代码，另一个是一份配置文件，该配置文件是内核特征（kernel features）的一个列表，用来告诉前端解析器，哪些内核模块需要编译，哪些不需要编译。编译器根据这两个输入去构建抽象语法树（AST）,并把它转化为中间表示语言（IR）,作为后序阶段的输入。  
2. 把上一个阶段得到的中间表示语言作为本阶段的输入，去遍历配置文件中列出来的每一个系统调用，对于每一个系统调用，K-Miner都生成以下数据结构：一个调用图（CG）、一个指针分析图（PAG）、一个数值流图（VFG）和几个相关的内部数据结构。有了以上结果，就可以对每一个系统调用进行相应的分析（例如，空指针解引用分析等）。
3. 对于第二步中的分析结果，如果检测到有内存漏洞，则在这一步中产生相应的分析报告（报告包含相应的内核版本、配置文件、系统调用以及程序执行路径等）。


# **六、K-Miner实现细节**

## **1. Global Context 初始化**  
&ensp;&ensp;&ensp;&ensp;Global Context初始化是分析每一次pass的前提基础，有效的管理Global Context是进行大规模代码分析的必备条件，它分为以下三个部分：  
- **内核上下文初始化**：由于内核的初始化是通过调用一系列的InitCalls来完成的，所以，作者通过模拟内核中的InitCalls的执行来填充（初始化）内核上下文，并将结果存入文件中，方便后序导入。
- **跟踪堆的分配**：由于在内核中分配内存有多种形式（slab allocator、low-level page-based allocator和various object caches等），因此，K-Miner把这些内存分配函数都标记为堆分配函数，以便于后序的数值流分析。
- **建立系统调用上下文**：在分析每一个系统调用图的过程中，作者通过收集它们所用到的所有全局变量和函数来建立一个memory context，然后再结合global context，来建立一个更加准确的memory context。

## **2. 分析每个系统调用的内核代码**
&ensp;&ensp;&ensp;&ensp;仅仅以系统调用为单位来分割内核代码，虽然能极大的减少每一次分析过程中的代码量，但是对于作者32G内存的实验环境，目前依然吃不消，需要进一步的优化。通过分析发现，消耗资源的主要原因是函数指针的剧增，因为在通常情况下，对于每一个系统调用来说，最朴素的想法是：使得所有的全局变量和函数都可达，这种方法确实会比较安全，但是也很低效，因此，作者使用了如下三个方法来改善这种朴素的想法：  
- **提高函数调用图的准确性**：在IR层分析所有函数的调用关系图，以确定某个函数是否可达（例如：通过判断全局函数是否被局部变量访问），这就使得对于每一个系统调用，可以得到一个更加简练的可达函数集合。
- **流敏感（Flow-Sensitive）指针分析**：首先通过一个基于闭包（inclusion-based）的指针分析策略来约束上一步的分析结果，然后再结合控制流分析，进一步简练每一个系统调用的可达函数集合。
- **结合全局内核上下文分析**：对于上一步的分析结果，把它结合全局内核上下文，再次分析每一个系统调用，进一步确定它所能到达的全局变量和全局函数，并把缺失的引用包含进来，得到一个更加完整的、更加精确的context。

## **3. 最小化误报率**
&ensp;&ensp;&ensp;&ensp;使用静态分析方法来近似的逼近程序行为，不可避免的产生误报情况，而作者在文章中使用Dangling Pointer为例，分析如何较少误报率。  
<div align="center">
    <img src="Material\\Figure4.jpg"/>
</div>  

- **精确的数据流分析**：  
&ensp;&ensp;&ensp;&ensp;如图4所示，子图a模拟了一个具有指针悬空漏洞的系统调用，在add_x函数中，局部指针变量p被赋值给全局指针变量global_p；在remove_x函数中，把全局指针变量global_p置空，但是在do_foo函数中，我们可以看到，只有在满足特定条件（③）下才执行remove_x函数，因此存在指针悬空的漏洞。  
&ensp;&ensp;&ensp;&ensp;在子图b中，local_o和global_o分别表示相应的内存对象，K-Miner通过两个步骤来查找该bug：第一步，在子图c中，前向分析local_x这个节点所经过的位置，在这个节点的合法范围内（do_foo函数内），如果离开这个区域的出口数量大于进入这个区域的入口数量，那么就存在一个非法的引用。对于local_x，存在一条路径：local_x->①->②->③->⑤->⑥，在这条路径的最后一个节点:⑥,离开了local_x的合法范文之内； 第二步，反向遍历VFG图，查找对该节点的引用，只要存在这样的引用，就表明它是一个指针悬空漏洞，在子图c中可以找到：⑥->⑤->③->②->global_p，即存在一个指针悬空漏洞。此外，由于有了PAG图，在反向查找的时候就可以避免访问不属于当前跟踪的内存对象的边（例如：⑤->④）。最后把该bug所经过的路径信息和相应的节点等保存下来，以便后续处理。
- **净化报告**：  
&ensp;&ensp;&ensp;&ensp;通过以上的数值流分析之后，作者再次检查报告中的候选结果，并去除一些在运行时不可能出现的结果（例如：调用函数g，然后从函数f中返回）。此外，对于同一个节点，如果产生重复的报告，则把它们合并为一个报告，精简最后的输出结果。  


# **七、 评估**
&ensp;&ensp;&ensp;&ensp;作者的测试环境是：  
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;
CPU：Xeon E5-4650、8核处理器、2.4GHz的主频  
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;
内存：32G

## **1. 测试时间评估**  

<div align="center">
    <img src="Material\\Table1.jpg"/>
</div>  
&ensp;&ensp;&ensp;&ensp;如表一所示，是对不同版本的Linux内核代码的测试结果，每个版本的内核都测试了DP（Dangling Pointer）、 UAF（Use After Free）和DF（Double Free）。结果表明，对于每一个系统调用，平均大约需要25分钟测试时间（第四列的结果）。对于每一个内核版本，对所有系统调用都进行三项测试（DP、UAF、DF），所需时间在70小时到200小时之间。表中的最后三列表示被检测出来的可能的漏洞数量，括号中的数值表示产生的警告数量。  

<div align="center">
    <img src="Material\\Figure5.jpg"/>
</div>   
&ensp;&ensp;&ensp;&ensp;图5给出了测试一部分系统调用所需的时间（这里只显示测试时间超过30分钟的系统调用），图中的每个柱子被划分为四节，每一节的长度代表测试该类型所需要的时间，从上到下依次代表的测试对象是：Double Free Checker、Memory Leak Checker（Use-After-Free Checker）、 Use-After-Return Checker（Dangling Pointer）和Context Handling。其中，对于测试每一个系统调用，主要花费的时间是在Context Handling 上，因为Contex Handling是进行其它各项分析（例如：数值流分析等）的基础，是开始测试的第一阶段。但是，这里也有几个异常情况，分别是：sys_execve、 sys_madvise 和 sys_keyctl，这几个系统调用的DP测试时间与Context Handling 时间基本相等。  

## **2. 内存消耗评估**  

<div align="center">
    <img src="Material\\Table2.jpg"/>
</div>  
&ensp;&ensp;&ensp;&ensp;在本文中，由于作者使用分割内核的技术，使得内存使用情况在可以接受的范围内，即，对于分析每一个系统调用，平均内存使用量在8.7G到13.2G之间，最大值达到26G。而对于测试每一个版本的内核，内存的使用情况如表2所示：  


# **八、 结论**
&ensp;&ensp;&ensp;&ensp;作者开发了一个新颖的内核代码检测工具：K-Miner，该工具是第一个对系统内核进行数值流分析的框架，它对内核进行大规模的、精确的分析，并能够找到内核中存在的漏洞：CVE-2014-3153（dangling pointer 漏洞）和CVE-2015-8962（double free 漏洞），此外，它还是第一个能够对具有庞大代码量的系统内核进行过程间分析的框架，它对不同的漏洞类型提供了不同的passes，目前论文中已经完成的passes有：dangling pointers、 use-after-free和 double free，其它类型的漏洞还在慢慢完善的过程中。  


# **九、 文章解决的问题**
&ensp;&ensp;&ensp;&ensp;运行时防护措施无法根除bug，而现存的静态源码分析方法无法检测大规模工程，只能基于过程内（intra-procedural）的检测，或者是基于单个文件的检测，对于代码量庞大的工程（例如Linux内核等）无能为力，甚至对于跨函数或者是跨文件的工程都难以检测。因此，作者的目标是实现一个能够检测Linux内核这种大规模代码工程的静态分析工具，但是由于静态分析工具存在一个致命的缺陷，那就是路径爆炸问题，为了解决这个问题，作者使用分割技术，按照系统调用为单位，对内核进行分割，以解决路径爆炸问题。因此，作者实现了一个基于LLVM和SVF的、强有力的静态分析工具：K-Miner。  








