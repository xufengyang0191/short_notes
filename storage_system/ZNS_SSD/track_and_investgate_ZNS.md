# ZNS SSD 相关的趋势

花了一点时间整理了一些 ZNS 近两年的文章，不涉及实验结果剖析，做一个简单的调查和回顾，追溯最新的研究热点。

论文列表：

1. ZNS: Avoiding the Block Interface Tax for Flash-based SSDs
2. ZNS+: Advanced Zoned Namespace Interface for Supporting In-Storage Zone Compaction
3. Don't be a blockhead: zoned namespaces make work on conventional SSDs obsolete
4. Constant Time Garbage Collection in SSDs
5. Benefits of ZNS in Datacenter Storage Systems
6. A New LSM-style Garbage Collection Scheme for ZNS SSDs
7. Accelerating RocksDB for small-zone ZNS SSDs by parallel I/O mechanism
8. An Efficient Order-Preserving Recovery for F2FS with ZNS SSD
9. Append is Near: Log-based Data Management on ZNS SSDs
10. Compaction-aware zone allocation for LSM based key-value store on ZNS SSDs
11. ConfZNS : A Novel Emulator for Exploring Design Space of ZNS SSDs
12. Efficient Data Placement for Zoned Namespaces (ZNS) SSDs（**<font color=FF0000>NPC'22，CCF-C</font>**）
13. eZNS: An Elastic Zoned Namespace for Commodity ZNS SSDs（**<font color=FF0000>OSDI'23，CCF-A</font>**）
14. Fair-ZNS: Enhancing Fairness in ZNS SSDs through Self-balancing I/O Scheduling
15. Lifetime-leveling LSM-tree compaction for ZNS SSD
16. Performance Characterization of NVMe Flash Devices with Zoned Namespaces (ZNS)
17. Understanding NVMe Zoned Namespace (ZNS) Flash SSD Storage Devices
18. What you can't forget: exploiting parallelism for zoned namespaces
19. ZapRAID: Toward High-Performance RAID for ZNS SSDs via Zone Append
20. RAIZN: Redundant Array of Independent Zoned Namespaces
21. zCeph: Achieving High Performance On Storage System Using Small Zoned ZNS SSD
22. ZenFS+: Nurturing Performance and Isolation to ZenFS

## 12 Efficient Data Placement for Zoned Namespaces (ZNS) SSDs

这篇文章提出 ZNS 的一个问题是在存储分配方面，ZNS SSD 要求插入数据时必须考虑到只能在 zone 内顺序写入的特性，即 ZNS 友好性。 ZNS SSD 的 GC 涉及整个 Zone 的重置，如果 zone 中存在大量有效数据，则会带来较高的数据迁移成本。因此，如何降低 zone 垃圾收集的成本，避免垃圾收集对系统性能稳定性的影响，是 ZNS SSD 数据管理必须考虑的另一个关键问题。

文章针对 ZNS 的写入过程和 GC 提出了一种高效的 ZNS SSD-aware data placement 机制：

- lifetime-based data insertion algorithm：通过划定 key->page->zone 的热度来管理数据写入。
- data lifetime variance-aware zone collection algorithm：通过 zone 中 page 的 lifetime variance 来判断工作负载的数据分布是否改变。

感觉这篇文章没有事先做一个前置实验说明存储分配的影响，同时对 key 的 lifetime 这一点设计存疑，以一个细粒度去映射粗粒度的 lifetime 效果不容易出，并且实验数据是自己生成的。
