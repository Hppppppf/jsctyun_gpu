# GPU算力管理

# 1.算力管理概览

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A9c4f93a2-ca87-4708-844d-bacb9bc90720%3Aimage.png?table=block&id=1c86a624-db43-8033-88f8-e64a9fe34402&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

抛开任务管理层面，也就是把上一节的‘云-边-端’和‘GPU集群任务调度’暂且不讨论，原因是这两部分已经不单单是某个产品，或某一个技术能实现的，这个需要客户从最开始的业务架构的规划、到AI任务管理平台，再到硬件层面的拓扑架构等多方面的技术整合和选型，才能满足所需的AI任务的预期管理。本节主要聚焦在为了满足业务的不同GPU资源使用的诉求，**单卡调度、单机多卡调度和多机多卡调度所发展出来的基础技术。**

# 2.单GPU卡算力管理

## 2.1. **MPS（单卡多线程并行）**

多进程服务（MPS）是对 CUDA 应用编程接口（API）的一种替代性实现，并且与原有的二进制代码兼容。MPS 的运行时架构旨在透明地支持协作式的多进程 CUDA 应用，主要是消息传递接口（MPI）任务。MPS时Nvidia在Pascal架构引入的，并在Volta架构中进一步的完善。Volat-MPS相比较之前的MPS，主要在以下几个方面做了提升：

1. 直接提交任务至 GPU：Volta MPS 客户端可以直接向 GPU 提交任务，无需通过 MPS 服务器。
2. 独立的 GPU 地址空间：每个 Volta MPS 客户端拥有独立的 GPU 地址空间，不与其他客户端共享。
3. 有限的执行资源分配以保证服务质量 (QoS)：Volta MPS支持有限的执行资源分配，以确保服务质量。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Aa75a6fc8-f626-4476-8cf0-af220c0e1d7f%3Aimage.png?table=block&id=1c86a624-db43-80ed-bfd5-d84dc0b4df04&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

**MPS优势**

1. **提高GPU利用率：**MPS使不同进程的内核和内存复制操作可以在GPU上并发执行，提升GPU利用率并加快任务完成速度。
2. **减少GPU上下文存储：**MPS服务器为所有客户端共享一份GPU存储和调度资源，减少了资源占用。
3. **减少GPU上下文切换：**通过共享调度资源，MPS消除了进程间切换的开销，提高了效率。

**适用应用场景**

MPS特别适用于那些单个进程不足以饱和GPU的应用。通过在同一节点上运行多个进程，MPS增强了并发性。适用于每网格块数较少的应用；若应用因每网格线程数少而GPU占用率低，使用MPS可提升性能。在强扩展场景中，MPS能有效利用剩余的GPU容量，让不同进程的内核并发执行，优化计算效率。

**限制**

**相关限制**

**A. 系统限制**

**操作系统支持：**MPS 仅支持 Linux 操作系统。在非 Linux 操作系统上启动 MPS 服务器将导致启动失败。

**Tegra 平台支持：**仅支持 Volta MPS 在 Tegra 平台上运行。

**GPU计算能力要求：**MPS 需要计算能力版本为 3.5 或更高的 GPU。如果在应用 CUDA_VISIBLE_DEVICES环境变量后可见的任何一个 GPU 的计算能力低于 3.5，MPS 服务器将无法启动。

**统一虚拟寻址 (UVA)：**CUDA 的 UVA 功能必须可用，默认情况下，任何在计算能力版本为 2.0 或更高的 GPU 上运行的 64 位 CUDA 程序都启用了 UVA。如果 UVA 不可用，MPS 服务器将无法启动。

**页锁定主机内存限制：**MPS 客户端可以分配的页锁定主机内存量受限于 tmpfs 文件系统 (/dev/shm) 的大小。

**独占模式限制：**独占模式限制应用于 MPS 服务器，而不是 MPS 客户端。Tegra 平台不支持 GPU 计算模式。

**用户限制：**一个系统上只能有一个用户拥有活跃的 MPS 服务器。

**用户排队机制：**MPS 控制守护进程会将来自不同用户的 MPS 服务器激活请求排队，不管GPU独占设置如何，也会导致用户之间对 GPU 的串行独占访问。

**行为归因：**系统监控和会计工具（例如 nvidia-smi、NVML API）将所有 MPS 客户端的行为归因于 MPS 服务器进程。

**B. GPU计算模式限制**

nvidia-smi支持设置三种Compute Mode：

- PROHIBITED：GPU对所有Compute应用都不可用
- EXCLUSIVE_PROCESS：GPU一次只分配给一个Context（CPU进程），单个CPU进程的线程可以同时向 GPU提交工作
- DEFAULT：多个CPU进程可以同时使用 GPU。每个进程的各个线程可以同时向 GPU 提交工作。

**使用 MPS 实际上会使 EXCLUSIVE_PROCESS 模式对所有 MPS 客户端表现得像 DEFAULT 模式一样。MPS 总是允许多个客户端通过 MPS 服务器使用 GPU。**在使用 MPS 时，建议启用 EXCLUSIVE_PROCESS 模式，以确保只有一个 MPS 服务器使用该 GPU。这提供了额外的保障，确保 MPS 服务器是该GPU上所有 CUDA程序的唯一入口。

**C. 应用限制**

NVIDIA编解码SDK不支持：NVIDIA视频编解码SDK在Volta架构之前的MPS客户端上不受支持。

仅支持64位应用：只有64位应用程序被支持。

CUDA驱动API版本要求：如果应用程序使用CUDA驱动API，则必须使用CUDA 4.0或更高版本的头文件。

不支持动态并行：如果模块使用了动态并行特性，CUDA模块加载将会失败。

UID一致性要求：MPS服务器仅支持与服务器具有相同用户ID（UID）的客户端运行。如果服务器和客户端的UID不同，客户端应用程序将无法初始化。

流回调不支持：在Volta架构之前的MPS客户端上不支持流回调。调用任何流回调API将返回错误。

CUDA图不支持：Volta MPS 之前的Client上，MPS 不支持具有主机节点的 CUDA Graph。

页锁定主机内存限制：MPS Client应用能够请求的page-locked host memory数量受tmpfs文件系统限制（/dev/shm）

MPS客户端终止的影响：在未同步所有未完成GPU工作的情况下终止MPS客户端（例如通过Ctrl-C、程序异常如段错误、信号等），可能会使MPS服务器和其他MPS客户端处于未定义状态，导致挂起、意外失败或数据损坏。

CUDA IPC支持：由MPS Client创建的CUDA Context与没有使用MPS 创建的CUDA Context之间的CUDA IPC通信在Volta MPS是支持的

参考网站：https://docs.nvidia.com/deploy/mps/index.html

## 2.2. **MIG（单卡单显存多租户线程物理隔离）**

使用MIG，每个实例的处理器在整个内存系统中都有独立的路径——片上交叉开关端口、L2缓存库、内存控制器和DRAM地址总线都唯一分配给一个实例。这确保了单个用户的工作负载可以以可预测的吞吐量和延迟运行，具有相同的L2缓存分配和DRAM带宽，即使其他任务正在冲击自己的缓存或饱和其DRAM接口。MIG可以分割可用的GPU计算资源（包括流式多处理器或SM和GPU引擎，如复制引擎或解码器），以提供定义的服务质量（QoS），并为不同的客户端（如虚拟机、容器或进程）提供故障隔离。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A3152a322-c920-4139-93f0-0820e14c2347%3Aimage.png?table=block&id=1c86a624-db43-805e-9e04-d76049ba9b63&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

从NVIDIA安培一代开始的gpu（即具有计算能力>= 8.0的gpu）支持MIG。下表提供了支持的gpu列表：

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A49f35b88-0290-4975-b8b7-9b44cbff75d4%3Aimage.png?table=block&id=1c86a624-db43-801b-9c38-e3a046651aa7&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

GPU在MIG中分为GI和CI单元，其中CI单元是GI单元的subset，共享GI单元的共享内存和带框。而GI之间的算力在物理层面是隔离的。而GI是 GPU 切片和 GPU 引擎（DMA、NVDEC 等）的组合。GI内的任何内容总是共享所有内存和其GPU，但其SM切片可以进一步细分为CI。GI提供内存级别的QoS，每个GPU slice包含专用的GPU内存资源，这些资源限制了可用容量和带宽，并提供内存QoS。每个GPU内存切片获得总物理层面GPU 内存资源的 1/8，每个 GPU SM 切片获得总物理层面SM数量的1/7。CI包含父GI的SM切片和其它GPU引擎（DMA、NVDEC 等）的子集，其共享内存和引擎。比如下图是A100卡可以划分的MIG类型

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Ab377b028-e024-436d-9e67-41d83da5a6b9%3Aimage.png?table=block&id=1c86a624-db43-8041-be94-e03d006e15fe&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

**相关特性：**

1. 提高资源利用率：MIG允许从NVIDIA Ampere架构开始的GPU被安全地分割成最多七个独立的GPU实例，用于CUDA应用程序，为多个用户提供独立的GPU资源以实现最佳的GPU利用率。
2. 多租户物理隔离：对于具有多租户使用案例的云服务提供商（CSP），MIG确保一个客户不会影响其他客户的工作或调度，同时提供增强的隔离。

**相关限制：**

A. 系统限制 操作系统支持：MIG 仅支持由 CUDA 支持的 Linux 操作系统发行版。

支持配置：裸金属、容器、

GPU需要重置：在 A100/A30 上设置 MIG 模式需要 GPU 重置（因此需要超级用户权限）。

配置持久化：MIG配置是持久性的，不会随着系统重启而被重置。

驱动限制：所有持有驱动程序模块句柄的守护进程需要在启用 MIG 之前停止。

权限配置：切换 MIG 模式需要 CAP_SYS_ADMIN 能力。其他 MIG 管理，如创建和销毁实例，默认需要超级用户，但可以通过调整 /proc/ 中 MIG 能力的权限委托给非特权用户。

B. 应用限制

图形不支持：不支持图形API（例如OpenGL、Vulkan等）。

不支持G2C：不支持GPU到GPU的P2P（无论是PCIe还是NVLink）。

物理隔离：CUDA应用程序将计算实例及其父GPU实例视为单个CUDA设备。

跨实例调用限制：不支持跨GPU实例的CUDA IPC。跨计算实例的CUDA IPC是支持的。

MPS支持：CUDA MPS在MIG之上支持。唯一的限制是最大客户端数（48）会根据计算实例的大小按比例降低。

GDR支持：当从GPU实例使用时，支持GPUDirect RDMA。

参考文档：https://docs.nvidia.com/datacenter/tesla/pdf/NVIDIA_MIG_User_Guide.pdf

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A98bef4b5-3ede-443e-90cf-18959a9d46d1%3Aimage.png?table=block&id=1c86a624-db43-802b-890b-d48455fdab7d&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

# 3. 单机多卡GPU算力管理

## 3.1.  PCIE&NVLink&NVSwitch（单机多卡组合）

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Abd0572d8-6b2a-4e89-9a20-2ffab21c31b2%3Aimage.png?table=block&id=1c86a624-db43-80a3-b67e-ca1c9a922d5d&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

PCIE设备在单机内可以实现多个设备的数据传输，比如上图就是一个典型的8卡机器内的，GPU通过PCIE实现设备之间的互联，但是NIC、NVME、GPU是共享一个PCIE的总带宽的。如果涉及跨CPU的GPU之间通信，则会被CPU之间的链路限制。比如图示的GPU0-GPU1会收到PCIE0的影响，GPU3-GPU5会经过两个CPU之间的交互。**所以资源调度使用的GPU间拓扑架构对任务整体表现是具有很大影响的。**

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A835c01ca-a718-4320-aa8f-b579fe16eaa5%3Aimage.png?table=block&id=1c86a624-db43-80db-b1ec-d984eef63a8c&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

除了PCIE可以实现将单机的GPU卡互联，实现算力的切换掉配，但是PCIE的带宽是多个设备共享，并且最新的gen5 双向带宽只有64GB/s，远远小于大模型时代数据搬运的需求。NVLink Nvidia开发的是一种双向 GPU-GPU 直接互连技术，其5.0版本可提供高达 1800 GB/s 的带宽，比 PCIe 5.0 高出14倍。它连接了英伟达的 Grace CPU 和 Hopper GPU，显著增强了处理大规模AI模型的能力。作为先进的互联标准，NVLink 不仅优化了总线设计和通信协议，还通过点对点架构和串行传输技术提高了数据交换效率。NVSwitch，作为一个独立的 NVLink 芯片，支持最多18个 NVLink 连接，进一步提升了多GPU环境中数据流通的速度和并行处理复杂任务的效率。

但是虽然GPU之间绕过了PCIE网卡的限制，但是GPU之间的通信能力，取决于NVLink的数量，如图所示GPU0-GPU6只有1条NVLink，GPU3-GPU5之间有2条。在Ampere架构之后，Nvidia引入了NVSwitch，使单个机器内任何GPU卡之间的带宽链路获得了一致性，并且在Hopper架构中将NVLink引入到了机器之间，实现了服务器组的多GPU卡联通的一致性。这也是为什么客户AI任务需要了解底层GPU拓扑架构，不同的架构也需要适配不同的算力调度分配。

# 4. 多机多卡GPU算力管理

## 4.1. NVSwitch

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A8f43be6f-bcb8-4df8-84da-d281fb7c3ce8%3Aimage.png?table=block&id=1c86a624-db43-8025-bdae-e9381aa22510&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

在Hopper架构中中，Nvidia引入了基于模块化的MGX平台的GB200服务器，最多可以链接32个GPU，所有 GPU 线程能够访问高达 19.5TB 的内存，总带宽高达 900GB/s，以及高达 14.4TB/s 的带宽。

参考资料：https://resources.nvidia.com/en-us-grace-cpu/nvidia-grace-hopper?ncid=so-link-252431-vt04

## 4.2. RDMA

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Ab56e644c-2cf4-4584-8f98-84ec6d81986f%3Aimage.png?table=block&id=1c86a624-db43-80ab-9cdc-d7834d7d714c&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

RDMA，全称 Remote Direct Memory Access（远程直接内存访问），是一种高性能网络通信机制。它使一台计算机能够直接访问另一台计算机的内存，而无需操作系统内核或CPU的干预。这种方式可以显著减轻CPU的负担，提升数据传输速度，降低延迟，并增强整体计算效率。

**技术特点-优点**

**高性能与低延迟：**应用程序直接和网卡交互进行网络通信，绕过了内核和CPU切换，降低了CPU使用率，降低了数据传输也延迟。

**高数据传输：**数据传输无需将复制到网络层，减少数据复制开销，提高数据传输效率。

**大规模并行计算：**支持多个独立的通信流，可在不同的计算节点之间实现高效的数据交换和同步。

**快速内存访问：**允许远程节点直接访问本地内存。

**硬件off-load：**将网络协议栈的处理卸载到网卡硬件上，减轻CPU的负担。

**技术特点-缺点**

**专用硬件设备：**需要特定的软硬件匹配，不是所有的操作系统和网络设备都支持RDMA。

**安全性：**允许远程节点直接访问本地内存，这可能会带来一些安全性问题。

**编程复杂性：**需要人员对内存管理和井发机制有深入的理解。

**网络规模限制：**RDMA的设计目标是高性能和低延迟，因此它对于网络的稳定性和可靠性要求高，

**高成本：**需要特定的网卡硬件支持，硬件厂商少。

**技术分类**

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A394b2595-5023-4bde-9781-3aed3dfe1b8b%3Aimage.png?table=block&id=1c86a624-db43-80b1-b81d-e1bb0ba5ba6e&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

按照技术分类，RDMA技术可以分为IB、RoCE和iWARP三种：

**IB：**L2到L4到需要自己的专用硬件，设备成本最高

**RoCE：**可以使用以太网的交换设备，将IB的报文封装成以太网包进行收发，低成本IB方案。

**iWarp：**基于TCP/IP协议，相比于RoCE v2和IB具有更好的可靠性，在大规模组网时，优势明显。但是性能iWARP要比UDP的RoCE v2和IB差。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A412d8f60-246f-4f79-b018-05cee6a1c5fc%3Aimage.png?table=block&id=1c86a624-db43-802d-97b1-ffd73aaf72c4&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1390&userId=&cache=v2)

## 4.3. 阿里云eRDMA

eRDMA是阿里云自研的云上弹性RDMA网络，底层链路复用VPC网络，采用全栈自研的拥塞控制CC（Congestion Control）算法，享有传统RDMA网络高吞吐、低延迟特性的同时，可支持秒级的大规模RDMA组网。可兼容传统HPC应用、AI应用以及传统TCP/IP应用。

传统TCP/IP协议由于其固有的局限性，如较大的拷贝开销、厚重的协议栈处理、复杂的拥塞控制（CC）算法以及频繁的上下文切换等，在面对数据中心业务对网络性能日益增长的需求时，逐渐成为应用性能提升的瓶颈。为解决这些问题，远程直接内存访问（RDMA）技术应运而生，它通过实现零拷贝和内核旁路等功能，大幅减少了数据传输中的延迟与CPU占用，并提高了吞吐量。然而，由于成本高昂且运维复杂，RDMA的应用范围受到了限制。针对这一问题，阿里云开发了增强型RDMA（eRDMA），旨在提供一种在云端普及的解决方案，既保持了RDMA低延迟的优势，又降低了部署和使用的门槛，使得更多应用程序能够受益于更优的网络性能。**eRDMA默认采用iWarp模式，相比较IB，RoCE，不需要专门的RDMA设备，与默认的VPC网络互通，成本最低，有丰富的组网和弹性拓展能力。**

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A9937b498-8117-4f5b-a09c-aab463e01a43%3Aimage.png?table=block&id=1c86a624-db43-80fa-b85c-f937184eedf5&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1290&userId=&cache=v2)

**功能优势**

高性能：ReRDMA继承RDMA优势，应用于VPC中，提供低延迟性能。

普惠：eRDMA免费启用，购买实例时勾选即可，无需额外付费。

规模部署：eRDMA采用自研CC算法，适应VPC网络变化，在有损环境中保持良好性能，简化大规模部署。

弹性扩展：eRDMA基于神龙架构，支持ECS实例中动态添加和热迁移，部署灵活。

共享VPC网络：eRDMA通过ENI复用现有网络资源，不改变业务组网即可激活RDMA功能，便于集成。

官方已经提供比较详细的eRDMA信息，可以参考：https://help.aliyun.com/zh/ecs/user-guide/elastic-rdma-erdma/

## 4.4.  GDR

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Adb1e8cf0-4b9b-4f86-812c-d7c01ff8b2bb%3Aimage.png?table=block&id=1c86a624-db43-8085-a669-d8d949a45ce2&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

GPUDirect RDMA 是一种在 Kepler GPU和CUDA 5.0 中引入的技术，它利用 PCI Express的标准功能实现 GPU和第三方设备之间的直接数据交换。第三方设备包括网络接口、视频采集设备和存储适配器。GPUDirect RDMA 支持 Tesla 和 Quadro GPU。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Ab62200c5-7dbd-4461-8e40-17c6a3dfc5fa%3Aimage.png?table=block&id=1c86a624-db43-80ca-8932-fa0616f21ce8&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

传统上，数据在GPU和另一个设备之间传输时，必须通过CPU，这导致潜在的性能瓶颈和延迟增加。GPUDirect技术则通过绕过CPU，直接访问和传输数据，显著提高系统性能。在设置 GPUDirect RDMA 通信时，从 PCI Express 设备的角度来看，所有物理地址对两个对等设备都是相同的。在这个物理地址空间中，有线性窗口称为 PCI BAR。每个设备最多有六个 BAR 寄存器，因此可以有最多六个活跃的 32 位 BAR 区域。64 位 BAR 消耗两个 BAR 寄存器。PCI Express 设备以与系统内存相同的方式对对等设备的 BAR 地址进行读写。

参考链接：https://docs.nvidia.com/cuda/gpudirect-rdma

