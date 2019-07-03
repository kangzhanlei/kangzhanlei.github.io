---
title: dragonboat学习笔记-raft核心
date: 2019-07-01 15:36:08
tags:
	- dragonboat
	- raft
	- 一致性协议
	- 源码分析
	- go
keywords: raft,一致性协议,dragonboat
categories:
	- 一致性协议
---

[TOC]



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



# 日志复制

从发起 proposal 开始介绍, 到接收端的处理, 以及 commited 和 apply 的整个流程

## 发起请求

`nodeHost.SyncPropose`是阻塞式发起请求,代码:

```go
// 为了实现命令去重等功能,每个客户端都有一个session 
func (nh *NodeHost) SyncPropose(ctx context.Context,
	session *client.Session, cmd []byte) (sm.Result, error) {
  //获取超时时间
	timeout, err := getTimeoutFromContext(ctx)
	if err != nil {
		return sm.Result{}, err
	}
  //发起请求,返回一个 ReadState 结构体, 接收返回来的信息
	rs, err := nh.Propose(session, cmd, timeout)
	if err != nil {
		return sm.Result{}, err
	}
  //在 rs 的 channel 中阻塞,等待返回结果
	result, err := checkRequestState(ctx, rs)
	if err != nil {
		return sm.Result{}, err
	}
  //rs 是对象池中的数据,用完了需要释放
	rs.Release()
	return result, nil
}

```



下面来看 Propose 的实现

```go
func (nh *NodeHost) propose(s *client.Session,
	cmd []byte, handler ICompleteHandler,
	timeout time.Duration) (*RequestState, error) {
  //取样
	var st time.Time
	sampled := delaySampled(s)
	if sampled {
		st = time.Now()
	}
  //根据 clusterID,得到一个 node 节点对象
	v, ok := nh.getClusterNotLocked(s.ClusterID)
	if !ok {
		return nil, ErrClusterNotFound
	}
	if !v.supportClientSession() && !s.IsNoOPSession() {
		plog.Panicf("IOnDiskStateMachine based nodes must use NoOPSession")
	}
  //使用该 node 发起 proposal 请求
	req, err := v.propose(s, cmd, handler, timeout)
  //发出 nodeReady 事件,让执行引擎知道有数据进来了.就可以去获取任务执行
	nh.execEngine.setNodeReady(s.ClusterID)
	if sampled {
		nh.execEngine.proposeDelay(s.ClusterID, st)
	}
	return req, err
}
```

来看`propose`方法:

```go
func (n *node) propose(session *client.Session,
	cmd []byte, handler ICompleteHandler,
	timeout time.Duration) (*RequestState, error) {
	if !session.ValidForProposal(n.clusterID) {
		return nil, ErrInvalidSession
	}
  // 检查消息大小,太大的不管
	if n.payloadTooBig(len(cmd)) {
		return nil, ErrPayloadTooBig
	}
  //加入到pendingProposal的队列中去, 下面具体来看,挺有意思
	return n.pendingProposals.propose(session, cmd, handler, timeout)
}

//这里注意看,实现了数据分片
func (p *pendingProposal) propose(session *client.Session,
	cmd []byte, handler ICompleteHandler,
	timeout time.Duration) (*RequestState, error) {
  //根据 session的客户端,得到一个 keyGenerator 生成一个 key
	key := p.nextKey(session.ClientID)
  //根据 key,得到一个proposalShard分片,在系统中一共有 16 个分片,这里取一个
  //各Raft组被分配到不同的执行shard上，以提供parallelism，每个shard又是一个多级流水线，不同处理阶段不同性质（IO密集、内存访问密集等）的处理在流水线不同级间并发完成，充分利用concurrency优势将所有消息传递、执行更新等操作异步化。
	pp := p.shards[key%p.ps]
  //使用这个分片发起请求.尽可能的减少阻塞的时间
	return pp.propose(session, cmd, key, handler, timeout)
}

//分片的 propose 请求
func (p *proposalShard) propose(session *client.Session,
	cmd []byte, key uint64, handler ICompleteHandler,
	timeout time.Duration) (*RequestState, error) {
  //取什么时候超时,依赖于 tick 时钟
	timeoutTick := p.getTimeoutTick(timeout)
	if timeoutTick == 0 {
		return nil, ErrTimeoutTooSmall
	}
	if uint64(len(cmd)) > maxProposalPayloadSize {
		return nil, ErrPayloadTooBig
	}
  //构造一个 Entry 数据结构,是一条请求的日志条目.
	entry := pb.Entry{
		Key:         key,
		ClientID:    session.ClientID,
		SeriesID:    session.SeriesID,
		RespondedTo: session.RespondedTo,
		Cmd:         prepareProposalPayload(cmd),
	}
  //从池子中获取一个RequestState对象返回,在使用结束后需要 release 归还.
	req := p.pool.Get().(*RequestState)
	req.clientID = session.ClientID
	req.seriesID = session.SeriesID
	req.completeHandler = handler
	req.key = entry.Key
	req.deadline = p.getTick() + timeoutTick
	if len(req.CompletedC) > 0 {
    //构造一个 channel, 在这个管道上阻塞, 有了结果从这里返回
		req.CompletedC = make(chan RequestResult, 1)
	}

	p.mu.Lock()
	if badKeyCheck {
		_, ok := p.pending[entry.Key]
		if ok {
			plog.Warningf("bad key")
			p.mu.Unlock()
			return nil, ErrBadKey
		}
	}
	p.pending[entry.Key] = req
	p.mu.Unlock()
	//加入到队列中.这个队列挺有意思的, 分为左右小数组 一直在左队列中累计,等时间片到了时候, 指针指向右
  //数组, 从而使消息在右数组中积攒. 而与此同时左数组中的数据会发送出去,然后左右数组来回切换
  //是个不错的设计
	added, stopped := p.proposals.add(entry)
	if stopped {
		plog.Warningf("dropping proposals, cluster stopped")
		p.mu.Lock()
		delete(p.pending, entry.Key)
		p.mu.Unlock()
		return nil, ErrClusterClosed
	}
	if !added {
		p.mu.Lock()
		delete(p.pending, entry.Key)
		p.mu.Unlock()
		plog.Warningf("dropping proposals, overloaded")
		return nil, ErrSystemBusy
	}
	return req, nil
}

```



消息添加到了entryQueue中, 前面的文章介绍到了. 执行引擎会不停的`handleEvent`, 会监测`handleProposals`的状况,发现有消息添加进来了, 就从 entryQueue 中 get 一批. 此时. 队列的左右数组会进行切换. 让收集和发送互不影响. 

简单看下`node.handleProposals`的代码:

```go
func (n *node) handleProposals() bool {
	rateLimited := n.p.RateLimited()
	if n.rateLimited != rateLimited {
		n.rateLimited = rateLimited
		plog.Infof("%s new rate limit state is %t", n.id(), rateLimited)
	}
  //获取任务,这个方法会被周期性的执行.
	entries := n.incomingProposals.get(n.rateLimited)
	if len(entries) > 0 {
		n.p.ProposeEntries(entries)
		return true
	}
	return false
}

func (rc *Peer) ProposeEntries(ents []pb.Entry) {
  // 交给 raft 节点去处理 propose 事件.
	rc.raft.Handle(pb.Message{
		Type:    pb.Propose,
		From:    rc.raft.nodeID,
		Entries: ents,
	})
}
```



下面终于到了 raft 节点了. 前面介绍了, 不同状态下,不同的消息,有不同的处理函数. 我们以 follower 来看下是怎么处理的.

```go
//follower 节点, 
func (r *raft) handleFollowerPropose(m pb.Message) {
	if r.leaderID == NoLeader {
		plog.Warningf("%s dropping proposal as there is no leader", r.describe())
		return
	}
  //直接转给 leader 
	m.To = r.leaderID
	// the message might be queued by the transport layer, this violates the
	// requirement of the entryQueue.get() func. copy the m.Entries to its
	// own space.
	m.Entries = newEntrySlice(m.Entries)
	r.send(m)
}
```



下面来看 leader 节点的处理方式:

```go
func (r *raft) handleLeaderPropose(m pb.Message) {
	if r.selfRemoved() {
		plog.Warningf("dropping a proposal, local node has been removed")
		return
	}
  //发生 leader 转移的时候,需要拒绝客户端的请求. 等 leader 转移完毕之后,新 leader 再服务.
	if r.leaderTransfering() {
		plog.Warningf("dropping a proposal, leader transfer is ongoing")
		return
	}
  //因为发送的不是一个消息,是一堆消息,这也是 batch 优化的地方.
	for i, e := range m.Entries {
		if e.Type == pb.ConfigChangeEntry {
      //配置变更的信息,暂时不关注
			if r.hasPendingConfigChange() {
				plog.Warningf("%s dropped a config change, one is pending",
					r.describe())
				m.Entries[i] = pb.Entry{Type: pb.ApplicationEntry}
			}
			r.setPendingConfigChange()
		}
	}
  //leader收到请求后,把日志追加到内存中
	r.appendEntries(m.Entries)
  //向其他节点发送 replicate 消息,下面来看是怎么发送的
	r.broadcastReplicateMessage()
}

//广播消息
func (r *raft) broadcastReplicateMessage() {
	for nid := range r.remotes {
		if nid != r.nodeID {
			r.sendReplicateMessage(nid)
		}
	}
	for nid := range r.observers {
		if nid == r.nodeID {
			panic("observer is trying to broadcast Replicate msg")
		}
    //向其他节点发送 replicate 消息
		r.sendReplicateMessage(nid)
	}
}

```



```go
func (r *raft) sendReplicateMessage(to uint64) {
	var rp *remote
	if v, ok := r.remotes[to]; ok {
		rp = v
	} else {
		rp, ok = r.observers[to]
		if !ok {
			panic("failed to get the remote instance")
		}
	}
	if rp.isPaused() {
		return
	}
  //先把 Entry 们序列化为 pb.message
	m, err := r.makeReplicateMessage(to, rp.next, settings.Soft.MaxEntrySize)
	if err != nil {
		// log not available due to compaction, send snapshot
		if !rp.isActive() {
			plog.Warningf("node %s is not active, sending snapshot is skipped",
				NodeID(to))
			return
		}
		index := r.makeInstallSnapshotMessage(to, &m)
		plog.Infof("%s is sending snapshot (%d) to %s, r.Next %d, r.Match %d, %v",
			r.describe(), index, NodeID(to), rp.next, rp.match, err)
		rp.becomeSnapshot(index)
	} else {
		if len(m.Entries) > 0 {
      //这里也是论文的一个优化,发送完了消息之后,先把 next 更新为最后一笔,如果没有成功没关系
      //有日志检查的保证,不行就重试呗
			lastIndex := m.Entries[len(m.Entries)-1].Index
			rp.progress(lastIndex)
		}
	}
  //加到待发送队列中去.
	r.send(m)
}
```



以上是 leader 把接收到的 entry, 整理整理之后,发送给其他节点等候大多数节点的返回. 在看一下 Entry 是怎么转换为 message的

```go
func (r *raft) makeReplicateMessage(to uint64,
	next uint64, maxSize uint64) (pb.Message, error) {
  // leader 为每个节点都维护了一个 next 指针和 match 指针. next 表示下一个准备从哪里发送,因为每个节点的速度不同,网络丢失什么的,都会有不同的 next.
  
  //先取一下上一笔的 term 和上一笔的 index, 发送 replicate 的时候, 需要带着这俩内容, 然其他节点做日志检查. 如果他们的日志里上一笔的信息和leader 发送的不一致,说明日志不匹配啊. 要从 match 的地方二分法找到匹配的日志,再发
	term, err := r.log.term(next - 1)
	if err != nil {
		return pb.Message{}, err
	}
  //之前 entry 累计到了内存中了,可能累计了 100 条,这里要求最多发 30 条,就得处理处理.
  //如果内存中不够,还要从本地的 logdb 中加载,所以呢,缓存是个好手段啊
	entries, err := r.log.entries(next, maxSize)
	if err != nil {
		return pb.Message{}, err
	}
	if len(entries) > 0 {
		if entries[len(entries)-1].Index != next-1+uint64(len(entries)) {
			plog.Panicf("expected last index in Replicate %d, got %d",
				next-1+uint64(len(entries)), entries[len(entries)-1].Index)
		}
	}
	return pb.Message{
		To:       to,
		Type:     pb.Replicate, //发送的消息类型为 Replicate,也就就是 Append entry
		LogIndex: next - 1, //上一条的索引
		LogTerm:  term, //上一条的任期
		Entries:  entries,//本次的批量 entry
		Commit:   r.log.committed,//leader 当前提交的最后一个 index,目的是告诉其他节点,通过了, 可以 apply 了
	}, nil
}
```



好了,到现在,leader 已经把消息发出去了.  全都是异步的.



## 消息接收方处理

消息到达 follower 节点之后,follower 节点的处理比较简单,按照论文的说法,直接给出代码即可:

```go

func (r *raft) handleFollowerReplicate(m pb.Message) {
  //把自己的计时清零一下吧,都接受到别人的消息了,说明别人还活着
	r.electionTick = 0
  //知道别人是 leader 了,这里更新一下
	r.setLeaderID(m.From)
  //开始处理消息
	r.handleReplicateMessage(m)
}

func (r *raft) handleReplicateMessage(m pb.Message) {
  //构造返回消息
	resp := pb.Message{
		To:   m.From,
		Type: pb.ReplicateResp,
	}
  //如果 leader 发来的上一条数据,已经 commited 了,
  //就告诉 leader 节点,更新 match 节点吧,别发多余的数据了
  //这种情况是怎么发生的? 是因为 leader 会根据 next 的数据给节点发送数据, 节点会比对前一笔对不对,
  //如果不对,会让 leader 把 next-1 重新发,知道匹配上为止
	if m.LogIndex < r.log.committed {
		resp.LogIndex = r.log.committed
		r.send(resp)
		return
	}
  //查看leader 的上一笔信息和本地 commited 的是不是相同,如果不同,需要让 leader 的 next-1 重新发
	if r.log.matchTerm(m.LogIndex, m.LogTerm) {
    //如果相同,根据日志匹配原则,就可以把数据存下来了. 把匹配索引之后的垃圾数据全部删除
    //比如有人当选了 leader 之后,发了一堆乱七八糟的数据,然后挂了,别人当选了 leader,发来了信息
    //这个时候这个节点就要以人家的 leader 数据为准,把自己多余的乱七八糟的数据都删除掉.
		r.log.tryAppend(m.LogIndex, m.Entries)
		lastIdx := m.LogIndex + uint64(len(m.Entries))
    //一个消息不是白发送的,他携带了leader 当前已经 commit到了哪里的信息,这里取俩个中最小值,向前推进.
    //这个信息同样在心跳中携带. 
    //既然日志都匹配了,也都存盘了,就可以进行 commit 了.
		r.log.commitTo(min(lastIdx, m.Commit)) 
    //告诉 leader,本节点处理到了 lastIdx 了,下次从这开始发
		resp.LogIndex = lastIdx
	} else {
    //说明是过期的数据,直接拒绝, 顺便告诉 leader, 本节点的日志才commited 到 lastIndex 这里,下次
    //从这个位置再发一次试试.
		plog.Warningf("%s rejected Replicate index %d term %d from %s",
			r.describe(), m.LogIndex, m.Term, NodeID(m.From))
		resp.Reject = true
		resp.LogIndex = m.LogIndex //m.LogIndex是 leader 的上一笔提交 index.
		resp.Hint = r.log.lastIndex() //本地最新的一笔索引
	}
	r.send(resp)
}

```



## leader 接受回复消息

同理,会触发到 leader 的`handleLeaderReplicateResp`方法:

```go
//leader 接受其它节点的回复消息
func (r *raft) handleLeaderReplicateResp(m pb.Message, rp *remote) {
	rp.setActive()
	if !m.Reject {
    //如果接收到的是成功消息,准备向前推进,但是需要注意,不能提交上个任期的数据.
		paused := rp.isPaused()
    //更新该节点的 next 和 match 属性.
		if rp.tryUpdate(m.LogIndex) {
			rp.respondedTo()
      //tryCommit做了件什么事情. 首先本 raft 节点修正个各 follower 节点的 match 情况.
      //然后对 follower 节点的 match 排序一下. 比如 10,11,12,13,14
      //q := r.matched[len(r.remotes)-r.quorum()] 取第 3 位 12,也就是说 12 之后的都被确定了
      //尝试提交, 论文中有提到,不能直接提交上个任期的数据.
			if r.tryCommit() {
        //leader 节点已经成功的commited 了一条消息.
        //提交成功后发一个广播,给节点更新.也可以伴随心跳下去吧??
        //因为上次 leader 发出去replicate 消息的时候,100 条可能只发了 30 条,这里继续发.
				r.broadcastReplicateMessage()
			} else if paused {
				r.sendReplicateMessage(m.From)
			}
			// according to the leadership transfer protocol listed on the p29 of the
			// raft thesis
			if r.leaderTransfering() && m.From == r.leaderTransferTarget &&
				r.log.lastIndex() == rp.match {
        //leader迁移的时候,目标节点立刻超时,具体见论文.
				r.sendTimeoutNowMessage(r.leaderTransferTarget)
			}
		}
	} else {
    //接受到了拒绝消息, 然后需要将 next-1,或者 match+1 继续发.
		// the replication flow control code is derived from etcd raft, it resets
		// nextIndex to match + 1. it is thus even more conservative than the raft
		// thesis's approach of nextIndex = nextIndex - 1 mentioned on the p21 of
		// the thesis.
		if rp.decreaseTo(m.LogIndex, m.Hint) {
			r.enterRetryState(rp)
			r.sendReplicateMessage(m.From)
		}
	}
}
```



## follower 更新本地 commited

Leader进行过多数派确认之后,commited 了一个 index, 那么需要让 follower 知道, 这些信息通过心跳或者广播消息,最终到点 follower, 下面看 follower 怎么处理的.

```go
//这是一部分,随着心跳的信息,向前推进commit, 另外上面也提到了handleReplicateMessage的时候也会推进
func (r *raft) handleHeartbeatMessage(m pb.Message) {
	r.log.commitTo(m.Commit)
	r.send(pb.Message{
		To:       m.From,
		Type:     pb.HeartbeatResp,
		Hint:     m.Hint,
		HintHigh: m.HintHigh,
	})
}
```



之前介绍过执行引擎, 会不断的检查节点是否有更新事件. 这次 commited 更新了, 大于了 apply 的.就会产生一个事件`hasMoreEntriesToApply `, 执行引擎就会去加入到 task 队列中,让状态机执行. 最后解除 proposal 的 channel 的等待状态, 返回给nodehost.SyncPropose方法结果.





## 总结一下

复盘一下整个的日志复制流程: 

1. 节点接收客户端链接,发起 proposal 请求 . 如果当前节点不是 leader, 就转发到 leader 上去执行. 
2. leader 节点会在内存中缓存一部分的请求, 在一定的时间间隔或者到达某个阈值后,统一发起一笔replicated消息
3. follower 接收到 replicated 消息, 进行投票:
   1. 如果满足日志匹配原则的,就同意投票, 删除自己多余的数据, 返回给 leader 当前的处理进度
   2. 如果不满足日志匹配原则,就拒绝投票, 告诉 leader 自己当前 match 的 index, leader 会重新发送.
4. leader 收到 follower 的返回信息
   1. 如果是拒绝消息,就根据节点的 match 信息+1, 继续复制日志.直到这个节点成功为止. 
   2. 如果是成功消息, 注意不能commite 上个任期的数据, 应该是本任期内的第一笔确认了之后, 再根据日志匹配原则, 把内容复制到其他节点.
5. follower 通过接收心跳或者复制消息,更新本地的 commited 记录
6. 执行引擎发现有更新,执行 update, 到状态机
7. 状态机执行完成之后, 解除 channel 的阻塞.





