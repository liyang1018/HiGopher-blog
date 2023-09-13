---
title: MIT6.824系列(1)-MapReduce
subtitle: MapReduce

# Summary for listings and search engines
summary: MIT6.824系列(1)-MapReduce

# Link this post with a project
projects: []

# Date published
date: '2023-09-13T00:00:00Z'

# Date updated
lastmod: '2023-09-13T00:00:00Z'

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

authors:
- admin

tags:
- Golang
- MapReduce

categories:
- Golang
- MapReduce
---

## 1、MapReduce简介
分布式计算方案MapReduce，是一种编程模型，用于大规模数据集的并行运算，其中包括Map（映射）和Reduce（归约）。
MapReduce既是一个编程模型，也是一个计算组件，处理的过程分为以下阶段：
- Map阶段： 在Map阶段中，MapReduce将输入数据分割成若干个小块，然后在分布式计算集群上同时执行多个Map任务，
每个任务都对一个小块的数据进行处理，并将处理结果输出为一系列键值对，Map任务的输出结果会被临时存储在本地磁盘或内存中，
以供Reduce任务使用。
- shuffle阶段：
  - 合并相同 Key 的 Value：Map 任务输出的键值对可能会包含相同的 Key，Shuffle 过程会将相同 Key 的 Value 合并在一起，减少 Reduce 任务需要处理的数据量。
  - 按照 Key 进行排序：Shuffle 过程会将 Map 任务的输出结果按照 Key 进行排序，这样 Reduce 任务可以顺序地处理键值对序列，避免在处理数据时需要进行额外的排序操作。
  - 划分数据并传输：Shuffle 过程会将 Map 任务的输出结果按照 Key 划分成多个分区，并将每个分区的数据传输到对应的 Reduce 任务中。这样，Reduce 任务可以从不同的 Map 任务中获取到数据，从而实现更好的并行化处理。
- Reduce阶段：负责把多个小任务的处理结果进行汇总。其中Map阶段主要输入是一对Key-Value，经过map计算后输出一对Key-Value值；
然后将相同Key合并，形成Key-Value集合；再将这个Key-Value集合转入Reduce阶段，经过计算输出最终Key-Value结果集。
![img](./1.png)
**本质上是一种分而治之、并行计算的思想，将大任务分解为多个小任务，然后将多个小任务的结果进行汇总，得到最终结果。**
