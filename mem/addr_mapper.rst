AddrMapper
=================

gem5 有一个 AddrMapper SimObject. 虽然没看到有模型用到它，但是需要的时候可以用它做个地址转换。

AddrMapper 基类的基本工作流程是：收到一个包时，调用 ``remapAddr(Addr)`` 接口转换地址，把包中的原始地址存入 SenderState，设置包的地址为转换后的地址，再下发这个包。在 Timing 模式收到响应，或 Atomic/Functional 发送结束后，将响应的包的地址恢复为原始地址。

gem5 实现了一个最简单的 AddrMapper: RangeAddrMapper, 就是把一个地址范围映射到一个等大小的地址范围。
