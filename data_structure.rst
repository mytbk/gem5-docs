gem5 的一些数据结构
=======================

这里介绍一些 gem5 使用的数据结构。

由于 gem5 的源码都在 src/ 下，为了方便，以下写源码路径时，都省略开头的 src/.

TimeBuffer
-------------

在 gem5 CPU 流水线实现的代码里面，使用了一种称为 TimeBuffer 的数据结构，源码在 cpu/timebuf.hh.

TimeBuffer 的主要用途是在流水线的不同流水级之间传递信息。在两个部件之间传递数据，可能需要一定的时间，为了在模拟器中让一个数据在若干周期后传到一个部件，gem5 使用了 TimeBuffer 结构。

对于一个多级流水线的模拟，一种做法是每周期模拟的时候，先执行靠后的流水级的模拟，再执行靠前的流水级的模拟。但是这样做有一个问题，它假设了每个流水级获得的流水级间传送的数据都来自前面的流水级，但靠后的流水级也可以往前传送信息，出现这种情况的时候，先模拟哪个流水级都会出现相互依赖的问题。TimeBuffer 数据结构是解决这种问题的一种方法，它模拟了组合逻辑之间交换数据的寄存器。

TimeBuffer 的构造函数有两个参数 past 和 future，意思是可以通过这个缓冲区访问 past 个时间单位之前，和 future 个时间单位之后的数据。TimeBuffer::wire 相当于一个指向缓冲区中某个时间相对点的数据的指针。以如下代码为例::

  TimeBuffer<MyClass> mybuf(3, 0);
  TimeBuffer<MyClass>::wire toNext = mybuf.getWire(0);
  TimeBuffer<MyClass>::wire fromPrev = mybuf.getWire(-3);

这里我实例化了一个元素类型是 MyClass 的 TimeBuffer，通过它可以访问 3 个时间单位前的数据和 0 个时间单位之后的数据。toNext 指向当前时间的数据，fromPrev 指向 3 个时间单位前的数据。当一个时间单位结束后，TimeBuffer 的 advance() 成员函数将其中的数据更新至一个时间单位之后，模拟了电路时序部件的更新，此时 toNext 和 fromPrev 指向的值都会更新。当产生数据的部件通过 toNext 这个 wire 写了个值之后，mybuf 做 3 次 advance() 操作，就可以通过 fromPrev 得到写入的值，这就模拟了一个数据经过 3 个周期后到达另一个部件。

TimeBuffer 的内部实现是一个大小为 past+future+1 的数组和一个随着每次 advance() 不断自增的索引 base. 有点类似环形队列的实现，但是可以在 -past 和 future 之间的任意位置读写。在 CPU 模型中，主要通过 wire 来读写数据。

bug: TimeBuffer 没有正确地实现复制构造函数和移动构造函数，如果在使用了 TimeBuffer 的代码中发生程序崩溃的现象，可能发生了 TimeBuffer 对象的复制。


引用计数指针 RefCountingPtr
-----------------------------

gem5 实现了一种引用计数指针，源码在 base/refcnt.hh.

C++11 标准引入了两种只能指针 unique_ptr 和 shared_ptr 用于减轻指针引起的内存问题。其中 shared_ptr 就是一种引用计数指针，用计数器记录它被引用多少次，如果发现不再被引用，则释放指针指向的对象。那么为什么 gem5 要弄一个 RefCountingPtr?

我们考虑一个对象的指针 pobj，用 pobj->method() 调用它的一个成员函数时，在成员函数内部，this 指向 pobj 指向的对象，但它是一个裸指针，如果要在其他位置存放 this 用于后续使用，例如用指针 p2 保存 this，那么在 method() 返回之后，pobj 有可能不再被使用，如果用的是 shared_ptr，那么 pobj 指向的对象会被释放，之后再通过 p2 使用这个对象的时候，就会发生 use-after-free 的错误。如果从 this 构造出一个 shared_ptr，那么新构建的 shared_ptr 实际上和原有的那个不一样，它的计数器是重新初始化的，两个包含同一指针的 shared_ptr 析构时，也会出现内存重复释放的错误。（这说明不能用一个裸指针构造多于一个 shared_ptr）

RefCountingPtr 的做法不一样，它不是在 RefCountingPtr 的结构里面存计数器，而是在对象里面存计数器。要这么实现，首先 refcnt.hh 里面定义了一个 RefCounted 类，它就是一个计数器。所有要用 RefCountingPtr<T> 的类 T 必须继承 RefCounted，于是 RefCountingPtr<T> 的对象只要是通过同一个裸指针构造的，无论构造多少次，指针指向的对象里面都有它的正确的引用计数，这样就不会出现内存泄漏或者释放后使用的问题了。

实际上 RefCountringPtr 在 C++11 里面有类似的东西，它是 std::enable_shared_from_this.
