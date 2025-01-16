---
date: 2025-01-12T19:44:21+01:00
title: "Metal parallelization of llm.c"
description: "An outline about a Metal implementation for llm.c"
featured_image: "featured-image.jpg"
tags: ["parallelcomputing", "Swift", "LLM", "llm.c", "CUDA", "Metal", "Apple", "macOS", "iOS"]
draft: false
---

[Metal](https://developer.apple.com/metal/) is Apple's low-level API for GPU programming and [llm.c](https://github.com/karpathy/llm.c) is Andrej Karpathy's plain C and CUDA implementation of GPT-2. The C version leverages [OpenMP](https://www.openmp.org/) to parallelize the layer functions on the CPU cores. The CUDA version is highly optimized for multi-node multi-accelerator parallelization on NVIDIA GPUs using [Open MPI](https://www.open-mpi.org/) and [NCCL](https://developer.nvidia.com/nccl).

I once ported the C version to Swift and used [Grand Central Dispatch](https://developer.apple.com/documentation/DISPATCH) (GCD) for CPU parallelization. The Xcode project is in [llm.swift](https://github.com/otabuzzman/llm.swift). Despite using the `-Ounchecked` Swift compiler option to generate fast code without bounds checks the C version compiled with `clang` runs about 6 times faster than Swift.

|llm.c|llm.swift|
|:---:|:---:|
|2.230 s|12.256 s|
{ class="center w-75 f5" }
Measures on MacBook Pro M2
{ class="w-75 f6" }

The times are the average time of the 10 loops that `test_gpt2` executes and logs to stdout for each loop. The reason for the difference in execution times might be my code, which follows llm.c almost one-to-one except for file I/O and GCD. Model data access uses pointer arithmetic just like llm.c. Anyway, I've postponed looking for the root cause and moved on to trying Metal for parallelization to see how far I can get with it.

I defined a `struct Launchpad` in `LaunchPad.swift`. The type contains the necessary data structures for Metal to execute code on GPU. I adapted this concept from [@regrettable-username](https://github.com/regrettable-username) from his Metal port of llm.c. He used C and thankfully shared it in [llm.metal](https://github.com/regrettable-username/llm.metal).

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

A global `var launchPad` in `train_gpt2.swift` stores an instance of `LaunchPad`. While not best practice for Swift, I stick with global objects because they're instantly available anywhere I need them without having to pass them around, and it's a lab, and llm.c does it too.


`LaunchPad` also provides methods for (de-)registering Metal buffers (`class MTLBuffer`) and functions (`class MTLFunction`) for shaders, the Metal analogue of CUDA kernels. Both refer to programs that are to be executed on GPUs.

  ```swift
  func registerKernel(name: String, _ preserve: Bool = true)

  func registerBuffer(address: UnsafeMutableRawPointer, length: Int, _ preserve: Bool = true)
  func unregisterBuffer(address: UnsafeMutableRawPointer)

  func dispatchKernel(name: String, context: KernelContext, params: [KernelParam])
  ```

The Metal buffers use memory already allocated by llm.swift during initialization. I inserted the Metal buffer registrations immediately after the allocations of the respective Swift buffers, namely `params_memory`, `acts_memory` (both in `train_gpt2.swift`), and `inputs_memory` and `targets_memory` in `llmc/dataloader.swift`.

  ```swift
  /// Metal buffer registration for already allocated `params_memory´

  // malloc all parameters all at once
  let params_memory = UnsafeMutableBufferPointer<Float>.allocate(capacity: num_parameters)

  let params_length = num_parameters * MemoryLayout<Float>.size
  try? launchPad?.registerBuffer(address: params_memory.baseAddress!, length: params_length)
  ```

On init, `LaunchPad` loads shader code from another global `var defaultLibrary` defined in `dev/metal/DefaultLibrary.swift`. It stores the source code of the shaders in a single string in [Metal Shader Language](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf) (MSL). The C++-based MSL defines shaders as functions. The naming concept follows llm.c. It appends `"_kernel"` and a version to the specific part of a shader name (e.g. `encoder_forward_kernel1`).

I decided against using `.metal` files to enable development on iPad with _Swift Playgrounds 4_, which does not support compiling `.metal` files. A future project might be to see if there is a hack for Playgrounds to load shader libraries precompiled with Xcode.

The architecture of llm.c implements the GPT model as a stack of sequentially called global layer functions, and so does llm.swift. There is one stack for the forward pass in `func gpt2_forward` and another for the backward (training) pass in `func gpt2_backward`, both in `train_gpt2.swift`. The names of the layer functions are concatenations of layer name and pass, e.g. encoder_forward for the encoder layer of the forward pass. The function definitions in `train_gpt2.swift` execute on CPU. Functions to be executed on GPU are each in its own layer-pass file (e.g. `dev/metal/encoder_forward.swift`) and have a version number appended to their names. The remaining signature corresponds to that of the CPU functions. The simple naming concept allows quick switching from CPU to GPU by simply adding or removing version numbers.

Each GPU layer function compiles parameter lists to dispatch shaders via the `dispatchKernel` method of `LaunchPad`. Some GPU layer functions just call a single shader (e.g. `func encoder_forward1`). Functions with more complex logic may call multiple shaders one after the other, since mapping the whole logic onto a single one would not provide significant performance gains. This is the case, for example, when the logic requires if/else branches, which are known to be GPU performance killers.

  ```swift
  // shader call in `func encoder_forward1´

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

Metal needs to know the names of the shaders it should execute on the GPU. I inserted the necessary registrations in `func test_gpt2` (`test_gpt2.swift`)and `func train_gpt2` before these functions call `func gpt2_forward` and `func gpt2_backward` (the latter in `train_gpt2.swift`).

  ```swift
  /// shader registration in `func test_gpt2´
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

llm.swift compiles each layer-pass file into a command line executable with the same name. These benchmark programs in folder `dev/metal` expect a version number, call the respective layer function with random data a few times and print the average execution times to stdout. This allows a quick comparison of different shader implementations. Compilation hints are in comments at the beginnings of the layer-pass files.

  ```bash
  # output (truncated) of `encoder_forward´ benchmark program
  #   ./dev/metal/encoder_forward 2
  
  # block sizes are from llm.c/ CUDA and have no use in Metal
  ... (lines removed)
  CPU time 2.7149 ms

  block_size 64   | time 0.0014 ms | bandwidth 71.00 GB/s
  block_size 128  | time 0.0014 ms | bandwidth 74.50 GB/s
  block_size 256  | time 0.0013 ms | bandwidth 75.27 GB/s
  block_size 512  | time 0.0013 ms | bandwidth 75.86 GB/s
  block_size 1024 | time 0.0013 ms | bandwidth 75.44 GB/s
  
  # the "napkin math" for bandwitch in llm.c/ CUDA
  # is probably not suitable for llm.swift/ Metal
  ```

The shaders in `var defaultLibrary` are mostly one-to-one ports of the CUDA kernels in llm.c. Deviations occur when CUDA features are available differently or not at all in MSP and therefore require specific implementations.

When the GPU layer functions return dispatched shaders must be committed to start execution. A subsequent call of the `commit` method of `LaunchPad` adds the shaders to the Metal compute pipeline, where the GPU fetches and executes them one by one. `commit` returns immediately to continue processing the layer stack on the CPU and thus continue calling GPU layer functions that feed shaders into the pipeline and keep the GPU busy. CPU and GPU run asynchronously. The commit after the last GPU layer function therefore waits until all shaders have been executed.

  ```swift
  // the CPU layer function...
  encoder_forward(acts.encoded, inputs, params.wte, params.wpe, B, T, C)
  
  // ...and the GPU replacement
  try encoder_forward3(acts.encoded, inputs, params.wte, params.wpe, B, T, C)
  try launchPad?.commit(wait: true) // wait until GPU finishes
  ```

**GPU layer function benchmarks**

|Program|Version|llm.swift|llm.c|
|:---:|:---:|:---:|:---:|
|encoder_forward|2|2.0 ms|0.0014 ms|
{ class="f5" }
llm.swift: Metal version on MacBook Pro M2{{< line_break >}}
llm.c: CUDA version on Lenovo IdeaPad with NVIDIA RTX 3050 Ti
{ class="f6" }

Obviously there are differences between the two systems that make the M2 (2023) chip look better than the RTX 3050 Ti (2021). One difference is the unified memory feature, which allows the M2 to share Metal buffers between CPU and GPU, while the 3050 has to copy the buffers. On the other hand, `encoder_forward` calls a single, rather simple shader with little data, and the 3050 might perform better in a more realistic configuration.

Currently I only have a GPU layer function for the encoder layer of the forward pass. Let's see how llm.swift behaves with more functions to come until both layer stacks run entirely on the GPU.
