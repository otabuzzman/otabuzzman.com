---
date: 2024-04-22T13:15:17+02:00
title: "Java parallelization with TornadoVM"
description: "Part 3: See how it works"
featured_image: "featured_image.jpg"
tags: ["CUDA", "parallelcomputing", "GPGPU", "TornadoVM", "AMD", "Intel", "NVIDIA"]
disable_share: true
draft: true
---

```bash
git clone https://github.com/beehive-lab/TornadoVM.git
cd TornadoVM

bin/tornadovm-installer --jdk jdk21 --backend opencl,ptx,spirv
```

In my original setup, the PJ2 library is pre-installed in a separate folder, so I installed TornadoVM the same way. The installation is well [documented](https://tornadovm.readthedocs.io/en/latest/installation.html) and essentially requires cloning the repository and running the Python installer.

```
source setvars.sh

tornado --devices
```

Once finished there is a batch file `setvars.sh` that must be sourced to run the `tornado` CLI, and ask it to list the available accelerators.

```
Number of Tornado drivers: 3
Driver: SPIRV
  Total number of SPIRV devices  : 1
  Tornado device=0:0  (DEFAULT)
    SPIRV -- SPIRV LevelZero - Intel(R) Iris(R) Xe Graphics
            Global Memory Size: 6,3 GB
            Local Memory Size: 64,0 KB
            Workgroup Dimensions: 3
            Total Number of Block Threads: [512]
            Max WorkGroup Configuration: [512, 512, 512]
            Device OpenCL C version:  (LEVEL ZERO)

Driver: OpenCL
  Total number of OpenCL devices  : 3
  Tornado device=1:0
    OPENCL --  [NVIDIA CUDA] -- NVIDIA GeForce RTX 3050 Ti Laptop GPU
            Global Memory Size: 4,0 GB
            Local Memory Size: 48,0 KB
            Workgroup Dimensions: 3
            Total Number of Block Threads: [1024]
            Max WorkGroup Configuration: [1024, 1024, 64]
            Device OpenCL C version: OpenCL C 1.2

  Tornado device=1:1
    OPENCL --  [Intel(R) OpenCL Graphics] -- Intel(R) Iris(R) Xe Graphics
            Global Memory Size: 6,3 GB
            Local Memory Size: 64,0 KB
            Workgroup Dimensions: 3
            Total Number of Block Threads: [512]
            Max WorkGroup Configuration: [512, 512, 512]
            Device OpenCL C version: OpenCL C 1.2

  Tornado device=1:2
    OPENCL --  [Intel(R) OpenCL] -- 11th Gen Intel(R) Core(TM) i5-11320H @ 3.20GHz
            Global Memory Size: 15,8 GB
            Local Memory Size: 32,0 KB
            Workgroup Dimensions: 3
            Total Number of Block Threads: [8192]
            Max WorkGroup Configuration: [8192, 8192, 8192]
            Device OpenCL C version: OpenCL C 3.0

Driver: PTX
  Total number of PTX devices  : 1
  Tornado device=2:0
    PTX -- PTX -- NVIDIA GeForce RTX 3050 Ti Laptop GPU
            Global Memory Size: 4,0 GB
            Local Memory Size: 48,0 KB
            Workgroup Dimensions: 3
            Total Number of Block Threads: [2147483647, 65535, 65535]
            Max WorkGroup Configuration: [1024, 1024, 64]
            Device OpenCL C version: N/A
```
