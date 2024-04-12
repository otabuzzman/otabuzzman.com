---
date: 2024-04-11T23:38:31+02:00
title: "Java parallelization with CUDA"
description: "A stepwise approach to parallelize Java code"
featured_image: "featured_image.jpg"
tags: ["CUDA", "parallelcomputing", "GPGPU", "TornadoVM", "AMD", "Intel", "NVIDIA"]
disable_share: true
draft: true
---

An infographic about my first approach to parallelizing Java code in 2017. It worked for me then and probably still does, but now there are tools that are much easier to use and are also much more flexible. One is [TornadoVM](https://www.tornadovm.org/), which essentially allows the programmer to mark up the code to be parallelized and does all the heavy lifting for execution on popular accelerators (AMD, Intel, NVIDIA) and multiple CPU cores. I created a tutorial on TornadoVM that was published in a German computer magazine: part [one](https://www.heise.de/select/ix/2024/2/2332508044372163045) and [two](https://www.heise.de/select/ix/2024/3/2332713580068863270).

Infographic made with [Pictochart](https://piktochart.com/).

![Infographic](parallel-java.jpg)
