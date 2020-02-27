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


gem5 CPU 访存流程
--------------------------

这部分介绍 gem5 的 CPU 模型如何完成一次存储访问。

Atomic read
~~~~~~~~~~~~~~

对于 Atomic 访问方式，CPU 模型执行访存指令都用指令类的 Execute 函数。以 RISC-V 体系结构为例，该函数又调用 arch/generic/memhelpers.hh 的 readMemAtomicLE 和 readMemAtomic 函数。总的调用流程如下::

  readMemAtomic(exec_context, traceData, addr, mem, flags)
    -> ExecContext::readMem(addr, data, size, flags)
         -> AtomicSimpleCPU::readMem(addr, data, size, flags)

在 AtomicSimpleCPU 中，readMem() 首先配置一个 Request 的虚拟地址和读取数据大小，接着调用体系结构的地址翻译功能，把虚拟地址转为物理地址，之后用这个 Request 生成一个 Packet，把这个 Packet 发往 D-cache 端口，完成一次访问。

Timing read
~~~~~~~~~~~~~~

对于 Timing 访问方式，CPU 模型执行访存指令用指令类的 initiateAcc 函数。以 TimingSimpleCPU 为例，调用流程如下::

  initiateAcc
    -> initiateMemRead(exec_context, traceData, addr, mem, flags)  (arch/generic/memhelpers.hh)
         -> ExecContext::initiateMemRead(addr, size, flags)
	      -> TimingSimpleCPU::initiateMemRead(addr, size, flags)
	           -> BaseTLB::translateTiming(req, thread_context, translation, mode)  (arch/generic/tlb.hh)
		        -> DataTranslation::finish(fault, req, thread_context, mode)  (cpu/translation.hh)
			     -> TimingSimpleCPU::finishTranslation(state)
                                  -> TimingSimpleCPU::sendData(req, data, res, read)
				       -> TimingSimpleCPU::handleReadPacket(pkt)
				            -> DCachePort::sendTimingReq(pkt)

这样就完成了一次将包含请求信息的数据包发往 D-cache 端口的访问。对于 Timing 访问，DCachePort 还需要实现 recvTimingResp 处理该请求的响应::

  TimingSimpleCPU::DcachePort::recvTimingResp(PacketPtr pkt)
    -> TimingSimpleCPU::TimingCPUPort::TickEvent::schedule(pkt, tick)
         -> TimingSimpleCPU::schedule(tickEvent, tick)

而 DCachePort 的 TickEvent 的 process() 内容如下::

  void
  TimingSimpleCPU::DcachePort::DTickEvent::process()
  {
      cpu->completeDataAccess(pkt);
  }

而 TimingSimpleCPU::completeDataAccess 会调用指令的 completeAcc 函数，从响应的包中获取读到的数据，更新至寄存器堆。


在 gem5 Python 配置中连接端口
-------------------------------

(TBD)


一个使用 SimpleMemory 的例子
-------------------------------

以下部分是我做的一些 hacking 工作，用于探索关于 Port 和存储访问的实现细节。

配置 gem5 中的部件和存储系统的连接的方法是在 Python 配置中将要连接的端口连接起来。以下描述如何创建一个可以连接存储系统的 SimObject，并将其连接到，连接到一个 SimpleMemory 的方法。这里以 learning_gem5/part2 中的 SimpleObject 为例。

首先要添加一个 MasterPort::

  mem_side = MasterPort("memory side port, send requests")

除了在 Python 里面添加此定义外，还需要在 SimpleObject 类中添加 MasterPort 的成员。MasterPort 是个虚基类，它的子类必须有以下虚成员函数::

  virtual bool recvTimingResp(PacketPtr pkt);
  virtual void recvReqRetry();

实现了一个 MasterPort 的子类 SimplePort 后，在 SimpleObject 添加该类的成员 memPort. 现在需要实现 getPort 函数，让 gem5 知道在绑定端口时和 memPort 绑定::

  virtual Port &getPort(const std::string &if_name,
                        PortID idx=InvalidPortID) override
  {
      if (if_name == "mem_side")
          return memPort;
      return SimpleObject::getPort(if_name, idx);
  }

这里 "mem_side" 用的是在 Python 类定义 SimpleObject.py 里面的端口名字。

要连接端口，首先在配置文件里面实例化一个 SimpleObject 和 SimpleMemory. AbstractMemory 要求 gem5 模拟的系统有 System 的实例，于是实例化一个 System，并把 SimpleMemory 的实例连接到 System 下::

  system = System()
  system.clk_domain = SrcClockDomain()
  system.clk_domain.clock = '1GHz'
  system.clk_domain.voltage_domain = VoltageDomain()

  system.memory = SimpleMemory(range=AddrRange('512MB'))

此外，System 有一个称为 system_port 的 MasterPort，可以把它连接到一个 SystemXBar 下::

  system.membus = SystemXBar()
  system.system_port = system.membus.slave

一个 XBar 需要有至少一个 MasterPort 和一个 SlavePort. 可以把 system.memory 连到 system.membus 的 MasterPort 上。同时，我们实例化一个 SimpleObject 并把它连到 system.membus 的 SlavePort 上::

  system.memory.port = system.membus.master

  system.hello = SimpleObject()
  system.hello.mem_side = system.membus.slave

连接了 SimpleObject 和 SimpleMemory 的实例后，就可以在 SimpleObject 的实例中往存储器读写数据了。但是在这之前，还有一些细节上的东西需要实现。

一个是 Request 里面要有 MasterID. 而 MasterID 是一个从 System 实例里面获取的值。所以在这里我们给 SimpleObject 添加一个 MasterID 字段，同时在 SimpleObject 中添加一个 system 属性::

  system = Param.System(Parent.any, "system object")

在构造函数中::

  SimpleObject::SimpleObject(SimpleObjectParams *params) :
      masterId(params->system->getMasterId(this))

这样 SimpleObject 的对象就有了 Request 需要的 MasterID 了。然后就可以构造请求和数据包用于读写数据了。

以下是我用 Atomic 和 Timing 两种模式读取存储器的例子，具体代码见 https://git.wehack.space/gem5/log/?h=simple-object-demo.

Atomic 模式
~~~~~~~~~~~~~~~

以下是用 Atomic 模式读写数据的代码，它在 0x200000 读写一个 4 字节的数据。

写数据::

  RequestPtr req = std::make_shared<Request>(0x200000, 4, 0, masterId);
  PacketPtr pkt = Packet::createWrite(req);

  uint32_t x = 0xdeadbeef;
  pkt->dataStatic(&x);

  Tick t = memPort.sendAtomic(pkt);

读数据::

  uint32_t val = 0;

  RequestPtr req = std::make_shared<Request>(0x200000, 4, 0, masterId);
  PacketPtr pkt = Packet::createRead(req);

  pkt->dataStatic(&val);

  Tick t = memPort.sendAtomic(pkt);

我构造了两个 SimpleObject 的对象 hello 和 goodbye 通过 SystemXBar 连接到 SimpleMemory 上，hello 读，goodbye 写，可以发现 hello 读出了 goodbye 写入到 0x200000 的 0xdeadbeef.

Timing 模式
~~~~~~~~~~~~~~

在 Atomic 模式的基础上，我有写了个 Timing 模式读取存储器的代码::

  void SimpleObject::readTiming()
  {
      RequestPtr req = std::make_shared<Request>(0x200000, 4, 0, masterId);
      PacketPtr pkt = Packet::createRead(req);
  
      // we cannot use a local stack variable in timing request
      pkt->dataDynamic(new uint32_t);
  
      bool res = memPort.sendTimingReq(pkt);
      if (res) {
          std::cout << "Successfully send timing request. Tick = " <<
              curTick() << std::endl;
      }
  }
  
和 Atomic 读取的代码类似，但有几个不同的地方。首先是不能用 dataStatic 让 pkt 使用栈上的变量，因为在 sendTimingReq 返回之后，访存的数据包仍然被使用。其次是 sendTimingReq 返回的是一个 bool 类型的结果，因为它可能会失败，我这里暂时不处理失败的情况。

过一段时间后，memPort 会收到这个请求的返回，需要实现 SimplePort 的 recvTimingResp 方法::

  bool SimpleObject::SimplePort::recvTimingResp(PacketPtr pkt)
  {
      std::cout << "Receive packet, val = 0x" <<
          std::hex << pkt->getLE<uint32_t>() <<
          ". Tick = " << std::dec << curTick() << std::endl;
      return true;
  }

在这里，我们可以从返回的包里面得到存储器中的值。同时输出当前 tick，可以发现和 sendTimingReq 时的 tick 相比，已经经过了 40000 ticks.我的配置里面频率是 1GHz，每周期是 1ns, 1000 ticks. 而在 SimpleMemory 的默认配置里面，延迟是 30ns，就是 30000 ticks. 而如果是 Atomic 模式，则直接返回 30ns 对应的 30000 tick. 通过调试发现，Timing 模式这多出来的 10000 ticks，也就是 10 周期，是从 SystemXBar 来的::

  class SystemXBar(CoherentXBar):
      # 128-bit crossbar by default
      width = 16
  
      # A handful pipeline stages for each portion of the latency
      # contributions.
      frontend_latency = 3
      forward_latency = 4
      response_latency = 2
      snoop_response_latency = 4
  
      # Use a snoop-filter by default
      snoop_filter = SnoopFilter(lookup_latency = 1)

这里 frontend_latency, forward_latency, response_latency 和 snoop_filter.lookup_latency 加起来刚好是 10 周期。在配置文件中修改这几个参数的值，都可以发现 recvTimingResp 中的周期发生相应的变化，足以说明多出来的 10 个周期是 SystemXBar 中的这几个延迟的总和。

.. [1] https://www.gem5.org/documentation/general_docs/memory_system/
.. [2] https://gem5.github.io/gem5-doxygen/classBaseCache.htm
.. [3] https://gem5.github.io/gem5-doxygen/classAbstractMemory.html
.. [4] https://github.com/dramninjasUMD/DRAMSim2
.. [5] https://en.wikipedia.org/wiki/Crossbar_switch
.. [6] https://www.gem5.org/documentation/general_docs/memory_system/gem5_memory_system/#coherent-bus-object
.. [7] https://gem5.github.io/gem5-doxygen/classBaseXBar.html
