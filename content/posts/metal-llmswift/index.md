---
date: 2025-01-12T19:44:21+01:00
title: "Metal parallelization of llm.c"
description: "An outline about a Metal implementation for llm.c"
featured_image: "featured_image.jpg"
tags: ["parallelcomputing", "Swift", "LLM", "Metal", "Apple", "macOS", "iOS"]
disable_share: true
draft: true
---



[Metal](https://developer.apple.com/metal/) is Apple's low-level API for GPU programming and [llm.c](https://github.com/karpathy/llm.c) is Andrej Karpathy's plain C and CUDA implementation of GPT-2. The C version leverages [OpenMP](https://www.openmp.org/) to parallelize the layer functions on the CPU cores. The CUDA version is highly optimized for multi-node multi-accelerator parallelization on NVIDIA GPUs using [Open MPI](https://www.open-mpi.org/) and [NCCL](https://developer.nvidia.com/nccl). I once ported the C version to Swift and used [Grand Central Dispatch](https://developer.apple.com/documentation/DISPATCH) (GCD) for CPU parallelization. My Xcode project in [llm.swift](https://github.com/otabuzzman/llm.swift) is set to generate fast code and to skip bounds checks using the Swift compiler option `-Ounchecked`. The C version compiled with `clang` runs about 6 times faster than Swift.

**Measures on MacBook Pro M2**

|llm.c/ CPU {{< line_break >}} avg ms|llm.swift/ CPU {{< line_break >}} avg ms|
|:---:|:---:|
|2230|12256|

The times are the average time of the 10 loops that `test_gpt2` performs and logs the time for each loop to stdout. The reason for this big difference in execution times might be my code, which follows llm.c almost one-to-one. The biggest differences are file I/O and GCD. Model data access uses pointer arithmetic just like llm.c. Anyway, I've postponed looking for the root cause and moved on to trying Metal for parallelization to see how far I can get with it.

  ```swift
  struct LaunchPad {
      private let device: MTLDevice
      private let queue: MTLCommandQueue

      private let library: MTLLibrary

      private var kernel = [String: MTLComputePipelineState]()
      private var buffer = [MTLBuffer?]()

      // transient objects
      private var command: MTLCommandBuffer?
      private var encoder: MTLComputeCommandEncoder?
  }
  ```

A global `var launchPad` (`train_gpt2.swift`) stores an instance of type `LaunchPad` (`LaunchPad.swift`) that contains the necessary data structures for Metal to execute code on GPU. I adapted this concept from [@regrettable-username](https://github.com/regrettable-username). He took the same approach for his Metal port of llm.c and thankfully shared it in [llm.metal](https://github.com/regrettable-username/llm.metal). While it's not the best practice for Swift, I stick with global objects because they're instantly available anywhere I need them without having to pass them around, and it's a lab, and llm.c does it too.

In addition to data structures, `LaunchPad` provides methods for (un)registering Metal buffers (`MTLBuffer`) and functions (`MTLFunction`) that specify shaders, the Metal analogue to CUDA kernels, both naming programs to be executed on GPUs.

  ```swift
  func registerKernel(name: String, _ preserve: Bool = true)

  func registerBuffer(address: UnsafeMutableRawPointer, length: Int, _ preserve: Bool = true)
  func unregisterBuffer(address: UnsafeMutableRawPointer)

  dispatchKernel(name: String, context: KernelContext, params: [KernelParam])
  ```

The Metal buffers use memory already allocated by llm.c during initialization. I inserted these buffer registrations immediately after allocating the respective Swift buffers, namely `params_memory`, `acts_memory` (`train_gpt2.swift`), `inputs_memory` and `targets_memory` (`llmc/dataloader.swift`).

  ```swift
  /// Metal buffer registration for already allocated `params_memory´

  // malloc all parameters all at once
  let params_memory = UnsafeMutableBufferPointer<Float>.allocate(capacity: num_parameters)

  let params_length = num_parameters * MemoryLayout<Float>.size
  try? launchPad?.registerBuffer(address: params_memory.baseAddress!, length: params_length)
  ```

On init, `LaunchPad` loads shader code from another global `var defaultLibrary` (`dev/metal/DefaultLibrary.swift`), which stores the Metal Shader Language (MSL) source code of all shaders in a single string. I decided against using `.metal` files to enable development on iPad with Swift Playgrounds 4, which does not support compiling `.metal` files. A future project might be to see if there is a hack that allows Playgrounds to load shader libraries precompiled with Xcode.

  ```swift
  /// shader registration in `test_gpt2´
  let kernels = [
      "encoder_forward_kernel1",
      "encoder_forward_kernel2",
      "encoder_forward_kernel3"
  ]
  for kernel in kernels {
      do {
          // register layer kernels (shader) for Metal if available
          try launchPad?.registerKernel(name: "\(kernel)")
      } catch { stdlog?("\(error.localizedDescription)\n") }
  }

  ```

llm.swift performs shader registrations in `func test_gpt2` (`test_gpt2.swift`) and `func train_gpt2` before these methods call `gpt2_forward` and `gpt2_backward` (`train_gpt2.swift`), which implement the forward and backward (training) passes of GPT-2.

The architecture of llm.c implements the GPT model as a stack of sequentially called layer methods, and so does llm.swift. There is one stack for the forward pass and another for the backward pass. The names of the layer methods are concatenations of layer name and pass, e.g. encoder_forward for the encoder layer of the forward pass. The methods do the math of the respective layer. Layer methods to be executed on CPU (e.g. `func encoder_forward`) are in `train_gpt2.swift`. Methods to be executed on GPU are each in its own layer-pass file, e.g. `dev/metal/encoder_forward.swift` and have a version number appended to their names. The remaining signature corresponds to that of the CPU methods. This concept allows quick switching from CPU to GPU by simply adding or removing version numbers.

The methods of the GPU layer prepare the parameter list in a Metal-specific manner and call the respective shader via `LaunchPad`. Some GPU layer methods do nothing more than this (e.g. `func encoder_forward1`) and call a single shader. Methods with more complex logic may call multiple shaders one after the other, since mapping the whole logic onto a single one would not provide significant performance. This is the case, for example, when the logic requires if/else branches, which are known to be GPU performance killers.

  ```swift
  // shader call in `encoder_forward1´

  func encoder_forward1(
      _ out: UnsafeMutablePointer<Float>,
      _ inp: UnsafePointer<Int32>,
      _ wte: UnsafePointer<Float>,
      _ wpe: UnsafePointer<Float>,
      _ B: Int, _ T: Int, _ C: Int,
      _ block_size: Int = 256) throws {
      let threadsPerGrid = B * T
      let context = KernelContext(
          threadsPerGrid: MTLSize(width: threadsPerGrid, height: 1, depth: 1))
  
      let params: [KernelParam] = [
          UnsafeMutableRawPointer(out),
          UnsafeMutableRawPointer(mutating: inp),
          UnsafeMutableRawPointer(mutating: wte),
          UnsafeMutableRawPointer(mutating: wpe),
          Int32(B), Int32(T), Int32(C)]
  
      try launchPad?.dispatchKernel(
          name: "encoder_forward_kernel1",
          context: context,
          params: params)
  }
  ```

Like llm.c, each layer-pass file is compiled into an executable with the same name. These benchmark programs expect a version number, execute the respective layer method with random data a few times and print the average to stdout. This allows a quick comparison of different implementations. Compilation and usage hints are in comments at the beginnings of the layer-pass files.

  ```bash
  # output (truncated) of `encoder_forward´ benchmark program
  # Xcode copiles the program into `dev/metal´ folder
  #   ./dev/metal/encoder_forward 2
  
  # block sizes are from llm.c/ CUDA and have no use in Metal
  ... (lines removed)
  CPU time 2.7149 ms

  block_size 64   | time 0.0014 ms | bandwidth 71.00 GB/s
  block_size 128  | time 0.0014 ms | bandwidth 74.50 GB/s
  block_size 256  | time 0.0013 ms | bandwidth 75.27 GB/s
  block_size 512  | time 0.0013 ms | bandwidth 75.86 GB/s
  block_size 1024 | time 0.0013 ms | bandwidth 75.44 GB/s
  ```

The shaders in `var defaultLibrary` are mostly one-to-one ports of the CUDA kernels in llm.c. Deviations occur when CUDA features are available differently or not at all in MSP and therefore require specific implementations.

The GPU layer methods return after dispatching the contained shaders for execution by Metal. A subsequent call of `launchPad.commit` adds the shaders to the Metal compute pipeline. The method returns immediately to continue executing layer methods on CPU, thus continuing to feed shaders into the pipeline to keep the GPU busy. CPU and GPU run asynchronously. The commit after the last GPU layer method therefore waits until all shaders have been executed before continuing execution on the CPU.

  ```swift
  // a method parameter tells `commit´ to wait for the GPU to finish
  
  // CPU layer method...
  //    encoder_forward(acts.encoded, inputs, params.wte, params.wpe, B, T, C)
  
  // ...and the GPU replacement
      try encoder_forward3(acts.encoded, inputs, params.wte, params.wpe, B, T, C)
      try launchPad?.commit(wait: true)
  ```

**GPU layer method benchmarks**

|GPU layer method|llm.swift/ Metal (ms) {{< line_break >}} (MacBook Pro M2)|llm.c/ CUDA (ms) {{< line_break >}} (NVIDIA RTX 3050 Ti)|
|:---:|:---:|:---:|
|encoder_forward|2.0|0.0014|

Despite being a bit older than M2 (2023), the RTX 3050 Ti (2021) isn't at all bad. Obviously there are differences between the two systems that make the M2 chip look better. One difference is the unified memory feature, which allows the M2 to share metal buffers between CPU and GPU, while the 3050 has to copy the buffers. On the other hand, encoder_forward calls a single, rather simple shader with little data, and the 3050 might perform better in a more realistic configuration. Currently I only have the GPU layer method `encoder_forward`. Let's see how the rest behaves, especially when the entire layer method stack is executed on the GPU during the forward pass.
