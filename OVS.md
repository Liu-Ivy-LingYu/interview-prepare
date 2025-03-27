[Intel® Ethernet Controller 700系列: Open vSwitch硬件加速应用说明](https://mp.weixin.qq.com/s?__biz=MzI3NDA4ODY4MA==&mid=2653337015&idx=1&sn=43b13a351e2ba4489a03093345a6acb5&chksm=f0cb4230c7bccb269d7a418d5545dfd1eafcb53bf1eb4a7ce8357533993f1a3d2ebc97c73a32&mpshare=1&scene=1&srcid=1126v66kn9OZFXffGwQv4gyA&sharer_sharetime=1606358522611&sharer_shareid=d336eec716f16e6430e624c43d70de51&exportkey=A%2BXWNLFxULD9GlBk%2F6%2Bzau8%3D&pass_ticket=0Wu39P9soW%2FRPvkW3dF3z3sCnKhCQaXVDXUMiJpz%2BJf934c23ZYJHSptsGVHArWJ&wx_header=0#rd)

[基于DPDK的Open vSwitch概述](https://mp.weixin.qq.com/s/HJixu8WpqIJHugXVujY_TQ)
原始的OVS:多层虚拟交换机。快速路径在内核空间。虽然不需要进行上下文切换但是受到Linux网络协议栈的带宽限制

OVS-DPDK:快速路径也在内核空间, 异常路径和原来OVS的路径一致

[基于Intel®E810 的OVS-DPDK VXLAN TUNNEL性能优化](https://mp.weixin.qq.com/s/nFG-eFiHiGuTfrjn6Cv0Ag)
对于VXLAN或者其他类型的隧道报文，packet recirculation导致一个报文处理周期会涉及到多次packet header解析和流表查询。

OVS内部的flow表示被转化成基于DPDK rte_flow的描述后通过rte_flow_create() API将流表规则下发到硬件，后续OVS接收到的报文将会被硬件打上flow mark标签，OVS可以迅速地从flow mark索引到相应的flow并执行其action。

[基于Intel®以太网800系列网络适配器的FDIR功能及原理介绍](https://mp.weixin.qq.com/s/2rvnJKTqDVyiVp46APieEQ)

[Intel E810 Advanced RSS介绍](https://mp.weixin.qq.com/s/l3R3OK5lbuzOu8u_ulS-Fg)

[DPDK Rx flexible descriptor 在Intel E810 网卡中的使用](https://mp.weixin.qq.com/s/qA-ry_3TXH5ZaJBCwXHMJg)