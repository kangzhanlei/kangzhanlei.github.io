---
title: dragonboat学习笔记-快照
date: 2019-07-05 16:21:12
tags:
	- raft
	- dragonboat
	- 源码分析
	- go
	- 一致性协议
categories:
	- 一致性协议
---

`nodeHost`中提供了`RequestSnapshot` 的方法. 以此为入口, 来分析快照是如何操作的.

```go
func (nh *NodeHost) RequestSnapshot(clusterID uint64,
	opt SnapshotOption, timeout time.Duration) (*RequestState, error) {
  //每个 raft 组都有自己的快照
	v, ok := nh.getCluster(clusterID)
	if !ok {
		return nil, ErrClusterNotFound
	}
  //发起请求,加入到事件队列
	req, err := v.requestSnapshot(opt, timeout)
  //通知执行引擎有事件到来
	nh.execEngine.setNodeReady(clusterID)
	return req, err
}
```

继续来看是怎么处理的

```go
//发起快照请求
func (n *node) requestSnapshot(opt SnapshotOption,
	timeout time.Duration) (*RequestState, error) {
	st := rsm.UserRequestedSnapshot
	if opt.Exported {
		plog.Infof("export snapshot called on %s", n.id())
		st = rsm.ExportedSnapshot
		exist, err := fileutil.Exist(opt.ExportPath)
		if err != nil {
			return nil, err
		}
		if !exist {
			return nil, ErrDirNotExist
		}
	} else {
		if len(opt.ExportPath) > 0 {
			plog.Warningf("opt.ExportPath set when not exporting a snapshot")
			opt.ExportPath = ""
		}
	}
  //加入到pendongSnapshot队列,跟之前代码分析一样, 最终会有执行引擎去处理. 
	return n.pendingSnapshot.request(st, opt.ExportPath, timeout)
}

```

执行引擎每隔一定的时间会执行node.handleEvents方法来处理 node 的事件. 直接来看关于 snapshot 的部分

```go
//处理快照请求
func (n *node) handleSnapshotRequest(lastApplied uint64) bool {
	var req rsm.SnapshotRequest
	select {
	case req = <-n.snapshotC:
	default:
		return false
	}
  //获取当前快照的索引是不是和 commited 是一致的,如果一样了就没有必要取快照了,因为该有的我都有了
	si := n.ss.getReqSnapshotIndex()
	if lastApplied == si {
		n.reportIgnoredSnapshotRequest(req.Key)
		return false
	}
  //设置是为了获取
	n.ss.setReqSnapshotIndex(lastApplied)
  //请求又加入队列 , 代码见下面
	n.pushTakeSnapshotRequest(req)
	return true
}

func (n *node) pushTakeSnapshotRequest(req rsm.SnapshotRequest) bool {
  //产生一个关于状态机的事件, 然后加入到队列中
	rec := rsm.Task{SnapshotRequested: true, SnapshotRequest: req}
	return n.pushTask(rec)
}
// taskQ是状态机的队列, 执行引擎每隔一段时间或者事件触发的时候都会执行
func (n *node) pushTask(rec rsm.Task) bool {
	n.taskQ.Add(rec)
	n.engine.setTaskReady(n.clusterID)
	return !n.stopped()
}
```



下面就会执行到执行引擎的`execSMs`方法, 主要处理状态机的事情

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
	for clusterID := range idmap {
		node, ok := nodes[clusterID]
		if !ok || node.stopped() {
			continue
		}
		if node.processSnapshotStatusTransition() {
			continue
		}
		task, err := node.handleTask(batch, entries)
		if err != nil {
			panic(err)
		}
    //这里判断是不是快照任务
		if task.IsSnapshotTask() {
      //见下面的分析
			node.handleSnapshotTask(task)
		}
	}
	if p != nil {
		p.exec.end()
	}
}
```

跟进来看

```go
func (n *node) handleSnapshotTask(task rsm.Task) {
  //如果当前正在处理快照了, 这里不重复执行
	if n.ss.recoveringFromSnapshot() {
		plog.Panicf("recovering from snapshot again")
	}
  //
	if task.SnapshotAvailable {
		plog.Infof("check incoming snapshot, %s", n.id())
		n.reportAvailableSnapshot(task)
	} else if task.SnapshotRequested {
    //根据任务类型, 走到这里
		plog.Infof("reportRequestedSnapshot, %s", n.id())
    //进行快照采集
		if n.ss.takingSnapshot() {
			plog.Infof("task.SnapshotRequested ignored on %s", n.id())
			n.reportIgnoredSnapshotRequest(task.SnapshotRequest.Key)
			return
		}
    //请求抓取快照
		n.reportRequestedSnapshot(task)
	} else if task.StreamSnapshot {
		if !n.canStreamSnapshot() {
			n.reportSnapshotStatus(task.ClusterID, task.NodeID, true)
			return
		}
		n.reportStreamSnapshot(task)
	} else {
		panic("unknown returned task rec type")
	}
}
```

来看如何抓取的快照:

```go
func (n *node) reportRequestedSnapshot(rec rsm.Task) {
  //设置标示,意思是当前正在抓取, 再有请求来了就可以根据标志拦截了
	n.ss.setTakingSnapshot()
  //通知可以抓取快照了. 来看这个的实现.
	n.ss.setSaveSnapshotReq(rec)
  // 事件通知,通知执行引擎来执行任务啦
	n.engine.setRequestedSnapshotReady(n.clusterID)
}

func (rs *snapshotState) setSaveSnapshotReq(t rsm.Task) {
  //这里比较简单, 就是向snapshotReady中设置一个任务, 那这个任务什么时候被执行呢? 
  //之前前文也提到了 ,执行引擎开启了 3 个定时任务
  // 1. nodeWorkerMain 观察节点的普通消息请求
  // 2. taskWorkerMain 观察节点的状态机事件
  // 3. snapshotWorkerMain 观察是否有抓取快照的请求. 
  // 这里第 3个的请求就体现了, 也就是每隔一定的时间, 就会检查snapshotReady中是否有抓取快照的请求
  // 发现请求到了, 就可以取出 task 来, 然后 saveSnapshot, 下面来看具体代码
	rs.saveSnapshotReady.setTask(t)
}

```



首先来看执行引擎的`snapshowtWorkerMain`方法: 

```go
func (s *execEngine) snapshotWorkerMain(workerID uint64) {
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()
	nodes := make(map[uint64]*node)
	cci := uint64(0)
	for {
		select {
    //snapshooter抓手的停止事件
		case <-s.snapshotStopper.ShouldStop():
			s.offloadNodeMap(nodes, rsm.FromSnapshotWorker)
			return
		case <-ticker.C:
      //定时检查是否有快照请求,然后抓取
			nodes, cci = s.loadSnapshotNodes(workerID, cci, nodes)
			for _, node := range nodes {
				s.recoverFromSnapshot(node.clusterID, nodes)
				s.saveSnapshot(node.clusterID, nodes)
				if s.snapshotWorkerClosed(nodes) {
					return
				}
			}
		case <-s.snapshotWorkReady.waitCh(workerID):
      // 当前快照准备完毕了, 可以应用快照了
			clusterIDMap := s.snapshotWorkReady.getReadyMap(workerID)
			for clusterID := range clusterIDMap {
				nodes, cci = s.loadSnapshotNodes(workerID, cci, nodes)
				s.recoverFromSnapshot(clusterID, nodes)
				if s.snapshotWorkerClosed(nodes) {
					return
				}
			}
		case <-s.requestedSnapshotWorkReady.waitCh(workerID):
      //收到请求,开始抓取快照
			clusterIDMap := s.requestedSnapshotWorkReady.getReadyMap(workerID)
			for clusterID := range clusterIDMap {
				nodes, cci = s.loadSnapshotNodes(workerID, cci, nodes)
				s.saveSnapshot(clusterID, nodes)
				if s.snapshotWorkerClosed(nodes) {
					return
				}
			}
		case <-s.streamSnapshotWorkReady.waitCh(workerID):
      //可以传输快照的信号来了, 开始传输
			clusterIDMap := s.streamSnapshotWorkReady.getReadyMap(workerID)
			for clusterID := range clusterIDMap {
				nodes, cci = s.loadSnapshotNodes(workerID, cci, nodes)
				s.streamSnapshot(clusterID, nodes)
				if s.snapshotWorkerClosed(nodes) {
					return
				}
			}
		}
	}
}
```

重点来看saveSnapshot这段

```go
func (s *execEngine) saveSnapshot(clusterID uint64, nodes map[uint64]*node) {
	node, ok := nodes[clusterID]
	if !ok {
		return
	}
  //获取请求的任务的结构体
	rec, ok := node.ss.getSaveSnapshotReq()
	if !ok {
		return
	}
	plog.Infof("%s called saveSnapshot", node.id())
  // 抓取, 下面是具体代码
	if err := node.saveSnapshot(rec); err != nil {
		panic(err)
	} else {
		node.saveSnapshotDone()
	}
}

func (n *node) saveSnapshot(rec rsm.Task) error {
  //真正的抓取 ,下面的代码重点来看.
	index, err := n.doSaveSnapshot(rec.SnapshotRequest)
	if err != nil {
		return err
	}
  //通知等待抓取结果的
	n.pendingSnapshot.apply(rec.SnapshotRequest.Key, index == 0, index)
	return nil
}
```



看如何抓取的快照: 

```go

func (n *node) doSaveSnapshot(req rsm.SnapshotRequest) (uint64, error) {
	n.snapshotLock.Lock()
	defer n.snapshotLock.Unlock()
  //如果当前状态机应用的索引,比请求的还小, 说明已经有镜像推送过来了还没有执行完
	if n.sm.GetLastApplied() <= n.ss.getSnapshotIndex() {
		// a snapshot has been pushed to the sm but not applied yet
		// or the snapshot has been applied and there is no further progress
		return 0, nil
	}
  //真正的抓取, 后面具体来看
	ss, ssenv, err := n.sm.SaveSnapshot(req)
	if err != nil {
		if err == sm.ErrSnapshotStopped {
      //抓取停止
			ssenv.MustRemoveTempDir()
			plog.Infof("%s aborted SaveSnapshot", n.id())
			return 0, nil
		} else if isSoftSnapshotError(err) || err == rsm.ErrTestKnobReturn {
			return 0, nil
		}
		plog.Errorf("%s SaveSnapshot failed %v", n.id(), err)
		return 0, err
	}
  //测试的代码, 可以忽略
	if tests.ReadyToReturnTestKnob(n.stopc, true, "snapshotter.Commit") {
		return 0, nil
	}
	plog.Infof("%s snapshotted, index %d, term %d, file count %d",
		n.id(), ss.Index, ss.Term, len(ss.Files))
  //设置一些快照文件的各种属性
	if err := n.snapshotter.Commit(*ss, req); err != nil {
		plog.Errorf("%s Commit failed %v", n.id(), err)
		if err == errSnapshotOutOfDate {
			ssenv.MustRemoveTempDir()
			return 0, nil
		}
		// this can only happen in monkey test
		if err == sm.ErrSnapshotStopped {
			return 0, nil
		}
		return 0, err
	}
	if req.IsExportedSnapshot() {
		return ss.Index, nil
	}
	if !ss.Validate() {
		plog.Panicf("%s generated invalid snapshot %v", n.id(), ss)
	}
	if err = n.logreader.CreateSnapshot(*ss); err != nil {
		plog.Errorf("%s CreateSnapshot failed %v", n.id(), err)
		if !isSoftSnapshotError(err) {
			return 0, err
		}
		return 0, nil
	}
  //如果大于阈值, 开启压缩
	if ss.Index > n.config.CompactionOverhead {
		n.ss.setCompactLogTo(ss.Index - n.config.CompactionOverhead)
		if err := n.snapshotter.Compact(ss.Index); err != nil {
			plog.Errorf("%s snapshotter.Compact failed %v", n.id(), err)
			return 0, err
		}
	}
	n.ss.setSnapshotIndex(ss.Index)
	return ss.Index, nil
}
```

