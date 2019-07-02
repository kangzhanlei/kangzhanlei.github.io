---
title: dragonboat学习笔记-架构模型
date: 2019-07-01 14:45:11
tags:
	- raft
	- dragonboat
	- 源码分析
keywords: dragonboat,一致性协议,raft,源码分析
categories:
	- 一致性协议
---

# 整体结构

![](dragonboat-study-1/dragonboat.png)



# 功能模块介绍

## NodeHost

* 维护raft组，每个raft组对应一个clusterID
* 维护配置信息
* 设置执行引擎
* 设置存储logdb
* 设置net通讯组件
* 设置消息处理器
* 性能取样统计等

## raft

* leader选举
* 日志复制
* 集群变更
* 线性读取

##  execEngine

* 节点加载，下线
* 抓取快照，传输快照，恢复快照
* 执行taskworkerMain, 触发状态机
* 执行nodeWorkerMain, 每隔一定的tick时间，执行execNode事件



# 启动流程

##  启动步骤

1. 构造配置信息: config.Config

   * 设置clusterID，每个raft组都分配一个固定的clusterID，各个raft组的数据互相不冲突。
   * 设置ElectionRTT，选举的round trip time，就是消息延迟，以这个作为逻辑时钟的依据
   * 设置heartbeatRTT， 就是每隔多少个RTT时间，发送一次心跳
   * 设置CheckQuorum，在读取的时候，是否要做一次leader地位的确认，就是检查消息是否能达到大多数节点并得到回应
   * 设置SnapshotEntries，当到达多少个entry的时候，生成一次快照
   * 设置CompactionOverhead，表示raft快照之后，剩余多少个entry被保留到日志中

2. 构造NodeHostConfig

   * 设置WAL的存储路径  (TODO)
   * 设置NodeHost的存储路径,也就是logdb的存储
   * 设置RTTMillisecond，设置平均延迟在节点之间，用毫秒表示，也就是逻辑时钟的依据
   * 设置raft节点的组成信息

3. 构造`dragonboat.NewNodeHost(config)`, 得到NodeHost对象

   * 构造NodeHost对象
   * 构造snapshotFeedback
   * 创建消息的回调函数处理器，所有的网络请求都会走到这个处理器来处理
   * 为每一个clusterID，创建一个MessageQueue，用来存放网络传递进来的消息
   * 创建RequestResult对象池，因为请求数量很多，可以用池化减少对象的分配
   * 创建Transport，用来接收网络请求
   * 创建logdb
   * 创建执行引擎ExecEngine
     * 启动n个workerMain的goroutine
       * 每隔一定的逻辑时钟，执行一次execNodes调用
       * 当有事件发生的时候，执行一次execNodes调用, 具体作用后续描述
     * 启动n个taskWorkerMain的goroutine，作用同上
     * 启动n个snapshotWorkerMain的goroutine，作用同上
   * 启动goroutine，每隔1毫秒，执行一次tickWorkerMain,直到达到一次`RTTMillisecond`时间间隔之后，发起一次TickMessage调用，这里充当逻辑时钟的功能。消息类型为：`LocalTick`

4. 调用`nodeHost.StartCluster` 启动raft

   * bootstrapCluster （TODO ）
   * 创建消息队列,多组raft共享同一个消息队列
   * 创建快照目录
   * 创建snapshotter，来抓取快照
   * 根据clusterID，创建状态机工厂
   * 根据clusterID，创建Node实例。也就是说每个clusterID，对应着一个node的实例结构体
     * 根据配置的最大长度等参数，创建`EntryQueue`,存储entry的队列
     * 根据参数，创建`ReadIndexQueue`，用于存储ReadIndex的队列
     * 创建PendingProposal实例，用于积攒proposal请求
     * 创建PendingReadIndex实例，用于积攒ReadIndex的请求
     * 创建PendingConfigChange实例，用于积攒ConfigChange请求
     * 创建PendingSnapshot实例，用于积攒snapshot请求
     * 创建logReader，用于读取logdb的内容
     *  创建状态机
     * 开始调用`raft.startRaft` 启动raft

5. 启动goroutine，开始读取或者发起请求。

   

## 网络请求路径

nodehost是raft的大总管，启动了网络端口监听，接收客户端的请求之后，根据请求的clusterID，找到对应的MessageQueue，并把请求加入到队列中，并发出nodeReady事件。

执行引擎每隔一个tick时间，或者接收到nodeReady事件之后，执行execNode方法，判断是否产生了变更Update, 如果有update，则需要执行变更。



# 执行引擎

执行引擎execEngine是最重要的模块，主要执行node事件和状态机。其本质是检查所有的raft组是否有变化发生，如果有就处理，没有变更，就等待下次的时间片到期或者事件触发的时候，再重复调用。

关键代码如下:

```go
func (s *execEngine) execNodes(workerID uint64,
	clusterIDMap map[uint64]struct{},
	nodes map[uint64]*node, stopC chan struct{}) {
  ...
  //遍历所有的cluster
for cid := range clusterIDMap {
		node, ok := nodes[cid]
		if !ok {
			continue
		}
  	//对每一个cluster，都要检查是否有update发生
		ud, hasUpdate := node.stepNode()
		if hasUpdate {
			if !pb.IsEmptySnapshot(ud.Snapshot) {
				hasSnapshot = true
			}
      //如果有更新，增加到队列中
			nodeUpdates = append(nodeUpdates, ud)
		}
	}
  
  ... 
  
  // see raft thesis section 10.2.1 on details why we send Relicate message
	// before those entries are persisted to disk
  //如果有变更，先把请求发送给follower，同时自己落盘保存，只要收到的信息大于大多数，就是安全的。并发提高了效率
	for _, ud := range nodeUpdates {
		node := nodes[ud.ClusterID]
		node.sendReplicateMessages(ud)
		node.processReadyToRead(ud)
	}
  
  ...
  //落盘
  if err := s.logdb.SaveRaftState(nodeUpdates, nodeCtx); err != nil {
		panic(err)
	}
  
  //处理
  	for _, ud := range nodeUpdates {
		node := nodes[ud.ClusterID]
    //执行变更，如果是状态机变更，就交给状态机的goroutine执行了
		cont, err := node.processRaftUpdate(ud)
		if err != nil {
			panic(err)
		}
		if !cont {
			plog.Infof("process update failed, %s is ready to exit", node.id())
		}
		s.processMoreCommittedEntries(ud)
		if tests.ReadyToReturnTestKnob(stopC, false, "committing updates") {
			return
		}
		node.commitRaftUpdate(ud)
	}
}
```



以上，是execNode的全部流程，下面具体分解一下各步骤都做了些什么事情。

## stepNode

主要作用是检查当前的Node是否有Update发生。

```go
func (n *node) stepNode() (pb.Update, bool) {
	n.raftMu.Lock()
	defer n.raftMu.Unlock()
  //如果节点没有初始化，返回空
	if n.initialized() {
    //判断是否有event需要处理，有则返回true，去获取更新内容。没有事件就返回空对象
		if n.handleEvents() {
      //是否是静默模式,发送请求给他，这段代码只执行一次
			if n.newQuiesceState() {
				n.sendEnterQuiesceMessages()
			}
      //获取node的更新
			return n.getUpdate()
		}
	}
	return pb.Update{}, false
}
```





这里解释一下什么是静默模式：([静默模式](https://github.com/brpc/braft/blob/master/docs/cn/raft_protocol.md))

>  RAFT的Leader向Follower的心跳间隔一般都较小，在100ms粒度，当复制实例数较多的时候，心跳包的数量就呈指数增长。通常复制组不需要频繁的切换Leader，我们可以将主动Leader Election的功能关闭，这样就不需要维护Leader Lease的心跳了。复制组依靠业务Master进行被动触发Leader Election，这个可以只在Leader节点宕机时触发，整体的心跳数就从复制实例数降为节点数。社区还有一种解决方法是[MultiRAFT](http://www.cockroachlabs.com/blog/scaling-RAFT/)，将复制组之间的心跳合并到节点之间的心跳。



重点关注`handleEvents`函数和`getUpdate`函数

```go
func (n *node) handleEvents() bool {
 //初始化时没有事件
	hasEvent := false
  //获取当前日志里最后一个apply的索引，更新n.smAppliedIndex
	lastApplied := n.updateBatchedLastApplied()
  //因为状态机执行完毕之后会更新smAppliedIndex的值，confirmedIndex表示node节点已经执行的值，如果confirmed大于
  // appliedIndex,说明这个节点TODO
	if lastApplied != n.confirmedIndex {
		hasEvent = true
	}
  //判断当前log队列中，已经commited，待apply的entry是不是还有，如果有的话，返回true，继续处理
	if n.hasEntryToApply() {
		hasEvent = true
	}
  //判断有没有挂起的ReadIndex请求，还在队列中(之前readindex的请求都加入到了队列中，这里会检查)
  //如果队列中有数据，那么increaseReadReqCount,计数器加1，后续会用到
	if n.handleReadIndexRequests() {
		hasEvent = true
	}
  //检查有没有从网络上收到的消息，如果有，则需要处理
  //1. 遍历所有的消息，如果是LocalTick消息，则计数器累加就可以了(因为每个1个tick执行一次时钟事件，累加就表示时钟了)
  //2. 消息类型是replicate消息或者当前正在处理快照，本次就不管了，因为快照处理完了就可以了，不需要再replicate了
  //3. 消息类型是Quiesce的情况下，tryEnterQuiesec进去静默模式,等等
  //4. 交给raft去处理raft消息,这个后续详细描述
  //5. 如果上面的ReadReqCount>0，表示有ReadIndex请求，这里构造一个pb消息，直接让raft根据消息类型ReadIndex执行
  //6. 如果是LocalTick消息，表示逻辑时钟。对pendingSnapshot,PendingProposals,
  //   PendingReadIndexes,PendingConfigChange都进行tick自增，表示时钟流逝了。
  //   因为时钟滴滴答答的向前走，raft也会进行tick处理，让当前的状态机执行LocalTck事件。状态机会根据当前所在的节点是否是leader，来决定leaderTick还是nonLeaderTick，nonLeadertick会判断当前的时间是否超过了election的时候，然后就会发起election消息，竞选leader
	if n.handleReceivedMessages() {
		hasEvent = true
	}
  //发送Propose事件，设置configChangeEntry，表示节点信息变更，有C-old和C-New共同作用。
	if n.handleConfigChangeMessage() {
		hasEvent = true
	}
  //查看当前node的proposal队列中是否有请求，如果有请求，则进行raft的proposal请求处理模块。
  //上面处理的是节点的接受消息，这里处理的是节点向外发送消息。俩个的优先级明显是别人的消息优先。
	if n.handleProposals() {
		hasEvent = true
	}
 //是否有快照的请求，TODO
	if n.handleSnapshotRequest(lastApplied) {
		hasEvent = true
	}
	if hasEvent {
    //挂起的请求如果很久都没有处理了，不能一直留在内存中，这里隔一定的时间之后，要处理掉。
		if n.expireNotified != n.tickCount {
			n.pendingProposals.gc()
			n.pendingConfigChange.gc()
			n.pendingSnapshot.gc()
			n.expireNotified = n.tickCount
		}
    //当状态执行完毕之后，别忘记了，还有人在等待着状态机的执行结果呢，这个就是ReadIndex，当处理到了ReadIndex的时候
    //要通知pendingReadIndexes对象，告诉他我已经apply到XXX的索引了，你该返回就返回，没有到的时候接着等。主要用来保证
    //线性一致性的。具体描述见其他文章。
		n.pendingReadIndexes.applied(lastApplied)
	}
	return hasEvent
}

```

执行引擎就是不停的重复上面的动作，来发现并处理事件。

处理完毕之后，要获取下是否有需要执行的变化。具体来看代码：

```go
//获取本次execNode时候，是否产生了变化的数据
func (n *node) getUpdate() (pb.Update, bool) {
  //判断当前状态机中是否有任务要执行
	moreEntriesToApply := n.canHaveMoreEntriesToApply()
  //1. 获取当前raft的状态和记录的上一个状态是不是一致(term,vote,commit)，不一致说明有执行了新的apply，
  //   需要更新给其他节点 
  //2. 快照 TODO
  //3. 判断当前node的msgs是否大于0，这个msgs表示待向外发送的pb消息，这是一个队列
  //4. 判断当前在内存中是否缓存了Entry，有的话也需要处理
  //5. 判断当前是否有readyToRead的消息，如果有，也需要处理，这个主要是ReadIndex协议有结果了，加到队列中来的。
	if n.p.HasUpdate(moreEntriesToApply) ||
		//在上次执行完变更之后，状态机又有了新的变化，这些是要同步给别人的。
		n.confirmedIndex != n.smAppliedIndex {
    //这基本不会发生吧。判断这个干啥
		if n.smAppliedIndex < n.confirmedIndex {
			plog.Panicf("last applied value moving backwards, %d, now %d",
				n.confirmedIndex, n.smAppliedIndex)
		}
    //获取变更的信息具体信息，包含了clusterID,messages,nodeID,lastAppliedIndex
		ud := n.p.GetUpdate(moreEntriesToApply, n.smAppliedIndex)
		for idx := range ud.Messages {
			ud.Messages[idx].ClusterId = n.clusterID
		}
    //把confirmedIndex修改为applyIndex，记录下来：我上次给别人发送的update是从这里开始的
    //下次一比对就知道有没有更新了
		n.confirmedIndex = n.smAppliedIndex
		return ud, true
	}
	return pb.Update{}, false
}
```

`Update`是一个需要变更的集合信息。包含的主要内容有：

* clusterID , NodeID
* State ，当前raft节点的状态，他必须在发送给其他节点之前被持久化下来。
* EntriesToSave， 准备写到logdb里的日志条目们
* CommitedEntries, 已经commited的日志，还没有被状态机应用
* MoreCommitedEntries, 表示这是否还有更多的committed entries准备被apply
* snapshot，快照被apply
* ReadyToRead，表示一个准备被本地读取的一个ReadIndex的请求结果信息
* Message ，向其他节点发送的消息的数组
* LastApplied，表示被状态机执行的最后一个index
* UpdateCommit 描述如何commit这个update，从而推进raft的状态变更

## SendReplicateMessages &&SaveRaftState

得到变更的消息之后，可以先不必落盘，而是先并行的发送给其他节点，这样可以做到网络和磁盘同时进行。为什么不落地直接发送是安全的？因为最终也是要判断是否大多数节点收到了这个消息，如果收到了，即使主节点挂了，依然可以通过选取新主来得到这个commited的数据的。



## processRaftUpdate

主要进行日志压缩，然后`runSyncTask`来进行状态机的改变

```go
func (n *node) runSyncTask() (bool, error) {
	if !n.sm.OnDiskStateMachine() {
		return true, nil
	}
	if n.syncTask == nil ||
		!n.syncTask.timeToRun(n.millisecondSinceStart()) {
		return true, nil
	}
	plog.Infof("%s is running sync task", n.id())
	if !n.sm.TaskChanBusy() {
    //把状态机的任务，增加到队列中去执行 . 执行引擎execEngine会执行这个任务。
		task := rsm.Task{PeriodicSync: true}
		if !n.pushTask(task) {
			return false, nil
		}
	}
  //快照的部分，后续梳理
	syncedIndex := n.sm.GetSyncedIndex()
	if err := n.shrinkSnapshots(syncedIndex); err != nil {
		return false, err
	}
	return true, nil
}
```



##  状态机执行

跟上面的`nodeWorkerMain`一样，`taskWorkerMain` 也是定时或者消息机制通知后，开始执行`execSMs`方法，具体代码如下：

```go
func (s *execEngine) execSMs(workerID uint64,
	idmap map[uint64]struct{},
	nodes map[uint64]*node, batch []rsm.Task, entries []sm.Entry) {
	if len(idmap) == 0 {
		for k := range nodes {
			idmap[k] = struct{}{}
		}
	}
	var p *profiler
	if workerCount == taskWorkerCount {
		p = s.profilers[workerID-1]
		p.newCommitIteration()
		p.exec.start()
	}
  //idmap中存的都是节点的cluster，表示哪个节点有事件了
	for clusterID := range idmap {
		node, ok := nodes[clusterID]
		if !ok || node.stopped() {
			continue
		}
    //处理快照状态，先不关注
		if node.processSnapshotStatusTransition() {
			continue
		}
    //让node去执行任务，传入的是一个batch，表示积攒的Task，entries表示本次所有的请求条目
		task, err := node.handleTask(batch, entries)
		if err != nil {
			panic(err)
		}
		if task.IsSnapshotTask() {
			node.handleSnapshotTask(task)
		}
	}
	if p != nil {
		p.exec.end()
	}
}
```

继续来看`node.handleTask`的实现, 直接交由状态执行`Handle`方法: 

```go
func (s *StateMachine) Handle(batch []Task, apply []sm.Entry) (Task, error) {
	batch = batch[:0]
	apply = apply[:0]
	processed := false
	defer func() {
		// give the node worker a chance to run when
		//  - batched applied value has been updated
		//  - taskC has been poped
		if processed {
			s.node.NodeReady()
		}
	}()
  //获取任务
	rec, ok := s.taskQ.Get()
	if ok {
		processed = true
    //是快照任务的话，返回，外面有对快照处理的方法
		if rec.IsSnapshotTask() {
			return rec, nil
		}
    //如果不是同步任务，就积攒到一个batch中，等待执行
		if !rec.isSyncTask() {
			batch = append(batch, rec)
		} else {
      //如果是同步任务，则落盘保存
			if err := s.sync(); err != nil {
				return Task{}, err
			}
		}
		done := false
		for !done {
			rec, ok := s.taskQ.Get()
			if ok {
				if rec.IsSnapshotTask() {
					if err := s.handle(batch, apply); err != nil {
						return Task{}, err
					}
					return rec, nil
				}
				if !rec.isSyncTask() {
					batch = append(batch, rec)
				} else {
					if err := s.sync(); err != nil {
						return Task{}, err
					}
				}
			} else {
				done = true
			}
		}
	}
  //在这里handle
	return Task{}, s.handle(batch, apply)
}
```



继续看状态机的handle方法

```go

func (s *StateMachine) handle(batch []Task, toApply []sm.Entry) error {
	batchSupport := batchedEntryApply && s.ConcurrentSnapshot()
	for b := range batch {
		if batch[b].IsSnapshotTask() || batch[b].isSyncTask() {
			plog.Panicf("%s trying to handle a snapshot/sync request", s.id())
		}
		input := batch[b].Entries
		allUpdate, allNoOP := getEntryTypes(input)
		if batchSupport && allUpdate && allNoOP {
      //判断这个entry是不是configChange，sessionManager，newSessionRequest，EndOfSessionRequest
			if err := s.handleBatch(input, toApply); err != nil {
				return err
			}
		} else {
			for i := range input {
				last := b == len(batch)-1 && i == len(input)-1
        //处理entry
				if err := s.handleEntry(input[i], last); err != nil {
					return err
				}
			}
		}
	}
	return nil
}
```

```go
func (s *StateMachine) handleEntry(ent pb.Entry, last bool) error {
	// ConfChnage also go through the SM so the index value is updated
	if ent.IsConfigChange() {
		accepted := s.handleConfigChange(ent)
		s.node.ConfigChangeProcessed(ent.Key, accepted)
	} else {
		if !ent.IsSessionManaged() {
			if ent.IsEmpty() {
				s.handleNoOP(ent)
				s.node.ApplyUpdate(ent, sm.Result{}, false, true, last)
			} else {
				panic("not session managed, not empty")
			}
		} else {
			if ent.IsNewSessionRequest() {
				smResult := s.handleRegisterSession(ent)
				s.node.ApplyUpdate(ent, smResult, isEmptyResult(smResult), false, last)
			} else if ent.IsEndOfSessionRequest() {
				smResult := s.handleUnregisterSession(ent)
				s.node.ApplyUpdate(ent, smResult, isEmptyResult(smResult), false, last)
			} else {
				if !s.entryAppliedInDiskSM(ent.Index) {
          //执行更新状态机的动作,得到返回信息
					smResult, ignored, rejected, err := s.handleUpdate(ent)
					if err != nil {
						return err
					}
					if !ignored {
            //解除readIndex的阻塞
            //解除proposal的阻塞
						s.node.ApplyUpdate(ent, smResult, rejected, ignored, last)
					}
				} else {
					// treat it as a NoOP entry
					s.handleNoOP(pb.Entry{Index: ent.Index, Term: ent.Term})
				}
			}
		}
	}
	index := s.GetLastApplied()
	if index != ent.Index {
		plog.Panicf("unexpected last applied value, %d, %d", index, ent.Index)
	}
	if last {
    //设置状态机最后更新的index
		s.setBatchedLastApplied(ent.Index)
	}
	return nil
}

```

