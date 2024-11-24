---
title: "Remap-SSD的论文阅读笔记"
collection: talks
type: "Paper Reading"
permalink: /talks/remap-ssd
venue: "MUC, Beijing"
date: 2024-11-23
location: "Beijing"
---

[https://www.usenix.org/system/files/fast21-zhou.pdf](https://www.usenix.org/system/files/fast21-zhou.pdf)

### SSD 的数据写入和修改机制
1. **不可直接覆盖写：**
    - NAND 闪存的一个物理块（Block）包含多个页面（Page）。
    - 页面只能从未使用（空白）状态写入一次，不能直接覆盖已有数据。
    - 如果要更新某个页面的数据，必须先将整个块擦除，再重新写入。
2. **写入新块：**
    - 当数据需要更新时，SSD 不会在原页面上写入，而是将新数据写入一个空闲页面。
    - 然后，SSD 更新逻辑到物理地址的映射表（L2P），使逻辑地址指向新页面。
3. **标记旧数据为无效：**
    - 原来的页面被标记为 "无效" 数据，等待垃圾回收（Garbage Collection，GC）时清理。

### **什么是dupicate write？**
**Duplicate write** 指的是在存储系统中，数据由于某些原因被写入了多个副本，从而导致冗余数据的现象。虽然有些场景（如数据库的主从复制或日志同步）是有意进行重复写入，但在某些情况下，它可能是由于错误或设计问题引起的，这种情况通常会导致性能问题、资源浪费或数据一致性问题。

> 示例：假设一个分布式数据库使用多副本机制来保证高可用性和数据可靠性。当一个客户端向数据库写入数据时，主节点会将数据同步到多个从节点。但如果由于网络抖动或超时问题，主节点未正确收到从节点的确认，可能会认为写入失败并重新尝试。这种情况下，从节点可能会多次接收到相同的数据写入请求，导致重复写。
>

#### 解决办法：
+ **幂等性设计**：确保每次写操作具有幂等性，即同一个写请求无论执行多少次，结果都保持一致。
+ **请求去重**：在写入请求中附加唯一标识（如事务 ID 或请求 ID），使系统能够识别并忽略重复的写入请求。
+ **分布式事务**：利用分布式锁或一致性协议（如 Paxos、Raft）来确保每次写操作的唯一性。

<font style="background-color:#D9EAFC;">论文中的说法：</font>

<font style="background-color:#D9EAFC;">a. Data Deduplication</font>

<font style="background-color:#D9EAFC;">这是因为数据去重可以直接修改Remap（L2P）指向已经存储的地址，而不需要真正地写入到flash page中</font>

b. Address Remapping

可以通过重映射来解决

但是存在问题：L2P是存储在DRAM中的，支持in-place edit，但是相应的P2L存储在flash memory中，不支持in-place edit。一旦断电，L2P会丢失。需要P2L来恢复。（为什么不直接将L2P存储在flash memory中？一样的，也会存在inconsistency的问题）

目前已有解决方案分类

P/U-type and D/N-type

P就是Remapping的key列表的大小是确定的，U就是不确定的

D就是在物理地址写入之前，LPNs和PPNs都确定了，N就是不确定

| 特性 | D-type（确定性映射） | N-type（非确定性映射） |
| --- | --- | --- |
| **映射时间** | 映射关系在 PPN 写入时已确定 | 映射关系在 PPN 写入时尚未确定 |
| **灵活性** | 固定映射，灵活性较低 | 映射延迟决策，灵活性较高 |
| **复杂性** | 简化了未来的操作（如读写或重新映射） | 未来可能需要额外的计算和元数据来解析映射关系 |
| **典型应用** | 写前日志（WAL）、传统数据库更新机制 | 日志结构化存储、垃圾回收机制动态决定映射关系 |


> 为什么U/D是不可能的？
>

### 什么是写放大？
**固态硬盘（SSD）中的写放大（Write Amplification）** 是在存储系统中描述的一种现象，指的是实际写入到底层存储设备的数据量远远大于用户写入的数据量。

> 由于 NAND 闪存的特性，写操作需要以整个擦除块（通常为几 MB）的粒度进行。即使用户只更新了几个字节的数据，SSD 可能需要读取整个块的数据、修改其中一部分并重写整个块。
>

# <font style="color:rgb(0, 0, 0);">Log-Structured 结构</font>
**Log-Structured 结构**（日志结构化存储，或称为日志结构化文件系统）是一种数据结构和存储设计理念，最初用于提高磁盘存储的性能，尤其在写入密集型的场景中具有显著优势。它的核心思想是将所有的数据操作（包括写入和更新）都按时间顺序追加到一个日志（或叫写入日志）中，而不是像传统文件系统那样直接修改数据块或覆盖原有的数据。这种方式常用于文件系统、数据库、存储系统等。

+ **写入操作**：所有数据写入都会被追加到日志的末尾。在数据库中，插入、更新、删除操作通常都会生成日志条目，表示数据的变更。这种方法的一个优势是能够很高效地进行顺序写入，避免了磁盘寻址和定位的开销。
+ **更新操作**：当数据需要更新时，不会直接修改原来的位置，而是写入一个新的日志条目，包含更新后的数据。只有在垃圾回收时，系统才会将过期的旧数据清理掉，回收空间。
+ **空间回收与整理（Compaction）**：随着时间的推移，日志会包含大量过时或无效的条目。为了防止存储空间被占满并且访问效率下降，系统会定期进行垃圾回收或整理（Compaction）操作，合并和清理日志中的无效数据。

Log-Structured 结构的最大优点是 **顺序写入**，使得写入操作变得非常高效。顺序写入大大降低了磁盘寻址的开销，尤其适用于存储介质（如硬盘和 SSD）中，顺序写入比随机写入更高效。

**缺点：**

+ **空间浪费与垃圾回收**：由于数据不会直接覆盖，随着时间的推移，日志中会有许多过期数据，这会导致空间浪费。因此，系统必须定期进行垃圾回收操作。垃圾回收会消耗一定的计算资源，并且如果操作不当，会导致性能波动。
+ **读取性能**：由于数据通常是追加写入的，读取操作可能需要检索多个日志条目，特别是对于更新频繁的数据。为了获取最新的数据，系统可能需要进行多次读取操作，从而影响读取性能。
+ **维护成本**：日志结构需要定期的整理（Compaction）操作来清理无效数据，这会增加系统的维护成本。在高并发的环境下，这些整理操作可能会导致 I/O 阻塞和性能下降。

Compaction介绍：

假如要设计一个统计B站视频观看次数的程序，一个简单的方式就是用一个k-v字典去技术，k为视频的哈希值，v为相应的观看次数。如果使用简单的数据库则需要考虑在并行处理情况下的各种情况。

> 这里解释一下数据库的三种情况：
>
> SQL 标准定义了 4 种事务隔离级别，从低到高依次是：
>
> 1. **Read Uncommitted（读未提交）**
> 2. **Read Committed（读已提交）**
> 3. **Repeatable Read（可重复读）**
> 4. **Serializable（可串行化）**
>

如果采用Log-Structure（假设A和B分别为不同视频的哈希值，每条记录只在末尾追加，可以视为存储在一个无限长度的数组中）：

![](https://cdn.nlark.com/yuque/0/2024/png/26280077/1731755358654-baf7de0a-1d36-4a55-aae6-0b383adb949f.png)

这时存在以下几个问题：

+ <font style="color:rgb(51, 51, 51);">一个数组不可能在内存中无限地增长下去，我们要如何处理呢？</font>
+ <font style="color:rgb(51, 51, 51);">如果每次想要知道结果，就必须遍历一遍这样的数组，时间复杂度会非常高，那该怎么优化呢？</font>
+ <font style="color:rgb(51, 51, 51);">平衡树是如何被应用在里面的呢？</font>

问题1的解决：

将数据分块（segment or block)，每个块是固定长度的。一旦一个数组中的大小大于设定的长度，则新建一个新的一维数组存储数据：

![](https://cdn.nlark.com/yuque/0/2024/png/26280077/1731755665387-dd77e405-6a03-4bf4-8080-bff624a94382.png)

问题2的解决：

在将数据分块之后，时间复杂度仍然很高，这时我们可以通过compaction将已存满的segment中的内容进行合并：

![](https://cdn.nlark.com/yuque/0/2024/png/26280077/1731755782489-765ac2a5-5c19-46f3-b311-07a6e08cb8f9.png)

这是无论是存储还是读取效率又提高了（但是合并带来的cpu开销也需要注意），如何进行快速的compaction呢？需要使用SSTable和LSM Tree进行优化

### SSTable & LSM Tree
<font style="color:rgb(51, 51, 51);">SSTable（Sorted String Table）数据结构是在 Log-Structured 结构的基础上，多加了一条规则，就是所有保存在 Log-Structured 结构里的数据都是键值对，并且键必须是字符串，在经过了 Compaction 操作之后，所有的 Compacted Segment 里保存的键值对都必须按照字符排序。</font>

<font style="color:rgb(51, 51, 51);">我们假设现在想利用 Log-Structured 结构来保存一本书里的词频，为了方便说明，把 Segment 的大小设为 4。在刚开始的时候，这个 Log-Structured 结构在经过了 Compaction 操作之后，内存图会变成如下图所示：</font>

![](https://cdn.nlark.com/yuque/0/2024/png/26280077/1731756321157-ecd97ab8-6510-4af3-b232-c59cf2cf8897.png)  
  
<font style="color:rgb(51, 51, 51);">可以看到，所有的 Compacted Segment 都是按照字符串排序的。当我们要查找一个单词出现的次数时，可以去遍历所有的 Compacted Segment，来看看这个单词的词频，当然了，因为所有数据都是按照字符串排好序的，如果当遍历到的字符串已经大于我们要找的字符串时，就表示并没有出现过这个单词。</font>

<font style="color:rgb(51, 51, 51);">这时候你可能会有一个疑问，Log-Structured 结构是指不停地将新数据加入到 Segment 的结尾，像这种 Compaction 的时候将字符串排序应该怎么做呢？此时我们就需要上一讲中所讲到的平衡树了。</font>

<font style="color:rgb(51, 51, 51);">我们先来复习一下二叉查找树里的一个特性：二叉查找树的任意一个节点都比它的左子树所有节点大，同时比右子树所有节点小，说到这里你是不是有点恍然大悟了。</font>

<font style="color:rgb(51, 51, 51);">如果我们将所有 Log-Structured 结构里的数据都保存在一个二叉查找树里，当写入数据时其实是按照任意顺序写入的，而当读取数据时是按照二叉查找树的中序遍历来访问数据的，其实就相当于按字符串顺序读取了。</font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);">在业界上，我们为了维护数据结构读取的高效，一般都会维护一个平衡树，比如，在上一讲中说到的红黑树或者 AVL 树。</font>

<font style="color:rgb(51, 51, 51);">而这样一个平衡树在 Log-Structured 结构里通常被称为 memtable。</font>

<font style="color:rgb(51, 51, 51);">而上面所讲到的概念，通过内部维护平衡树来进行 Log-Structured 结构的 Compaction 优化，这样一种数据结构被称为是 LSM 树（Log-Structured Merge-Tree），它是由 Patrick O'Neil 等人在 1996 年所提出的。</font>

### LSM Tree的应用
<font style="color:rgb(51, 51, 51);">在数据库里面，有一项功能叫做 Range Query，用于查询在一个下界和上界之间的数据，比如，查找时间戳在 A 到 B 之内的所有数据。许多著名的数据库系统，像是 HBase、SQLite 和 MongoDB，它们的底层索引因为采用了 LSM 树，所以可以很快地定位到一个范围。</font>

<font style="color:rgb(51, 51, 51);">比如，如果内存里保存有以下的 Compacted Segments：</font>

![](https://cdn.nlark.com/yuque/0/2024/png/26280077/1731756391833-47c2bd8b-d206-4a05-aeb8-c823c3b422d1.png)

<font style="color:rgb(51, 51, 51);">  
</font><font style="color:rgb(51, 51, 51);">如果我们的查询是需要找出所有从 Home 到 No 的数据，那我们就知道，可以从 Compacted Segment 2 到 Compacted Segment 3 里面去寻找这些数据了。</font>

<font style="color:rgb(51, 51, 51);">同样的，采用 Lucene 作为后台索引引擎的开源搜索框架，</font><font style="color:rgb(255, 0, 0);">像 ElasticSearch 和 Solr，底层其实运用了 LSM 树。</font>

<font style="color:rgb(51, 51, 51);">因为搜索引擎的特殊性，有可能会遇到一些情况，那就是：所搜索的词并不在保存的数据里，而想要知道一个数据是否存在 Segment 里面，必须遍历一次整个 Segment，时间开销还并不是最优化的，所以这两个搜索引擎除了采用 LSM 树之外，还会利用 </font><font style="color:rgb(255, 0, 0);">Bloom Filter</font><font style="color:rgb(51, 51, 51);"> 这个数据结构，它可以用来判断一个词是否一定不在保存的数据里面。</font>

### **数据库中常见的现象**
#### **脏读（Dirty Read）**
+ **定义**：一个事务可以读取到另一个事务尚未提交的数据。如果另一个事务随后回滚了，这些被读取的数据就变成了无效数据。
+ **示例**：
    1. 事务 A 修改了某条记录，但尚未提交。
    2. 事务 B 读取了这条未提交的数据。
    3. 如果事务 A 随后回滚，事务 B 所读取的数据就是脏数据。
+ **隔离级别**：仅在 **Read Uncommitted（读未提交）** 隔离级别下可能发生。

---

#### **不可重复读（Non-Repeatable Read）**
+ **定义**：在一个事务内，多次读取同一条记录时，数据内容不一致，可能是由于另一个事务修改并提交了该记录。
+ **示例**：
    1. 事务 A 读取某条记录，值为 `X`。
    2. 事务 B 修改了该记录的值并提交，值变成 `Y`。
    3. 事务 A 再次读取时，值变成了 `Y`，与第一次读取不一致。
+ **隔离级别**：在 **Read Committed（读已提交）** 隔离级别下可能发生。**Repeatable Read（可重复读）** 可以防止。

---

#### **幻读（Phantom Read）**
+ **定义**：一个事务内多次查询时，结果集的行数发生变化。通常是因为另一个事务插入或删除了满足查询条件的新记录。
+ **示例**：
    1. 事务 A 查询 `WHERE salary > 1000`，结果有 5 条记录。
    2. 事务 B 插入了一条满足条件的新记录并提交。
    3. 事务 A 再次查询时，结果变成了 6 条记录。
+ **隔离级别**：在 **Repeatable Read（可重复读）** 隔离级别下可能发生。**Serializable（可串行化）** 可以防止。

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| --- | --- | --- | --- |


| **Read Uncommitted** | 可能 | 可能 | 可能 |
| --- | --- | --- | --- |


| **Read Committed** | 不可能 | 可能 | 可能 |
| --- | --- | --- | --- |


| **Repeatable Read** | 不可能 | 不可能 | 可能 |
| --- | --- | --- | --- |


| **Serializable** | 不可能 | 不可能 | 不可能 |
| --- | --- | --- | --- |






# 问题
1. <font style="background-color:#D9DFFC;">为什么要有P2L</font>

> 比如逻辑页 L2 原来在物理地址 P9，但由于GC操作，L2 的数据被搬迁到了新的物理地址 P15。为了确保数据一致性，系统需要P2L映射来找到 P9 对应的逻辑页 L2，并更新为新的物理地址 P15。
>

2. 为什么L2P改变之后，P2L不能进行相应的变化？

> 这里说GC之后L2P会改变
>

3. 以前的解决方式（将P2L使用log-structured manner存储）会导致log日志比较大

> 现在spark等大数据架构都是采用这种方式来处理的吧，可以使用上面所说的combine来不定时地合并一些数据，减少log文件大小，<font style="background-color:#CEF5F7;">为什么原文中还说这个会导致Log比较大，只能限制log文件大小呢？</font>
>

4. 模拟数据读取和写入：

```cpp
#include <iostream>
#include <unordered_map>
#include <vector>
#include <string>

// 模拟每个物理块的数据结构
struct FlashBlock {
    int ppn; // 物理页号
    std::string data; // 存储的数据
};

// L2P映射表
std::unordered_map<int, int> l2pMapping; // 逻辑页号 -> 物理页号

// 存储flash的所有物理块
std::vector<FlashBlock> flashBlocks;

// 模拟写入函数
void writeData(int lpn, const std::string& data) {
    int ppn = flashBlocks.size(); // 新的数据写入一个新的物理页
    flashBlocks.push_back({ppn, data});
    l2pMapping[lpn] = ppn; // 更新L2P映射表
    std::cout << "Writing LPN " << lpn << " to PPN " << ppn << " with data: " << data << "\n";
}

// 模拟读取函数
void readData(int lpn) {
    if (l2pMapping.find(lpn) != l2pMapping.end()) {
        int ppn = l2pMapping[lpn];
        std::cout << "Reading LPN " << lpn << " from PPN " << ppn << " with data: " << flashBlocks[ppn].data << "\n";
    } else {
        std::cout << "LPN " << lpn << " not found in L2P mapping table." << "\n";
    }
}

int main() {
    // 模拟写入4个逻辑页的数据
    writeData(1, "Data for LPN 1");
    writeData(2, "Data for LPN 2");
    writeData(3, "Data for LPN 3");
    writeData(4, "Data for LPN 4");

    std::cout << "\n";

    // 模拟读取逻辑页的数据
    readData(1);
    readData(2);
    readData(3);
    readData(4);

    return 0;
}

```

5. PebbleSSD 也提出了基于 NVRAM 的框架，但是原文中说OOB的大小是固定的，所以只适合P类型。那么Remap-SSD也没能解决这个问题吧？
6. 为什么不能使用 dedicated log （这个才是log-structure storage吧）的形式去持久化P2L呢？
这个其实很好解释，log-structure storage需要combing等，日志比较大
7. 为什么说基于NVRAM存储RMM的形式也是 log-structure storage？不应该是write-ahead logging （WAL）吗？因为此处的log并不是真正的数据，而是产生的日志文件，用来恢复P2L，并保证原子性的
8. 为什么RMM不能存储在DRAM中，主要原因不是因为断电恢复吗？为什么原文中会和cache扯上关系呢？他的目的本身就是持久化
9. RMM的 invalidate 的逻辑没有详细说明，以及 Segment 的回收也没有说明，感觉这两个点比较关键

# 设计部分
![](https://cdn.nlark.com/yuque/0/2024/png/26280077/1732370075208-1b5d0508-2437-4532-80ff-7ae1854f2b22.png)

<font style="background-color:#FBF5CB;">使用了NVRAM来保存（那么1 OOB容量是固定的，对于U-type这种不知道多大的怎么办？ 2 存储空间的利用率可能很低，因为为每个flash page都采用固定大小的容量，一些可能用不上）</font>

关键点：(Remapping metadata）RMM

<font style="background-color:#D9EAFC;">GC以及Data relocate是以superblock为单位的</font>

（难道意思是之前的Log文件需要逐行扫描，但是这种就类似于Timescale时序数据一样，可以直接定位到对应的entry？）

数<font style="background-color:#CEF5F7;">据迁移之前，难道最新的数据不应该在L2P中吗？（relocation是先将physical data 迁移到新的superblock中，然后去找之前P2L对应的逻辑编号，然后修改L2P）</font>

<font style="background-color:#CEF5F7;">relocation和remap不同</font>

<font style="background-color:#CEF5F7;">relocation，或者叫数据迁移，是为了write （GC）</font>

<font style="background-color:#CEF5F7;">remap是为了防止duplicate write</font>

设计的NVRAM分段处理，每一段只为一个super-block服务。【我认为如果superblock的RMM大小超出一个段，会重新再分配一个，或者执行cleaning or combining】

segment validity bitmap (SV-bitmap) 还有一个flags来指示某个seg是否被使用

<font style="background-color:#FBDFEF;">P2L mapping entry: {LPN, PPN, relocateFlag, SRC-LPN | NULL}</font>

<font style="background-color:#FBDFEF;">LPN和PPN表示对应需要的值，relocateFlag 表示是否需要将之前的逻辑页标记为invalid，如果是，则标记后面的SRC-LPN为invalid「那么为什么需要这个relocateFlag？直接用最后一位是否为NULL来判断不就行了吗？」</font>



RMM需要保证：

1. 内容上
2. 原子性，命令的原子性，以及单个RMM写入的原子性

remap(tgtLPN, srcLPN, length, remapFlag)



![](https://cdn.nlark.com/yuque/0/2024/png/26280077/1732461644433-b8d409c0-a184-4829-8052-025bcaa1446f.png)

上述是RMM的设计（由于目前处理器支持8bytes写入的原子性，所以RMM的大小和8bytes成正比）

元数据：

 1. segment validity bitmap (SV-bitmap) 用来判断segment是否在使用

 2. reference counting table (RC-table) to track the number of references to each flash page. 用来追踪每个flash的被引用的数量，实际上是FTL来管理的，此处remap-ssds使用4bit的计数器「以前都只用1bit?」

> The RC-table (with 4-bit counters) size in Remap-SSD is 640MB, while that (with 1-bit counters) in conventional SSDs is 160MB.
>

 3. LR-bitmap，里面记录了当前的L2P是否由remap操作生成。因为一旦执行了remap，意味着有一个P2L（RMM）就已经失效了，那么FTL就会将对应superblock里面关于segment的无效RMM + 1

> The FTL tracks the number of invalid RMM entries in each segment group. Specifically, a bitmap is used to indicate whether the current L2P mapping of each LPN is established by a remap or write operation, called LR-bitmap.
>

