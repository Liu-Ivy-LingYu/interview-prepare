## DMA
QEMU通过VFIO接口实现设备模拟（对container, group, device）

### 设置DMA 重映射（IOVA -> 进程虚拟地址）

ioctl(VFIO_IOMMU_MAP_DMA)创建设备IO地址（IOVA）到宿主机物理地址的映射

为虚拟地址vaddr找到物理页并pin住（因为设备DMA是异步的，随时可能发生，物理页面不能交换出去）

IOMMU页表中记录的是**IOVA和vaddr对应的物理页（HPA）**之间的映射关系

IOMMU将GVA->HPA的映射关系存储在**DMAR表（DMA Remapping table）**

### DMA访问：
Guest VM通过dma_alloc_coherent()分配连续的虚拟内存GVA，并获取对应的物理地址IPA/GPA。QEMU捕获这一操作，并在内部维护GVA->HPA的映射。QEMU向IOMMU提交DMAR表条目（DMA Remapping Table Entry）（ioctl(iommu_fd, VFIO_IOMMU_MAP_DMA, &map)）。

设备发起DMA请求，虚拟设备（如网卡）通过DMA控制器读取/写入缓冲区，触发硬件级的DMA引擎操作。

DMA控制器使用DMAR表，将设备的DMA物理地址（DPA）转换为宿主机HPA。IOMMU同时检查访问权限​（如读/写）和隔离域，防止非法访问。


设备透传就是由虚机直接接管设备，虚机可以直接访问（什么叫直接访问？答：IOVA（GPA）->HPA，寄存器虚拟地址为BAR3_BASE+offset，CPU初次访问时触发MMU页表故障，QEMU截获（如果IPA->HPA的映射已经存在，就可以直接访问HPA了），检查该地址属于已注册的PCI内存映射区域，通过IOMMU进行转换或者通过QEMU软件模拟）MMIO空间，VMM配置好IOMMU之后，设备DMA读写请求也无需VMM介入，需要注意的是设备的配置空间没有透传，因为VMM已经配置好了BAR空间，如果将这部分空间也透传给虚机，虚机会对BAR空间再次配置，会导致设备无法正常工作。

直通设备的BAR地址空间映射到QEMU（分配物理内存，IOVA->IPA->HPA）。通过mmap系统调用。

## MSI中断
- ​MSI优势：相比传统的INTx中断，MSI通过PCI配置空间中的中断消息寄存器（MSI Cap）​直接向CPU发送中断向量，避免了物理引脚的模拟，支持多中断向量和高性能。
- ​VFIO直通要求：VFIO需要将物理设备的MSI能力完整暴露给虚拟机，包括中断向量数量、消息地址等配置。QEMU通过vfio-pci驱动读取设备的PCI配置空间，在虚拟机的PCI配置空间中模拟MSI Cap结构，供虚拟设备驱动读取。

### QEMU/KVM处理VFIO-MSI中断的流程图：
物理设备触发MSI中断 → KVM捕获物理IRQ → QEMU通过eventfd接收通知 → 
QEMU模拟MSI消息写入（通过KVM的kvm_mmu_write()接口写入VM的物理内存/如果驱动读取的是APIC寄存器，QEMU通过KVM的kvm_irq_delivery接口直接触发虚拟CPU的中断） → 虚拟机APIC处理中断 → 设备驱动执行ISR → 清除中断

### KVM irqfd机制：
KVM通过irqfd将物理IRQ（IRQ 17）绑定到QEMU的eventfd文件描述符。QEMU创建一个eventfd​（如/dev/eventfd），KVM通过ioctl将物理IRQ与eventfd绑定，当物理设备触发IRQ时，KVM向eventfd发送一个事件（字节0x01或0x02）。

ls /run/qemu/vfio-00:1f.3/msi会显示设备的中断eventfd文件

### irq_bypass
irq_bypass允许KVM绕过宿主机的IRQ子系统，通过vfio-pci驱动直接监控物理设备的PCI配置空间和内存访问（绑定到vfio驱动的设备都可以bypass irq）（原本的路径是什么样的？需要依赖宿主机的IRQ路由表——how？），…..（posted interrupt）…… KVM使用vCPU的IRR（interrupt request register）来轮询未处理的中断向量，发现中断后，通过kvm_irq_delivery接口向目标vCPU注入中断，vCPU执行相应的ISR。（向目标vCPU注入中断的结果是什么？irqfd在哪里用了？）。

使用kvm_irqfd绑定物理irq到KVM的eventfd，而不是通过宿主机的IRQ系统。（不能bypass的时候是怎么绕的？）

### posted interrupt
而posted interrupt是Intel CPU的一种功能，允许将中断延迟处理（将中断向量写入APIC的ICR（interrupt command register），但并不立即触发处理），中断状态被保留在IRR中，直到下一个CPU周期，从而减少中断处理的开销。

使用Interrupt posting模式时，vcpu可以直接在non-root模式下处理中断而不会被vm-exit到宿主机。
中断被缓存在PIR中，只有绑定的VM能处理这些中断。

VFIO注册生产者信息，并为producer初始化token和irq，token为eventfd的文件描述符，irq为host irq

KVM使用vfio生成的eventfd注册consumer

[QEMU-KVM中的VFIO MSI机制](https://www.cnblogs.com/haiyonghao/p/14709880.html)

[IOMMU DMA VFIO 一站式分析](https://www.owalle.com/2021/11/01/iommu-code/)
