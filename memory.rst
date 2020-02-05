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

.. [1] https://www.gem5.org/documentation/general_docs/memory_system/
