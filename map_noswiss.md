# Go Map (noswiss) 实现深度解析

这份文档，旨在从第一性原理出发，层层深入，彻底搞懂Go语言经典`map`（`noswiss`版本）的内部实现。

---

### 1. `map` 的核心使命：为什么需要哈希表？

- **根本问题**: 我们需要一种能够快速存取键值对（Key-Value）的数据结构。
- **数组的优点与局限**: 数组通过下标访问，速度极快，时间复杂度为`O(1)`。但它的`Key`必须是整数，且通常要求是连续的。
- **链表的优点与局限**: 链表可以存储任意`Key`，但查找一个元素，需要从头遍历，时间复杂度为`O(n)`，效率低下。
- **哈希表的诞生**: 为了结合两者的优点，哈希表（Hash Table）应运而生。它通过一个**哈希函数(Hash Function)**，将任意类型的`Key`，转换成一个整数“指纹”（哈希值），然后用这个哈希值，作为数组的下标，去定位数据。理想情况下，这让`map`的存取操作，都能接近`O(1)`的效率。

---

### 2. 冲突的宿命：链表法 vs. 开放寻址法

- **核心矛盾**: 哈希函数，无法保证，对不同的`Key`，一定能生成不同的哈希值。当两个不同的`Key`，经过计算后，落在了数组的同一个下标上时，就发生了**哈希冲突(Hash Collision)**。
- **通用解决方案**: 主要有两种：
    1.  **链表法 (Chaining)**: 在数组的每个槽位（在Go中称为**桶/Bucket**）上，都挂载一个链表。所有冲突的元素，都依次，添加到这个链表中。**Go的`noswiss` map，采用的就是这种方法。**
    2.  **开放寻址法 (Open Addressing)**: 当发现槽位被占用时，通过一种探测算法（线性探测 hash(key) + 1、二次探测等），去寻找数组中的下一个可用槽位。**Go的`swiss` map，采用的是这种思想的变体。**

#### `noswiss` 的基本数据结构

Go的`map`，由一个[hmap](https://github.com/golang/go/blob/release-branch.go1.24/src/runtime/map_noswiss.go#L113-L128)结构体和多个[bmap](https://github.com/golang/go/blob/release-branch.go1.24/src/runtime/map_noswiss.go#L148-L158)（桶）结构体组成。

```mermaid
graph TD
    subgraph hmap
        count[count: 元素数量]
        B[B: 桶数量的对数]
        hash0[hash0: 哈希种子]
        buckets[buckets 指针] --> MainBuckets
        oldbuckets[oldbuckets 指针] --> OldBuckets
        extra[extra 指针] --> OverflowPool
    end

    subgraph MainBuckets [主桶数组]
        B0(Bucket 0)
        B1(Bucket 1)
        B2(...)
    end

    subgraph bmap [桶结构]
        tophash[tophash[8]]
        keys[keys[8]]
        values[values[8]]
        overflow[overflow 指针]
    end

    subgraph OverflowPool [溢出桶内存池]
        O1(Overflow Bucket 1)
        O2(...)
    end

    B1 -- overflow --> O1
```
### 3. 动态的艺术：扩容规则

随着元素不断增加，`map`需要扩容，来保证查找效率。`noswiss` map主要有两种扩容触发条件：

1.  **负载因子 (Load Factor) 过高**
    - **条件**: `map中的元素数量 / 桶的数量 > 6.5`。
    - **行为**: 触发一次**2倍扩容**。桶的数量，会翻倍（`B++`）。这是一个常规的、为了增加容量的扩容。

2.  **溢出桶 (Overflow Buckets) 过多**
    - **你问的“什么叫过多”**: 当`map`中的溢出桶数量，约等于主桶数量时，就会被判定为“过多”。
    - **精确条件**:
        - 当`B < 16`时，`noverflow`精确计数，条件是 `noverflow >= (1 << B)`。
        - 当`B >= 16`时，`noverflow`概率性计数，当计数值也达到阈值时触发。
    - **行为**: 触发一次**同尺寸扩容 (same-size grow)**。桶的数量，**保持不变**，但Go会更换一个哈希种子，并将所有元素**重新散列(Rehash)**一遍。这相当于一次“**洗牌**”，目的是为了消除因哈希分布不佳，而导致的长溢出链，以及清理“墓碑”，优化内存。

#### 渐进式扩容 (Incremental Growing)

无论是哪种扩容，Go都不会，一次性，把所有数据，从旧桶(`oldbuckets`)搬到新桶(`buckets`)，因为这会导致，一次非常耗时的全局暂停。相反，它采用“渐进式”的策略：

- **时机**: 只在**写操作**（增、删、改）时，才触发搬迁工作。
- **双重机制**: 每次写操作，会触发**两种**搬运动作：
    1.  **机会主义搬运**: 搬运本次写操作，随机命中的那个旧桶。
    2.  **顺序保底搬运**: 按照`h.nevacuate`计数器的顺序，额外再搬运一个旧桶。
- **目的**: 这种“随机触发 + 顺序保底”的机制，确保了，即使在最坏的情况下，搬迁工作，也能稳步地、线性地，向前推进，并最终完成。

---

### 4. `map` 的核心操作：查找与删除

#### 查找 (`mapaccess`)

查找一个`key`，是一个充满细节的、“先决策，后查找”的过程。

1.  **决策阶段 (处理扩容)**: 在查找循环开始前，函数会先做出一系列宏观决策：
    - `map`在扩容吗？(`h.oldbuckets != nil`)
    - 是哪种扩容？([h.sameSizeGrow()](https://github.com/golang/go/blob/release-branch.go1.24/src/runtime/map_noswiss.go#L1185-L1188))
    - 我要找的`key`，对应的旧桶，搬了没有？(`!evacuated(oldb)`)
    - **最终决策**: 本次查找，是去`oldbuckets`里找，还是去`buckets`里找？

2.  **查找阶段 (两阶段查找)**: 在确定了目标表（新表或旧表）后，开始遍历桶和溢出链：
    - **第一阶段：[tophash](cci:1://file:///usr/local/go/src/runtime/map_noswiss.go:193:0-200:1)快速筛选**: [tophash](https://github.com/golang/go/blob/release-branch.go1.24/src/runtime/map_noswiss.go#L193-L200)存储了`key`哈希值的高8位。在一个桶内，可以一次性，在8个[tophash](https://github.com/golang/go/blob/release-branch.go1.24/src/runtime/map_noswiss.go#L193-L200)值中，进行快速比较，迅速排除掉绝大部分不匹配的槽位。
    - **第二阶段：完整`key`比较**: 只有在[tophash](https://github.com/golang/go/blob/release-branch.go1.24/src/runtime/map_noswiss.go#L193-L200)匹配成功后，才会进行一次完整的、开销更大的`key`比较。

#### 删除 ([mapdelete](https://github.com/golang/go/blob/release-branch.go1.24/src/runtime/map_noswiss.go#L736-L859))

删除操作，并不会，真正地，整理内存，而是采用“**墓碑 (Tombstone)**”策略。

1.  **查找**: 首先，像查找一样，定位到目标`key`所在的槽位。
2.  **标记**: 找到后，将该槽位的[tophash](https://github.com/golang/go/blob/release-branch.go1.24/src/runtime/map_noswiss.go#L193-L200)值，设置为`emptyOne` (值为1)。

```mermaid
graph TD
    subgraph tophash 状态机
        E["emptyRest (0)<br>空，且后面都空<br>终止查找"]
        O["emptyOne (1)<br>墓碑，仅此为空<br>继续查找"]
        X["evacuatedX (2)<br>已搬迁(前半部分)"]
        Y["evacuatedY (3)<br>已搬迁(后半部分)"]
        EE["evacuatedEmpty (4)<br>空，且桶已搬迁"]
        V[">= minTopHash (5+)<br>有效的哈希值"]
    end
```
- `emptyOne`这个“墓碑”，告诉后续的查找操作：“这里是空的，但请不要停，继续往后找”。
- 这些大量的“墓碑”，正是“同尺寸扩容”需要被触发，来进行“大扫除”的原因之一。

---
Go 的map介绍结束了---------吗？

### 5. `map` 的未来：`swiss` map 简介

Go 1.2x版本后，实验性的`swiss` map，旨在解决这些问题。

- **核心思想**: 借鉴Google的Swiss Table，它是一种基于**开放寻址法**的变体，并结合了**SIMD（单指令多数据）**优化。
- **关键优势**:
    1.  **缓存友好**: 它将元数据（类似于[tophash](https://github.com/golang/go/blob/release-branch.go1.24/src/runtime/map_noswiss.go#L193-L200)）和`key/value`数据，分开存储。在查找时，可以先，用SIMD指令，在一个16字节的元数据块中，并行地，比较16个槽位的状态，一次性，过滤掉大量不匹配的项。这极大地，提高了CPU缓存的命中率。
    2.  **消除指针跳转**: 由于采用开放寻址，它没有[overflow](https://github.com/golang/go/blob/release-branch.go1.24/src/runtime/map_noswiss.go#L207-L209)指针。数据，在物理上，是连续存储的，消除了遍历溢出链时，大量的指针跳转开销，这对CPU的预取机制，非常友好。
    3.  **更优的空间利用率**: 相比链表法，它通常，有更高的空间利用率。

`swiss` map的实现，比`noswiss`更复杂，它代表了哈希表技术，在现代硬件上，发展的最新方向。