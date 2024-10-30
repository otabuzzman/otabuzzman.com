---
date: 2024-10-28T16:58:13+02:00
title: "Run llm.c in TornadoVM"
description: "A lab on executing GPT-2 in a Java app on accelerators"
featured_image: "featured_image.jpg"
tags: ["CUDA", "parallelcomputing", "Java", "LLM"]
disable_share: true
draft: false
---

[TornadoVM](https://github.com/beehive-lab/TornadoVM) lets Java programs execute on accelerators. [llm.c](https://github.com/karpathy/llm.c) is a plain C implementation of OpenAI‘s GPT-2, the LLM that powered the 1st ChatGPT. Released in fall ‘22, it sparked an AI hype that still lasts. Both are not a perfect fit at first glance, but a Java version of llm.c could make them friends, so I tried to bring them together.

Although there was already a Java port of llm.c, I made my own to get (back) into the groove of Java. I defined some obvious classes, turned C functions into Java methods, replaced pointers with array inidices, used Java Streams instead of OpenMP to parallelize `for`-loops, and leveraged the Java Vector API for matrix multiplication (the latter taken from [llama2.java](https://github.com/mukel/llama2.java), thx for sharing [@TheMukel](https://github.com/mukel)).

The port is in the `main` branch of [llm.java](https://github.com/otabuzzman/llm.java). Performance compared to llm.c is almost equal. The measures taken on my [Lenovo Ideapad Gaming 3](https://psref.lenovo.com/syspool/Sys/PDF/IdeaPad/IdeaPad_Gaming_3_15IHU6/IdeaPad_Gaming_3_15IHU6_Spec.html) (i5 4-core/ 16 GB, Intel Irix Graphics, NVIDIA RTX 3050 ti 4 GB) are for a single _forward_ pass that applies the entire GPT stack exactly once.

<!-- https://stackoverflow.com/a/65784095 -->
|Version|Vector API {{< line_break >}} enabled|forward {{< line_break >}} pass (ms)|
|:---|:---:|:---:|
|llm.c/ C|n/ a|3712|
|llm.c/ Java|yes|3656|
|llm.c/ Java|no|4225|

The layer methods contained in the stack are ideal for parallelization because they all contain nested `for`-loops, and TornadoVM is designed to run the body of a `for`-loop in parallel on an accelerator (to make a long story short).

## TornadoVM with a single task graph

The plan was to put the whole stack of layer methods into a single task graph. TornadoVM would then arrange for the tasks in the graph to be executed one by one with parallelized `for`-loops on the accelerator.

I defined a `TaskGraph` instance and added layer by layer using `.addTask`, with each layer defined by a `TaskPackage` instance. Before the first `.addTask` and after the last I inserted `.transferToDevice` and `.transferToHost` respectively to move data and result buffers forth and back.

```java
// code snippet

// the array indices passed via method parameters to the layer tasks
TensorPointers t = new TensorPointers(params, acts, B, T, C, NH);
// the input block
encoder_forward(acts.encoded, params.wte, params.wpe, B, T, C); // encoding goes into residual[0]

TaskGraph gpt2 = new TaskGraph("gpt2");
// transfer buffers to accelerator
gpt2.transferToDevice(DataTransferMode.EVERY_EXECUTION, params_memory, acts_memory);

// control variable `l´ counts the transformer blocks
for (int l = 0 ; l < L ; l++) {
    // update layer indices in `t´ for this iteration
    t.updateForLayer(l);

    // add layer tasks of transformer block
    gpt2.addTask(new TaskPackage("ln1" + "-l" + l, GPT2::layernorm_forward, params_memory, acts_memory, t.ln1, t.ln1_mean, t.ln1_rstd, t.residual, t.ln1w, t.ln1b, B, T, C));
    gpt2.addTask(new TaskPackage("mm1" + "-l" + l, GPT2::matmul_forward, params_memory, acts_memory, t.qkv, t.ln1, t.qkvw, t.qkvb, B, T, C, 3 * C));
    gpt2.addTask(new TaskPackage("at" + "-l" + l, GPT2::attention_forward, acts_memory, t.atty, t.preatt, t.att, t.qkv, B, T, C, NH));
    gpt2.addTask(new TaskPackage("mm2" + "-l" + l, GPT2::matmul_forward, params_memory, acts_memory, t.attproj, t.atty, t.attprojw, t.attprojb, B, T, C, C));
    gpt2.addTask(new TaskPackage("rs1" + "-l" + l, GPT2::residual_forward, acts_memory, t.residual2, t.residual, t.attproj, B * T * C));
    gpt2.addTask(new TaskPackage("ln2" + "-l" + l, GPT2::layernorm_forward, params_memory, acts_memory, t.ln2, t.ln2_mean, t.ln2_rstd, t.residual2, t.ln2w, t.ln2b, B, T, C));
    gpt2.addTask(new TaskPackage("mm3" + "-l" + l, GPT2::matmul_forward, params_memory, acts_memory, t.fch, t.ln2, t.fcw, t.fcb, B, T, C, 4 * C));
    gpt2.addTask(new TaskPackage("ge" + "-l" + l, GPT2::gelu_forward, acts_memory, t.fch_gelu, t.fch, B * T * 4 * C));
    gpt2.addTask(new TaskPackage("mm4" + "-l" + l, GPT2::matmul_forward, params_memory, acts_memory, t.fcproj, t.fch_gelu, t.fcprojw, t.fcprojb, B, T, 4 * C, C));
    gpt2.addTask(new TaskPackage("rs2" + "-l" + l, GPT2::residual_forward, acts_memory, t.residual3, t.residual2, t.fcproj, B * T * C));
}
// add tasks of final output block
int residual = acts.residual3 + (L - 1) * B * T * C; // last residual is in residual3
gpt2.addTask(new TaskPackage("ln", GPT2::layernorm_forward, params_memory, acts_memory, acts.lnf, acts.lnf_mean, acts.lnf_rstd, residual, params.lnfw, params.lnfb, B, T, C));
gpt2.addTask(new TaskPackage("mm", GPT2::matmul_forward, params_memory, acts_memory, acts.logits, acts.lnf, params.wte, -1, B, T, C, Vp));
gpt2.addTask(new TaskPackage("sm", GPT2::softmax_forward, acts_memory, acts.probs, acts.logits, B, T, V, Vp));
// retrieve result buffer from accelerator
gpt2.transferToHost(DataTransferMode.EVERY_EXECUTION, acts_memory);

// execute the full GPT-2 stack in TaskGraph `gpt2´
TornadoExecutionPlan gpt2_runner = new TornadoExecutionPlan(gpt2.snapshot());
gpt2_runner.execute();
try { gpt2_runner.close(); } catch (TornadoExecutionPlanException e) { throw new UnexpectedException(null); }

```

It seemed that 12 transformer blocks, with 10 layers each, plus 1 input and 3 output layers was too much, as TornadoVM was showing errors that essentially said two internal arrays are too small (see issues [#1](https://github.com/otabuzzman/llm.java/issues/1) and [#2](https://github.com/otabuzzman/llm.java/issues/2)). By experimentally rebuilding TornadoVM with increased array sizes, the task graph was executed, but now the stack produced incorrect results.

|Version|Vector API {{< line_break >}} enabled|forward {{< line_break >}} pass (ms)|
|:---|:---:|:---:|
|single TaskGraph|n/ a|49729|

The single task graph implementation was also quite slow (see issue [#3](https://github.com/otabuzzman/llm.java/issues/3), which was probably due to resource limitations. It's in the [`tornado-single-tg` branch](https://github.com/otabuzzman/llm.java/tree/tornado-single-tg/src/com/otabuzzman/llmj/GPT2.java) of the llm.java repo.

The positive thing is that I had now made all the code changes required for TornadoVM, namely
1. replaced the Java Math API with the TornadoMath API,
2. replaced Java Buffers for `int`s and `float`s by those that TornadoVM can move between host and accelerator (e.g. replaced `IntBuffer` (Java) with `IntArray` (TornadoVM)),
3. annotated `for`-loops inside the layer methods to be parallelized by TornadoVM,
4. appropriate TornadoVM buffers added to the signatures of the layer methods,
5. code changes to execute the task graph instead of individual layer methods.

## TornadoVM with multiple task graphs

I set up another configuration to allows a seamless mix of original layer methods and TornadoVM tasks and let me proceed layer by layer and fix errors as they occur.

I defined a task graph for a single transformer block before iterating over the stack of blocks. At first the task graph would contain only the task for the first layer method in a transformer block. Each iteration would then run the partial block in the task graph followed by the original layer methods for missing tasks.

```Java
// code snippet

// the array indices in `ind´ are passed to the accelerator via memeory transfer
// and accessed by the layer tasks via indices.
TensorIndices ind = new TensorIndices(params, acts, B, T, C, NH);

// task graph of a single transformer block
TaskGraph transformer_block = new TaskGraph("tb")
.transferToDevice(DataTransferMode.FIRST_EXECUTION, params_memory, acts_memory)
.transferToDevice(DataTransferMode.EVERY_EXECUTION, ind.tensors)
.task("ln1", GPT2::layernorm_forward, params_memory, acts_memory, ind.tensors, ind.ln1, ind.ln1_mean, ind.ln1_rstd, ind.residual, ind.ln1w, ind.ln1b, B, T, C)
.task("mm1", GPT2::matmul_forward, params_memory, acts_memory, ind.tensors, ind.qkv, ind.ln1, ind.qkvw, ind.qkvb, B, T, C, 3 * C)
.task("at", GPT2::attention_forward, acts_memory, ind.tensors, ind.atty, ind.preatt, ind.att, ind.qkv, B, T, C, NH)
.task("mm2", GPT2::matmul_forward, params_memory, acts_memory, ind.tensors, ind.attproj, ind.atty, ind.attprojw, ind.attprojb, B, T, C, C)
.task("rs1", GPT2::residual_forward, acts_memory, ind.tensors, ind.residual2, ind.residual, ind.attproj, B * T * C)
.task("ln2", GPT2::layernorm_forward, params_memory, acts_memory, ind.tensors, ind.ln2, ind.ln2_mean, ind.ln2_rstd, ind.residual2, ind.ln2w, ind.ln2b, B, T, C)
.task("mm3", GPT2::matmul_forward, params_memory, acts_memory, ind.tensors, ind.fch, ind.ln2, ind.fcw, ind.fcb, B, T, C, 4 * C)
.task("ge", GPT2::gelu_forward, acts_memory, ind.tensors, ind.fch_gelu, ind.fch, B * T * 4 * C)
.task("mm4", GPT2::matmul_forward, params_memory, acts_memory, ind.tensors, ind.fcproj, ind.fch_gelu, ind.fcprojw, ind.fcprojb, B, T, 4 * C, C)
.task("rs2", GPT2::residual_forward, acts_memory, ind.tensors, ind.residual3, ind.residual2, ind.fcproj, B * T * C)
.transferToHost(DataTransferMode.UNDER_DEMAND, acts_memory);
TornadoExecutionPlan transformer_runner = new TornadoExecutionPlan(transformer_block.snapshot());
TornadoExecutionResult transformer_result = null;

// the input block
encoder_forward(acts.encoded, params.wte, params.wpe, B, T, C); // encoding goes into residual[0]

// control variable `l´ counts the transformer blocks
for (int l = 0 ; l < L ; l++) {
    // update layer indices in `ind.tensors´ for this iteration
    ind.updateForLayer(l);
    // execute this transformer block's Task graph `tb´
    transformer_result = transformer_runner.execute();
}
// transfer results from accelerator to host after all blocks have benn executed
transformer_result.transferToHost(acts_memory);
// define and run the output block's task graph
TaskGraph output_layer = new TaskGraph("ol")
.transferToDevice(DataTransferMode.EVERY_EXECUTION, params_memory, acts_memory) // only one execution here
.task("ln", GPT2::layernorm_forward, params_memory, acts_memory, acts.lnf, acts.lnf_mean, acts.lnf_rstd, residual, params.lnfw, params.lnfb, B, T, C)
.task("mm", GPT2::matmul_forward, params_memory, acts_memory, acts.logits, acts.lnf, params.wte, -1, B, T, C, Vp)
.task("sm", GPT2::softmax_forward, acts_memory, acts.probs, acts.logits, B, T, V, Vp)
.transferToHost(DataTransferMode.EVERY_EXECUTION, acts_memory);
TornadoExecutionPlan output_runner = new TornadoExecutionPlan(output_layer.snapshot());

output_runner.execute();
```

The layer methods receive array indices via method parameters which reflect the current layer. The original code updates these indices as it iterates layer by layer over the transformer blocks. For the now predefined transformer block, its (now as well predefined) parameters must become indices pointing to the actual (real) indices, the latter updated with each iteration (just as the original code did). This additional level of indirection requires another buffer to be transferred before each task graph execution.

This approach eventually worked and let me execute the full stack on accelerators with TornadoVM. It's in the [`tornado-multi-tg` branch](https://github.com/otabuzzman/llm.java/blob/tornado-multi-tg/src/com/otabuzzman/llmj/GPT2.java) of the llm.java repo.

|Version|Backend|forward pass {{< line_break >}} duration (ms)|
|:---|:---:|:---:|
|multiple TaskGraph|PTX|2353|
|multiple TaskGraph|OpenCL|2511|
|multiple TaskGraph|SPIR-V|6701|

## Findings

1. The matmul task for matrix multiplication yields a runtime error depending on the code structure. Could be fixed by moving parts of code around (trial and error).
```java
private static void matmul_forward(FloatArray params, FloatArray acts, IntArray pointers, int out, int inp, intweight, int bias, int B, int T, int C, int OC) {
    for (@Parallel int b = 0 ; b < B ; b++) {
        for (@Parallel int t = 0 ; t < T ; t++) {
            int bt = b * T + t;
            for (@Parallel int o = 0 ; o < OC ; o++) {
                // ERROR : clBuildProgram -> Returned: -11
                // '}' mismatch, moved code behind for-loop
                // int _bias = pointers.get(bias);
                // float val += (_bias != -1) ? params.get(_bias + o) : 0.0f;
                float val = 0.0f;
                for (int i = 0 ; i < C ; i++) {
                    val += acts.get(pointers.get(inp) + bt * C + i) * params.get(pointers.get(weight) + o * C + i);
                }
                int _bias = pointers.get(bias);
                val += (_bias != -1) ? params.get(_bias + o) : 0.0f;
                acts.set(pointers.get(out) + bt * OC + o, val);
    }}}}
```

2. The inner `for`-loop in the attention task (not the annotated parallelized loops) ends early when the termination condition references a control variable of one of the surrounding parallelized `for`-loops. The generated kernel shows a wrong block sequence that lets it terminate immediately after entering the respective `for`-loop. Could be fixed by changing the termination logic for the respective inner `for`-loop. A message from TornadoVM's option `tornado.parallelise.auto=true` for automatic parallelization (which I happened to try when I was running out of reasonable ideas) pointed me in the right direction: it said that it doesn't support loop dependencies.

3. The attention task yields a runtime error depending on a combination of code structure and backend. The SPIR-V backend works with a condition check at the beginning of an inner `for`-loop, whereas the OpenCL backend needs the check at the end of the loop. Could be fixed by using the PTX backend which works in both cases.

```java
private static void attention_forward(FloatArray acts, IntArray pointers, int out, int preatt, int att, int inp, int B, int T, int C, int NH) {
    int C3 = C * 3;
    int hs = C / NH; // head size
    float scale = 1.0f / TornadoMath.sqrt(hs);
    int _inp = pointers.get(inp);

    for (@Parallel int b = 0; b < B; b++) {
        for (@Parallel int t = 0; t < T; t++) {
            for (@Parallel int h = 0; h < NH; h++) {
                int query_t = _inp + b * T * C3 + t * C3 + h * hs;
                int preatt_bth = pointers.get(preatt) + b * NH * T * T + h * T * T + t * T;
                int att_bth = pointers.get(att) + b * NH * T * T + h * T * T + t * T;

                float maxval = -10000.0f; // TODO something better
                // ERROR with variabl `t´instead of `T´
                for (int t2 = 0; t2 < T; t2++) {
                    // SPIR-V, PTX backends ok
                    // if (t2 > t) continue; // avoid ERROR by looping if termination criterion is met

                    int key_t2 = _inp + b * T * C3 + t2 * C3 + h * hs + C; // +C because it's key

                    // (query_t) dot (key_t2)
                    float val = 0.0f;
                    for (int i = 0; i < hs; i++) {
                        val += acts.get(query_t + i) * acts.get(key_t2 + i);
                    }
                    val *= scale;
                    if (val > maxval) {
                        maxval = val;
                    }

                    // OpenCL, PTX backends ok
                    if (t2 > t) continue;
                    acts.set(preatt_bth + t2, val); // avoid ERROR by looping if termination criterion is met
                }
                // ...
    }}}}
```

4. Using break in a `for`-loop inside a task yields runtime error. Could be fixed by refactoring program logic.

## Conclusion

This was my second project to refactor a codebase to implement TornadoVM. The logic was more complex this time, but it was still no problem to use the TornadoVM APIs.

Playing around with the code structure of the task methods to get TornadoVM to render proper kernels was challenging, but also fun – and that’s what counts...
