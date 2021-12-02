gem5 中的存储器类型
===========================

SimpleMemory
---------------

SimpleMemory 是一个单端口存储器模型，其带宽和延迟可配置。

使用Timing模式访问时，SimpleMemory在接收请求后，将自身设置为busy状态，根据带宽在一定时间后转为非busy状态。在busy状态时接收请求，则访问失败，需要请求端重发请求。

连接到CoherentXBar下时，访问失败的原因还有XBar端口被占用，此时由XBar而不是Memory返回访问失败的信息。
