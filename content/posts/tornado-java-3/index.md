---
date: 2024-04-22T13:15:17+02:00
title: "Parallel Java with TornadoVM (3)"
description: "Part 3: See how it works"
featured_image: "featured_image.jpg"
tags: ["CUDA", "parallelcomputing", "GPGPU", "TornadoVM", "AMD", "Intel", "NVIDIA"]
disable_share: true
draft: true
---

In my original setup, the PJ2 library is pre-installed in a separate folder, so I installed TornadoVM the same way. The installation is well [documented](https://tornadovm.readthedocs.io/en/latest/installation.html) and essentially requires cloning the repository and running the installer. **Note** that the script uses Python and installs some modules. To keep the Python system configuration clean from this stuff I prefer setting up a Python _Virtual Environment_ before running the installer.

```bash
git clone https://github.com/beehive-lab/TornadoVM.git
cd TornadoVM

# create Python Virtual Environment (venv)
# python -m venv .venv
# activate venv for bash
# source .venv/bin/activate

bin/tornadovm-installer --jdk jdk21 --backend opencl,ptx,spirv
```

Once finished (after a couple of minutes and tons of logging on screen) there is a batch file `setvars.sh` that must be sourced to run the `tornado` CLI, and ask it to list the available accelerators.

```
source setvars.sh

tornado --devices
```

TornadoVM uses tuples with driver and device values as an accelerator reference. A driver (also called backend) roughly corresponds to the access method and a device corresponds to a specific CPU or GPU.

The following sample output from `tornado --devices` on my Lenovo notebook lists three drivers for SPIR-V, OpenCL, and PTX (as passed to the installer). The drivers, in turn, are tied to specific devices, namely SPIR-V to the integrated Intel GPU, PTX to the NVIDIA GPU and OpenCL to both GPUs. The OpenCL driver also supports the multicore CPU and therefore seems to be the most versatile.

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
