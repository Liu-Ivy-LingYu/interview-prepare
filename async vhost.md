[异步模式下的 Vhost Packed Ring 设计介绍](https://mp.weixin.qq.com/s/h_LTJiscJ17cKPQLNCCcgA)

[英特尔DSA-加速DPDK Vhost](https://mp.weixin.qq.com/s/O_S9v6DGoc4HmamkJyuKEA)

DPDK Vhost库是一个高性能的VirtIO网络设备的后端实现，它是Open vSwitch中广泛使用的虚拟网络接口，实现高速地接收来自虚拟机的数据包和传输数据包给虚拟机。

异步卸载流水线，使用户程序能够将Intel DSA的数据包拷贝和上层业务执行并行起来。

### Split Virtqueue
一个描述符表、一个可用环和一个已用环。

描述符表是描述数据包缓冲区的描述符数组，每个可用环和已用环都是指向描述符表中描述符的索引数组。可用环和已用环之间的区别在于，可用环存储 Vhost 可用的描述符索引，但使用的环存储 VirtIO 前端可用的描述符索引。每个环还有一个尾部指针，用于指示 VirtIO 前端（用于可用环）或 Vhost（用于使用环）将放置下一个描述符索引的位置，并且在将描述符索引填充到环后，尾指针会增加。尾部指针分别称为可用环和已用环的avail_idx和used_idx。

一个enqueue或dequeue操作可以分为两个子操作：处理Virtqueue和数据包拷贝。VirtIO描述符所占的存储空间较小，而Core十分擅长对小数据空间的读和写，因此Core可以快速地完成处理Virtqueue的更新操作。  与处理Virtqueue相比，数据包的拷贝会占用更多的Core时钟周期，尤其是在数据包较大时。因此，在Intel DSA加速的DPDK Vhost方案中，我们将enqueue或dequeue操作中的所有数据包拷贝都卸载到Intel DSA（以蓝色箭头显示），但将 Virtqueue 的处理仍留给Core（如绿色箭头所示）。

DPDK Vhost利用**DPDK dmadev**库（一个通用的DMA引擎库）来操作硬件DMA设备。因此，除了 Intel DSA以外，DPDK Vhost还可以支持其他类型的DMA引擎，如Crystal Beach DMA。值得注意的是，DPDK dmadev 库将每个Intel DSA WQ呈现为一个具有一个虚拟通道的 DMA设备实例。在下文中，我们将使用 “DMA 设备实例” 和 “虚拟通道” 来描述如何使用DPDK Vhost 的异步API。

DPDK Vhost的异步API 由**控制路径API**和**数据路径 API**组成。

- 在开始数据路径操作之前，用户程序需要调用控制路径API来使能DMA加速，具体需要以下三个步骤。

1. 首先，调用**rte_vhost_async_dma_configure（）**告诉DPDK Vhost 将要使用的全部DMA设备实例和虚拟通道。

2. 其次，在调用**rte_vhost_driver_register（）**时设置RTE_VHOST_USER_ASYNC_COPY标识，用于通知DPDK Vhost准备DMA需要的数据路径环境。

3. 最后，对于将要使用DMA加速的Virtqueue，在其启用时（即接收到VHOST_USER_SET_VRING_ENABLE消息），在DPDK Vhost的回调函数vring_state_changed（）中调用**rte_vhost_async_channel_register（）**。之后，用户程序就可以调用异步的数据路径API将Virtqueue上的数据包拷贝卸载到DMA。

- DPDK Vhost的异步数据路径API包含enqueue操作和dequeue操作。使用**enqueue**操作的异步API需要两个 步骤。

1. 首先，调用**rte_vhost_submit_enqueue_burst（）**以获取可用的描述符，并将数据包的拷贝操作提交到指定的DMA 设备实例和虚拟通道。在调用rte_vhost_submit_enqueue_burst（）之后，DMA 设备便开始执行拷贝操作。

2. 其次，调用**rte_vhost_poll_enqueue_completed（）**以获取指定的DMA设备实例和虚拟通道上拷贝完成的数据包，并将这些数据包对应的描述符写回Virtqueue。在调用rte_vhost_poll_enqueue_completed（）之后，VirtIO 前端便能接收DMA设备发送完成的数据包。

- 值得注意的是，用户程序需要在适当的时候调用 rte_vhost_poll_enqueue_completed（）。否则，即使DMA设备已经将数据包拷贝到了VM内存，VirtIO 前端也无法接收到数据包。

**dequeue**操作的异步API只有rte_vhost_try_dequeue_burst（）。rte_vhost_try_dequeue_burst（）将新的数据包拷贝操作提交到给定的 DMA 设备实例和虚拟通道。此外，它还会检查给定的DMA设备以获取拷贝完成的数据包，并将完成的数据包返回给用户程序。 

- 当要禁用Virtqueue或者销毁Vhost 设备时，应用程序需要按照以下三个步骤调用控制路径API以禁用Virtqueue的DMA加速。

1. 首先，等待DMA设备完成Virtqueue 中所有的数据包拷贝操作。这可以通过在 DPDK Vhost的回调函数destroy_device（）中调用 rte_vhost_clear_queue（） 或在DPDK Vhost 的回调函数 vring_state_changed（） 中调用 rte_vhost_clear_queue_thread_unsafe（） 来实现。

2. 其次，注销Virtqueue的DMA加速。这可以通过在 DPDK Vhost的回调函数 destroy_device（）中调用 rte_vhost_async_channel_unregister（）或在DPDK Vhost 的回调函数 vring_state_changed（）中调用rte_vhost_async_channel_register_thread_unsafe（） 来实现。

3. 最后，调用rte_vhost_async_dma_unconfigure（）通知DPDK Vhost不再使用的DMA 设备实例和虚拟通道。

![图8. 使用DPDK Vhost的异步API示例]()


![图4. 采用Intel DSA 加速的 DPDK Vhost 的设计视图]()

- 重排序数组，Vhost Core只会通知VM拷贝全部完成的数据包，从而保证enqueue或dequeue操作后数据包不乱序。

### Packed Virtqueue
一个具有不同描述符格式的描述符环。

在本地保存一个已用环形缓冲区的备份，以减小更新descriptor ring的频率，提升运行效率。

Async shadow used ring就是Vhost后端在本地缓存的一个used ring的缓冲区，后端取用descriptor后，由于DMA拷贝的特性，拷贝任务并不会马上完成，所以在DMA拷贝完成之前，我们会将这个descriptor的信息暂存在Async shadow used ring中，在DMA拷贝完成后将 Async shadow used ring中的内容一起写回 descriptor ring中，并通知前端进行回收处理。

**rte_vhost_submit_enqueue_burst()**会将相应需要enqueue的包进行enqueue操作， **rte_vhost_poll_enqueue_completed()**会查询目前DMA数据拷贝的完成情况，只有查询到包的数据已经完成了相应的拷贝，才能将对应的暂存在Async shadow used descriptor中的元素写回 used ring。这是在 API 层面上异步路径和同步路径最主要的差别。

rte_vhost_submit_enqueue_burst()函数的调用会将内存拷贝交由DMA处理后继续处理下一批数据包，在这里将descriptor处理和内存拷贝两件工作在时间上重叠起来，提升了数据传输的总体性能。


[利用英特尔数据流加速器（DSA）加速 DPDK Vhost虚拟接口](https://mp.weixin.qq.com/s/jsFzwbeWQRwYxxAU5wV28w)


[OVS DPDK async](https://github.com/istokes/ovs/tree/dpdk-dma-tracking)