---
title: dragonboat学习笔记-raft核心
date: 2019-07-01 15:36:08
tags:
	- dragonboat
	- raft
	- 一致性协议
	- 源码分析
keywords: raft,一致性协议,dragonboat
categories:
	-一致性协议
---

代码在`raft.go`文件中, 在不同节点状态下，对不同的消息，有不同的处理函数：

```go
func (r *raft) initializeHandlerMap() {
	// candidate
	r.handlers[candidate][pb.Heartbeat] = r.handleCandidateHeartbeat
	r.handlers[candidate][pb.Propose] = r.handleCandidatePropose
	r.handlers[candidate][pb.Replicate] = r.handleCandidateReplicate
	r.handlers[candidate][pb.InstallSnapshot] = r.handleCandidateInstallSnapshot
	r.handlers[candidate][pb.RequestVoteResp] = r.handleCandidateRequestVoteResp
	r.handlers[candidate][pb.Election] = r.handleNodeElection
	r.handlers[candidate][pb.RequestVote] = r.handleNodeRequestVote
	r.handlers[candidate][pb.ConfigChangeEvent] = r.handleNodeConfigChange
	r.handlers[candidate][pb.LocalTick] = r.handleLocalTick
	r.handlers[candidate][pb.SnapshotReceived] = r.handleRestoreRemote
	// follower
	r.handlers[follower][pb.Propose] = r.handleFollowerPropose
	r.handlers[follower][pb.Replicate] = r.handleFollowerReplicate
	r.handlers[follower][pb.Heartbeat] = r.handleFollowerHeartbeat
	r.handlers[follower][pb.ReadIndex] = r.handleFollowerReadIndex
	r.handlers[follower][pb.LeaderTransfer] = r.handleFollowerLeaderTransfer
	r.handlers[follower][pb.ReadIndexResp] = r.handleFollowerReadIndexResp
	r.handlers[follower][pb.InstallSnapshot] = r.handleFollowerInstallSnapshot
	r.handlers[follower][pb.Election] = r.handleNodeElection
	r.handlers[follower][pb.RequestVote] = r.handleNodeRequestVote
	r.handlers[follower][pb.TimeoutNow] = r.handleFollowerTimeoutNow
	r.handlers[follower][pb.ConfigChangeEvent] = r.handleNodeConfigChange
	r.handlers[follower][pb.LocalTick] = r.handleLocalTick
	r.handlers[follower][pb.SnapshotReceived] = r.handleRestoreRemote
	// leader
	r.handlers[leader][pb.LeaderHeartbeat] = r.handleLeaderHeartbeat
	r.handlers[leader][pb.CheckQuorum] = r.handleLeaderCheckQuorum
	r.handlers[leader][pb.Propose] = r.handleLeaderPropose
	r.handlers[leader][pb.ReadIndex] = r.handleLeaderReadIndex
	r.handlers[leader][pb.ReplicateResp] = lw(r, r.handleLeaderReplicateResp)
	r.handlers[leader][pb.HeartbeatResp] = lw(r, r.handleLeaderHeartbeatResp)
	r.handlers[leader][pb.SnapshotStatus] = lw(r, r.handleLeaderSnapshotStatus)
	r.handlers[leader][pb.Unreachable] = lw(r, r.handleLeaderUnreachable)
	r.handlers[leader][pb.LeaderTransfer] = lw(r, r.handleLeaderTransfer)
	r.handlers[leader][pb.Election] = r.handleNodeElection
	r.handlers[leader][pb.RequestVote] = r.handleNodeRequestVote
	r.handlers[leader][pb.ConfigChangeEvent] = r.handleNodeConfigChange
	r.handlers[leader][pb.LocalTick] = r.handleLocalTick
	r.handlers[leader][pb.SnapshotReceived] = r.handleRestoreRemote
	r.handlers[leader][pb.RateLimit] = r.handleLeaderRateLimit
	// observer
	r.handlers[observer][pb.Heartbeat] = r.handleObserverHeartbeat
	r.handlers[observer][pb.Replicate] = r.handleObserverReplicate
	r.handlers[observer][pb.InstallSnapshot] = r.handleObserverSnapshot
	r.handlers[observer][pb.Propose] = r.handleObserverPropose
	r.handlers[observer][pb.ReadIndex] = r.handleObserverReadIndex
	r.handlers[observer][pb.ReadIndexResp] = r.handleObserverReadIndexResp
	r.handlers[observer][pb.ConfigChangeEvent] = r.handleNodeConfigChange
	r.handlers[observer][pb.LocalTick] = r.handleLocalTick
	r.handlers[observer][pb.SnapshotReceived] = r.handleRestoreRemote
}
```



# leader选举

从候选者发起投票,接受投票结果,以及其他节点如何处理消息来展开.



##  候选者发起投票

系统启动的时候，执行`Launch` 函数，启动raft节点，然后直接进入`r.becomeFollower(1, NoLeader)` 状态。

执行引擎启动之后,很快就知道时间片到期了,该选举了, 然后就会通过定时tick 执行到:

```go
func (r *raft) handleLocalTick(m pb.Message) {
	if m.Reject {
		r.quiescedTick()
	} else {
    //执行时钟事件
		r.tick()
	}
}


func (r *raft) tick() {
	r.tickCount++
	if r.state == leader {
    //如果是 leader 的话,执行 leaderTick,向别人发送心跳
		r.leaderTick()
	} else {
    //如果不是 leader,竞选 leader
		r.nonLeaderTick()
	}
}

```

下面来看`nonLeaderTick`是怎么做的:

```go
func (r *raft) nonLeaderTick() {
	if r.state == leader {
		panic("noleader tick called on leader node")
	}
	r.electionTick++
	if r.timeForRateLimitCheck() {
		if r.rl.Enabled() {
			r.rl.HeartbeatTick()
			r.sendRateLimitMessage()
		}
	}
	// section 4.2.1 of the raft thesis
	// non-voting member is not to do anything related to election
	if r.isObserver() {
    //观察者不参与投票,只用来接收信息
		return
	}
	// 6th paragraph section 5.2 of the raft paper
  //时间片到期了,就构造一个消息,重用 Handle 函数来执行 election 事件.
	if !r.selfRemoved() && r.timeForElection() {
		r.electionTick = 0
		r.Handle(pb.Message{
			From: r.nodeID,
			Type: pb.Election,
		})
	}
}
```



根据本文置顶的函数转发器,可以知道, 处理`pb.Election`的函数是`handleNodeElection`:

```go
func (r *raft) handleNodeElection(m pb.Message) {
	if r.state != leader {
		// there can be multiple pending membership change entries committed but not
		// applied on this node. say with a cluster of X, Y and Z, there are two
		// such entries for adding node A and B are committed but not applied
		// available on X. If X is allowed to start a new election, it can become the
		// leader with a vote from any one of the node Y or Z. Further proposals made
		// by the new leader X in the next term will require a quorum of 2 which can
		// has no overlap with the committed quorum of 3. this violates the safety
		// requirement of raft.
		// ignore the Election message when there is membership configure change
		// committed but not applied
		if r.hasConfigChangeToApply() {
			plog.Warningf("%s campaign skipped due to pending Config Change",
				r.describe())
			return
		}
		plog.Infof("%s will campaign at term %d", r.describe(), r.term)
    //开始竞选
		r.campaign()
	} else {
		plog.Infof("leader node %s ignored Election", r.describe())
	}
}
```



竞选的代码如下:

```go
func (r *raft) campaign() {
	plog.Infof("%s campaign called, remotes len: %d", r.describe(), len(r.remotes))
  //首先变为 candidate 状态,term+1
	r.becomeCandidate()
  //得到当前的 term,就是 term+1 之后的值
	term := r.term
  //自己给自己投票,就不用构造消息投递了,直接直接 handle 方法就可以了. 票数就会加一
	r.handleVoteResp(r.nodeID, false)
  //如果只有一个节点,直接自己是 leader
	if r.isSingleNodeQuorum() {
		r.becomeLeader()
		return
	}
	var hint uint64
	if r.isLeaderTransferTarget {
		hint = r.nodeID
		r.isLeaderTransferTarget = false
	}
	for k := range r.remotes {
		if k == r.nodeID {
			continue
		}
    //向其他节点发送心跳信息.
		r.send(pb.Message{
			Term:     term, //我当前的 term 
			To:       k, //目标
			Type:     pb.RequestVote, //请求投票给我
			LogIndex: r.log.lastIndex(), //我的最后一个 index 
			LogTerm:  r.log.lastTerm(), //我的最后一个 term,这两个用于接收方判断谁大,谁大就投给谁
			Hint:     hint,
		})
		plog.Infof("%s sent RequestVote to node %s", r.describe(), NodeID(k))
	}
}
```



发起投票之后就等到回复消息了,收到回复之后就会执行

```go
//收到投票消息
func (r *raft) handleCandidateRequestVoteResp(m pb.Message) {
	_, ok := r.observers[m.From]
	if ok {
		plog.Warningf("dropping a RequestVoteResp from observer")
		return
	}
 //处理投票的结果,同意 count+1,否则就不加
	count := r.handleVoteResp(m.From, m.Reject)
	plog.Infof("%s received %d votes and %d rejections, quorum is %d",
		r.describe(), count, len(r.votes)-count, r.quorum())
	// 3rd paragraph section 5.2 of the raft paper
	if count == r.quorum() {
    //如果达到了大多数,就 becomeLeader ,成为 leader, 写入本地日志.
		r.becomeLeader()
		// get the NoOP entry committed ASAP
    //必须发送一条空的心跳消息,用来确立自己的 leader 地址,可以见论文或者本系列的其他文章描述
		r.broadcastReplicateMessage()
	} else if len(r.votes)-count == r.quorum() {
		// etcd raft does this, it is not stated in the raft paper
		r.becomeFollower(r.term, NoLeader)
	}
}
```



## 其他节点如何响应投票信息

对应的, 候选者发送了RequestVote消息之后,  无论当前接受消息的服务器节点处于什么状态, 都会执行

```go
// 处理候选者发来的投票请求
func (r *raft) handleNodeRequestVote(m pb.Message) {
  //构造返回信息
	resp := pb.Message{
		To:   m.From,
		Type: pb.RequestVoteResp,
	}
	// 3rd paragraph section 5.2 of the raft paper
  // 每个服务器将在给定的任期内最多投票给一个候选人,谁的消息先来就投给谁.
  // 判断条件是: r.vote == NoNode || r.vote == m.From || m.Term > r.term
	canGrant := r.canGrantVote(m)
	// 2nd paragraph section 5.4 of the raft paper
  //判断消息里的 term 和本节点之前处理的最后一个日志的 term 谁大, 如果消息大,就可以投票给他
  //如果 term 一样,以谁的日志更大,就投给谁.
	isUpToDate := r.log.upToDate(m.LogIndex, m.LogTerm)
	if canGrant && isUpToDate {
		plog.Infof("%s cast vote from %s index %d term %d, log term: %d",
			r.describe(), NodeID(m.From), m.LogIndex, m.Term, m.LogTerm)
		r.electionTick = 0
		r.vote = m.From
    //投票成功
	} else {
		plog.Infof("%s rejected vote %s index%d term%d,logterm%d,grant%v,utd%v",
			r.describe(), NodeID(m.From), m.LogIndex, m.Term,
			m.LogTerm, canGrant, isUpToDate)
    //拒绝投票
		resp.Reject = true
	}
  //消息回复给发起方
	r.send(resp)
}
```

