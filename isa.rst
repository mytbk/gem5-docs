gem5 的 ISA 支持
======================

gem5 在 src/arch 下有多种体系结构的支持代码。其中有 CPU 的指令系统 alpha, arm, x86, riscv, mips, sparc, power, 还有 GPU 的 hsail. 以下通过分析代码，讨论在 gem5 中实现 ISA 的一些要点。

由于 gem5 的源码都在 src/ 下，为了方便，以下写源码路径时，都省略开头的 src/.


ISA 相关头文件和 namespace
----------------------------

gem5 是在构建的时候就配置了 CPU 使用的指令系统的。一个 gem5 模拟器只能模拟一种 CPU 指令系统，这是因为在 CPU 的实现代码中，已经在编译时配置了指令系统的信息。

在 cpu/exec_context.hh 中，可以看到包含了两个头文件 arch/registers.hh 和 config/the_isa.hh，但是在 src/ 下却没找到它们。实际上，这两个头文件是构建的时候在 build/ 下生成的。在构建 gem5 的时候，每个头文件都会在 build/<TARGET>/ 下有一个副本（大多数是符号链接），而在 src/ 下找不到的头文件都是构建系统在 build/ 下生成的，编译器寻找头文件是在 build/ 下找。例如在构建 build/ARM/gem5.opt 时，可以在 build/ARM/ 下找到 config/the_isa.hh 和 arch/registers.hh.

其中 config/the_isa.hh 的部分内容如下::

  #define THE_ISA ARM_ISA
  #define TheISA ArmISA
  #define THE_ISA_STR "arm"

而 arch/registers.hh 只有一行::

  #include "arch/arm/registers.hh"

也就是说 arch/register.hh 就是 arch/arm/registers.hh. CPU 的代码在包含 arch/registers.hh 的时候，无需考虑是哪个体系结构的 CPU，构建系统会让编译器使用用户配置的 CPU 对应的头文件。

config/the_isa.hh 的作用主要是定义 TheISA 等宏，在 exec_context.hh 里面有::

    typedef TheISA::PCState PCState;

    using VecRegContainer = TheISA::VecRegContainer;
    using VecElem = TheISA::VecElem;
    using VecPredRegContainer = TheISA::VecPredRegContainer;

于是在 CPU 的代码里面，所有和 ISA 有关的细节都不需要考虑，只需要一个 TheISA 就行了，编译器会把 TheISA 替换为 ArmISA 等具体的 ISA.

而 ArmISA 是一个 namespace, 里面可以定义各种常量和数据类型。在 arch/arm/register.hh 里面，定义了和 ARM 寄存器相关的常量和类型信息::

  using VecElem = uint32_t;
  using VecReg = ::VecRegT<VecElem, NumVecElemPerVecReg, false>;
  using ConstVecReg = ::VecRegT<VecElem, NumVecElemPerVecReg, true>;
  using VecRegContainer = VecReg::Container;

  using VecPredReg = ::VecPredRegT<VecElem, NumVecElemPerVecReg,
                                   VecPredRegHasPackedRepr, false>;
  using ConstVecPredReg = ::VecPredRegT<VecElem, NumVecElemPerVecReg,
                                        VecPredRegHasPackedRepr, true>;
  using VecPredRegContainer = VecPredReg::Container;

  // Constants Related to the number of registers
  // Int, Float, CC, Misc
  const int NumIntArchRegs = NUM_ARCH_INTREGS;
  const int NumIntRegs = NUM_INTREGS;
  const int NumFloatRegs = 0; // Float values are stored in the VecRegs
  const int NumCCRegs = NUM_CCREGS;
  const int NumMiscRegs = NUM_MISCREGS;
  
其中包括了整点、浮点标记寄存器的数量，还有向量寄存器元素的类型等等。这些常量在 CPU 的代码中都会被使用。

这些头文件生成的方式可以见 arch/SConscript::

  env.SwitchingHeaders(
      Split('''
          decoder.hh
          isa.hh
          isa_traits.hh
          kernel_stats.hh
          locked_mem.hh
          microcode_rom.hh
          mmapped_ipr.hh
          process.hh
          pseudo_inst.hh
          registers.hh
          remote_gdb.hh
          stacktrace.hh
          types.hh
          utility.hh
          vtophys.hh
          '''),
      env.subst('${TARGET_ISA}'))
  
  if env['BUILD_GPU']:
      env.SwitchingHeaders(
          Split('''
              gpu_decoder.hh
              gpu_isa.hh
              gpu_types.hh
              '''),
          env.subst('${TARGET_GPU_ISA}'))

SwitchingHeaders 里面的那些头文件都是会根据在 scons 中配置的 ISA 在构建时生成的，它们都会指向对应体系结构的头文件。从上面代码可以看出，GPU 相关的代码也会这么做，在构建期间会根据 GPU 的 ISA 生成 3 个头文件。


指令实现和 ISA 文件
---------------------

要模拟 ISA 的功能，一个重要的部分是怎么实现各个指令。

在 gem5 中，每个指令都用 StaticInst 类表示，StaticInst 类有对应的引用计数指针 StaticInstPtr. 同一个指令对应的 StaticInst 的内容是相同的。为了保存指令的动态信息，一般会根据处理器类型添加一种 DynInst 类型，在 CPU 模型中，minor 和 o3 都有各自的 DynInst 类型。StaticInst 里面保存了指令的各种信息，包括指令的二进制编码，重要字段的值，还有一个成员函数描述它执行时的行为。具体可以查看 cpu/static_inst.hh. 指令系统的每个操作码都对应 StaticInst 的一个子类。

为了减少重复代码的编写，gem5 使用了一种自定义的语言，用于描述指令的译码和每个指令的功能。解析这个指令描述的程序是 arch/isa_parser.py，它会生成用于译码和执行指令的 C++ 代码。

实际上，由于有的指令系统过于复杂，有的指令系统的实现并没有完全使用 gem5 的这套系统。GPU 的 HSAIL 用的是另一个手写的代码生成器。x86 的指令不定长，有大量指令扩展，而且处理器实现中有一个指令翻译为微操作的过程，因此 x86 的 ISA 模拟器的代码也用了额外的代码生成工具。下面讲一下 isa_parser.py 使用的指令描述文件的构造。我们用简单的指令系统 RISC-V 作为例子。

首先，arch/SConscript 里面有几行 add_gen 说明了 isa_parser.py 会生成哪些文件。在构建的时候，这些文件可以在 build/<TARGET>/arch/<ARCH>/generated 里面找到。

实际上，我们也可以手动执行 isa_parser.py 产生这些文件。我们切换到 gem5 的 src/arch 下，然后执行以下命令，可以在 /tmp/out 生成 RISC-V 指令系统的模拟指令译码和执行的程序::

  PYTHONPATH=../python:../../ext/ply python2 isa_parser.py riscv/isa/main.isa /tmp/out/

指定 PYTHONPATH 是因为 isa_parser.py 使用了 src/python 和 ext/ply (注意不在 src/ 下) 里面的库，其中 PLY 是用 Python 实现的功能类似于 lex 和 yacc 的库。

可以看到 /tmp/out 下有不少 .cc 和 .hh 文件，有几个 .cc 的内容只有一些 include，可以暂时不看它。有几个文件可以看一下：

- decode-method.cc.inc: 这里面是译码的函数 decodeInst, 它的参数是一个 TheISA::ExtMachInst 类型的指令表示，返回一个 StaticInstPtr. ExtMachInst 的类型在 ISA 的头文件里面定义，一般来说 uint64_t 足够表示一个指令的所有信息了。不过像 x86 这样的指令系统，ExtMachInst 是用一个结构体表示的。函数的内容是一串 switch-case 解析指令的各个字段，最后识别指令对应的操作码。
- decoder-ns.cc.inc: 这个文件保存每个指令对应的类的构造函数。
- decoder-ns.hh.inc: 这个文件保存每个指令对应的类的声明，还有一堆表示指令字段的宏定义。
- exec-ns.cc.inc: 这个文件里面保存每个指令的执行函数，即指令功能的实现。
- max_inst_regs.hh: 这个文件定义了所有指令的最大源操作数和目的操作数的数量。

接下来可以分析 isa_parser.py 的输入，即 riscv/isa/main.isa，它里面只有几个 include 行，也就是说，这些 isa 文件可以写成一个，只是为了维护方便，把它拆成了多个文件。

- bitfields.isa: 定义了各个字段，语法很简单，就是指定字段名字和这个字段的区域，字段区域<n:m>表示从第m比特到第n比特，包括第m和第n比特，如果只有一比特，可以简写为<n>. 这些字段定义最终成为 decoder-ns.hh.inc 里面的字段的宏定义。
- operands.isa: 定义操作数类型和名称，这样在指令模板（后面会提到）中用到这些操作数时，isa_parser.py 会识别出来，从而知道一个指令有多少个源操作数和目的操作数，还有它们对应的指令字段。这部分描述的详细信息可以见 isa_parser.py 里面 def_operands 和 def_operand_types 相关的描述。操作数的类型有 IntReg 等，可见 isa_parser.py 里面 class Operand 及其子类。
- formats.isa: 这里面 include 了不少文件，都用于定义指令模板。
- decoder.isa: 这里面表示如何译码一个指令。

有了这些 isa 文件，就可以用 isa_parser.py 生成一套包含了译码和指令执行逻辑的代码。最后我们讲述指令模板和译码逻辑的编写。


指令模板
~~~~~~~~~~

我们看 arch/riscv/isa/formats/basic.isa.

文件里面的几个 ``def template TemplateName {{ code }}`` 定义了多个模板，模板名称为 TemplateName，内容是 code. 其中 code 部分有 ``%(code)``, ``%(op_decl)`` 这种变量字符串，只要传入一个字典，isa_parser.py 就可以把这些变量替换为字典里面定义的字符串。

最后是 ``def format SomeOp(arg1, arg2, ..., *kargs) {{ }}``. 它定义了一种指令格式，通过 SomeOp 和传入的参数 arg1, arg2 等，就可以生成一组 C++ 代码，其中里面定义了一些值：

- iop: 产生一个字典，部分值来自 SomeOp 的参数
- header_output: 产生指令类的声明，即最终生成文件里的 decoder-ns.hh.inc 中各个类的声明
- decoder_output: 产生指令的构造函数，即 decoder-ns.cc.inc 中各个类的构造函数
- decode_block: 产生调用指令类构造函数的方法，最终出现在 decode-method.cc.inc 里面
- exec_output: 产生指令类的各个成员函数的定义，一般是指令的执行方法，最终出现在 exec-ns.cc.inc 里面


译码逻辑
~~~~~~~~

isa 文件的译码逻辑一般位于 decoder.isa 里面。译码部分比较容易编写，就是 ``decode 字段 {}`` 里面是字段的多个可能的值，如果一个字段不能翻译出指令，那就继续接 decode 块。

如果在读取多个字段后能译出指令，那就可以写是什么指令了，格式是 ``SomeOp::InstName(args...)``. 其中 SomeOp 就是在指令模板里面最后定义的指令格式，参数 args 就是 SomeOp 的参数，这些参数里面的内容最终会根据 SomeOp 里面的定义替换掉一个模板里面的一些变量字符串，而 InstName 就是这个指令的名称。

为了方便编写，如果一个译码块里面大多数指令都用 SomeOp 这种指令格式，那么可以把 decode 块里面的内容包装在 ``format SomeOp {}`` 里面，这样使用 SomeOp 这种格式的可以省略开头的 ``SomeOp::``.
