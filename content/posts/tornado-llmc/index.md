---
date: 2024-10-28T16:58:13+02:00
title: "Run llm.c in TornadoVM"
description: "A lab on executing GPT-2 in a Java app on accelerators"
featured_image: "featured_image.jpg"
tags: ["CUDA", "parallelcomputing", "Java", "LLM"]
disable_share: true
draft: true
---

[TornadoVM](https://github.com/beehive-lab/TornadoVM) lets Java programs execute on accelerators, and [llm.c](https://github.com/karpathy/llm.c) is a plain C implementation of OpenAI‘s GPT-2, the LLM that powered the 1st ChatGPT. Released in fall ‘22, it sparked an AI hype that still lasts. Both are not a perfect fit at first glance, but a Java version of llm.c could make them friends, so I tried to bring them together.

Although there was already a Java port of llm.c, I made my own to get (back) into the groove of Java. I defined some obvious classes, turned C functions into Java methods, replaced pointers with array inidices, used Java Streams instead of OpenMP for parallelization, and leveraged the Java Vector API for matrix multiplication (the latter taken from [llama2.java](https://github.com/mukel/llama2.java), thx for sharing [@TheMukel](https://github.com/mukel)).

My port is in the `main` branch of [llm.java](https://github.com/otabuzzman/llm.java). Performance compared to llm.c is almost equal. The measures from my [Lenovo Ideapad Gaming 3](https://psref.lenovo.com/syspool/Sys/PDF/IdeaPad/IdeaPad_Gaming_3_15IHU6/IdeaPad_Gaming_3_15IHU6_Spec.html) (i5 4-core/ 16 GB, Intel Irix Graphics, NVIDIA RTX 3050 ti 4 GB) are for a single _forward_ pass that applies the entire GPT stack exactly once.

|Version|Vector API|forward pass (ms)|
|:---|:---:|:---:|
|llm.c/ C|n/ a|3712|
|llm.c/ Java|yes|3656|
|llm.c/ Java|no|4225|

The layer methods contained in the stack are ideal for parallelization because they all contain nested `for`-loops, and TornadoVM is designed to run the body of a `for`-loop in parallel on an accelerator (to make a long story short).

## TornadoVM with a single task graph

The plan was to put the whole stack of layer methods into a single task graph. TornadoVM would then arrange for the tasks in the graph to be executed one by one with parallelized `for`-loops on the accelerator.

I defined a `TaskGraph` instance and added layer by layer using `.addtask`, with each layer defined by a `TaskPackage` instance. Before the first and after the last `.addtask` I inserted `.transferHosttodevive` and `.transferdevicetohost` respectively to move data and result buffers forth and back.

It seemed that 12 transformer blocks, with 10 layers each, plus 1 input and 3 output layers was too much, as TornadoVM was showing errors that essentially said two internal arrays are too small (see issues [#1](https://github.com/otabuzzman/llm.java/issues/1) and [#2](https://github.com/otabuzzman/llm.java/issues/2)). By experimentally rebuilding TornadoVM with increased array sizes, the task graph was executed, but now the stack produced incorrect results.

table (updated) llm.c llm.java/ main llm.java/ tornado

The single task graph implementation took 115818 ms for the _forward_ pass which was rather slow (see issue [#3](https://github.com/otabuzzman/llm.java/issues/3), but may have been due to resource limitations. It's in the [`tornado-single-tg` branch](https://github.com/otabuzzman/llm.java/tree/tornado-single-tg/src/com/otabuzzman/llmj/GPT2.java) of the llm.java repo.

On the plus side, however, were all the code changes required for TornadoVM, namely
1. replaced the Java Math API with the Tornadomath API,
2. replaced Java Buffers for `int`s and `float`s by those that TornadoVM can move between host and accelerator,
3. annotated `for`-loops inside the layer methods to be parallelized by TornadoVM,
4. appropriate TornadoVM buffers added to the signatures of the layer methods,
5. code changes to execute the task graph instead of individual layer methods.

## TornadoVM with multiple task graphs

I set up another configuration that allows a seamless mix of original layer methods and TornadoVM tasks and let me proceed layer by layer and fix errors as they occur.

I defined a task graph for a single transformer block in advance, before iterating over the stack of blocks. At first the task graph would contain only the task for the first layer method in a transformer block. Each iteration would then run the partial block in the task graph followed by the original layer methods for missing tasks.

The layer methods receive array indices via method parameters to reflectthe current layer. The original code updates these indices as it iterates layewr by layer over the transformer blocks. For the now predefined transformer block, its (now as well predefined) parameters must become indices pointing to the actual (real) indices, updated with each iteration. This additional level of indirection requires another buffer to be transferred before each task graph execution.

This approach eventually worked and let me execute the full stack on accelerators with TornadoVM. It's in the [`tornado-multi-tg` branch](https://github.com/otabuzzman/llm.java/blob/tornado-multi-tg/src/com/otabuzzman/llmj/GPT2.java) of the llm.java repo.

table (updated) llm.c llm.java/ main llm.java/ tornado

## Findings

1. The matmul task for matrix multiplication yields a runtime error depending on the code structure. Could be fixed by moving parts of code around (trial and error).

2. The inner `for`-loop in the attention task (not the annotated parallelized loops) ends early when the termination condition references a control variable of one of the surrounding parallelized `for`-loops. The generated kernel shows a wrong block sequence that lets it terminate immediately after entering the respective `for`-loop. Could be fixed by changing the termination logic for the inner `for`-loop in question. A message from TornadoVM's option `-Dtornado.parallelise.auto=true` for automatic parallelization (which I happened to try when I was running out of reasonable ideas) pointed me in the right direction: it said that it doesn't support loop dependencies.

3. The attention task yields a runtime error depending on a combination of code structure and backend. The SPIR-V backend works with a condition check at the beginning of an inner `for`-loop, whereas the OpenCL backend needs the check at the end of the loop. Could be fixed by using the PTX backend which works in both cases.

4. Putting all tasks of a transformer block into a single task graph yields a runtime error when executing with the OpenCL backend (see issue [#4](https://github.com/otabuzzman/llm.java/issues/4)). Could be fixed by using the SPIR-V or the PTX backend.

5. Using break in a `for`-loop inside a task yields runtime error. Could be fixed by refactoring program logic.

## Conclusion

Lorem ipsum...
