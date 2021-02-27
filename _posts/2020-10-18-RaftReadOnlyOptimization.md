---
title: 《Raft只读优化的思考》
description: 由于Raft的Safety很大程度上是由其Strong Leader的特性来保证的，使用Strong Leader虽然能简化raft协议的可理解性，但是也带来了很大的性能损失。比如raft的读写操作都必须由Leader来处理。这使得无论集群中有多少台服务器，都只有 Leader 副本能够对外提供服务，另外的Follower节点除了时刻同步数据，以及参加选举之外，所有的cpu，io和网络资源都被浪费掉了。这对读操作密集的服务来说是非常难以接收的，即使可以使用multi-raft来对数据进行分片来解决单leader性能瓶颈的问题，但是集群中2/3的cpu，io，和网络资源还是别浪费了。因此本文主要介绍raft的作者在他PhD论文[1]中提到的两种优化方法。 
date:  Sun, 18 Oct 2020 21:25:54
tags:
  - Distributed System
  - Raft
categories:
- Distributed System
---

## Raft 只读操作优化

​	由于Raft的Safety很大程度上是由其Strong Leader的特性来保证的，使用Strong Leader虽然能简化raft协议的可理解性，但是也带来了很大的性能损失。比如raft的读写操作都必须由Leader来处理。这使得无论集群中有多少台服务器，都只有 Leader 副本能够对外提供服务，另外的Follower节点除了时刻同步数据，以及参加选举之外，所有的cpu，io和网络资源都被浪费掉了。这对读操作密集的服务来说是非常难以接收的，即使可以使用multi-raft来对数据进行分片来解决单leader性能瓶颈的问题，但是集群中2/3的cpu，io，和网络资源还是别浪费了。因此本文主要介绍raft的作者在他PhD论文[1]中提到的两种优化方法，以及一些思考。

### 一. 使用readIndex

​	由于只读操作并不会改变数据。因此在只读操作的时候，如果不提交read log则可以减少数据落盘的开销。正常情况下这样做是没问题的，但是在split brain的情况下可能会导致stale read。例如：一个leader与集群的其他部分分开，集群的其他部分可能已经选择了一个新的leader并向raft提交了新的条目。如果分区leader在没有咨询其他服务器的情况下响应只读查询，那么它将返回旧的数据破坏线性一致性。因此作者提出了以下五点约束

> 1. If the leader has not yet marked an entry from its current term committed, it waits until it has done so. The Leader Completeness Property guarantees that a leader has all committed entries, but at the start of its term, it may not know which those are. To find out, it needs to commit an entry from its term. Raft handles this by having each leader commit a blank no-op entry into the log at the start of its term. As soon as this no-op entry is committed, the leader’s commit index will be at least as large as any other servers’ during its term.
> 2. The leader saves its current commit index in a local variable readIndex. This will be used as a lower bound for the version of the state that the query operates against.
> 3. The leader needs to make sure it hasn’t been superseded by a newer leader of which it is unaware. It issues a new round of heartbeats and waits for their acknowledgments from a majority of the cluster. Once these acknowledgments are received, the leader knows that there could not have existed a leader for a greater term at the moment it sent the heartbeats. Thus, the readIndex was, at the time, the largest commit index ever seen by any server in the cluster.
> 4. The leader waits for its state machine to advance at least as far as the readIndex; this is current enough to satisfy linearizability.
> 5. Finally, the leader issues the query against its state machine and replies to the client with the results.

**总结以下就是：**

> 1. 需要注意：当一个leader刚被当选时，虽然他肯定拥有最完整的信息，但leader无法确认自己的commitIndex是最新的。所以这个leader必须发一个no-op的空白log。直到这个空白log被commit了，保证了commitIndex为最新的。再将最新的commitIndex做为readIndex。
> 2. 发起一次心跳，当收集到这次心跳返回ack中的commitIndex >= readIndex 且 commitIndex达到法定人数时。确认leader拥有最新被提交的日志且自己就是集群中的合法leader。
> 3. leader的状态机至少执行到readIndex，保证线性一致性。
> 4. 返回给状态机，执行这个只读的操作给客户。

readIndex这种方法，虽然避免不了网络请求的开销，但是减少了raft的log，也避免了读操作落磁盘的开销。此方法实现以后，用户可以向follower请求只读操作，以保证负载均衡，增强系统读的吞吐率。这里需要注意，follower的数据可能落后当前leader很多，或者网络分区后跟随了错误的leader。所以，leader必须提供一个接口以返回当前的readIndex。follower请求这个接口，拿到当前的readIndex，然后leader执行1-2，被请求的follower在自己的状态机中执行3-4步。因此在读压力比较大的时候，client的读操作不一定非要由leader来处理，使得follower的io和网络资源就被利用起来了。

### 二. 使用租期(lease read)

readIndex方法虽然减少了raft的log开销，但还是需要网络的请求操作。因此作者又提出了lease read:在租期未过期时，在不进行任何网络请求下，保证用户只读请求。

虽然leader还是需要记录commitIndex作为readIndex，但是不会专门发起一次心跳。因为在leader的租期不过期的情况下，肯定不会有新的leader产生。因此只要保证leader的状态机至少执行到readIndex，便可以给用户返回只读操作。

这里使用论文[1]中的一副图来说明lease read

![1607151461882](http://img.jackdu.cn/1607151461882.png)

因为要考虑到进程调度，垃圾回收，虚拟机迁移，时钟频率不同等种种原因，需要设置一个bound（bound是大于1的）。其中start为leader向集群follower发送心跳的时间点。那么租期为：

![[公式]](https://www.zhihu.com/equation?tex=+lease%5C_time+%3D+%5Cfrac%7Belection%5C+timeout%7D%7Bclock%5C+drift%5C+bound%7D+)

当收到法定人数的心跳点后，延长一个租期(lease_time)。虽然使用lease这种方法减少了时钟开销，但是如果这里的bound设置与现实不符，或者出现时钟严重漂移等问题。仍会导致旧leader可能给客户返回陈旧数据。

作者在文章[1]中最后又提出了一种方法来避免这种情况:

> Fortunately, a simple extension can improve the guarantee provided to clients, so that even under asynchronous assumptions (even if clocks were to misbehave), each client would see the replicated state machine progress monotonically (sequential consistency). For example, a client would not see the state as of log index n, then change to a different server and see only the state as of log index n − 1. To implement this guarantee, servers would include the index corresponding to the state machine state with each reply to clients. Clients would track the latest index corresponding to results they had seen, and they would provide this information to servers on each request. If a server received a request for a client that had seen an index greater than the server’s last applied log index, it would not service the request (yet).

服务器在返回apply日志时，把这个日志对应的index也一并返回给用户，用户把这个日志的index保存起来，客户将跟踪与他们所看到的结果相对应的最新索引，并在每次请求时将这些信息提供给服务端。如果服务端收到的客户端请求的index大于服务器最新的applied的日志的index，则服务器不会为该请求提供服务。

### 三. TiKV的实践

由于Tikv使用的是raft作为一致性协议，因此在实践过程中也做了相同的优化并且做了分享，具体可见下面两篇blog:

https://pingcap.com/blog-cn/follower-read-the-new-features-of-tidb/

https://pingcap.com/blog-cn/lease-read/

**Reference:**

[1]《CONSENSUS: BRIDGING THEORY AND PRACTICE》 http://doc.jackdu.cn/OngaroPhD.pdf

[2] https://zhuanlan.zhihu.com/p/104651506