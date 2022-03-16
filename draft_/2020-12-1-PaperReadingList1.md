---
title: 《论文阅读笔记1》
description:  
date: Tue, 01 Dec 2020 19:41:02 +0800
tags:
  - Distributed System
  - Raft
categories:
- Distributed System
---

### 1. CRaft

**《CRaft: An Erasure-coding-supported Version of Raft for Reducing Storage Cost and Network Cost》**

https://www.usenix.org/conference/fast20/presentation/wang-zizhong

​	因为最近也在做一篇raft性能优化方面的论文，所以在fast2020上看到清华的这篇论文觉得非常有意思。毕竟他们声称通过Erasure-coding能大量的降低网络带宽和存储io和极大的提高性能：CRaft could save 66% of storage, reach a 250% improvement on write throughput and reduce 60.8% of write latency compared to original Raft.



