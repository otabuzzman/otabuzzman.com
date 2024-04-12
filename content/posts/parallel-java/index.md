---
date: 2024-04-25T23:38:31+02:00
title: "Java parallelization with CUDA"
description: "A stepwise approach to parallelize Java code"
tags: ["CUDA", "parallelcomputing", "GPGPU", "TornadoVM", "AMD", "Intel", "NVIDIA"]
disable_share: true
draft: true
---

This is an infographic showing my first approach to parallelizing Java code. It worked for me then and probably still does now, but now there are tools that are much easier to use. One of them is TornadoVM, which essentially allows the programmer to mark up the code to be parallelized and does all the heavy lifting for execution on popular accelerators (AMD, Intel, NVIDIA).

![Infographic](parallel-java.png)
