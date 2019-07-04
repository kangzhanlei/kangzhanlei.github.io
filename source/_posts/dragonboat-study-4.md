---
title: dragonboat学习笔记-leader转移
date: 2019-07-04 07:12:00
tags:
	- raft
	- 一致性协议
	- go
	- dragonboat
	- 源码分析
categories:
	- 一致性协议
---

在 tidb 中, region分裂的时候需要进行 raft 节点间的 leader 转移, 是很常用的。所以这里来分析一下 leader 转移的大致流程. 先参考一下论文的3.10节

1. 某些时候leader必须被下线， 比如它可能被重启了，或者心跳超时了其他节点当选为leader
2. 某些场景下， 某些节点更适合成为leader，比如性能比较高的服务器，带宽较大的服务器，权重大的服务器

# leader 切换原理

在raft中， 如果需要做 leader 的切换, 首先leader 需要发送本节点上所有的 log entries到目标节点, 然后目标节点直接开始发起 leader 的竞选投票,而不用等他的electionTimeout .   leader 会确保在当前Term 下目标节点已经持久化了所有的 commited entries, 然后让新节点开始竞选. 细节如下:

* 当前的 leader 首先停止接收客户端的请求 (假设 leader 转给别人了,但是这里还接收请求,那么新 leader 可能没有这个最新的日志, 因为旧 leader 只要大多数就可以了,不一定发送到新 leader 上)
* 当前节点更新目标节点的 log 日志,确保这 2 个节点的日志是一样的 . 可以使用日志复制机制来保证
* 当前 leader 给目标节点发送一个`timeoutNow`的请求, 目标节点接收到之后,应该立刻开始新一轮的竞选: 也就是增加自己的 term,然后变成candidate, 开始发起投票. 



# 代码实现

这里依然涉及到发送方和接收方的消息处理, 代码大都在 raft.go文件中. 

## 发送方

在 dragonboat 中, nodehost 有个`RequestLeaderTransfer`的方法,用于转移 leader. 下面从这个入口来分析

```go
//leader 转移的请求,clusterID 表示当前的组, targetNodeID 表示需要转移到的新节点的的 ID
func (nh *NodeHost) RequestLeaderTransfer(clusterID uint64,
	targetNodeID uint64) error {
  //获取对应 clusterID 的 node 节点
	v, ok := nh.getCluster(clusterID)
	if !ok {
		return ErrClusterNotFound
	}
	plog.Infof("RequestLeaderTransfer called on cluster %d target nodeid %d",
		clusterID, targetNodeID)
  //发起leaderTransfer 的请求
	v.requestLeaderTransfer(targetNodeID)
	return nil
}

```

也就是说, 每个 cluster, 都可以进行 leader 转移, 多个 raft 组互相不会影响. 下面来看node 的requestLeaderTransfer方法

```go
//node的 leader 转移方法,直接交给peer来发起请求
func (n *node) requestLeaderTransfer(nodeID uint64) {
	n.p.RequestLeaderTransfer(nodeID)
}

//发起转移请求, 因为当前运行的节点有可能不是 leader, 如果是其他状态的话, 需要转发到 leader 上.
//所以构造了一个LeaderTransfer的请求,到 raft 节点去执行.
func (rc *Peer) RequestLeaderTransfer(target uint64) {
	plog.Infof("RequestLeaderTransfer called, target %d", target)
	rc.raft.Handle(pb.Message{
		Type: pb.LeaderTransfer,
		To:   rc.raft.nodeID,
		From: target,
		Hint: target,
	})
}
```

raft的`Handle`方法, 就会根据当前 raft 节点的状态(leader,follower, candidate, observer) 来决定去向. 来看 follower

```go
func (r *raft) handleFollowerLeaderTransfer(m pb.Message) {
	if r.leaderID == NoLeader {
		plog.Warningf("%s dropped LeaderTransfer as no leader", r.describe())
		return
	}
	plog.Infof("rerouting LeaderTransfer for %d to %d",
		r.clusterID, r.leaderID)
	m.To = r.leaderID
  //直接发送到 leader 节点去执行. 
	r.send(m)
}
```

下面来看 leader 节点的处理方式:

```go

func (r *raft) handleLeaderTransfer(m pb.Message, rp *remote) {
	target := m.Hint
	plog.Infof("handleLeaderTransfer called on cluster %d, target %d",
		r.clusterID, target)
  //没有目标节点,忽略请求
	if target == NoNode {
		panic("leader transfer target not set")
	}
  //当期处理转移状态中, 不会重复处理
	if r.leaderTransfering() {
		plog.Warningf("LeaderTransfer ignored, leader transfer is ongoing")
		return
	}
  //自己转移给自己没有意义
	if r.nodeID == target {
		plog.Warningf("received LeaderTransfer with target pointing to itself")
		return
	}
  //设置转移节点, 暂存起来,如果不满足发送条件,等待日志复制到匹配位置之后,再进行发送, 见下面的注释
  //同时也会告诉propose请求,我当前正在处理 leader 转移,不接收任何的请求.
	r.leaderTransferTarget = target
  //让自己的时间片清 0,防止发送 leader 租期给其他节点.
	r.electionTick = 0
	// fast path below
	// or wait for the target node to catch up, see p29 of the raft thesis
  //如果目标节点没有完全的日志, 暂时不发送. 
  //那么这个转移请求是不是真的忽略了呢? 其实并不是. 上文说过了,只有对方节点拥有了完整的日志才可以转移
  //所以需要在发送日志复制消息之后,得到节点的返回信息,如果两者匹配了, 那么就会立刻发送TimeoutNow的消息 
  //这也就是为什么需要设置r.leaderTransferTarget=target的标志. 
	if rp.match == r.log.lastIndex() {
    //发送TimeoutNow消息给对应的节点.让他开始竞选
		r.sendTimeoutNowMessage(target)
	}
}
```



在 leader 转移的过程当中,leader 不应该响应任何 propose 请求,来看代码

```go
func (r *raft) handleLeaderPropose(m pb.Message) {
	if r.selfRemoved() {
		plog.Warningf("dropping a proposal, local node has been removed")
		return
	}
  //判断当前是否在进行转移, 转移过程中不响应事件. 因为如果响应了, 新的 leader 可能没有最全的数据
	if r.leaderTransfering() {
		plog.Warningf("dropping a proposal, leader transfer is ongoing")
		return
	}
	for i, e := range m.Entries {
		if e.Type == pb.ConfigChangeEntry {
			if r.hasPendingConfigChange() {
				plog.Warningf("%s dropped a config change, one is pending",
					r.describe())
				m.Entries[i] = pb.Entry{Type: pb.ApplicationEntry}
			}
			r.setPendingConfigChange()
		}
	}
	r.appendEntries(m.Entries)
	r.broadcastReplicateMessage()
}
```





## 接收方

目标节点响应`TimeoutNow`的消息,  因为是 leader 转移,所以其他节点都会是 follower, 直接看他的实现

```go
func (r *raft) handleFollowerTimeoutNow(m pb.Message) {
	// the last paragraph, p29 of the raft thesis mentions that this is nothing
	// different from the clock moving forward quickly
  //论文中提到了, 发送 timeoutNow 消息, 和时钟走的非常迅速,直到electionTimeout到期,效果是一样的
	plog.Infof("TimeoutNow received on %d:%d", r.clusterID, r.nodeID)
  //设置 election 的 Tick 时间,目的是为了告诉 tick 时钟,我的时间片到啦,该竞选啦
	r.electionTick = r.randomizedElectionTimeout
  //标记当前节点正处于 leader 转移的过程当中
	r.isLeaderTransferTarget = true
  //然后直接发起 tick 事件, 这里会比对当前的electionTick和 electionTimeout, 如果大于表示超时了
  //因为刚才已经重置为最大的了,这里 tick 一下加 1,肯定过了时间片了, 开始竞选
	r.tick()
	if r.isLeaderTransferTarget {
		r.isLeaderTransferTarget = false
	}
}
```

好, 此时接收方就很有可能竞选成功, 因为在这个时间内, 很少有其他节点来同时竞选 leader 的,因为他们的时间片还没有到期. 



## 超时失败

如果被转移的节点,迟迟收不到消息, 或者消息丢失了, 那么 leader 节点已经关闭了 propose请求,岂不是系统就要停止服务了. 所以一定会有一个超时时间,和错误判断, 来处理这种情况.

leader 会进行逻辑时钟的 tick, 每次 tick 都会判断当前是否超过了一次心跳的时间了,在这个时间内没有得到新 leader 的消息, 那么就会终止LeaderTransfer 的请求.  开始接收 propose请求

代码如下

```go

func (r *raft) leaderTick() {
	if r.state != leader {
		panic("leaderTick called on a non-leader node")
	}
	r.electionTick++
	if r.timeForRateLimitCheck() {
		if r.rl.Enabled() {
			r.rl.HeartbeatTick()
		}
	}
  //判断时间片是否到期,到期之后,允许接收请求
	timeToAbortLeaderTransfer := r.timeToAbortLeaderTransfer()
	if r.timeForCheckQuorum() {
		r.electionTick = 0
		if r.checkQuorum {
			r.Handle(pb.Message{
				From: r.nodeID,
				Type: pb.CheckQuorum,
			})
		}
	}
	if timeToAbortLeaderTransfer {
    //关闭拦截 ,允许 leader 接受请求
		r.abortLeaderTransfer()
	}
	r.heartbeatTick++
	if r.timeForHearbeat() {
		r.heartbeatTick = 0
		r.Handle(pb.Message{
			From: r.nodeID,
			Type: pb.LeaderHeartbeat,
		})
	}
}
```



这里也有几种情况, 比如

* 当前leader 消息发出去了,  新的 leader 产生了,但是旧 leader 没有收到消息, 还继续 tick,继续接受 propose 消息会有问题吗? 
  * 答案是不会的, 因为新的 term 已经产生, 旧 leader 的 propose 请求都会被拒绝的. 所以没有问题
* leader 消息发出去了, 但是接收方没有收到, 而 leader 的时间片到期了,又开始接收 propose 请求, 这个时候接收方收到了TimeoutNow消息,开始竞选了,会有问题吗? 
  * 假设 leader 又接收了 propose 请求, 而请求没有达到大多数, 那么接收方就有机会晋升为 leader, 那么旧 leader 的 propose 消息也就作废了,没有生效. 所以没关系
  * 假设 leader 又接收了 propose 请求,而达到大多数,已经 commited, 那么也没关系, 因为接收方想要成为 leader, 必须拥有最新的日志, 而接收方当前并不是最新的了, 所以他成为不了 leader. 



## 总结

leader 转移还是比较简单的. 只要对方节点拥有了 leader 节点当前 term 的所有的日志, 就可以进行转移了. 其实也就是判断其他节点的 matchIndex 和 当前 leader 的 commited 是不是相同即可. 

转移失败了, 也没有关系, leader 节点恢复接收propose 消息,就当什么事情都没有发生过.