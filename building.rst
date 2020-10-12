gem5 的构建
================

gem5 使用 `scons <https://scons.org/>`__ 构建系统。从 v20.0.0.0 开始，支持 Python 3 的 scons 构建，如果使用老版本的 gem5，则需要 Python 2 的 scons.

构建 gem5 的命令行很简单，以 X86 CPU 架构为例::

  scons -j9 build/X86/gem5.opt

那么除了 X86 之外还有什么架构可以使用呢？可以看 ``build_opts`` 目录，里面有 X86, ARM, MIPS 等文件，把以上命令行的 X86 换成这些文件名，就可以构建其他配置的 gem5. 这些文件配置的都是 gem5 构建时要配置的变量。

build_opts/X86::

  TARGET_ISA = 'x86'
  CPU_MODELS = 'AtomicSimpleCPU,O3CPU,TimingSimpleCPU'
  PROTOCOL = 'MI_example'

而配置 X86_MESI_Two_Level 的 PROTOCOL 配置有变化，它用了不同的缓存一致性协议 [1]_ 。

build_opts/X86_MESI_Two_Level::

  TARGET_ISA = 'x86'
  CPU_MODELS = 'TimingSimpleCPU,O3CPU,AtomicSimpleCPU'
  PROTOCOL = 'MESI_Two_Level'
  NUMBER_BITS_PER_SET = '128'

其他架构如 ARM::

  TARGET_ISA = 'arm'
  CPU_MODELS = 'AtomicSimpleCPU,TimingSimpleCPU,O3CPU,MinorCPU'
  PROTOCOL = 'MOESI_CMP_directory'

gem5 还支持 GPU，如果想构建一个支持 GPU 的模拟器，可以用 HSAIL_X86::

  PROTOCOL = 'GPU_RfO'
  TARGET_ISA = 'x86'
  TARGET_GPU_ISA = 'hsail'
  BUILD_GPU = True
  CPU_MODELS = 'AtomicSimpleCPU,O3CPU,TimingSimpleCPU'

构建 gem5 所用的 TARGET_ISA, CPU_MODELS 等变量可以在 SConstruct 里面找到，位置在 sticky_vars.AddVariables 下面::

  sticky_vars.AddVariables(
      EnumVariable('TARGET_ISA', 'Target ISA', 'alpha', all_isa_list),
      EnumVariable('TARGET_GPU_ISA', 'Target GPU ISA', 'hsail', all_gpu_isa_list),
      ListVariable('CPU_MODELS', 'CPU models',
                   sorted(n for n,m in CpuModel.dict.iteritems() if m.default),
                   sorted(CpuModel.dict.keys())),
      ...

gem5.opt 表示用 opt 配置的编译和链接参数构建 gem5. 除了 opt 之外，还可以使用 debug, fast, prof, perf 配置，具体可以见 src/SConscript::

  ccflags = {'debug' : [], 'opt' : ['-g'], 'fast' : [], 'prof' : ['-g', '-pg'],
             'perf' : ['-g']}
  
  ldflags = {'debug' : [], 'opt' : [], 'fast' : [], 'prof' : ['-pg'],
             'perf' : ['-Wl,--no-as-needed', '-lprofiler', '-Wl,--as-needed']}
  
  if 'debug' in needed_envs:
      makeEnv(env, 'debug', '.do',
              CCFLAGS = Split(ccflags['debug']),
              CPPDEFINES = ['DEBUG', 'TRACING_ON=1'],
              LINKFLAGS = Split(ldflags['debug']),
              disable_partial=disable_partial)

对于使用者来说，gem5.debug 编译优化最少，包含调试信息，适合用于调试。gem5.opt 比较常用，它在构建时使用了较多的编译优化。gem5.fast 比 gem5.opt 更快，但是使用 gem5.fast 时没有调试输出（在 gem5 里面用 DPRINTF 可以输出调试信息）。而 prof 和 perf 是用于对 gem5 的代码进行运行剖视（profiling）使用的。

同时，可以注意到，构建 gem5.debug 时，目标文件的后缀是 ".do". 而从 src/SConscript 可以看到，用不同的配置生产的目标文件的后缀都不同，因此构建不同配置的二进制文件，可以不用删掉 build 目录，直接用 scons 构建即可。

gem5 的 SConstruct 配置了一些常用的构建选项，可以在 scons 的命令行中加入，例如::

  scons -j9 build/ARM/gem5.opt --verbose --with-asan

--verbose 选项使得构建时可以打印出构建每个目标文件使用的完整命令行。--with-asan 可以在编译时使用 address sanitizer [2]_.

还可以在命令行中配置 gem5 的变量，例如我们不想构建 SystemC 相关代码时（src/systemc/SConscript 使用该变量），可以用::

  scons -j9 build/ARM/gem5.opt USE_SYSTEMC=False

.. [1] https://www.semanticscholar.org/paper/A-Primer-on-Memory-Consistency-and-Cache-Coherence-Sorin-Hill/10f1faeec4ee2158b8535b249a20de5419998153
.. [2] https://github.com/google/sanitizers/wiki/AddressSanitizer
