---
date: 2024-04-18T13:15:17+02:00
title: "Java parallelization with TornadoVM"
description: "Refactor Java code for parallelization with TornadoVM"
featured_image: "featured_image.jpg"
tags: ["CUDA", "parallelcomputing", "GPGPU", "TornadoVM", "AMD", "Intel", "NVIDIA"]
disable_share: true
draft: true
---

A few years ago I created a Java app that was quite slow. It ran a nested for-loop with some mathematical calculations up to a hundred times with a varying number of up to a few million iterations.

The iterations were independent of each other and therefore this for-loop was an ideal candidate for parallelization. So I decided to offload the for-loop to an accelerator and chose CUDA for this task.

While searching for a CUDA JNI I came across [Alan Kamnisky](https://www.cs.rit.edu/~ark/)'s [Parallel Java 2 library](https://www.cs.rit.edu/~ark/pj2.shtml) (PJ2). Alan is a (now retired) professor at the Rochester Institute of Technology. For his lectures he wrote PJ2 and the accompanying book [Big CPU, Big Data](https://www.cs.rit.edu/~ark/bcbd_2/). There were other CUDA JNI implementations at the time (e.g. jCUDA), but I stuck with PJ2 because it also provides APIs for Java parallelization on multiple CPU cores, as well as abstractions for executing code on multiple nodes.

PJ2 made things a lot easier, but there was still a need to write and compile a CUDA kernel. These tasks in turn made it necessary to acquire knowledge of programming NVIDIA accelerators and learn how to use the corresponding tools, APIs, and SDKs. The result was a codebase and build system that is somewhat confusing and therefore more difficult to maintain than in pure Java. Last but not least, the implementation is tied to NVIDIA. For other, also common accelerators (e.g. AMD or Intel), the learning curve must be gone through again.

This is where [TornadoVM](https://www.tornadovm.org/) enters the stage to eliminate these problems. It provides a simple API for scheduling Java code to run on popular accelerators (AMD, Intel, NVIDIA) without requiring programmers to leave the Java ecosystem. TornadoVM does all the heavy lifting of preparing and executing Java code on accelerators, including transferring data between host and device. Sounds tempting. Here is my report on implementing TornadoVM in my Java application.

The for-loop I parallelized projects one planar rectangular area onto another in three-dimensional space. Each area represents an image and is thus defined by a bitmap, with each element representing a pixel.

Here is a sketch of the strategy I implemented:
- Find target bitmap for a given source image
- For each pixel in target image
  - Find corresponding pixel in source image
  - Set target pixel to color of source

`<SOAP>`
The overall use case of my app is creating star maps. There is a function of mapping star constellations onto pictures, for example, to highlight the constellations of the zodiac through images of an artist's impressions of the corresponding constellations. It turns out that the strategy I used for this function resulted in rather slow execution. That's why I chose parallelization, even though I could have just as easily started looking for a more efficient strategy (or doing both...).
`</SOAP>`

The code that performs the projection is in class `Artwork`. There are several nested classes within `Artwork` that do the real work: `Artwork$PJ2TextureMapperGpu` implements the projection using CUDA, `Artwork$PJ2TextureMapperSmp` uses the available CPU cores, and `Artwork$PJ2TextureMapperSeq` is a sequential implementation without any parallelization. It's a matter of configuration which of these classes the app actually uses by calling the instance method `main` of the respective class.

For simplicity I decided to continue with this concept and simply create another class `Artwork$PJ2TextureMapperTvm` for TornadoVM. The [_Loop Parallel API_](https://tornadovm.readthedocs.io/en/latest/programming.html#loop-parallel-api) is for parallelization of for-loops (as the name might imply). In an ideal world, using this API doesn't require much more effort than adding an annotation to the (sequential) for-loop in question and some code that converts the method containing the for-loop into a kernel and ultimately makes that kernel running on an accelerator.

That said I started out with my original sequential implementation and copied the respective nested class into a new one. Then I moved the code from the `main` method into a static method and named it `kernel`. This method, actually the code in the body of the nested for-loops it contains, is subject to parallelization by TornadoVM. The signature of `kernel` foresees two buffers for the source and result images. The [buffer types](https://tornadovm.readthedocs.io/en/latest/programming.html#data-representation) are from the TornadoVM API and can be copied between host and accelerator. The new code in `main` is responsible for buffer allocation and transfers and eventually runs `kernel` on the accelerator.
