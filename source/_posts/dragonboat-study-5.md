---
title: dragonboat学习笔记-集群变更
date: 2019-07-04 08:42:03
tags:
	- raft
	- 一致性协议
	- go
	- dragonboat
	- 源码分析
categories:
	- 一致性协议
---

raft论文的Chapter4介绍了有关集群变更需要注意的诸多细节. 主要从以下几点展开描述

# 安全性

在成员变更时,无法做到: 在同一个时刻, 让所有的节点都从旧配置切换到新配置. 

因为有的节点接收的消息比较快,有的接收的比较慢,  这样就会有一个问题: 假设现在有 ABC 三个节点构成的机器, 要向集群中增加 DE 两个节点, 那么在刚加入的时候, C 节点可能收到了变更配置的请求, 但是 AB可能因为网络原因还没收到, 就会造成, AB 认为这个集群只有 3 个节点, 只要 AB 投票了, 那么 A 就可以当选为 leader, 而 CDE 节点中,C认为有 5 个节点,他只需要拉拢 DE 就构成了多数派, 也变成了 leader. 

# 可用性

新加入的节点可能是空节点, 他需要复制很多日志到自己的机器上, 这个过程是很缓慢的. 也就就是论文中的catching up 

解决办法就是, 新加入的节点还不参与投票, 先充当 observer, 当他的日志差不多跟上了 leader了, 再开始变更



# 联合配置

下面给出解决集群配置的方法:

1. 创建一个 C-old 和 C-new 共同构成的配置信息, 作为一条 entry, 复制到大多数机器上. 
2. 竞选的时候, 需要拿到 C-old 的大多数, 同时也要拿到 C-new 的大多数, 才可以成为 leader 
3. leader 发送一条 C-new 日志条目到大多数节点

# 代码实现

下面来看代码实现 , 入口还是在`nodehost`中 , 先来看增加一个节点的流程

```go
// 增加一个节点, 参数分别为:
// 1. clusterID 表示给那个 raft 组增加节点
// 2. nodeID 新增的哪个节点
// 3. address 地址
// 4. configChangeIndex 可以先忽略,传0 或者通过SyncGetClusterMembership得到,后面再讲
// 5. timeout 超时时间
func (nh *NodeHost) RequestAddNode(clusterID uint64,
	nodeID uint64, address string, configChangeIndex uint64,
	timeout time.Duration) (*RequestState, error) {
  //获取 node
	v, ok := nh.getCluster(clusterID)
	if !ok {
		return nil, ErrClusterNotFound
	}
  //发起请求,加入到队列
	req, err := v.requestAddNodeWithOrderID(nodeID,
		address, configChangeIndex, timeout)
  //通知执行引擎有变化了, 执行引擎开始取数据发送出去
	nh.execEngine.setNodeReady(clusterID)
	return req, err
}

```

跟进`requestAddNodeWithOrderID`来看

```go
//发起增加 node 请求
func (n *node) requestAddNodeWithOrderID(nodeID uint64,
	addr string, orderID uint64, timeout time.Duration) (*RequestState, error) {
  //走配置变更
	return n.requestConfigChange(pb.AddNode,
		nodeID, addr, orderID, timeout)
}

func (n *node) requestConfigChange(cct pb.ConfigChangeType,
	nodeID uint64, addr string, orderID uint64,
	timeout time.Duration) (*RequestState, error) {
  //构造一个 ConfigChange 的请求消息 , 消息类型为: AddNode,removeNode,AddObserver
	cc := pb.ConfigChange{
		Type:           cct,
		NodeID:         nodeID,
		ConfigChangeId: orderID,
		Address:        addr,
	}
  //依然是加入到队列中. 
	return n.pendingConfigChange.request(cc, timeout)
}

```



执行引擎每隔一个 tick, 执行一次`n.handleEvents`, 其中就会对配置变更做检查

```go

func (n *node) handleConfigChangeMessage() bool {
	if len(n.confChangeC) == 0 {
		return false
	}
	select {
    //从队列中获取消息
	case req, ok := <-n.confChangeC:
		if !ok {
			n.confChangeC = nil
		} else {
			n.recordActivity(pb.ConfigChangeEvent)
			var cc pb.ConfigChange
			if err := cc.Unmarshal(req.data); err != nil {
				panic(err)
			}
      //调用 peer 的配置变更方法
			n.p.ProposeConfigChange(cc, req.key)
		}
	case <-n.stopc:
		return false
	default:
		return false
	}
	return true
}

```

下面来看 peer 的代码

```go
// ProposeConfigChange proposes a raft membership change.
func (rc *Peer) ProposeConfigChange(configChange pb.ConfigChange, key uint64) {
	data, err := configChange.Marshal()
	if err != nil {
		panic(err)
	}
  // 配置变更, 其实也是发起了一笔普通的 raft 日志请求, 类型是 ConfigChangeEntry
	rc.raft.Handle(pb.Message{
		Type:    pb.Propose,
		Entries: []pb.Entry{{Type: pb.ConfigChangeEntry, Cmd: data, Key: key}},
	})
}
```

也就是说, 当前节点就会执行到 raft 节点的函数转发这里, 还是要根据当前节点处于 follower 状态还是 leader 状态, 走不同的处理方式 , follower把请求都递转给 leader, leader 开启日志复制的流程, 等到收到了大多数之后, 执行状态机, 下面来看代码: 

```go
func (s *StateMachine) handleEntry(ent pb.Entry, last bool) error {
	// ConfChnage also go through the SM so the index value is updated
	if ent.IsConfigChange() {
    //状态机处理变更事件.下面具体来看
		accepted := s.handleConfigChange(ent)
		s.node.ConfigChangeProcessed(ent.Key, accepted)
	} 
  //其他代码已经分析过了,这里不做分析了,参考其他文章
  .....
}
```



```go
//状态机处理配置变更
func (s *StateMachine) handleConfigChange(ent pb.Entry) bool {
	var cc pb.ConfigChange
  //把命令反序列化, 得到 propose 时候的那个报文
	if err := cc.Unmarshal(ent.Cmd); err != nil {
		panic(err)
	}
  //如果是增加节点, 并没有提供地址,拉倒吧
	if cc.Type == pb.AddNode && len(cc.Address) == 0 {
		panic("empty address in AddNode request")
	}
	s.mu.Lock()
	defer s.mu.Unlock()
  //更新状态机的 term 和 index, 惯例,不用理会
	s.updateLastApplied(ent.Index, ent.Term)
  //在这里是核心代码 . 下面来看
	if s.members.handleConfigChange(cc, ent.Index) {
    //如果状态机执行成功, 那么就可以让 node 去添加节点信息了, 也就是把服务列表更新一下. 后续再看. 
		s.node.ApplyConfigChange(cc)
		return true
	}
	return false
}

```

到这里分 2 部分来看, 一部分是状态机如何执行这个指令, 另外一个就是状态机执行成功之后, 如何添加服务器信息的

1. 状态机执行

   直接来看`s.memebers.handleConfigChange`方法 

   ```go
   //状态机执行变更, 返回 true 表示同意添加节点, false 则不同意
   func (m *membership) handleConfigChange(cc pb.ConfigChange, index uint64) bool {
   	accepted := false
   	//用户发起的配置变更请求的 ID
   	ccid := cc.ConfigChangeId
     //判断你是不是AddObserver消息
   	nodeBecomingObserver := m.isAddingNodeAsObserver(cc)
     //判断是不是已经存在了节点
   	alreadyMember := m.isAddingExistingMember(cc)
     //判断是是不是增加已经被 removed 的机器
   	addRemovedNode := m.isAddingRemovedNode(cc)
     //判断本次请求的 ID 是不是和之前记录的 ID 相同,如果不相同就不是最新的配置更新
   	upToDateCC := m.isConfChangeUpToDate(cc)
     //判断是不是删除唯一节点
   	deleteOnlyNode := m.isDeletingOnlyNode(cc)
     //判断是不是 observer 节点提升到 follower 节点
   	invalidPromotion := m.isInvalidObserverPromotion(cc)
   	if upToDateCC && !addRemovedNode && !alreadyMember &&
   		!nodeBecomingObserver && !deleteOnlyNode && !invalidPromotion {
   		// current entry index, it will be recorded as the conf change id of the members
       // 执行变更,把新的服务器列表加到 memebership 中去.
   		m.applyConfigChange(cc, index)
   		if cc.Type == pb.AddNode {
   			plog.Infof("%s applied ConfChange Add ccid %d, node %s index %d address %s",
   				m.id(), ccid, logutil.NodeID(cc.NodeID), index, string(cc.Address))
   		} else if cc.Type == pb.RemoveNode {
   			plog.Infof("%s applied ConfChange Remove ccid %d, node %s, index %d",
   				m.id(), ccid, logutil.NodeID(cc.NodeID), index)
   		} else if cc.Type == pb.AddObserver {
   			plog.Infof("%s applied ConfChange Add Observer ccid %d, node %s index %d address %s",
   				m.id(), ccid, logutil.NodeID(cc.NodeID), index, string(cc.Address))
   		} else {
   			plog.Panicf("unknown cc.Type value")
   		}
   		accepted = true
   	} else {
   		if !upToDateCC {
   			plog.Warningf("%s rejected out-of-order ConfChange ccid %d, type %s, index %d",
   				m.id(), ccid, cc.Type, index)
   		} else if addRemovedNode {
   			plog.Warningf("%s rejected adding removed node ccid %d, node id %d, index %d",
   				m.id(), ccid, cc.NodeID, index)
   		} else if alreadyMember {
   			plog.Warningf("%s rejected adding existing member to raft cluster ccid %d "+
   				"node id %d, index %d, address %s",
   				m.id(), ccid, cc.NodeID, index, cc.Address)
   		} else if nodeBecomingObserver {
   			plog.Warningf("%s rejected adding existing member as observer ccid %d "+
   				"node id %d, index %d, address %s",
   				m.id(), ccid, cc.NodeID, index, cc.Address)
   		} else if deleteOnlyNode {
   			plog.Warningf("%s rejected removing the only node %d from the cluster",
   				m.id(), cc.NodeID)
   		} else if invalidPromotion {
   			plog.Warningf("%s rejected invalid observer promotion change ccid %d "+
   				"node id %d, index %d, address %s",
   				m.id(), ccid, cc.NodeID, index, cc.Address)
   		} else {
   			plog.Panicf("config change rejected for unknown reasons")
   		}
   	}
   	return accepted
   }
   
   ```

   

2. 来看 node 节点怎么处理

   ```go
   func (n *node) ApplyConfigChange(cc pb.ConfigChange) {
   	n.raftMu.Lock()
   	defer n.raftMu.Unlock()
     //执行一个 raft membership 的变更到本地 node 节点
     //其实是发送了一个ConfigChangeEvent的 raft 事件
   	n.p.ApplyConfigChange(cc)
   	switch cc.Type {
   	case pb.AddNode:
   		n.nodeRegistry.AddNode(n.clusterID, cc.NodeID, string(cc.Address))
   	case pb.AddObserver:
   		n.nodeRegistry.AddNode(n.clusterID, cc.NodeID, string(cc.Address))
   	case pb.RemoveNode:
   		if cc.NodeID == n.nodeID {
   			plog.Infof("%s applied ConfChange Remove for itself", n.id())
   			n.nodeRegistry.RemoveCluster(n.clusterID)
   			n.requestRemoval()
   		} else {
   			n.nodeRegistry.RemoveNode(n.clusterID, cc.NodeID)
   		}
   	default:
   		panic("unknown config change type")
   	}
   }
   
   // addnode 就是把当前的服务节点信息,加到Nodes集合中
   func (n *Nodes) AddNode(clusterID uint64, nodeID uint64, url string) {
   	n.mu.Lock()
   	defer n.mu.Unlock()
   	key := raftio.GetNodeInfo(clusterID, nodeID)
   	if v, err := newAddr(url); err != nil {
   		panic(err)
   	} else {
   		if _, ok := n.mu.addr[key]; !ok {
   			rec := record{
   				address: v.String(),
   				key:     n.getConnectionKey(v.String(), clusterID),
   			}
   			n.mu.addr[key] = rec
   		}
   	}
   }
   
   ```

   下面来看看上面提到的 raft 的 ConfigChnageEvent的消息是如何处理的:

   ```go
   func (r *raft) handleNodeConfigChange(m pb.Message) {
   	if m.Reject {
   		r.clearPendingConfigChange()
   	} else {
   		cctype := (pb.ConfigChangeType)(m.HintHigh)
   		nodeid := m.Hint
   		switch cctype {
   		case pb.AddNode:
   			r.addNode(nodeid) //增加节点
   		case pb.RemoveNode:
   			r.removeNode(nodeid)
   		case pb.AddObserver:
   			r.addObserver(nodeid)
   		default:
   			panic("unexpected config change type")
   		}
   	}
   }
   
   
   func (r *raft) addNode(nodeID uint64) {
   	r.clearPendingConfigChange()
   	if _, ok := r.remotes[nodeID]; ok {
   		// already a voting member
   		return
   	}
   	if rp, ok := r.observers[nodeID]; ok {
   		// promoting to full member with inheriated progress info
   		r.deleteObserver(nodeID)
   		r.remotes[nodeID] = rp
   		// local peer promoted, become follower
   		if nodeID == r.nodeID {
   			r.becomeFollower(r.term, r.leaderID)
   		}
   	} else {
       //设置 raft 对应的 remote 信息
       //变更这个节点的 matchIndex 和 nextIndex ,因为leader 中维护着每个节点的待发送的日志复制的起始位置
   		r.setRemote(nodeID, 0, r.log.lastIndex()+1)
   	}
   }
   ```

   



