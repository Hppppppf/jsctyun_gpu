---
title: "英伟达GPU介绍"
meta_title: "英伟达GPU介绍"
description: "英伟达GPU介绍"
date: 2025-04-01T16:03:00+08:00
categories: ["算力百科"]
tags: ["英伟达", "GPU"]
draft: false
---

# 英伟达GPU介绍

## 架构演进

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A9c9ced77-ecf7-447f-bccf-93785651b9bb%3Aimage.png?table=block&id=1c86a624-db43-805d-8745-f7d8d542bfa9&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

## 主要架构

### Pascal（2016）

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Aa720f919-0951-43c5-9580-5a508402a3d7%3Aimage.png?table=block&id=1c86a624-db43-80de-861b-c53b9ce7f760&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

关键特性包括：

- CUDA core：每个 SM 包含 64 个单精度（FP32）CUDA core，分为两个处理块，每个块有 32 个 FP32 CUDA core。虽然这比 Maxwell SM 的 CUDA core数量少了一半，但它保持了类似的寄存器文件大小和 warp/线程块占用率。
- 寄存器和线程：尽管每个 SM 的 CUDA core较少，但 GP100 拥有更多的 SM（60个），因此总寄存器数量更多，并支持更多的线程、warp 和线程块同时运行。
<!--more-->
每个SM具有FP16计算的Cuda core 64个， FP32的Cuda core 32个.所以一个GP100总的core是： FP16: 64*60=3840， FP32:32*60=1920个

- 共享内存：由于 SM 数量增加，GP100 GPU 的总共享内存量也增加了，聚合共享内存带宽实际上翻倍。
- 高效执行：改进后的 SM 中共享内存、寄存器和 warp 的比例使得代码执行更加高效，具有更多的 warp 可供指令调度器选择，更高的加载启动次数以及每线程到共享内存的更高带宽。
- 高级调度：每个 warp 调度器（每个处理块一个）能够在每个时钟周期调度两个 warp 指令。
- 新功能：可以处理 16 位和 32 位精度的指令和数据，FP16 操作吞吐量最高可达 FP32 操作的两倍。（DPunit 单元是 Core 的一半）

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A4b57442b-ac1c-4f64-8db6-bef234309fd6%3Aimage.png?table=block&id=1c86a624-db43-8085-b452-f4e0501d4edb&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

Tesla P100 引入了 NVIDIA 的新型高速接口 NVLink，它能够提供高达 160 GB/s 的双向带宽，是 PCIe Gen 3 x16 带宽的 5 倍。上图显示了 NVLink 如何在一个混合立方网格拓扑中连接八个 Tesla P100 加速器，从图中可以看到，任意一个GPU都有4个NVLink与其他的GPU相连。下图展示了GPU互联的路径：

其中NVLINK表示两个GPU是通过NVLink链接，可以利用总带宽160GB/s带宽（双向），单个GPU-to-GPU之间带宽是40GB/s（双向），单向是20GB/s；

PCIE表示需要走PCIE--- CPU--- PCIE 链接，在Pasca架构，是第三代PCIE，理论最大带宽是16GB/s（单向）

虽然NVLInk 1.0 GPU-to-GPU单向只有20GB/s，相比较PCIE的16GB/s提高幅度没有很震撼。但是需要清楚的是NVLINK只是G2G独享，但是PCIE的单向16GB/s是两个GPU，2张NIC网卡共用的，真正用于GPU-to-GPU数据传输的其实远远达不到16GB/s。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A70a7e6b3-5dee-4bde-bb67-14124cdfdb0b%3Aimage.png?table=block&id=1c86a624-db43-800f-9d2c-dc32222bae15&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

NVLink：表示最大双向40GB/s

PCIE：表示最大双向32GB/s

所以可以看到，如果GPU需要使用PCIE方式去读区其他GPU上的数据，必然数据传输速度收到了PCIE的影响。从物理架构层面受到PCIE链接带宽限制，AI任务调度方面要尽可能让任务调度到NVLink的关联GPU上。详情可参加nvidia官网介绍：https://images.nvidia.com/content/pdf/tesla/whitepaper/pascal-architecture-whitepaper-v1.2.pdf

### Volta（2017）

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A1e6ca697-a428-4626-9793-f081f6c5f1bb%3Aimage.png?table=block&id=1c86a624-db43-80ed-9da3-cd26980f3a44&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

关键特性：第二代 NVIDIA NVLink：单GPU支持六条 NVLink 链路，总带宽300 GB/s。HBM2 内存，16 GB HBM2 内存子系统，带宽 900 GB/s。L1和共享内存合并，由4个纹理单元共享，可以看到内存L1/L2分级和扩容都是为了避免数据从内存或硬盘读取，内存分级也是任务运算的瓶颈之一。Volta 多进程服务 (MPS)，提供QOS和隔离。（这一部分会在下一篇文章结合ACK容器GPU算力调度一起说明）

GV100，含有6个GPC，每个GPC拥有7个TPC，14个SM。每个SM拥有：64 个 FP32core+64 个INT32 core+32个 FP64core+8个 Tensor core+4个纹理单元。含84个SM 的完整GV100GPU，总共拥有 5376个 FP32 core、5376 个 INT32 core、2688个 FP64 core、672个 Tensor core以及 336个纹理单元。

V100 GPU 包含640个 Tensor core：每个SM有8个core，SM内的每个处理块（分区）有2个core。在VoltaGV100中，每个 Tensor core每时钟执行64次浮点 FMA 运算，一个5M中的8个 Tensor core每时钟总共执行512次 FMA 运算（或1024次单个浮点运算）。每个 HBM2 DRAM 堆栈由一对内存控制器控制。完整的GV100 GPU 总共包含6144KB 的L2缓存。Tesla V100 加速器拥有80个SM。

新的张量core使 Volta 架构得以训练大型神经网络，GPU 并行模式可以实现深度学习功能的通用计算，最常见卷积/矩阵乘（Conv/GEMM）操作，依旧被编码成融合乘加运算 FMA（Fused Multiply Add），硬件层面还是需要把数据按照：寄存器-ALU-寄存器-ALU-寄存器方式来回来回搬运数据，因此专门设计 Tensor Core 实现矩阵乘计算。

NVLInk 2.0 GPU-to-GPU单向提高到25GB/s，相比较1.0提高了5GB/s，但是每个GPU可以链接Link数量提高了到了6条，所以单GPU双向最大带宽来到了25*2*6=300GB/s，相比Pascal架构提升了一倍左右。同时，引入NVSwitch1.0 旨在提高 GPU 之间的通信效率和性能。NVSwitch1.0 可以支持多达 16 个 GPU 之间的通信，可以实现 GPU 之间的高速数据传输。可以看到Nvidia除了疯狂的堆SM和core，也在想尽一切办法提升GPU-to-GPU之间的带宽，使数据尽可能在GPU间快速读取。隐约可以遇见，如何绕开PCIE，绕开CPU和内核切换是AI时代的瓶颈，毕竟大模型时代，数据量是几何倍数的增长。

#### NVLINK：第一代GPU-to-GPU

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A922f62e0-9e5e-4e83-b8de-ba9dce607417%3Aimage.png?table=block&id=1c86a624-db43-804a-81f1-c445082c68c0&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1050&userId=&cache=v2)

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A75595bb9-949a-40c9-a55a-bb64d0bfbaba%3Aimage.png?table=block&id=1c86a624-db43-8071-bb87-eeb2d46e4702&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

NVLink1：表示最大双向50GB/s

NVLink2：表示最大双向100GB/s

PCIE：表示最大双向32GB/s

由此可以看见Volat架构正在努力的将GPU变成一个整体的GPU对外提供GPU能力，但是不通GPU之间的数据传输还是不一样的，这个对于任务调度GPU计算资源提出了挑战。

详情可参加nvidia官网介绍：https://www.nvidia.cn/content/dam/en-zz/zh_cn/Solutions/Data-Center/volta-gpu-architecture/Volta-Architecture-Whitepaper-v1.1-CN.compressed.pdf

### Turing（2018）

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A025781cf-a03e-4bec-b498-23c241be46f5%3Aimage.png?table=block&id=1c86a624-db43-80f5-a95e-f665c1bfaba7&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

关键特性：

包含 2,560 个 CUDA core和 320 个 Tensor core。

继承自 Volta 架构的增强 MPS 功能：Turing 继承并进一步优化了 Volta 架构中首次引入的多进程服务功能，Tesla T4 上的 MPS 在小批量推理任务中提供了更好的性能，减少了启动延迟，提高了服务质量（QoS），并且能够处理更多的并发客户端请求。

大幅提升的内存配置：Tesla T4 配备了 16 GB 的 GPU 内存和 320 GB/s 的内存带宽，这几乎是其前代产品 Tesla P4 GPU 的两倍。

每个SM的纹理处理器都引入了wrap进行scheduler调度，并且每个纹理处理器都有自己的寄存器进行数据切换。

Turing 架构中的 Tensor Core（张量core）增加了对 INT8/INT4/Binary 的支持，加速神经网络训练和推理函数的矩阵乘法core。一个 TU102 GPU 包含 576 个张量core，每个张量core可以使用 FP16 输入在每个时钟执行多达 64 个浮点融合乘法加法（FMA）操作。SM 中 8 个张量core在每个时钟中总共执行 512 次 FP16 的乘法和累积运算，或者在每个时钟执行 1024 次 FP 运算，新的 INT8 精度模式以两倍的速率工作，即每个时钟进行 2048 个整数运算。

可以看到Turing架构主要是Volta的改版，主要引入了光线追踪的功能，而这个功能更多的是利用在3D大型游戏领域。

T4最适合：小型模型的推理。关键特性： 比 L4 更旧且速度较慢。适合小规模实验和原型设计。例如，可以用T4 开始项目，然后在生产环境中使用 L4 或 A10 运行相同的代码。参考：https://images.nvidia.com/aem-dam/en-zz/Solutions/design-visualization/technologies/turing-architecture/NVIDIA-Turing-Architecture-Whitepaper.pdf

### Ampere（2020）

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A6054645e-4e97-4b67-9feb-9de1d5c2da28%3Aimage.png?table=block&id=1c86a624-db43-806f-81ab-e827c7262f65&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

关键特征：

1. 每一个SM含有4个第三代Tensor Cores，每一个拥有256个FP16/FP32，意味着每一个SM拥有1025个。L1共享内存增加到了192KB。其中A100增加到了108SM。
2. 多实例 GPU (MIG)：允许 A100 Tensor Core GPU 安全地分割成最多 7 个独立的 GPU 实例，每个实例的处理器在整个内存系统中都有单独且相互隔离的路径，片上交叉端口、L2 缓存、内存控制器和 DRAM 地址总线都被唯一地分配给一个单独的实例，确保单个用户的工作负载可以在可预测的吞吐量和延迟下运行，同时具有相同的 L2 缓存分配和 DRAM 带宽，即使其他任务正在读写缓存或 DRAM 接口。用户可以将这些虚拟 GPU 实例当成真的 GPU 进行使用，为云计算厂商提供算力切分和多用户租赁服务。（这一部分会在下一篇文章结合ACK容器GPU算力调度一起说明）
3. 第三代 NVLink：第三代 NVLink 的数据速率为 50 Gbit/sec 每对信号，并首次引入NVLink switch full to mesh 的概念。
4. PCIe Gen 4：支持 PCIe Gen 4，提供 31.5 GB/s 的带宽。40 GB HBM2 和 40 MB L2 缓存：

#### NVLink：第三代

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Ac8eafb38-e9d2-49d2-9d36-b3fc419f03d3%3Aimage.png?table=block&id=1c86a624-db43-803d-a778-eef165d7db19&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

上图是一个8个A100组网称的一个大型GPU。可以看到引入了6个NvSwitch的概念。每个GPU链接每个NvSwitch2个Link，每个LInk单向25GB/s，双向50GB/s。由于NvSwitch的池化作用，所以理论上任何一个GPU与其他GPU进行数据交换的理论速度达到了双向50*12=600GB/s。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Aff47357e-d324-47b4-bc44-26c114d21969%3Aimage.png?table=block&id=1c86a624-db43-8097-b405-fa3d0d9eb443&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

NVLink12：表示最大双向600GB/s

在Ampere架构，Nvidia利用引入的NvSwitch实现了GPU full mesh组网，实现了8卡或4卡整体对外组网提供一致性服务的能力。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Ae9dee107-7e85-4e25-a8e4-301b0991add4%3Aimage.png?table=block&id=1c86a624-db43-802a-8fd4-c04b4fe5c3f2&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

**除了GPU之间显存的交互，我们还需要注意到PCIE、NIC和GPU之间的组网方式。**

解决了GPU之间的数据交换速度不一致情况，我们看一下外部清理。一般情况下，8张NIC网卡，两两bond对外以一张网卡形式对外提供服务。所以其实理论上在系统层面会把8张物理网卡识别为4张软件层面网络设备（NIC0-NIC4）。所以这里又会涉及到NIC和NIC之间、NIC 和CPU之间。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Ada34e577-69b0-443c-ba1a-350e6805a4a3%3Aimage.png?table=block&id=1c86a624-db43-8009-a59d-f85066e13e32&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1260&userId=&cache=v2)

**PCIE：表示数据只需要经过PCIE交换，A100使用第四代PCIE，双向达到64GB/s**

**SYS：表示数据需要经过CPU处理，有上下文和内核切换**

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A18e94b48-2fc6-45bf-b6cf-3efbf93fdf26%3Aimage.png?table=block&id=1c86a624-db43-8014-86a0-c8fa56409367&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1080&userId=&cache=v2)

**PCIE：表示数据只需要经过PCIE交换，A100使用第四代PCIE，双向达到64GB/s**

**CPU：表示数据经过的是同一个CPU处理，只需要跨PCIE和 PCIE host bridge**

**SYS：表示数据需要经过CPU处理，有上下文和内核切换**

可以看到数据的远距离调用和切换，对于任务的运行，耗时、计算等都会产生影响，而这个影响是物理层面的瓶颈，只能尽可能的想尽办法将任务调度得更‘近’一点

#### 多级带宽

最下面为这次架构升级所引入 NVLink 技术，它主要来优化单机多块 GPU 卡之间的数据互连访问。在传统的架构中，GPU 之间的数据交换受到CPU 和 PCIe 总线的瓶颈。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A6ea9d677-a276-4510-834d-610fc7264e2c%3Aimage.png?table=block&id=1c86a624-db43-803e-81f5-e7c21d363528&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1260&userId=&cache=v2)

再往上一层为 L2 Cache 缓存和 DRAM，它负责的是每块 GPU 卡内部的存储。L2 Cache 缓存作为一个高速缓存，用于存储经常访问的数据，以减少对 DRAM 的访问延迟。DRAM 则提供了更大的存储空间，用于存储 GPU 计算所需的大量数据。这两者的协同工作，使得 GPU 能够高效地处理大规模数据集。

再往上一层为共享内存和 L1 Cache，它们负责 SM 中数据存储，共享内存允许同一 SM 内的线程快速共享数据，通过共享内存，线程能够直接访问和修改共享数据，从而提高了数据访问的效率和并行计算的性能。

而最上面是针对具体的计算任务 Math 模块，负责 GPU 数学处理能力。Math 模块包括 Tensor Core 和 CUDA Core，分别针对不同的计算需求进行优化。Tensor Core 是专为深度学习等计算密集型任务设计的，能够高效地执行矩阵乘法等张量运算。而 CUDA Core 则提供了更广泛的计算能力，支持各种通用的 GPU 计算任务。

在 Ampere 之前的 GPU 架构中，如果要使用共享内存（Shared Memory），必须先把数据从全局内存（Global Memory）加载到寄存器中，然后再写入共享内存。这不仅浪费了宝贵的寄存器资源，还增加了数据搬运的时延，影响了 GPU 的整体性能。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A249ae2b3-709c-4be7-92f6-933af4de16b1%3Aimage.png?table=block&id=1c86a624-db43-8096-a4f2-f3b26f7461d3&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

如上图所示，Ampere 架构中提供异步内存拷贝机制，通过新指令 LDGSTS（Load Global Storage Shared），实现全局内存直接加载到共享内存，避免了数据从全局内存到寄存器再到共享内存的繁琐操作，从而减少时延和功耗。

另外，A100 还引入了软件层面的 Sync Copy，这是一种异步拷贝机制，可以直接将 L2 Cache 中的全局内存传输到 SMEM 共享内存，然后直接执行，减少了数据搬运带来的时延和功耗。

A100最适合：训练和推理较大模型（70 亿到 700 亿参数）。关键特性：NVIDIA 的主力 GPU，适用于 AI、数据分析和高性能计算（HPC）任务。提供 40GB 和 80GB 两种版本。对于内存受限的工作负载（如在小批量上运行大模型），A100 可能比 H100 更具成本效益。

A10最适合：小型到中型模型（70 亿参数或以下，如大多数基于扩散的图像生成模型）的推理，以及小型模型的小规模训练。关键特性：与 A100 架构相同，因此大多数能在 A100 上运行的代码也能在 A10 上运行。小型工作负载的性能与成本比率良好。

参考：

https://images.nvidia.cn/aem-dam/en-zz/Solutions/data-center/nvidia-ampere-architecturewhitepaper.pdf

https://developer.download.nvidia.com/video/gputechconf/gtc/2020/presentations/s21730-inside-the-nvidia-ampere-architecture.pdf

## Ada Lovelace(2022)

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A4758040a-e5cd-4cb5-ba6b-a6365b3795d9%3Aimage.png?table=block&id=1c86a624-db43-809c-a1b0-db872c3dce4a&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

关键特征：

AD102包含12个GPC，72个TPC，144个SM。

每个SM包含128 个 CUDA core，一个第三代 RT Core，四个第四代 Tensor core，四个纹理单元，256 KB 寄存器文件，128 KB 的 L1/共享内存，可以根据图形或计算工作负载的需求配置为不同的内存大小。

RT Core 在 Turing 和 Ampere GPU 中：包含专用硬件单元，用于加速数据结构遍历，执行关键的光线追踪任务。

L4最适合：小型到中型模型（70 亿参数或以下，如大多数基于扩散的图像生成模型）的推理。关键特性： 成本效益高，但仍具备强大性能。

VRAM 容量与 A10 相同，但内存带宽仅为一半。性能比 T4 高出 2 到 4 倍。

参考：https://images.nvidia.cn/aem-dam/Solutions/Data-Center/l4/nvidia-ada-gpu-architecture-whitepaper-v2.1.pdf

### Hopper（2022）

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Ad29d5fbe-7258-4883-9224-0a39dde5bc09%3Aimage.png?table=block&id=1c86a624-db43-8067-8fb8-f9838471e699&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

关键信息：

结合软件和定制的 Hopper Tensor core：专门设计用于加速 Transformer 模型训练和推理。智能管理 FP8 和 16 位计算，自动处理重铸和缩放，提供高达 9 倍的 AI 训练速度和 30 倍的大型语言模型推理速度。

提供近 2 倍的带宽提升：H100 SXM5 GPU 是首款采用 HBM3 内存的 GPU，提供 3 TB/s 的内存带宽。

50 MB L2 缓存：缓存大量模型和数据集，减少对 HBM3 的重复访问。

第二代MIG技术，提供约 3 倍的计算能力和近 2 倍的内存带宽。

每个 GPU 实例支持多达 7 个独立的实例，每个实例有自己的性能监控工具。（这个为云产生切分GPU给多个租户创造条件）

第四代NVLink：提供 3 倍的带宽提升：总带宽为 900 GB/s，是 PCIe Gen 5 的 7 倍。

第三代NVLink Switch，最多可连接 32 个节点或 256 个 GPU。

提供 128 GB/s 的总带宽：每个方向 64 GB/s，是 PCIe Gen 4 的两倍

SM提供256 KB 的共享内存和 L1 数据缓存，允许直接 SM 间通信，用于加载、存储和原子操作，跨越多个 SM 共享内存块，引入TMA。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Af3d633aa-2607-49ff-a91d-a837b4a33948%3Aimage.png?table=block&id=1c86a624-db43-80c4-af2d-d83f6875f2c3&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1190&userId=&cache=v2)

在第 4 代 Tensor Core 中，一个显著的创新是引入了 Tensor Memory Accelerator（TMA），这一功能被称为增量内存加速。这一硬件化的数据异步加载机制使得全局内存的数据能够更为高效地异步加载到共享内存，进而供寄存器进行读写操作。传统的 Warp-Level 编程模式要求所有线程都参与数据搬运和计算过程，这不仅消耗了大量的资源，还限制了计算规模的可扩展性。而单线程 schedule 模型则打破了这一束缚，它允许 Tensor Core 在不需要所有线程参与的情况下进行运算。这种设计大大减少了线程间的同步和协调开销，提高了计算效率。

H100最适合：训练和推理非常大的模型（700 亿参数及以上），基于 Transformer 的架构，低精度（8 位）推理。关键特性：

截至2024年底在售的最强大的 NVIDIA 数据中心 GPU。

大多数工作负载比 A100 快约两倍，但更难获取且价格更高。

优化用于大型语言模型任务，提供超过 3 TB/s 的内存带宽，对于需要快速数据传输的 LLM 推理任务至关重要。

包含专门用于低精度（FP8）操作的计算单元。

参考：https://resources.nvidia.com/en-us-tensor-core

### Blackwell

目前据说已经推迟到2025上半年商业化，已经跳票好多次，目前官方还没有发布详细的白皮书信息。以下信息来自官方Breif说明。

- 新型 AI 超级芯片：Blackwell 架构 GPU 具有 2080 亿个晶体管，采用专门定制的台积电 4NP 工艺制造。所有 Blackwell 产品均采用双倍光刻极限尺寸的裸片，通过 10 TB/s 的片间互联技术连接成一块统一的 GPU。
- 第二代 Transformer 引擎：将定制的 Blackwell Tensor Core 技术与英伟达 TensorRT-LLM 和 NeMo 框架创新相结合，加速大语言模型 (LLM) 和专家混合模型 (MoE) 的推理和训练。
- NVLink 5.0：为了加速万亿参数和混合专家模型的性能，新一代 NVLink 为每个 GPU 提供 1.8TB/s 双向带宽，支持多达 576 个 GPU 间的无缝高速通信，适用于复杂大语言模型。
- RAS 引擎：Blackwell 通过专用的可靠性、可用性和可服务性 (RAS) 引擎增加了智能恢复能力，以识别早期可能发生的潜在故障，从而更大限度地减少停机时间。
- 安全 AI：内置英伟达机密计算技术，可通过基于硬件的强大安全性保护敏感数据和 AI 模型，使其免遭未经授权的访问。
- 解压缩引擎：拥有解压缩引擎以及通过 900GB/s 双向带宽的高速链路访问英伟达 Grace CPU 中大量内存的能力，可加速整个数据库查询工作流，从而在数据分析和数据科学方面实现更高性能。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A71d63d9f-53df-4a87-a887-f390638af9dc%3Aimage.png?table=block&id=1c86a624-db43-8063-8035-e978efa67412&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

参考：

https://resources.nvidia.com/en-us-blackwell-architecture

## NVLink和NVSwitch

随着迈入大模型主导的时代，训练这些复杂的大型模型绝非易事，不仅因为需要耗费大量的GPU资源和时间成本，还因为单个GPU的内存容量有限，无法独自承载许多大型模型的数据量。为了解决这一挑战，业界转向了多GPU协作训练的方式，即分布式计算，分布式通信的概念是将多个计算单元（如服务器或GPU）互联，让它们能够协同工作以完成一个共同的任务。这种方式依赖于节点间的高效通信机制。

PCI express每一代带宽都是前一代的2倍，PCIE gen 5 x16 即为64GB/s，而且目前能生产PCIE gen5 和gen6的厂商全球仅有2-3家，产能极其有限。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A8924c394-0f6e-4414-85d1-128ed5178cd0%3Aimage.png?table=block&id=1c86a624-db43-806c-b980-f24ea71ff251&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

H100的32位浮点计算能力为67TFLOPS，如果每个浮点都是用来从GPU外搬运数据，而不是复用旧的数据，则需要67 * 10 ^ 3 * 32 Gbps的带宽，268000GB/s。显然这个带宽远远超过了PCIE的能力，同时由于客户复用数据的水准是有一定瓶颈的，那么为了避免更多的计算单元闲置浪费，就需要更多的带宽去做数据处理。

NVLink 是双向直接 GPU-GPU 互连，NVLink 5.0连接主机和加速处理器的速度高达每秒 1800GB/s，这是传统 x86 服务器的互连通道——PCIe 5.0 带宽的 14 倍多。英伟达 NVLink-C2C 还将 Grace CPU 和 Hopper GPU 进行连接，加速异构系统可为数万亿和数万亿参数的 AI 模型提供加速性能。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A707750f7-77d8-400e-812f-17007d148800%3Aimage.png?table=block&id=1c86a624-db43-802f-a670-d586b2abc916&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

NVLink代表了一种前沿的互联标准，它不仅包括总线设计，还涵盖了通信协议，旨在优化CPU与GPU之间，以及多个GPU之间的直接连接。通过采用点对点架构和串行传输技术，NVLink能够提供比传统接口更高效的数据交换能力。而NVSwitch则进一步推进了这一技术，它是一种专为高性能计算设计的高速互连解决方案。作为一款独立的NVLink芯片，NVSwitch能够支持多达18个NVLink连接，从而实现多GPU配置中的极速数据流通，极大地促进了复杂计算任务的并行处理效率。

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3Ad2423c70-0c02-4449-89bd-c8dc037b44bf%3Aimage.png?table=block&id=1c86a624-db43-809b-85bc-fed4946cbf31&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

利用NVSwitch，可以打破GPU2GPU的单点链接实现full mesh的GPU全项链接，详情请见【2.4 Ampere】

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A07d63f27-127d-4047-8da3-99328e188f21%3Aimage.png?table=block&id=1c86a624-db43-8072-a183-f582caa0a485&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

如上图所示，在没有 NVSwitch 的配置中，GPU 之间的连接是通过 NVLinks 来实现的，而不同GPU之间的NVLink的数量是不一样的。所以任意两个 GPU 之间的最大带宽受限于他们之间的 NVLink 数量和带宽。但是利用NVSwitch，可以打破GPU2GPU的单点链接实现full mesh的GPU全项链接，实现任意GPU间的一致性性。详情请见【2.2.4 Ampere】

参考：https://www.nvidia.cn/data-center/nvlink/

## CUDA

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A0b363bb9-10cd-4fd3-8378-f235eb38bf3f%3Aimage.png?table=block&id=1c86a624-db43-80b7-91ed-c11cb40da2ce&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1420&userId=&cache=v2)

GPU资源使用涉及两个方面：cuda river和cuda toolkit（runtime和libraries）。程序调用GPU资源其实是调用cuda toolkit，具体底层GPU资源的利用其实是由cuda river去驱动。（不恰当的比喻：可以把cuda driver当作contained，toolkit是kubelet。pod创建其实是发信号给kubelet，具体pod如何创建出来是由containered去实现的）

![image.png](https://speckled-amber-aa6.notion.site/image/attachment%3A9e400fba-97c2-446a-b68d-cdced1c56701%3Aimage.png?table=block&id=1c86a624-db43-801c-8351-ee9465992f81&spaceId=bd920ac3-a269-416a-b879-0fea0c915514&width=1340&userId=&cache=v2)

这里其实细分可以化成3个层级概念：

CUDA Toolkit: 这里是面向开发者、应用程序暴露的直接调用GPU能力的runtime，libraries

CUDA user-mode driver:用户态cuda驱动

CUDA kernel-mode driver:内核态驱动

所以这里涉及几个需要关注的部分：GPU卡所能支持的CUDA Driver的版本， 以及CUDA Driver版本和CUDA Toolkit的兼容性：

GPU卡和CUDA Driver版本的信息表

Download The Latest Official NVIDIA Drivers：https://www.nvidia.com/en-us/drivers/

Toolkit和Nvidia驱动版本兼容表

https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html

CUDA12.5 Update 1 Release Notes

参考：1. Why CUDA Compatibility — CUDA Compatibility r555 documentationhttps://docs.nvidia.com/deploy/cuda-compatibility/
