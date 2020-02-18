gem5 的 ExecContext 和 Thread
====================================

看 gem5 的 CPU 模型代码时，可以看到 CPU 模型中用到了 ExecContext 类，而和 ExecContext 一起使用的还有一个 SimpleThread 类。下面我们分析它们的功能。

ExexContext
---------------

ExecContext 是一个抽象类，在 CPU 模型中会定义各自的 ExecContext 的子类，作为 CPU 模型的一部分。ExecContext 在 CPU 模型中的一个作用，是它会作为指令 StaticInst 执行时 execute 方法的一个参数使用。

从 cpu/exec_context.hh 的代码注释中，可以看到它的作用是提供了 ISA 操作 CPU 状态所使用的接口。它提供的接口是类似 readIntRegOperand 这一类的函数，也就是说，指令可以通过这些接口，读取 CPU 中的寄存器等资源。以 SimpleCPU 的 SimpleExecContext 为例，readIntRegOperand 的实现如下::

    /** Reads an integer register. */
    RegVal
    readIntRegOperand(const StaticInst *si, int idx) override
    {
        numIntRegReads++;
        const RegId& reg = si->srcRegIdx(idx);
        assert(reg.isIntReg());
        return thread->readIntReg(reg.index());
    }

这个函数接收两个参数，一个是 StaticInst 所表示的指令，一个是索引 idx. 这里 idx 指的是指令 si 的第 idx 个源操作数。从指令 si 的信息中读取源操作数的寄存器号之后，从 thread 中读出这个寄存器号对应的寄存器的值。这就是 readIntRegOperand 的作用。其中 SimpleExecContext 中还有一个统计数据 numIntRegReads，用于记录 CPU 运行过程中读了多少次寄存器。


Thread
----------

从上面的代码可以看出，ExecContext 调用了 ``thread->readIntReg``. 这里 thread 是一个 SimpleThread 的指针。而 SimpleThread 又是 ThreadContext (cpu/thread_context.hh) 的一个子类。

首先，线程（thread）是处理器的一个概念。一个线程中保存了 ISA 定义的所有的处理器状态，包括一套独立的 ISA 寄存器，独立的 PC. 如今 x86 处理器和 POWER 处理器中的同时多线程（SMT）就是一种多线程微体系结构。多线程处理器在应用程序看起来就是多个逻辑处理器核心，而程序运行在一个逻辑核心上，但不同线程上的程序实际上占用了处理器核心的不同时间片段。

所以 ExecContext 表示的是一个有多个线程的处理器核，而 ThreadContext 表示处理器核的一个线程。最终访问 ISA 状态的是 ThreadContext 提供的一系列抽象接口。

以 SimpleThread 为例。SimpleThread 中的成员包含了一个 ISA 里面的所有寄存器，读寄存器的方法 readIntReg 做的事情就是把里面整点寄存器的值取出来然后返回。


O3CPU 中的 ExecContext 和 Thread
------------------------------------------

我们再看比较复杂的 O3CPU 的相关实现。在 cpu/o3 下，没找到 ExecContext 相关的代码，但实际上 O3CPU 中是 BaseO3DynInst 继承了 ExecContext 类，实现了这一功能，具体可以看 cpu/base_dyn_inst.hh.

而 O3CPU 的 Thread 不是 SimpleThread，而是 O3ThreadContext (cpu/o3/thread_context.hh).


GPU 模型中的 GPUExecContext
---------------------------------

gem5 中有 GPU 模型 gpu-compute，它用的是一套和 CPU 模型完全独立的代码。GPU 模型里面也有 GPUExecContext，实现比较简单，只有 wavefront(), computerUnit(), readMiscReg() 和 writeMiscReg() 这几个函数，内部有一个 GPUISA 的接口。
