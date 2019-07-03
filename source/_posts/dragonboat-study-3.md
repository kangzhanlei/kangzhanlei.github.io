---


title: dragonboat学习笔记-session
date: 2019-07-03 09:23:54
tags:
	- raft
	- dragonboat
	- 一致性协议
	- 源码分析
	- go
categories:
	- 一致性协议
---

[TOC]

根据论文描述的 session 细节, 实现了"至多一次"的语义. 

# 调用案例

在 test 文件中,已经有了明确的 session 的使用方法, 抽取一个简单的例子来看:

```go

func ExampleNodeHost_SyncGetSession() {
	//1. 先同步获取一个 session
	cs, err := enh.SyncGetSession(ectx, 100)
	if err != nil {
		return
	}
	defer func() {
    //4. 用完了关闭这个 session
		if err := enh.SyncCloseSession(ectx, cs); err != nil {
			log.Printf("close session failed %v\n", err)
		}
	}()
	//2. 以这个 session 发起 propose 请求
	rs, err := enh.Propose(cs, []byte("test-data"), 2000*time.Millisecond)
	if err != nil {
		return
	}
  //在 channel 上阻塞
	s := <-rs.CompletedC
	if s.Timeout() {
		
	} else if s.Completed() {
		rs.Release()
    //3. 请求已经被提交了, 调用ProposalCompleted, 以便推荐 session 中的序列.
		cs.ProposalCompleted()
	} else if s.Terminated() {
	} else if s.Rejected() {
		panic("client session already evicted")
	}

}

```



# 实现分析

分别从SyncGetSession , Propose, ProposalCompleted, SyncCloseSession, 以及状态机的实现谈起.

一个 session 代表了一个客户端实例, 每个session 对发起的请求都应该有一个序列号, 标示着该 session 下第几笔请求. 服务端处理过该笔请求之后,会记录下来当前 session 的序列. 如果 session 重新发起一个小于该序列的请求,或者等于该序列的请求, 都可以直接拒绝或者幂等的回复. 实现了 at-most-once 语义. 具体描述见论文的 6.3 节.

## SyncGetSession

直接看代码

```go
func (nh *NodeHost) SyncGetSession(ctx context.Context,
	clusterID uint64) (*client.Session, error) {
  //获取一个超时时间
	timeout, err := getTimeoutFromContext(ctx)
	if err != nil {
		return nil, err
	}
  //创建一个 session 对象,根据随机种子,生成一个ClientID,具体 ClientID 生成是什么规则
  //取决于这个种子. 序列 SeriesID 初始设置为:NoOPSeriesID+1 
	cs := client.NewSession(clusterID, nh.serverCtx.GetRandomSource())
  //设置cs.SeriesID = SeriesIDForRegister
  //表示当前需要的 session 还没有注册,第一次请求到 leader 的时候,会进行注册.
	cs.PrepareForRegister()
  //发起 propose 请求,携带着刚建立的session 对象(其实就是 clientID 和 seriesID)以及超时时间.
  //leader接受到之后会根据 seriesID 和 clientID 进行区别对待.
	rs, err := nh.ProposeSession(cs, timeout)
	if err != nil {
		return nil, err
	}
  //等待 leader 处理结束.
	result, err := checkRequestState(ctx, rs)
	if err != nil {
		return nil, err
	}
	if result.Value != cs.ClientID {
		plog.Panicf("unexpected result %d, want %d", result.Value, cs.ClientID)
	}
  //第一次建立的时候,SeriesID 为 : SeriesIDForRegister
  //等leader 处理完毕,建立完成之后, SeriesID 调整为 : SeriesIDFirstProposal 表示可以发起请求了
	cs.PrepareForPropose()
	return cs, nil
}

```



下面来看`ProposeSession`的实现:

```go
//发起 session 请求
func (nh *NodeHost) ProposeSession(session *client.Session,
	timeout time.Duration) (*RequestState, error) {
  //根据 clusterID,获取对应的 node 节点, 也就说明了 session 也是分组的,
  //客户端在每个 raft 都有一个 session
	v, ok := nh.getCluster(session.ClusterID)
	if !ok {
		return nil, ErrClusterNotFound
	}
  //sesson 如果是 NO-op的,不会产生任何影响,所以 session 就没什么作用了, 这里忽略他们就可以了.
	if !v.supportClientSession() && !session.IsNoOPSession() {
		plog.Panicf("IOnDiskStateMachine based nodes must use NoOPSession")
	}
  //添加到请求队列中
	req, err := v.proposeSession(session, nil, timeout)
  //发出消息通知, 队列中有事件了,让执行引擎醒来处理事件.
	nh.execEngine.setNodeReady(session.ClusterID)
	return req, err
}
```

继续跟进`proposeSession`的实现

```go
func (n *node) proposeSession(session *client.Session,
	handler ICompleteHandler, timeout time.Duration) (*RequestState, error) {
	if !session.ValidForSessionOp(n.clusterID) {
		return nil, ErrInvalidSession
	}
  //把 session和 nil(cmd) 加到了待发送队列中.
  //其实就是把 session 提取出: ClientID ,SeriesID, ResponedTo 这几个属性,加到消息内容中
	return n.pendingProposals.propose(session, nil, handler, timeout)
}
```

到此, 发送session 的请求就处理完毕了, 下面就需要 leader 来接受请求执行了.  因为这个 session 的建立,也是一个普通的 Propose 请求,和之前的普通 cmd 请求没有什么区别. 前文已经描述过过程, 直接来看 leader 的状态机执行部分: 

```go
//状态机处理一条 Entry 信息.
func (s *StateMachine) handleEntry(ent pb.Entry, last bool) error {
	// ConfChnage also go through the SM so the index value is updated
	if ent.IsConfigChange() {
		accepted := s.handleConfigChange(ent)
		s.node.ConfigChangeProcessed(ent.Key, accepted)
	} else {
    //首先看有没有 session 管理,没有 session 的,报错退出 
    //No-op 的例外,因为他不会对状态机造成任何影响, 所以他根本不需要 session
		if !ent.IsSessionManaged() {
			if ent.IsEmpty() {
				s.handleNoOP(ent) //直接处理 No-op 消息
				s.node.ApplyUpdate(ent, sm.Result{}, false, true, last)
			} else {
        //没有 session,还不是no-op的,就有可能造成多次提交的风险, 所以报错出去.
				panic("not session managed, not empty")
			}
		} else {
      //这里开始对 session 进行处理, 分三部分内容:
      // 1. 新建立 session
      // 2. 关闭 session
      // 3. 处理 session 的请求
      
      //判断是否是 NewSession,依据是:SeriesID ==SeriesIDForRegister 
      //还记得吗:  session 建立的时候第一次传递的 ID 就是SeriesIDForRegister
			if ent.IsNewSessionRequest() {
        //建立 session
				smResult := s.handleRegisterSession(ent)
				s.node.ApplyUpdate(ent, smResult, isEmptyResult(smResult), false, last)
      } else if ent.IsEndOfSessionRequest() {
        //判断是不是终止 session ,依据是 SeriesID 是不是==SeriesIDForUnregister
				smResult := s.handleUnregisterSession(ent)
				s.node.ApplyUpdate(ent, smResult, isEmptyResult(smResult), false, last)
			} else {
        //处理普通的 session 请求
				if !s.entryAppliedInDiskSM(ent.Index) {
					smResult, ignored, rejected, err := s.handleUpdate(ent)
					if err != nil {
						return err
					}
					if !ignored {
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
		s.setBatchedLastApplied(ent.Index)
	}
	return nil
}
```



下面来看如何处理建立和销毁session 的

```go
//处理 session 的建立请求
func (s *StateMachine) handleRegisterSession(ent pb.Entry) sm.Result {
	s.mu.Lock()
	defer s.mu.Unlock()
  //状态机里维护了一个 SessionManager 的数据结构,用来存储所有的 session
  //因为客户端可能会随时崩溃, 所有这个 SessionManager 结构必须可以清理 session
  //还需要对 session 进行超时的管理. 以便移除过期的 session
  //所以 SessionManager 应该需要一个 LRU 等的一个 cache 结构. 
  
  //这里是向 SessionManager 的数据结构中注册这个 ClientID, 下面具体来看实现.
	smResult := s.sessions.RegisterClientID(ent.ClientID)
	if isEmptyResult(smResult) {
		plog.Errorf("%s register client failed %v", s.id(), ent)
	}
	s.updateLastApplied(ent.Index, ent.Term)
	return smResult
}
```



```go
func (ds *SessionManager) RegisterClientID(clientID uint64) sm.Result {
  //首先获取一下,看看这个 session 存不存在
	es, ok := ds.sessions.getSession(RaftClientID(clientID))
	if ok {
		if es.ClientID != RaftClientID(clientID) {
      //这怎么会呢? 
			plog.Panicf("returned an expected session, got id %d, want %d",
				es.ClientID, clientID)
		}
		plog.Warningf("client ID %d already exist", clientID)
		return sm.Result{}
	}
  //创建一个 session 对象, 注册到SessionManager中来. 
	s := newSession(RaftClientID(clientID))
	ds.sessions.addSession(RaftClientID(clientID), *s)
	return sm.Result{Value: clientID}
}
```

后面具体来说一下SessionManager的实现机制 , 这里只需要认为, 针对 session 有专门的管理员, 来保存,删除,处理这些 session 数据就可以了. 

下面来看销毁处理

```go

func (s *StateMachine) handleUnregisterSession(ent pb.Entry) sm.Result {
	s.mu.Lock()
	defer s.mu.Unlock()
  //直接调用反注册方法, 参与依然是一个客户端的标识
	smResult := s.sessions.UnregisterClientID(ent.ClientID)
	if isEmptyResult(smResult) {
		plog.Errorf("%s unregister %d failed %v", s.id(), ent.ClientID, ent)
	}
	s.updateLastApplied(ent.Index, ent.Term)
	return smResult
}

```



```go
func (ds *SessionManager) UnregisterClientID(clientID uint64) sm.Result {
	es, ok := ds.sessions.getSession(RaftClientID(clientID))
	if !ok {
		return sm.Result{}
	}
	if es.ClientID != RaftClientID(clientID) {
		plog.Panicf("returned an expected session, got id %d, want %d",
			es.ClientID, clientID)
	}
  //最终还是交由了 sessionManager 来删除这个 sesison 对象. 
	ds.sessions.delSession(RaftClientID(clientID))
	return sm.Result{Value: clientID}
}
```



好, 现在 leader 已经建立完成了 session, 把处理结果返回给了请求方, 就可以通过 channel 发出结果, 解除请求方的阻塞, 请求方和 leader 都已经建立好了 session, 之后就可以拿这个 session 进行通讯了. 



## Propose

propose和之前的普通请求没有什么区别,第一个参数传递的都是 session 这个对象. 用于标识客户端的 ClientID 和 SeriesID, 具体来看 leader 接收到之后, 如何过滤的请求,以及如何幂等.  代码在上文中已经提到了. 直接来看.

```go
//状态机接收到普通的请求之后
func (s *StateMachine) handleUpdate(ent pb.Entry) (sm.Result, bool, bool, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	var ok bool
  //准备session
	var session *Session
  //合规新检查后更新 term 和 index
	s.updateLastApplied(ent.Index, ent.Term)
	if !ent.IsNoOPSession() {
    //如果是带有 session 的,走这边
    //首先从 sessinManager 中, 根据 ClientID ,获取 Session
		session, ok = s.sessions.ClientRegistered(ent.ClientID)
		if !ok {
			//如果获取不到, 说明客户端崩溃了,超时了, 或者其他情况, 反正 leader 这没有了.不处理.
			return sm.Result{}, false, true, nil
		}
    //记录下来当前这个 session 处理的序列号, 以后比这个序列号小的,都不会再被处理了.
    //这也就实现了请求过滤的功能.
		s.sessions.UpdateRespondedTo(session, ent.RespondedTo)
    // 这里也是保证唯一的关键代码, 判断本次请求是否已经处理过了
    // 如果已经 apply 了,不会执行第二次 apply, 会把上次 apply 之后的结果,再次返回给请求方
    // 也就实现了幂等性.
		result, responded, updateRequired := s.sessions.UpdateRequired(session,
			ent.SeriesID)
		if responded {
			// should ignore. client is expected to timeout
			return sm.Result{}, true, false, nil
		}
		if !updateRequired {
			// server responded, client never confirmed
			// return the result again but not update the sm again
			// this implements the no-more-than-once update of the SM
			return result, false, false, nil
		}
	}
	if !ent.IsNoOPSession() && session == nil {
		panic("session not found")
	}
  //走状态机的代码, 返回数据.
	result, err := s.sm.Update(session, ent)
	if err != nil {
		return sm.Result{}, false, false, err
	}
	return result, false, false, nil
}
```

来看下`UpdateRequired`的代码

```go
  // 返回的 2 个 bool 类型分别是: responded, updateRequired 
  // 表示: 1.是否已经响应过了  2.是否需要执行状态的 update
func (ds *SessionManager) UpdateRequired(session *Session,
	seriesID uint64) (sm.Result, bool, bool) {
  //判断依据是: id <= s.RespondedUpTo
  //也就是说,session 会记录一个当前已经响应过的最大值, 低于这个值的时候,直接返回true,表示已经响应过了
  //本次不处理
	if session.hasResponded(RaftSeriesID(seriesID)) {
		return sm.Result{}, true, false
	}
  //session 中缓存了上次seriesID 对应的处理结果, 如果是该笔请求,直接把结果返回给他,实现幂等
	v, ok := session.getResponse(RaftSeriesID(seriesID))
	if ok {
    //第 2 个 false,表示不需要执行状态机 .第一个返回 true 或者 false 都可以,接收方会判断.
		return v, false, false
	}
  //返回 false 表示没处理过请求,第二个 true 表示可以执行状态机
	return sm.Result{}, false, true
}
```



## ProposalCompleted

用一个 session 发起了请求, 得到回复之后, 需要对这个 session 的 SeriesID 进行自增, 用于区别多次请求.

```go
func (cs *Session) ProposalCompleted() {
  //一个小检查,判断是不是不启用 session 管理或者是 NO-OP SeriesID
	cs.assertRegularSession()
  //因为这个方法是暴露给应用程序调用的,为了防止应用端多次非法调用,做一个小小的检查
 	//主要是要让 SeriesID 进行递增操作.
	if cs.SeriesID == cs.RespondedTo+1 {
		cs.RespondedTo = cs.SeriesID
		cs.SeriesID++
	} else {
		panic("invalid responded to/series id values")
	}
}
```



# session 的管理器

sessionManager 用来管理所有的 session, 提供了注册,反注册,清理等功能.来看具体的实现

```go
//数据结构
type SessionManager struct {
  //关联了一个 LRUsession结构 
	sessions *lrusession
}

func NewSessionManager() *SessionManager {
	return &SessionManager{
    //分配了LRUSession , count 通过配置得到的
		sessions: newLRUSession(LRUMaxSessionCount),
	}
}

```

来看具体的 LRUSession 

```go
func newLRUSession(size uint64) *lrusession {
	if size == 0 {
		panic("lrusession size must be > 0")
	}
  //构造函数的数量, 以及一个有序的缓存.并且分片了一个清除策略 LRU
	rec := &lrusession{
		size:     size,
		sessions: cache.NewOrderedCache(cache.Config{Policy: cache.CacheLRU}),
	}
  //判断什么时候改清理了, 也就是当超过最大容纳数量的时候
	rec.sessions.Config.ShouldEvict = func(n int, k, v interface{}) bool {
		if uint64(n) > rec.size {
			clientID := k.(*RaftClientID)
			plog.Warningf("session with client id %d evicted, overloaded", *clientID)
			return true
		}
		return false
	}
	rec.sessions.Config.OnEvicted = func(k, v, e interface{}) {}
	return rec
}
```

来看添加 session

```go
//增加 session
func (rec *lrusession) addSession(key RaftClientID, s Session) {
	rec.Lock()
	defer rec.Unlock()
  //让缓存增加 session,见下面的代码
	rec.addSessionLocked(key, s)
}

```
这里 OrderedCache 继承了baseCache,  先看下 baseCache 的增加和清理..
```go

//交到 lurSession 中的 OrderedCache 来增加
func (bc *baseCache) add(key, value interface{}, entry, after *Entry) {
  //如果已经在缓存中存在了,那么重新access他,刷新他的鲜活时间
	if e := bc.store.get(key); e != nil {
    //如果是 LRU 算法,就把当前的 session 移动到队列的最前面.说明还在使用者, 别清理掉这个新的.
		bc.access(e)
		e.Value = value
		return
	}
	e := entry
	if e == nil {
		e = &Entry{Key: key, Value: value}
	}
  //如果没有指定顺序,就直接压到最前面,否则按照顺序插入
	if after != nil {
		bc.ll.insertBefore(e, after)
	} else {
		bc.ll.pushFront(e)
	}
	bc.store.add(e)
	//这里并没有看到关于 session 超时的设置, 即使超时了也依然在内存中, 只是判断数量而已.
  //evict的代码如下
	for bc.evict() {
	}
}
```

```go
func (bc *baseCache) evict() bool {
	if bc.ShouldEvict == nil || bc.Policy == CacheNone {
		return false
	}
	l := bc.store.length()
	if l > 0 {
    //从最后一个开始取
		e := bc.ll.back()
    //判断这个是不是到了被清理的条件
		if bc.ShouldEvict(l, e.Key, e.Value) {
      //从队列中清除掉
			bc.removeElement(e)
			return true
		}
	}
	return false
}
```

而 OrderedCache 实现了 Add 和 evict 方法. 来看下他的实现

```go
func (oc *OrderedCache) add(e *Entry) {
	oc.llrb.Insert(e)
}
//llrb 是一个 左倾红黑树 的数据结构
type Tree struct {
	Root  *Node // Root node of the tree.
	Count int   // Number of elements stored.
}


```



//TODO 