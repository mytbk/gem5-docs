gem5 的存储模型
==================

gem5 对多种存储模型进行了建模，可以模拟 CPU 的高速缓存和 DRAM，支持多种缓存管理算法，通过 Ruby 可以支持自定义的缓存一致性模型。gem5 的官方稳定有对 gem5 存储系统的介绍。[1]_

由于 gem5 的源码都在 src/ 下，为了方便，以下写源码路径时，都省略开头的 src/.

存储访问模拟精度
----------------

gem5 有 Atomic, Functional, Timing 三种模拟存储访问的方式。一下是 [1]_ 中对于它们的介绍：

- Timing: 最详细的访存模拟。模拟了包括队列延迟和资源竞争等现实情形。
- Atomic: 最快的模拟方式，访存请求发送后就会得到响应。
- Functional: 和 Atomic 一样立刻得到响应。Functional 可以和 Timing/Atomic 一起使用。

mem/protocol 下有这三种访问方式的接口 {Timing,Atomic,Functional}{Request,Response}Protocol.

Port
----

端口(port) 是 gem5 中不同对象之间交互的接口。MasterPort 是请求发起的端口，SlavePort 是接收请求的端口。在 python/m5/params.py 和 sim/port.hh 有 port 的 Python 和 C++ 两部分的实现。


gem5 支持的存储部件
-------------------

现有的文档提到连接存储系统的类都继承 MemObject. 但最新的 gem5 已经没有 MemObject. 在最新的代码中，gem5 中的存储部件大致分两类，一类是缓存，它基于 BaseCache 类 [2]_ ，一类表示一片连续的内存区域，基于 AbstractMemory 类 [3]_. BaseCache 和 AbstractMemory 都直接继承 ClockedObject. 此外，除了较为简单的高速缓存模型外，gem5 还有来自原 GEMS 模拟器的 Ruby 缓存模型。

Memory
~~~~~~~~~~~~~~~~~~~

gem5 内存模型的类都继承 AbstractMemory (mem/AbstractMemory.py, mem/abstract_mem.hh). 由 [3]_ 可以看出有 SimpleMemory, DRAMCtrl 等内存模型，由于它们是最下层的内存存储层次，所以只有 SlavePort 而没有 MasterPort.

- SimpleMemory(mem/SimpleMemory.py): 最简单的内存模型，只有延迟 (latency)、延迟变化 (latency variance) 和带宽 (bandwidth) 三个属性
- QosMemCtrl(mem/qos/QoSMemCtrl.py): 带 QoS 的存储模型，有大量可配的 QoS 参数，它有以下两个子类
  - DRAMCtrl(mem/DRAMCtrl.py): 非常详细的 DRAM 模型，可配置的参数包括多种 DRAM 时序参数
  - QoS::MemSinkCtrl(mem/qos/QosMemSinkCtrl.py): 较为简单的 QoS 内存控制器模型
- DRAMSim2: 使用 DRAMSim2 [4]_ 进行 DRAM 模拟的模型

Cache
~~~~~~~~~~~~~~~~~~

gem5 的高速缓存模型基于 BaseCache 类，由 [2]_ 可以知道有 Cache 和 NoncoherentCache，即 gem5 支持一致性和非一致性的缓存。

gem5 的 BaseCache 的参数包括大小，相联度，访问 tag 和数据的延迟，响应延迟，MSHR 个数，写缓冲长度，替换、预取、压缩的算法。它还支持非包含的缓存。

Ruby
~~~~~~~~~~~~~~~~~

Ruby 模型支持模拟多种缓存存储层次，多种缓存一致性模型，互联网络等。RubyPort 类 (mem/ruby/system/Sequencer.py) 包含 MasterPort 和 SlavePort. RubySequencer 是 Ruby 系统中模拟存储系统的主要部分，它基于 RubyPort 类，包含了 I-Cache 和 D-Cache, 它们是 RubyCache (mem/ruby/structures/RubyCache.py) 的实例。

Python 配置 gem5 时，如果用了 Ruby 系统，可能会调用 configs/ruby/Ruby.py 中的 create_system 方法，然后调用一致性协议中的 create_system, 如 configs/ruby/MESI_Two_Level.py 里的 create_system 方法，其中会出现 L1Cache_Controller, L2Cache_Controller 这些在 src/ 下没有出现的类。在 build/ 下可以找到，可以发现是 gem5 中的 SLICC 解释器解析状态机源文件生成出来的。

相比于 Cache 支持多种预取器，Ruby 的 RubyPrefetcher 只有一种预取算法。Ruby 也没有缓存压缩机制。


XBar
~~~~

crossbar (XBar) [5]_ 用于让一个端口能和多个端口连接，在配置 gem5 的时候实例化一个 SystemXBar 类型的存储总线，用于连接主存和多个 D-Cache, 如 [6]_ 中的结构图所示。

XBar 类的定义在 mem/XBar.py. BaseXBar 类 [7]_ 中包含多个 MasterPort 和 SlavePort, 可以配置延迟和带宽。XBar 有一致性和非一致性两种。SystemXBar 是一致性的 XBar, 它一端连接 CPU、GPU 和支持一致性 I/O 的主设备，另一端连接主存。一致性需要 SnoopFilter 维护。

.. [1] https://www.gem5.org/documentation/general_docs/memory_system/
.. [2] https://gem5.github.io/gem5-doxygen/classBaseCache.htm
.. [3] https://gem5.github.io/gem5-doxygen/classAbstractMemory.html
.. [4] https://github.com/dramninjasUMD/DRAMSim2
.. [5] https://en.wikipedia.org/wiki/Crossbar_switch
.. [6] https://www.gem5.org/documentation/general_docs/memory_system/gem5_memory_system/#coherent-bus-object
.. [7] https://gem5.github.io/gem5-doxygen/classBaseXBar.html
