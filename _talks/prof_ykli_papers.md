---
title: "李永坤老师的论文阅读笔记"
collection: talks
type: "Paper Reading"
permalink: /talks/prof_ykli_papers
venue: "MUC, Beijing"
date: 2024-09-18
location: "Beijing"
---


李老师的著作：真的太牛了

1.    Tao Li, Yongkun Li*, Wenzhe Zhu, Yinlong Xu and John C.S. Lui. MinFlow: High-performance and Cost-efficient Data Passing for I/O-intensive Stateful Serverless Analytics. The 22nd USENIX Conference on File and Storage Technologies (FAST 2024), Santa Clara, CA, USA, February 27–29, 2024.

2.    Wenzhe Zhu, Yongkun Li*, Erci Xu, Fei Li, Yinlong Xu, John C.S. Lui. DiffForward: On Balancing Forwarding Traffic for Modern Cloud Block Services via Differentiated Forwarding.ACM (SIGMETRICS 2023), Orlando, Florida, USA, June 19-23, 2023.

3.    Yipeng Xing, Yongkun Li*, Zhiqiang Wang, Yinlong Xu, John C.S. Lui. LightTraffic: On Optimizing CPU-GPU Data Traffic for Efficient Large-scale Random Walks. EEE International Conference on Data Engineering (ICDE 2023), Anaheim, California, USA, April 3-7, 2023.

4.    Qiang Zhang, Yongkun Li*, Patrick P. C. Lee, Yinlong Xu, and Si Wu. DEPART: Replica Decoupling for Distributed Key-Value Storage. Proceedings of the 20th USENIX Conference on File and Storage Technologies (FAST 2022), Santa Clara, CA, USA, February 22-24, 2022.

5.    Yongkun Li, Zhen Liu, Patrick P. C. Lee, Jiayu Wu, Yinlong Xu, Yi Wu, Liu Tang, Qi Liu, and Qiu Cui. Differentiated Key-Value Storage Management for Balanced I/O Performance. Proceedings of the 2021 USENIX Annual Technical Conference (ATC 2021), July 14-16, 2021.

6.    Rui Wang, Yongkun Li*, Hong Xie, Yinlong Xu, John C.S. Lui. GraphWalker: An I/O-Efficient and Resource-Friendly Graph Analytic System for Fast and Scalable Random Walks. Proceedings of the 2020 USENIX Annual Technical Conference (ATC 2020), Boston, MA, USA, July 2020.

7.    Zhang Qiang, Yongkun Li*, Patrick P. C. Lee, Yinlong Xu, Qiu Cui, Liu Tang. UniKV: Toward High-Performance and Scalable KV Storage in Mixed Workloads via Unified Indexing. Proceedings of the 36th IEEE International Conference on Data Engineering (ICDE 2020), Dallas, TX, USA, April 20-24, 2020.

8.    Yongkun Li, Chengjin Tian, Fan Guo, Cheng Li, and Yinlong Xu. ElasticBF: Elastic Bloom Filter with Hotness Awareness for Boosting Read Performance in Large Key-Value Stores. Poceedings of the 2019 USENIX Annual Technical Conference (ATC 2019), RENTON, WA, USA, July 10-12, 2019.

9.    Helen H. W. Chan, Yongkun Li*, Patrick P. C. Lee, and Yinlong Xu. HashKV: Enabling Efficient Updates in KV Storage via Hashing. Proceedings of the 2018 USENIX Annual Technical Conference (ATC 2018), Boston, MA, USA, July 2018.

10.    Fan Guo, Yongkun Li*, Yinlong Xu, Song Jiang, John C. S. Lui. SmartMD: A High Performance Deduplication Engine with Mixed Pages. Proceedings of the USENIX Annual Technical Conference (ATC 2017), Santa Clara, CA, USA, July 12-14, 2017.


从最后一篇开始读吧：

SmartMD: A High Performance Deduplication Engine with Mixed Pages
===

#### 基础知识：

**translation lookaside buffers (TLB)**: 是计算机体系结构中的高速缓存，用来加速虚拟地址到物理地址的转换过程。虚拟内存系统的工作原理是将程序使用的虚拟地址映射到物理地址，而这个映射关系通常存储在页表（page table）中。然而，由于页表可能非常大，因此每次访问内存都需要访问页表，这会带来性能上的开销。TLB 通过缓存最近使用的虚拟地址到物理地址的映射，减少了访问页表的频率，从而提高了内存访问的效率。具体来说，当处理器需要访问内存时，它首先在 TLB 中查找虚拟地址的映射。如果找到（称为 TLB 命中），处理器可以立即获得物理地址并访问相应的内存位置。如果未找到（称为 TLB 丢失），则需要访问页表以获取映射，并将该映射存储到 TLB 中以便后续使用。

TLB 的主要特点：

    性能提升：通过缓存页表中的部分映射，减少了频繁访问内存和页表的开销。
    TLB 命中：在 TLB 中找到虚拟地址对应的物理地址。
    TLB 丢失：在 TLB 中未找到映射，此时需要查阅页表并更新 TLB。

TLB 的工作流程：

    处理器生成虚拟地址并查找 TLB。
    如果 TLB 命中，直接获得物理地址并访问内存。
    如果 TLB 丢失，访问页表获取映射，并将其插入 TLB 中。
    重试内存访问操作。
处理器在访问内存时首先生成虚拟地址，主要是因为现代操作系统使用了虚拟内存（Virtual Memory）技术。虚拟内存提供了一种抽象，使得每个进程能够以统一的方式访问大量的内存地址空间，而无需直接与底层的物理内存进行交互。接下来，我将详细解释为什么处理器首先生成虚拟地址，并虚拟内存技术的必要性。【**内存里面存储了系统内核、关键数据结构、数据、页表等内容，页表主要是将虚拟内存地址映射为在内存中的物理地址，TLB是为了加速这一过程**】

**Aggressive deduplication approach (ADA)**, which aggressively splits large pages (e.g., 2M-pages) to base pages (e.g., 4K-pages) and performs deduplication among base pages
### 论文核心
由于TLB出现命中失败时，需要查找页表。因此越大的页表（比如2M），能够减少查找页表的次数。然而大页表会导致去重效率低，内存利用率不高。因此文章提出了一种混合式页表的方法，称为SmartMD(Smart Memory Deduplciation)。具体做法就是监督页的状态，自适应的将多个页合并为一个大页，或者将一个大页拆分为多个小页。
The main idea is to split cold large pages with high repetition rate to save memory space, and at the same time, to reconstruct split large pages when they become hot to improve memory access performance.

**关键内容：** ADA 划分为base page之后，TLB也要相应的更新。比如TLB也是个表，其总行数可以视为固定的，因此细粒度存储的base page，导致TLB的覆盖范围更少了。如果将split page再reconstruct 为一个large，目前的OS不支持。


**主要难点：**how to efficiently monitor repetition rate and access frequency of pages, and how to dynamically conduct conversions between large pages and base pages so as to achieve both high deduplication rate and high memory access performance.


Serverless
===
在使用Serverless架构进行数据处理时，尤其是在涉及到“shuffle”操作的情况下，数据传输的问题。在这种场景中，每个函数的输出需要传递给下一个 stage 的所有函数，这导致了大量的数据传输请求。下面是对这个情况的详细解释：

1. 函数的并行性与数据传递
在 Serverless 架构中，每个 stage 可以包含多个并行执行的函数。在你的例子中，一个 stage 有 500 个函数。这种架构的一个关键特点是它的无状态性，即每个函数执行时不会保留之前状态的任何信息，因此所有必要的数据都需要通过网络从其他地方获取。

2. Shuffle 操作的数据传输需求
在数据处理的 shuffle 阶段，每个函数的输出不仅仅是传递给下一个 stage 的一个函数，而是要传递给下一个 stage 的所有函数。这种类型的数据传递是全连接的（all-to-all connectivity），即每个函数的输出都需要被下一个 stage 的每个函数获取。

3. 计算 PUT 和 GET 请求次数
PUT 请求次数：如果 stage A 有 500 个函数，每个函数都需要将其输出结果存储起来供其他函数访问。因为每个函数的输出需要被下一个 stage 的所有 500 个函数访问，所以每个输出都要被 PUT 到一个可以被这些函数访问的地方（比如 S3）。因此，每个函数都执行了 500 次 PUT 操作（每次 PUT 操作对应一个接收者函数），总共是 500 × 500 = 250,000 次 PUT。
GET 请求次数：同理，每个函数都需要获取其他所有 499 个函数（假设自己的输出不需要自己再次获取）的输出，也就会执行 500 次 GET 请求，总共也是 500 × 500 = 250,000 次 GET。
4. 对 Serverless 平台的影响
由于每个函数都依赖于远程对象存储（如 S3）来传递数据，因此大量的 PUT 和 GET 请求可能会导致达到这些服务的请求率上限。当请求量超过 S3 允许的限制时，会导致延迟增加，从而影响整个数据处理过程的效率和响应时间。

总结
在这种高并行度和全连接数据传输需求的场景中，数据传输请求（PUT 和 GET）的数量会非常巨大，这可能会超过远程数据存储服务的处理能力，从而成为系统性能的瓶颈。这也突出了在设计 Serverless 数据处理架构时，需要考虑如何优化数据传输和存储访问模式的重要性。

Shuffle 操作的目的
在数据处理和分布式计算中，shuffle 操作的主要目的是重新分配数据，使得数据可以根据某些特定的键（如分组键、排序键等）跨多个处理节点进行重新组合。这通常是为了执行某些需要广泛数据交换的操作，如汇总、排序或连接不同数据集。

Shuffle 是连接不同计算阶段（stages）的桥梁，确保数据按照处理逻辑被正确分配到下一个阶段的每个函数或任务。