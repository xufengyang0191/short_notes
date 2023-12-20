# Storage System Basic Concept 1_NVMf

[TOC]

## 1. 定义

NVMe over Fabrics（NVMe-oF）是一种基于网络传输协议的新型存储技术，它允许将NVMe协议扩展到远程存储设备上。与传统的SAN（Storage Area Network）相比，NVMe-oF可以提供更低的延迟和更高的带宽，同时提供更好的可扩展性和灵活性。这使得NVMe-oF成为处理大量高速数据的应用程序的理想选择，例如机器学习、人工智能、高性能计算等。

从技术角度来看，NVMe-oF利用高速网络传输协议，如RDMA（Remote Direct Memory Access）和TCP/IP，将NVMe协议扩展到远程存储设备上。它可以在多种网络传输协议上运行，例如RoCE（RDMA over Converged Ethernet）、InfiniBand、Fibre Channel等。这些协议可以提供更低的延迟和更高的带宽，从而实现了高效的远程存储访问。

总之，NVMe-oF是一项极具前景的技术，它能够为企业提供更好的数据存储和访问方式，同时也能够提高企业在处理大量高速数据时的效率。随着技术的不断发展，NVMe-oF的应用前景也将越来越广泛。

NVMf 的架构？

## 2. NVMf 和 RDMA 的关系？

NVMf 在kernel中复用了 RDMA。

RDMA stack 一般是位于 kernel space 的，因为 RDMA 的实现需要访问底层硬件和内核态的数据结构。不过，也有一些 RDMA stack 可以在 user space 运行，例如 OpenFabrics Interfaces (OFI)。OFI 是一套 API，允许用户空间的应用程序通过不同的传输协议访问 RDMA 设备。OFI 实现了一些 RDMA stack 的功能，并将它们暴露给用户空间的应用程序，从而允许应用程序在不需要内核支持的情况下使用 RDMA 功能。OFI 还支持多种 RDMA 网络技术，例如 InfiniBand、RoCE 和 iWARP，这使得应用程序能够适应不同类型的网络环境。

RDMA（Remote Direct Memory Access）协议栈是一种在网络协议栈中提供高性能、低延迟网络传输的技术。它可以让应用程序直接读写远程主机上的内存，从而避免了操作系统内核的参与，减少了网络传输的延迟和CPU资源的消耗。

RDMA协议栈由软件和硬件两部分组成，软件部分负责处理网络协议的数据包，硬件部分则是一种特殊的网络适配器，提供高速的数据传输和内存读写能力。RDMA协议栈通常包括以下组件：

1. 用户空间库（user-space library）：提供RDMA的API接口，让应用程序可以直接访问RDMA协议栈的功能，而无需了解底层的协议细节。
2. 网络协议栈（network protocol stack）：处理RDMA协议的数据包，包括数据包的解析、重组、路由和传输等功能。RDMA协议栈可以基于TCP/IP或InfiniBand等不同的底层网络协议实现。
3. DMA引擎（DMA engine）：负责将数据从应用程序内存中复制到RDMA适配器的内存中，并将数据从适配器的内存中复制到远程主机的内存中。RDMA协议栈通过DMA引擎实现数据零拷贝，提高了数据传输的效率。

总的来说，RDMA协议栈可以提供高速、低延迟、高吞吐量的网络传输能力，适用于需要处理大量数据的高性能计算、云计算、存储等应用场景。

## 3. NVMe-oF 在 FC 和 RDMA 上的性能评估

当我们想要共享存储的时候，有两种主要的选择：Ethernet 和 Fibre Channel

Ethernet-iSCSI，transitioning to NVMe/RoCE and NVMe/TCP

Fibre Channel-FCP(SCSI), transitioning to FC-NVMe

Ethernet（以太网）是一种广泛使用的局域网（LAN）协议，最初是由Xerox、Intel和Digital Equipment Corporation共同开发的，现在已经成为了工业标准。Ethernet通常使用双绞线、光纤或无线电波进行数据传输，可达到不同的速率，从10 Mbps到100 Gbps不等。Ethernet的优点是成本低廉，易于扩展和维护，并且可以应用于许多不同的网络环境，包括家庭、办公室和数据中心等。

Fibre Channel（光纤通道）是一种高速的、基于光纤的存储区域网络（SAN）协议，可用于连接计算机和存储设备。Fibre Channel支持点对点、交换和环形拓扑结构，并可达到高达32 Gbps的速度。Fibre Channel具有高可靠性和低延迟的优点，并且支持高级功能，例如可靠性、负载平衡和故障隔离等。Fibre Channel主要用于数据中心和企业级应用程序等需要高性能、高可靠性和可扩展性的场景。

总的来说，Ethernet和Fibre Channel都是非常重要的计算机网络协议，它们在不同的场景下发挥着不同的作用。Ethernet主要应用于局域网和互联网等广泛应用场景，而Fibre Channel则主要用于高性能、高可靠性和可扩展性的存储网络。

最近的会议引发了关于哪个传输通道使用 NVMe-oF 协议提供最佳性能的争论。一些供应商坚信 RDMA 是实现更高吞吐量的更好选择，许多供应商坚持使用光纤通道以获得性能优势。

这两种网络结构技术都有自己的优点和陷阱。

### 3.1 基于 FC 的 NVMe

NVMe-oF 是 NVM Express 组织提供的协议，用于支持通过网络结构传输 NVMe 流量，而 FC-NVMe 是特定于光纤通道的传输标准。大多数企业已经在使用光纤通道技术来处理其进出存储阵列的关键数据。

光纤通道是专门为存储设备和系统设计的，它是企业存储区域网络 （SAN） 解决方案的事实标准。光纤通道技术的主要优点是，它为**现有的传统存储协议（SCSI）和新的NVMe协议提供并发通信量**，这些协议在存储基础架构中使用相同的硬件资源。SCSI 和 NVMe 在光纤通道上的这种共存使大多数企业受益，因为它们只需简单的软件升级即可实现 NVMe 操作。

2018 年，NVM Express 在 NVMe-oF 协议中添加了一项名为非对称命名空间访问 （ANA） 的新功能。这允许在多个主机和命名空间之间支持多路径 I/O。

第 5 代和第 6 代是光纤通道的新版本。第 6 代支持高达 128Gbs 的传输速度，即存储网络中最高的传输速度。此外，Gen 6 还支持监视和诊断功能，可以查看延迟级别和 IOPS。NVMe-oF 与两个新版本的光纤通道协议无缝集成。

根据Demartek的报告，与基于SCSI的光纤通道协议相比，光纤通道上的NVMe可提供58%的IOPS和34%的延迟。大型企业倾向于使用 FC-NVMe 来处理关键工作负载，因为它具有简单性、可靠性、可预测性和性能。

但是，此实施需要更多的存储网络级别的专业知识，这可能会增加成本。

### 3.2 使用 RDMA 的 NVMe over Fabrics

RDMA 提供了光纤通道的替代方案。根据 [WhatIs.com](https://searchstorage.techtarget.com/definition/Remote-Direct-Memory-Access) 的说法，“远程直接内存访问（RDMA）是一种技术，它允许网络中的计算机在主内存中交换数据，而不涉及任何一台计算机的处理器，缓存或操作系统。“

换句话说，RDMA 允许应用程序绕过软件堆栈来处理网络流量。由于 RDMA 数据传输不涉及如此多的资源，因此 RDMA 可帮助企业以更低的延迟实现更高的吞吐量和更好的性能。启用了 NVMe 的存储设备似乎靠近具有 RDMA 的主机。

RDMA可以在存储网络中启用，如RoCE（融合以太网上的RDMA），iWARP（互联网广域RDMA协议）和Infiniband等协议。

iWARP大致是TCP/IP上的RDMA。它使用 TCP 和流控制传输协议 （SCTP） 进行数据传输。

RoCE 支持基于以太网的 RDMA。它被描述为以太网上的Inifiniband。RoCE v1 和 RoCE v2 有两个版本。由于传输机制不同，这两种协议彼此不兼容。

Inifiniband主要由提供高性能计算解决方案的供应商提供支持。它是最快的RDMA存储网络技术，数据传输速度约为100 Gbs，而Gen 128 FC-NVME提供的数据传输速度高达6 Gb/s。与FC-NVMe一样，Infiniband是一种无损传输协议，提供服务质量（QoS）机制以及基于信用的流量控制。

**一些供应商认为RDMA与NVMe用例高度兼容，因为它们使用相同的队列结构。使用基于 RDMA 的技术的主要原因是命令传输不需要任何类型的命令封装和转换，因为两者都使用类似的排队结构进行数据传输，而无需 CPU 干预。这样，RDMA 节省了 CPU 周期，从而降低了从主机到存储设备的数据传输延迟。**

最后，我们回到NVMe over Fabrics，以client的一个写请求的处理过程来展示NVMf如何利用RDMA技术。

1，NVMe Queue与Client端RDMA QP一一对应，把NVMe Submission Queue中的NVMe Command存放到RDMA QP注册的内存地址中（有可能带上I/O Payload），然后通过RDMA Send Queue发送出去；

2，当Target的NIC收到Work Request后把NVMe Command DMA到Target RDMA QP注册的内存中，并在Targe的QP的Completion Queue中设置一个Work Completion。

3，Target处理Work Completion时，把NVMe Command发送到后端PCIe NVMe驱动，如果本次传输没有带上I/O Payload，则使用RDMA Read获取；

4，Target收到PCIe NVMe驱动处理结果后通过QP的Send Queue把NVMe Completion结果返回给client。

### 3.3 主要差异化因素

- 借助光纤通道，企业可以保护其现有的硬件投资，同时充分利用支持 NVMe 的完整存储基础架构。但是，基于Infiniband，RDMA（iWARP或RoCE）和以太网的NVMe-oF实现通常需要企业的新硬件资源。
- 光纤通道结构具有流量控制的“缓冲区到缓冲区信用”功能，通过提供无损网络流量来确保企业的服务质量 （QoS）。RDMA 以太网（iWARP 和 RoCE）需要额外的协议支持才能启用此功能。
- 与其他网络结构选项相比，光纤通道启动网络通信所需的配置更少。
- 光纤通道结构具有自动发现和添加主机启动器和目标存储设备及其属性的功能。RDMA以太网（iWARP和RoCE）和Infiniband缺乏这种能力。

## 3. NVMf 对比实验

对比了 local NVMe, iSCSI and NVMe/RoCE，延迟增加28%，IOPS 接近

对比了 local NVMe, FCP and FC-NVMe，延迟增加一些，IOPS 接近

对比了 FC-NVMe, NVMe/TCP and NVMe/RoCE ，效果和成本
