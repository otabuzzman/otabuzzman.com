---
date: 2024-04-18T13:15:17+02:00
title: "Java parallelization with TornadoVM (Part 1)"
description: "Setting things up"
featured_image: "featured_image.jpg"
tags: ["CUDA", "parallelcomputing", "GPGPU", "TornadoVM", "AMD", "Intel", "NVIDIA"]
disable_share: true
draft: true
---

A few years ago I created a Java app that was quite slow. It ran a nested for-loop with some mathematical calculations up to a hundred times with a varying number of up to a few million iterations.

The iterations were independent of each other and therefore this for-loop was an ideal candidate for parallelization. So I decided to offload execution to an accelerator and chose CUDA for this task.

While searching for a CUDA JNI I came across [Alan Kamnisky](https://www.cs.rit.edu/~ark/)'s [Parallel Java 2 library](https://www.cs.rit.edu/~ark/pj2.shtml) (PJ2). Alan is a (now retired) professor at the Rochester Institute of Technology. For his lectures he wrote PJ2 and the accompanying book [Big CPU, Big Data](https://www.cs.rit.edu/~ark/bcbd_2/). There were other CUDA JNI implementations at the time (e.g. jCUDA), but I stuck with PJ2 because it also provides APIs for Java parallelization on multiple CPU cores, as well as abstractions for executing code on multiple nodes.

PJ2 made things a lot easier, but there was still a need to write and compile a CUDA kernel. These tasks in turn made it necessary to acquire knowledge of programming NVIDIA accelerators and learn how to use the corresponding tools, APIs, and SDKs. The result was a codebase and build system that is somewhat confusing and therefore more difficult to maintain than in pure Java. Last but not least, the implementation is tied to NVIDIA. For other, also common accelerators (e.g. AMD or Intel), the learning curve must be gone through again.

This is where [TornadoVM](https://www.tornadovm.org/) enters the stage to eliminate these problems. It provides a simple API for scheduling Java code to run on popular accelerators (AMD, Intel, NVIDIA) without requiring programmers to leave the Java ecosystem. TornadoVM does all the heavy lifting of preparing and executing Java code on accelerators, including transferring data between host and device. Sounds tempting. Here is my report on implementing TornadoVM in my Java application.

The for-loop I parallelized projects one planar rectangular area onto another in three-dimensional space. Each area represents an image and is thus defined by a bitmap, with each element representing a pixel.

Here is a sketch of the strategy I implemented:
- Find target bitmap dimensions for source image
- For each pixel in target image
  - Find corresponding pixel in source image
  - Set target pixel to color of source

`<SOAP>`
The overall use case of my app is creating star maps. There is a function of mapping star constellations onto pictures, for example, to highlight the constellations of the zodiac through images of an artist's impressions of the corresponding constellations. It turns out that the strategy I used for this function resulted in rather slow execution. That's why I chose parallelization, even though I could have just as easily started looking for a more efficient strategy (or doing both...).
`</SOAP>`

```java
public class Artwork extends org.chartacaeli.model.Artwork implements PostscriptEmitter {
    private int[] texture ;  // source image
    private int dimo, dimp ; // coordinates are o, p
    private int maxo, maxp ;

    private int[] mapping ;  // result image
    private int dims, dimt ; // coordinates are s, t

    private double ups ; // units per dot

    // GPU implementation
    private class PJ2TextureMapperGpu extends Task {
        public void main( String[] argv ) throws Exception { ... }
    }

    // CPU cores implementation
    private class PJ2TextureMapperSmp extends Task {
        ...
    }

    // sequential implementation
    private class PJ2TextureMapperSeq extends Task {
        ...
    }
}
```

The code that performs the projection is in class `Artwork`. There are several nested classes within `Artwork` that do the real work:
- `Artwork$PJ2TextureMapperGpu` implements the projection using CUDA,
- `Artwork$PJ2TextureMapperSmp` uses the available CPU cores, and
- `Artwork$PJ2TextureMapperSeq` is a sequential implementation without any parallelization.

It's a matter of configuration which of these classes the app actually uses by calling the method `main` of the respective class instance.

For simplicity I decided to continue with this concept and simply create another class `Artwork$PJ2TextureMapperTvm` for TornadoVM.

The [_Loop Parallel API_](https://tornadovm.readthedocs.io/en/latest/programming.html#loop-parallel-api) is for parallelization of for-loops (as the name might imply). In an ideal world, using this API doesn't require much more effort than adding the `@Parallel` annotation to the (sequential) for-loop in question and some code that converts the method containing the for-loop into a kernel and eventually makes that kernel running on an accelerator.

```java
// a new inner class utilizes TornadoVM
private class PJ2TextureMapperTvm extends Task {
    private double[] st ;
    private Coordinate uv ;

    private org.chartacaeli.Coordinate eq ;
    private double[] ca ;

    public PJ2TextureMapperTvm() {
        st = new double[] { 0, 0, 1 } ;
        uv = new Coordinate() ;

        eq = new org.chartacaeli.Coordinate( 0, 0, 0 ) ;
        ca = new double[] { 0, 0, 0, 1 } ;
    }

    // formerly `main´ with TornadoVM extensions
    static void kernel( IntArray source, IntArray result ) {
        double t0[], op[] ;
        Coordinate t1 ;
        Vector3D vca, xca ;
        double o, p ;

        // the @Parallel annotation instructs TornadoVM
        // to parallelize the body
        for ( @Parallel int t=0 ; dimt>t ; t++ ) {
            st[1] = t*ups ;

            for ( @Parallel int s=0 ; dims>s ; s++ ) {
                st[0] = s*ups ;

                t0 = tmM2P.operate( st ) ;
                uv.x = t0[0] ;
                uv.y = t0[1] ;

                eq.setCoordinate( projector.project( uv, true ) ) ;
                t1 = eq.cartesian() ;

                vca = new Vector3D( t1.x, t1.y, t1.z ) ;
                xca = spT.intersection( new Line( Vector3D.ZERO, vca, 1.0e-10 ) ) ;
                ca[0] = xca.getX() ;
                ca[1] = xca.getY() ;
                ca[2] = xca.getZ() ;

                op = tmH2T.operate( ca ) ;
                o = op[0] ;
                p = op[1] ;

                if ( 0>o || 0>p || o>=maxo || p>=maxp )
                    continue ;

                result[t*dims+s] = source[(int) p*dimo+(int) o] ;
            }
        }
    }

    public void main( String[] argv ) throws Exception {
        IntArray source = new IntArray(texture.length);
        IntArray result = new IntArray(mapping.length) ;

        // copy image (texture) to source buffer here ...

        // create a TaskGraph with a unique id
        TaskGraph taskGraph = new TaskGraph("s0")
             // 1st node: copy source image to accelerator
            .transferToDevice(DataTransferMode.FIRST_EXECUTION, source)
             // 2nd node: execute kernel on accelerator
            .task("t0", Artwork.PJ2TextureMapperTvm::kernel, source, result)
             // 3rd node: copy projection result to host
            .transferToHost(DataTransferMode.EVERY_EXECUTION, result);

        // make task graph read-only
        ImmutableTaskGraph immutableTaskGraph = taskGraph.snapshot();
        // TornadoVM defines plans for task graph execution
        TornadoExecutionPlan executionPlan = new TornadoExecutionPlan(immutableTaskGraph);
        // wait to complete execution on accelerator
        TornadoExecutionResult executionResult = executionPlan.execute();

        // copy result buffer to mapping here ...
    }
}
```

That said I started out with my original sequential implementation and copied the respective nested class into a new one. Then I moved the code from the `main` method into a static method and named it `kernel`. This method is subject to parallelization by TornadoVM. It is actually the code in the body of the nested for-loops it contains. The `@Parallel` annotations cause TornadoVM to compile the body into a kernel and run that kernel on an accelerator.

The signature of `kernel` foresees two buffers for the source and result images. The [buffer type](https://tornadovm.readthedocs.io/en/latest/programming.html#data-representation) `IntArray` is one of several from the TornadoVM API that can be copied between host and accelerator.

The new code in `main` is responsible for buffer allocation and transfers and eventually runs `kernel` on the accelerator. This is the minimum code required to make TornadoVM run a piece of Java code on an accelerator. Time to check if the compilation works and maybe even runs successfully. The latter requires a TornadoVM installation and I have postponed it for now.

The former required some adjustments to my Maven setup as the first runs of `mvn compile` pointed out. First I had to update my `pom.xml` to use Java 21 for compilation (set properties `maven.compiler.source` and `maven.compiler.target` to 21).

Another requirement for TornadoVM is to enable the preview feature of the JDK (set `<enablePreview>` to `true` for Maven compiler plugin).

Finally, I had to add a repository and dependencies to point to the actual API, i.e. the Jars. The TornadoVM website provides up to date [sniplets](https://tornadovm.readthedocs.io/en/latest/installation.html#tornadovm-maven-projects) for copy and paste. That's what it took to make Maven happy.

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.10.1:compile (default-compile) on project chartacaeli-app: Compilation failure: Compilation failure:
[ERROR] /C:/Users/iuerg/src/chartacaeli-app/org/chartacaeli/Artwork.java:[555,51] non-static variable dimt cannot be referenced from a static context
[ERROR] /C:/Users/iuerg/src/chartacaeli-app/org/chartacaeli/Artwork.java:[556,33] non-static variable st cannot be referenced from a static context
[ERROR] /C:/Users/iuerg/src/chartacaeli-app/org/chartacaeli/Artwork.java:[556,43] non-static variable ups cannot be referenced from a static context
[ERROR] /C:/Users/iuerg/src/chartacaeli-app/org/chartacaeli/Artwork.java:[558,59] non-static variable dims cannot be referenced from a static context
[ERROR] /C:/Users/iuerg/src/chartacaeli-app/org/chartacaeli/Artwork.java:[559,41] non-static variable st cannot be referenced from a static context
... many more lines
```

Once the compiler can access the TornadoVM API, there are many complaints due to instance variables referenced by the static `kernel` method. The code in `kernel` was previously in method `main` of a nested class, and as an instance method `main` is allowed to access fields with variables of the surrounding `Artwork` class.

`<LESSON>`This concept will not work any more now having code from an instance method moved to a static one.`</LESSON>`

The static method `kernel` can be considered as a program that runs completely isolated on the accelerator, so everything it needs to do its job must be provided at the start of execution (for example, through method parameters).

Now comes the tedious part of redesigning the original program logic to adapt it to new circumstances. The job is relatively simple to describe: all data needed on the accelerator must be copied somehow, either by memory transfers or via method parameters. The latter is slightly faster but limited to 15 parameters. Memory state, on the other hand, is maintained for the entire task graph on the accelerator. It can therefore be used to store intermediate results that are to be processed sequentially by several methods like `kernel`. In this case, one would simply add task nodes to the task graph as needed, each calling its respective static method.

```java
// create a TaskGraph with three sequential executions of kernel
TaskGraph taskGraph = new TaskGraph("s0")
    .transferToDevice(DataTransferMode.FIRST_EXECUTION, source)
     // 1st task node: execute kernel on accelerator
    .task("t0", Artwork.PJ2TextureMapperTvm::kernel, source, result)
     // 2nd task node: execute kernel on accelerator
    .task("t1", Artwork.PJ2TextureMapperTvm::kernel, source, result)
     // 3rd task node: execute kernel on accelerator
    .task("t2", Artwork.PJ2TextureMapperTvm::kernel, source, result)
    .transferToHost(DataTransferMode.EVERY_EXECUTION, result);
```

Getting it right is, to some extent, a matter of trial and error. A rule of thumb is that less references of instance variables in code that is subject for execution on accelerators means less work. Another fact is that this job of refactoring is not specific to TornadoVM, but is required for any parallelization on accelerators. A big advantage of TornadoVM is that it does not require leaving the Java ecosystem.

How I did it for my specific app and what I was able to benefit from my first parallelization with native CUDA is the subject of part 2 of this report.
