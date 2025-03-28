## Question
a challenging project you worked on and how you overcame the difficulties
Contribute E810 live migration to Linux kernel(two kinds implementation: iommufd, vfio; qemu, kernel, driver(iommu driver, device driver, vfio); security)

1. Introduction & Background
Q1: Could you briefly introduce yourself and describe your professional journey, focusing on your experience in networking and virtualization?
(Goal: Highlight relevant work, key achievements, and career growth.)


Q2: In your role at Intel, you implemented live migration for Intel E810 NIC and IPU devices. Can you explain:
    • What the challenges were during live migration?
    • How you optimized the migration downtime from 600ms to 95ms?
(Goal: Assess your understanding of live migration, IOMMU, and DMA dirty page tracking.)
Intel 4th xeon DMA device dirty page track - SSAD

Q3: You mentioned implementing Scalable IOV (SIOV) for Intel NICs.
    • What are the key differences between SR-IOV and SIOV?
    • How did you ensure performance improvements of 4x during packet processing?
(Goal: Evaluate your expertise in NIC virtualization and performance optimization.)


Q4: Can you discuss your experience with DPDK? Specifically:
    • How did you enable custom GRE header support in DPDK for Intel E810 NIC?
    • Why was deep packet inspection required, and how did you validate the solution?
(Goal: Understand your DPDK experience, packet processing, and testing methodology.)
DDP

3. Programming & Problem-Solving
Q5: This role requires strong C/C++ and Python skills.
    • Could you provide an example where you used Python for automating testing or debugging in your NIC driver projects?
    • How do you ensure your C/C++ code is efficient and optimized for hardware performance?
(Goal: Evaluate your coding proficiency and problem-solving mindset.)

4. Networking Protocols & Virtualization
Q6: You listed VLAN, GRE, VXLAN, MPLS, and IPSEC in your skills.
    • Can you explain a project where you worked with VXLAN or GRE protocols?
    • How did you troubleshoot or optimize performance issues related to these protocols?
(Goal: Assess your networking protocol knowledge and troubleshooting skills.)

Q7: This role emphasizes cloud-networking and virtualization technologies.
    • How have you collaborated with customers or architecture teams to address cloud-networking challenges?
    • Can you share an example of a prototype or POC you delivered that later became production-ready?
(Goal: Understand your ability to work in real-world cloud and customer-facing scenarios.)

5. Behavioral & Team Collaboration
Q8: Describe a time when you worked cross-functionally with hardware and software teams.
    • How did you align everyone toward a common goal, and what challenges did you face?

## Answer
Q1.
Hello, my name is Lillian. I graduated from Tongji University in 2018 with a Master's in Computer Science. After graduation, I joined Intel, where I have worked for seven years across two key roles.
In my first role, I worked in the Baseboard Management Controller (BMC) team, focusing on firmware development for Intel’s server platforms. I contributed to improving system reliability and implementing secure firmware features.
In my second role, I joined Intel's Network Platform Group, specializing in NIC driver development. My work includes developing high-performance DPDK and Linux kernel drivers, with a strong focus on virtualization features such as live migration and Scalable I/O Virtualization (SIOV). I successfully implemented these features on Intel’s E810 NIC and the first generation of smart NIC, MEV, ensuring seamless performance in virtualized and cloud environments.

Q2.
The primary challenge of live migration for VFIO devices is their direct passthrough to the virtual machine, unlike VirtIO devices, which are abstracted. This means different NIC vendors implement VFIO devices differently, with no standardized method for live migration. To address this, we needed a thorough understanding of the VFIO device lifecycle, including creation, packet transmission, and reception. Additionally, we analyzed the interactions between QEMU, the kernel, IOMMU, NIC drivers, and the migration protocol to design a reliable solution.

To optimize migration downtime, we leveraged the Dirty Page Tracking feature of Intel's 4th generation Xeon processors. This feature uses a bit in the second-stage page table entries to track modifications to memory pages. Live migration is inherently iterative: the VM's memory and vCPU state are transmitted first, followed by the VFIO device's state. Without Dirty Page Tracking, the network is down throughout the VFIO device migration. By utilizing this feature, we were able to transmit modified pages during the migration process. Once the number of dirty pages fell below a threshold, we paused the source VFIO device, transferred the remaining data pages, and resumed the device at the destination. This approach significantly reduced downtime and improved migration reliability.

Q3.
SIOV provides finer-grained resource allocation compared to SR-IOV. While SR-IOV distinguishes streams using Bus-Device-Function (BDF), SIOV leverages Process Address Space Identifiers (PASID), enabling more granular resource management.
In a traditional SR-IOV setup, when a VF device is passed through to a virtual machine, the IAVF driver must read or write VF device registers and queues. These operations often trigger traps in QEMU, causing the CPU to switch from virtual to non-virtual mode. This frequent context switching leads to significant overhead and negatively impacts performance.
To address this, I implemented a solution that directly maps VF device registers and queues to memory accessible by the IAVF driver. By eliminating the need for QEMU traps and the associated context switching, I reduced the overhead and improved throughput. This optimization resulted in a 4x improvement in packet processing performance, enabling significantly better efficiency in virtualized environments.
(DMA设备重映射)

Q4.
The Intel E810 NIC supports Dynamic Device Personalization (DDP), which allows users to configure specific packet processing rules. These rules are set via the PF driver, enabling the NIC to direct incoming packets matching specific criteria to corresponding queues. For example, in this case, we configured rules to handle different types of GRE packets.
To test this functionality, we used Scapy to construct custom GRE packets with varying header fields to represent different packet types. Using DPDK, we configured the NIC to route packets of each type to a dedicated queue based on the DDP rules. Once the packets were sent, we verified the functionality by checking whether the queues contained the expected number of packets for each type.
This approach ensured that the DDP rules were correctly applied and that the NIC handled packet classification as intended. By leveraging Scapy and DPDK, we validated the feature in a controlled and repeatable manner, demonstrating the NIC's ability to efficiently process and direct GRE packets in high-performance environments.

