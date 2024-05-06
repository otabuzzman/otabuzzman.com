---
date: 2024-04-18T13:15:17+02:00
title: "Parallel Java with TornadoVM (2)"
description: "Part 2: Refactor for execution on accelerator"
featured_image: "featured_image.jpg"
tags: ["CUDA", "parallelcomputing", "GPGPU", "TornadoVM", "AMD", "Intel", "NVIDIA"]
disable_share: true
draft: true
---

This second part of the report focuses on the changes necessary to separate a section of code and run it on an accelerator. Since I used TornadoVM's [_Loop Parallel API_](https://tornadovm.readthedocs.io/en/latest/programming.html#loop-parallel-api) the code in question is in the body of a for-loop.

In my case the for-loop is inside an instance method of a nested class and shares variables with that class and also with the parent class. I had to change this concept because TornadoVM requires static methods for parallelization. So I defined method parameters for all shared variables and let TornadoVM pass them to the kernel. Some of these variables have multiple elements (e.g. matrices). I put these in memory buffers that TornadoVM passes to the accelerator.
