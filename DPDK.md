## mbuf

## Example DPDK Network Application
```c
int main(int argc, char **argv)
{
/* Initialize environment */
rte_eal_init(argc, argv);
rte_eal_pci_probe();
/* Allocate memory for packet buffers */
buf_pool = rte_mempool_create("pool", num_mbufs,..., socket_id, FLAGS);
/* Configure device */
rte_eth_dev_configure(port_id, num_rxqs, num_txqs, &port_conf);
/* Configure device Rx/Tx queues */
rte_eth_rx_queue_setup(port_id, queue_id, num_rxds, socket_id, &rx_conf, buf_pool);
rte_eth_tx_queue_setup(port_id, queue_id, num_txds, socket_id, &tx_conf);
/* Start the device */
rte_eth_dev_start(port_id);
/* Echo back any received packets */
while (!done) {
/* Receive packets */
num_rx = rte_eth_rx_burst(port_id, queue_id, pkt_list, MAX_BURST);
if (!num_rx) continue;
/* Transmit packets */
num_tx = rte_eth_tx_burst(port_id, queue_id, pkt_list, num_rx);
while (num_tx != num_rx)
/* uh oh... drop some pkts */
rte_pktmbuf_free(pkt_list[num_tx++]);
}
/* Stop the device */
rte_eth_dev_stop(port_id);
}
```

### sample application - exception path
使用 DPDK 为数据包设置一条通过 Linux* 内核的异常路径。这是通过使用虚拟 TAP 网络接口来实现的。DPDK 应用程序可以读取和写入这些接口，并将其作为标准网络接口显示给内核。

应用程序为每个正在使用的 NIC 端口创建两个线程。一个线程从端口读取数据并将数据不加修改地写入特定于线程的 TAP 接口。第二个线程从 TAP 接口读取数据并将数据不加修改地写入 NIC 端口。

### sample application - vhost

示例程序根据媒体访问控制 (MAC) 地址或虚拟局域网 (VLAN) 标签在虚拟机之间执行简单的数据包交换。

数据包首先从物理 NIC 端口接收并进入 VM 的 Rx 队列。通过客户机 testpmd 的默认转发模式（io forward），这些数据包将被放入 Tx 队列。dpdk-vhost 示例依次获取数据包并将其放回相同的物理 NIC 端口。

```c
/*
 * Main function of vhost-switch. It basically does:
 *
 * for each vhost device {
 *    - drain_eth_rx()
 *
 *      Which drains the host eth Rx queue linked to the vhost device,
 *      and deliver all of them to guest virito Rx ring associated with
 *      this vhost device.
 *
 *    - drain_virtio_tx()
 *
 *      Which drains the guest virtio Tx queue and deliver all of them
 *      to the target, which could be another vhost device, or the
 *      physical eth dev. The route is done in function "virtio_tx_route".
 * }
 */
static int
switch_worker(void *arg __rte_unused)
{
	unsigned i;
	unsigned lcore_id = rte_lcore_id();
	struct vhost_dev *vdev;
	struct mbuf_table *tx_q;

	RTE_LOG(INFO, VHOST_DATA, "Processing on Core %u started\n", lcore_id);

	tx_q = &lcore_tx_queue[lcore_id];
	for (i = 0; i < rte_lcore_count(); i++) {
		if (lcore_ids[i] == lcore_id) {
			tx_q->txq_id = i;
			break;
		}
	}

	while(1) {
		drain_mbuf_table(tx_q);
		drain_vhost_table();
		/*
		 * Inform the configuration core that we have exited the
		 * linked list and that no devices are in use if requested.
		 */
		if (lcore_info[lcore_id].dev_removal_flag == REQUEST_DEV_REMOVAL)
			lcore_info[lcore_id].dev_removal_flag = ACK_DEV_REMOVAL;

		/*
		 * Process vhost devices
		 */
		TAILQ_FOREACH(vdev, &lcore_info[lcore_id].vdev_list,
			      lcore_vdev_entry) {
			if (unlikely(vdev->remove)) {
				unlink_vmdq(vdev);
				vdev->ready = DEVICE_SAFE_REMOVE;
				continue;
			}

			if (likely(vdev->ready == DEVICE_RX))
				drain_eth_rx(vdev);

			if (likely(!vdev->remove))
				drain_virtio_tx(vdev);
		}
	}

	return 0;
}
```

virtio_net_device_ops

### prog guid - vhost
[async vhost](async vhost.md)


### Switch Representation within DPDK Applications
E-Switch representation offload

port representors(DPDK)

- Ethernet switch device driver model (switchdev)(Linux)

以太网交换机设备驱动程序模型 (switchdev) 是用于交换机设备的内核驱动程序模型，可将转发（数据）平面从内核中卸载。
[Ethernet switch device driver model (switchdev)](https://www.kernel.org/doc/Documentation/networking/switchdev.txt)

