# 为什么是 a fully differentiable programming language?

在描述能力上，理论上 （1）"imperative language" 加上 （2）"if" and "goto" statements, 或者 "branch if zero" 就足以描述所有能够被计算机处理的计算逻辑，high-level programming 会设计出很多semantics和语言特性来方便用户来使用。

-   导数计算是机器学习任务的核心，并且是一个非常规则化和流程化的计算过程，因此非常适合用计算机程序自动化完成导数的计算。
-   但另一方面，high-level programming language 提供的一些语言特性远远超过了 "描述任计算过程" 所需要的。
-   在自动实现AD的过程中，通过尽可能的保留 programming language 的语言特性，同时应用足够的程序优化技术，来改善用户体验。
    -   imperative programming language 的灵活性和表达能力 + compiled programming language 的性能和优化。
    -   解决 two language contexts带来的用户体验痛点。

# 为什么一定要把AD实现在 language/compiler level？

更多是为了优化，在编译器做更多的事情。

例如，对JPL这种有staged programming思想和（一定的）能力的语言，结合独特的类型系统，通过value types + generated function：（1）可以在编译期特化出针对具体的矩阵大小的实现；（2）在用户真正写程序的那一刻很多shape信息都已经是确定的，可以实现编译期的各种shape合法性检查。

通常有两种实现AD的方式：

1.  动态的Tracing / operator overloading
    -   不需要显示地处理control-flow，运行时会得到一个**线性的**function call traces（概念上的Tape），通过反向replay Tape 来实现自动的反向。
    -   实现很简单，并且有足够的灵活性。缺点是失去了很多静态分析和全局优化的机会。
2.  静态的STC（source code transoformation）
    -   需要显示地处理 control-flow。程序被表示成 basic block 为节点，block之间控制流跳转为边的 control flow graph。
    -   生成反向计算时需要解决 control-flow reverse 的问题。
    -   对一些动态类型语言需要解决Type inference 问题。
    -   数学函数$f(x)$是可导的，但$f(x)$的实现不一定能直接生成反向计算，需要静态检查。

机器学习任务的输入通常是高维向量，输出（loss function）是一个scalar。反向AD的复杂度与输出维度线性相关，反向AD更适合机器学习任务， 不论是Tracing还是STC都需要引入runtime data structure 来记录前向计算的中间结果，在通常的实现中，会引入stack来保存前向计算一个basic block中的context。

# 为什么一层IR对language-level的AD是必须的 ？

编译器的后端最终会负责最终的binary code optimization，而反向的计算在构建AST，做完type inference，parser之后就可以拿到所需要的全部信息。在进入编译器后端的代码生成之前这一层"language IR"上做更方便。但并不是每一个language在parser之后都有这样一层暴露给开发者的IR。

在STC实现的AD中，遇到过两种形式 IR。（1）源自lamba演算的 functional reprensentation。这层IR非常接近LISP的语法；（2）SSA。

1.  把语言的各种表面语法归一化。
2.  构建CFG，方便进行各种data flow analysis。
3.  控制 side effects （从这一点上看，Python非常不适合SCT：变量类型运行时决定并且可变；imperative language 大量利用了side effects）
    -   side effects 会产生复杂的读写依赖，让data dependency analysis 和 code optimization 复杂化。
    -   reverse mode AD的计算逻辑对保留前向计算结果有一定的要求，在反向计算完之前需要被保留，不可以被修改。需要控制用户能够使用的语言特性，保证反向计算可以被静态地生成出来。
    -   控制side effects的一层IR对生成high-order gradient function很重要。使得可以对生成出来的反向，重复的应用SCT去生成高阶导数的计算。
