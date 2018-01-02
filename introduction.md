# Loo.py编程系统的实现方式以及多面体模型


## 程序转换技术  

**程序转换(Program Transformation)**是指将一个计算机程序进行转换，并根据需要生成另一个程序的技术。大多数情况下，被转换的程序需要与原始程序在语义上等价，但在很少的情况下，转换导致程序在语义上也可与原始程序在可预见的方式上有所不同。一般来说，语义等价性是一种程序细化(Program Refinement)的概念：如果一个程序终止于原始程序终止的所有初始状态，且对于每个这样的状态，确保它可以在原始程序的某个可能的终止状态终止，则这个程序是原始程序的细化。换句话说，一个程序的细化比这个源程序更加明确，且如果两个程序互为对方的细化，则这两个程序是等价的。   

虽然这种转换可以是手动的人工转换，但对于更大规模的代码或者需要针对多种不同情况，使用一个程序转换系统来进行所需要的转换更加实用。程序转换可以是修改表示程序的数据结构（如抽象语法树AST）的自动程序，也可以通过使用特定模式来表示参数化源代码文本片段来指定。  

源代码转换系统通常需要整合需要转化的编程语言的完整前端，包括源代码解析，代码结构的内部程序表示的建立，程序符号的含义，有效的静态分析，以及根据转换后的程序表示重新生成的有效源代码。  

程序转换技术已广泛运用于许多软件工程领域，如程序综合，程序优化，程序重构，逆行工程，及文档生成等。本组所研究的Loo.py即为一种基于程序转换的针对GPU和CPU的代码生成工具。  


## 使用多面体模型表示程序  

**多面体模型**是一种基于线性代数来表示程序和程序转换的计算模型，它应用了丰富的数学理论和直观的几何解释，且作为AST的改进，适合表示串行以及并行程序，并为分析和应用程序转换提供了方便的抽象模型。在程序的自动并行化和优化的处理方面上，已经通过应用多面体模型，已经取得了巨大的成效。

大致来说，多面体模型的应用框架是传统编译过程的一个插件。对它的应用分为以下步骤：
1. 首先，从抽象语法树开始，将适合多面体模型的部分程序翻译成线性代数表示；
2. 下一步，通过使用一种重新排序函数，来选择新的代码执行顺序。如何来寻找最为适合的代码执行顺序正是大多于对于多面体模型研究的重点；
3. 最后，进行代码生成，返回原有抽象生成树，或实现根据代码重排函数指定的执行顺序的新的源代码。

多面体模型是一种既适用于串行程序，也适用于并行程序的计算模型和表示方法。它对应了命令式语言的一种被称为静态控制程序的子集，包括C, FORTRAN等语言。这些语言的性质可被大致概括为：
1. 控制语句是具有仿射边界的do循环语句和具有仿射条件的if条件语句；
2. 仿射边界和条件仅取决于外围循环计数器和常量参数。

在任何程序中，连续的具有静态控制的语句的最大集合被成为静态控制部分。下面的程序即为一个严格的静态控制程序的例子：
```FORTRAN
    do i = 1, n
S1      x = a(i,i)
        do j = 1, i - 1
S2          x = x - a(i,j) ** 2
S3      p(i) = 1.0 / sqrt(x)
        do j = i + 1, n
S4          x = a(i,j)
            do k = 1, i - 1
S5              x = x - a(j,k) * a(i,k)
S6          a(j,i) = x * p(i)
```

这类程序中的循环可以用迭代矢量——n维列向量**x**=(i<sub>1</sub>,i<sub>2</sub>,...,i<sub>n</sub>)<sup>T</sup>表示，其中i<sub>k</sub>是第k层循环的循环迭代变量（如在最外层`do i = 1, n`循环中，循环变量为i，k为1，故i<sub>k</sub>=i），n是最内层循环的层数。考虑这类静态控制集合，对于每个语句可以用两种属性来表示，以此便可以完整地描述程序的执行，这两种属性分别为：
1. 迭代域(iteration domain)，即该语句需要执行的迭代矢量的取值的集合。  
When a statement is surrounded with static control, its iteration domain can always be specified by a set of linear inequalities defining a polyhedron. The tern polyhedron will be used in a broad sense to denote a convex set of points in a lattice, i.e. a set of points in a Z vector space bounded by affine inequalities.  
S2语句的迭代矢量为(i,j)<sup>T</sup>，故它的迭代域为由i，j在迭代中的取值范围确定的矢量空间，在外层循环中，i的取值为从1到n，内层循环中，j的取值为1到i-1。故S2语句的矢量空间为由i≥1，i≤n，j≥1和j\<i确定的域，如下图：  
![The correspondence between static control and polyhedral domains for the statement S2 of the program above](https://github.com/wwqqqqq/loopy-introduction/raw/master/pic/polyhedron.png)  


1. 散射函数(scattering function) θ(**x**)：an affine function specifying for each integral point in the iteration domain a new coordinate for the corresponding statement instance
根据上下文的不同，散射函数可能有如下的几种解释：根据空间（跨处理器）分配iterations，或根据时间对它们进行排序，或两者都有。
若是空间映射(space-mapping)的情况，对于给定的语句，θ(**x**)函数返回执行这条语句的处理器号；若用于n维的时间安排(time-schedule)，散射函数返回一个n维向量，表示该语句执行的逻辑时间(logical date)。逻辑时间为(a<sub>1</sub>...a<sub>n</sub>)的语句在对应时间为(b<sub>1</sub>...b<sub>n</sub>)的语句之前执行，当且仅当存在i，1≤i\<n，使得(a<sub>1</sub>...a<sub>i</sub>)=(b<sub>1</sub>...b<sub>i</sub>)，且a<sub>i+1</sub>\<b<sub>i+1</sub>，即不同语句的执行顺序遵循它们逻辑时间的字典序。    

这样以来，我们可以通过每个语句所处的循环嵌套情况和顺序，通过改进的AST来表示该程序的顺序执行顺序。以上面的程序为例，它的AST表示为：  
![AST of the program above, using polyhedral model](https://github.com/wwqqqqq/loopy-introduction/raw/master/pic/AST.png)  
散射函数为：  
θ<sub>S1</sub>(**x**<sub>S1</sub>)=(0,i,0)<sup>T</sup>  
θ<sub>S2</sub>(**x**<sub>S2</sub>)=(0,i,1,j,0)<sup>T</sup>  
θ<sub>S3</sub>(**x**<sub>S3</sub>)=(0,i,2)<sup>T</sup>    

使用多面体模型的程序转换可以用适当的散射函数来确定。They modify the source polyhedra into target polyhedra containing the same points but in a new coordinate system, thus with a new lexicographic order.


## Loo.py简介
Today's highly heterogeneous computing landscape places a burden on programmers wanting to achieve high performance on a reasonably broad cross-section of machines.  
To do so, computations need to be expressed in many different but mathematically equivalent ways, with, in the worst case, one variant per target machine.
Loo.py, a programming system embedded in Python, meets this challenge by defining a data model for array-style computations and a library of transformations that operate on this model.  
Offering transformations such as loop tiling, vectorization, storage management, unrolling, instruction-level parallelism, change of data layout, and many more, it provides a convenient way to capture, parametrize, and re-unify the growth among code variants.
Optional, deep integration with numpy and PyOpenCL provides a convenient computing environment where the transition from prototype to high-performance implementation can occur in a gradual, machine-assisted form.  

#### Introduction:  
As computer achitectures and execution models diversify, the number of mathematically equivalent ways a single computation can be expressed is growing rapidly. Unfortunately, only very few of these program variants achieve good machine utilization, as measured in, e.g. percentages of peak memory bandwidth or floating point throughput.  
//(传统approach)  
Optimization compilers that, with or without the help of user annotations, equivalently rewrite user code into a higher-performing variant have been the standard solution to this issue, although the goal of a compiler whose built-in optimization passes robustly make the sometimes complicated trade-offs needed to achieve good performance has remained somewhat elusive.
//Loo.py
Loo.py takes a different approach. Loo.py code is most often embedded in an outer controlling program in the high-level programming language Python. The user first specifies the computation to be carried out in a language consisting of a tree of polyhedra describing loop bounds along with a list of instructions, each tied to a node in the tree of polyhedra. The specification provided by the user is deliberately only weakly ordered, providing freedom to the code generator.

Once a computation is specified as described above, its description is held within an object which is open to inspection and manipulation from within the host language. These manipulations occir by applying a variety of transformations that Loo.py exactly preserve the semantics of the specified code. This is different from the conventional compiler approach in a number of important ways:
- Intermediate representations are deliberately open and intended to be inspected and manipulated by the user. An advanced user can easily implement their own transformations, extending the library already available.
- Instructions, loop bounds, and transformations together uniquely specify the code to be generated. Loo.py does not attempt to be intelligent or make choices on behalf of the user, all while retaining an interface high-level enough to be usable by moderately technical end users.
- Conventional compilers carry a considerable burden in proving that any rewriting they apply does not change the observable behavior of the program. Explicitly invoked transformations allow more flexibility. By invoking a transformation, the user may assume partial responsibility for its correctness. This puts changes within reach that would be difficult or impossible to apply with conventional compiler architectures, such as changes to globally visible data layouts.
- Unlike traditional 'pragma'-type compiler directives, transformations are applied under the control of a full scale programming-language. This means that code generation can react to the target hardware or the workload at hand.

In addition, control from a high-level programming environment encourages reuse and abstraction within the space of transformations, which aids users in dealing with larger-scale code generation tasks, in which, possibly, a large number of similar computational kernels need to be generated.


## 参考文献

[Code Generation in the Polyhedral Model Is Easier Than You Think](https://dl.acm.org/citation.cfm?id=1025992) 

[Loo.py: transformation-based code generation for GPUs and CPUs](https://arxiv.org/abs/1405.7470)


## 相关链接

[GitHub repository](https://github.com/01-Loopy/loo.py-intro)

[项目简介、分工以及进展记录](https://github.com/01-Loopy/2017fall-student-teamworks/blob/master/01-loopy.md)