## PFR
[Intel Platform Firmware Resilience](https://blog.csdn.net/zdx19880830/article/details/84190005)

## 单片机启动过程
加电后，运行芯片内固有程序（一般在flash中），即启动代码。系统中断被禁止，PC到地址0处取指令执行。

启动代码主要完成两方面的工作，一是初始化执行环境，例如中断向量表、堆栈、I/O等；二是初始化c库和用户应用程序。

在第一阶段，启动代码的过程可以描述为：

建立中断向量表；
初始化存储器；
初始化堆栈寄存器；
初始化i/o以及其他必要的设备；
根据需要改变处理器的状态。

最后，调用系统的初始化函数，将控制权交给了操作系统。

Linux： 安装页表， 驱动载入 ，运行环境准备 。

init (Busybox) ：启动 shell。

## BIOS
从功能上看，BIOS分为三个部分：

自检及初始化
这部分负责启动电脑，具体有三个部分：

第一个部分是用于电脑刚接通电源时对硬件部分的检测，也叫做加电自检（Power On Self Test，简称POST），功能是检查电脑是否良好，通常完整的POST自检将包括对CPU，640K基本内存，1M以上的扩展内存，ROM，主板，CMOS存储器，串并口，显示卡，软硬盘子系统及键盘进行测试，一旦在自检中发现问题，系统将给出提示信息或鸣笛警告。自检中如发现有错误，将按两种情况处理：对于严重故障（致命性故障）则停机，此时由于各种初始化操作还没完成，不能给出任何提示或信号；对于非严重故障则给出提示或声音报警信号，等待用户处理。

第二个部分是初始化，包括创建中断向量、设置寄存器、对一些外部设备进行初始化和检测等，其中很重要的一部分是BIOS设置，主要是对硬件设置的一些参数，当电脑启动时会读取这些参数，并和实际硬件设置进行比较，如果不符合，会影响系统的启动。

第三个部分是引导程序，功能是引导DOS或其他操作系统。BIOS先从软盘或硬盘的开始扇区读取引导记录，如果没有找到，则会在显示器上显示没有引导设备，如果找到引导记录会把电

## redfish host interface(http over usb)
[LINUX USB虚拟网卡 RNDIS](http://blog.sina.com.cn/s/blog_4d4617410100idp5.html)

[让Linux支持usb虚拟网卡。](https://blog.csdn.net/qq_31878855/article/details/80742593)

[usbnet驱动深入分析-usb虚拟网卡host端](https://blog.csdn.net/zh98jm/article/details/6339320?spm=a2c6h.12873639.0.0.3cac681fPYmi2p)


## 驱动
[linux 中bus驱动解析](https://www.cnblogs.com/downey-blog/p/10507703.html)

##
[Linux进程分配内存的两种方式--brk() 和mmap()](https://www.cnblogs.com/diegodu/p/9230280.html)
