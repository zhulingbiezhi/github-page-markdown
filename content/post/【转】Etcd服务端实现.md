---
title: "【转】Etcd服务端实现"
date: 2018-08-06T11:18:15+08:00
weight: 170
keywords: ["etcd"]
description: "Etcd服务端实现"
tags: ["etcd", "分布式"]
categories: ["分布式"]
author: "去去"
---  
7.1　Etcd启动
----------

Etcd有多种启动方式，我们从最简单的方式入手，也就是从embed的etcd.go开始启动，最后会启动EtcdServer。

先看看etcd.go中的启动代码：

    func StartEtcd(inCfg *Config) (e *Etcd, err error)
    
    

从StartEtcd方法启动etcd服务，参数是初始配置信息config，启动集群间监听进程和客户端监听进程，最后启动EtcdServer。

主要代码：

    e = &Etcd{cfg: *inCfg, stopc: make(chan struct{})}
    cfg := &e.cfg
    if e.Peers, err = startPeerListeners(cfg); err != nil {
       return
    }
    if e.sctxs, err = startClientListeners(cfg); err != nil {
       return
    }
    if e.Server, err = etcdserver.NewServer(srvcfg); err != nil {
       return
    }
    e.Server.Start()
    
    

startPeerListeners启动Peer监听，等待集群中其他机器连接自己。startClientListeners启动客户端监听Socket，等待客户端请求并响应。最后调用Start方法启动EtcdServer。

7.2　EtcdServer
--------------

EtcdServer位于etcdserver/server.go，定义了Server接口和EtcdServer对象。EtcdServer从逻辑上讲代表了一个完整的Etcd服务。

![{%}](http://www.ituring.com.cn/figures/2017/Etcd/022.png)

**图7.1　Etcd服务端的功能示意图**

Etcd服务端主要提供两大类客户端接口：

（1）集群配置

由memberHandler负责，提供添加集群成员，删除成员，更新成员信息三种接口服务。

（2）KV键值：由keysHandler负责。

KeysHandler接收到客户端请求后，调用EtcdServer的Do方法处理请求，Watcher类的客户端请求信息同样包含在keysHandler中了。KV键值响应主要在v2_server.go中定义，etcd新版本同时还提供了v3操作命令集，本文不讨论v3的源码实现。

### 7.2.1　接口定义

Etcdserver/server.go中定义了Server接口，是服务端的主接口，其中Do方法处理客户端请求。

Server.go中定义了EtcdServer对象，它是Server接口的实现类。Server中的Do接口是专门用来响应客户端请求的。

Server接口定义：

*   **start**
    
    读取配置文件，启动本Server。
    
*   **stop**
    
    停止本Server
    
*   **ID**
    
    获取本节点server的ID，集群中所有的机器都有唯一ID，用于标识自己。
    
*   **Leader**
    
    获取leader的ID
    
*   **Do**
    
    处理客户群请求，返回处理结果。
    
    定义：
    
        func (s *EtcdServer) Do(ctx context.Context, r pb.Request) (Response, error)
        
        
    
    在server.go中并没有看到Go接口的实现，其实它是在v2_server.go文件中定义的。
    
*   **Process**
    
        Process(ctx context.Context, m raftpb.Message) error
        
        
    
    处理Raft消息。
    
*   **AddMember**
    
    向Etcd集群中增加一台服务器，新增服务器的ID必须唯一标识。
    
*   **RemoveMember**
    
    从集群删除一台服务器，删除服务器的ID必须已经存在于集群中。
    
*   **UpdateMember**
    
    修改集群成员属性，如果成员ID不存在则返回ErrIDNotFound错误。
    

### 7.2.2　实体定义

EtcdServer表示一个独立运行的Etcd节点。

    type EtcdServer struct {
        inflightSnapshots int64
        appliedIndex      uint64
        committedIndex    uint64.
        consistIndex consistentIndex
        Cfg          *ServerConfig
        readych chan struct{}
        r       raftNode
        snapCount uint64
        w wait.Wait
        readMu sync.RWMutex
        readwaitc chan struct{}
        readNotifier *notifier
        stop chan struct{}
        stopping chan struct{}
        done chan struct{}
        errorc     chan error
        id         types.ID
        attributes membership.Attributes
        cluster *membership.RaftCluster
        store store.Store
        applyV2 ApplierV2
        applyV3 applierV3
        applyV3Base applierV3
        applyWait   wait.WaitTime
        kv         mvcc.ConsistentWatchableKV
        lessor     lease.Lessor
        bemu       sync.Mutex
        be         backend.Backend
        authStore  auth.AuthStore
        alarmStore *alarm.AlarmStore
        stats  *stats.ServerStats
        lstats *stats.LeaderStats
        SyncTicker *time.Ticker
        compactor *compactor.Periodic
        peerRt   http.RoundTripper
        reqIDGen *idutil.Generator
        forceVersionC chan struct{}
        wgMu sync.RWMutex
        wg sync.WaitGroup
        ctx    context.Context
        cancel context.CancelFunc
        leadTimeMu      sync.RWMutex
        leadElectedTime time.Time
    }
    
    

### 7.2.3　Do

Do定义在v2_server.go中，处理客户群请求包，调用raftNode的Propose方法。在上一章已经介绍过。

对于KV键值请求，Do方法是在etcdServer/v2_server.go中定义的，它的相关代码逻辑如下：

    func (s *EtcdServer) Do(ctx context.Context, r pb.Request) (Response, error) {
        r.ID = s.reqIDGen.Next()
        if r.Method == "GET" && r.Quorum {
            r.Method = "QGET"
        }
        v2api := (v2API)(&v2apiStore{s})
        switch r.Method {
        case "POST":
            return v2api.Post(ctx, &r)
        case "PUT":
            return v2api.Put(ctx, &r)
        case "DELETE":
            return v2api.Delete(ctx, &r)
        case "QGET":
            return v2api.QGet(ctx, &r)
        case "GET":
            return v2api.Get(ctx, &r)
        case "HEAD":
            return v2api.Head(ctx, &r)
        }
        return Response{}, ErrUnknownMethod
    }
    
    

可以看到对客户端的KV键值请求，最终是通过v2apiStore的相关方法来实现。客户端的命令前缀为"/v2/keys"。支持的命令有以下这些：

*   GET/QGET：读取键值
    
*   POST:创建一个新的KV键值
    
*   PUT：重新设置键值的值
    
*   DELETE：删除已有键值
    

v2apiStore包含了EtcdServer引用。

    type v2apiStore struct{ s *EtcdServer }
    
    

除了GET命令，其余Post，Put和Delete每个写操作请求最后都是通过processRaftRequest方法来处理的。

我们先看看GET命令的处理：

    func (a *v2apiStore) Get(ctx context.Context, r *pb.Request) (Response, error) {
        if r.Wait {
            wc, err := a.s.store.Watch(r.Path, r.Recursive, r.Stream, r.Since)
            if err != nil {
                return Response{}, err
            }
            return Response{Watcher: wc}, nil
        }
        ev, err := a.s.store.Get(r.Path, r.Recursive, r.Sorted)
        if err != nil {
            return Response{}, err
        }
        return Response{Event: ev}, nil
    }
    
    

看到对于普通的GET操作，直接调用store.Get方法获取KV值返回给客户端，如果是Watcher操作，则返回Watcher给客户端，客户端后续通过Watcher接口读取变化值。

对于POST，PUT，DELETE命令，走下述Propose流程处理。

![](http://www.ituring.com.cn/figures/2017/Etcd/023.png)

**图7.2　Propose流程示意图**

比如"DELETE"命令。

    func (a *v2apiStore) Delete(ctx context.Context, r *pb.Request) (Response, error) {
        return a.processRaftRequest(ctx, r)
    }
    
    

processRaftRequest方法的源码如下：

    func (a *v2apiStore) processRaftRequest(ctx context.Context, r *pb.Request) (Response, error) {
        data, err := r.Marshal()
        if err != nil {
            return Response{}, err
        }
        ch := a.s.w.Register(r.ID)
        start := time.Now()
        a.s.r.Propose(ctx, data)
        proposalsPending.Inc()
        defer proposalsPending.Dec()
        select {
        case x := <-ch:
            resp := x.(Response)
            return resp, resp.err
        case <-ctx.Done():
            proposalsFailed.Inc()
            a.s.w.Trigger(r.ID, nil) // GC wait
            return Response{}, a.s.parseProposeCtxErr(ctx.Err(), start)
        case <-a.s.stopping:
        }
        return Response{}, ErrStopped
    }
    
    

*   data, err := r.Marshal()语句：
    
    这条语句从pb.request得到请求数据data
    
*   ch := a.s.w.Register(r.ID)语句：
    
    注册chain，一直等待直到ch有响应数据。
    
    Register方法是wait的Register方法。该方法直到调用wait的Trigger方法后才会有数据从而触发select在该Register Id上线程被唤醒。Wait在pkg/wait中定义。
    
*   a.s.r.Propose(ctx, data)
    
    Propose方法在node中定义，raftNode在etcdserver/node.go文件中。Propose将写事务请求发给Leader，等待集群间同步。Propose集群间同步消息完成后会唤醒a.s.w.Register语句。
    
    调用raft/node的Propose方法处理写事务请求，进一步调用step方法将写事务封装成MsgProp消息并传递给集群中其他机器。
    
        func (n *node) Propose(ctx context.Context, data []byte) error {
            return n.step(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
        }
        
        
    
    step会调用StepFunc函数来处理MsgProp消息，根据leader，follower，candidate等运行状态分别调用不同的实现函数。
    
*   select语句
    
    select … case …语句类似于Socket通信中的select语句，它的含义是只要任意一个case语句有数据返回就往下执行，否则就阻塞在这里让出CPU给其他线程执行。
    
    case x := <-ch:当ch有值时，将ch赋值给x变量，同时唤醒case语句被执行，这里将执行以下代码：
    
        resp := x.(Response)
        return resp, resp.err
        
        
    
    此时将ch中的返回结果Response回复给调用者（即客户端）。
    
    case <-ctx.Done()：说明上下文被中断，Context的Done()被触发，此时写事务执行失败，返回空Response。
    

### 7.2.4　初始化

Etcd服务端主要由5大组件构成，他们的分工如下：

*   etcdServer：主进程，相当于整个Etcd的容器，包含了raftNode，WAL，snapshotter等多个关键组件。
    
*   raftNode：执行raft协议，保证写事务的集群一致性维护。
    
*   Store：管理维护Etcd数据库
    
*   Wal：管理事务日志
    
*   Snapshotter：负责数据快照，管理store数据库在内存中和磁盘上的相互转换。
    

raftNode除了负责集群间raft消息交互，还负责事务和快照的存储，保持数据一致性。

Etcd定义了一个storage数据结构，一起负责事务和快照。

    type storage struct {
        *wal.WAL
        *snap.Snapshotter
    }
    
    

storage中没有指定WAL和Snapshotter的变量名称，这两个类的方法都可直接通过storage来调用，比如WAL的Save方法，可以通过storage.Save来调用，也可以通过storage.WAL.Save来调用，这两者是等价的，在阅读源码的时候要注意这一点，否则对Go语法不太了解的读者会感到迷惑。

    func NewServer(cfg *ServerConfig) (srv *EtcdServer, err error) {
        st := store.New(StoreClusterPrefix, StoreKeysPrefix)
        var (
            w  *wal.WAL
            n  raft.Node
            s  *raft.MemoryStorage
            id types.ID
            cl *membership.RaftCluster
        )
        haveWAL := wal.Exist(cfg.WALDir())
        ss := snap.New(cfg.SnapDir())
        bepath := filepath.Join(cfg.SnapDir(), databaseFilename)
        beExist := fileutil.Exist(bepath)
    
        switch {
        case haveWAL:
            snapshot, err = ss.Load()
            if snapshot != nil {
                if err = st.Recovery(snapshot.Data); err != nil {
                    plog.Panicf("recovered store from snapshot error: %v", err)
                }
            }
            cfg.Print()
            if !cfg.ForceNewCluster {
                id, cl, n, s, w = restartNode(cfg, snapshot)
            } else {
                id, cl, n, s, w = restartAsStandaloneNode(cfg, snapshot)
            }
            cl.SetStore(st)
            cl.SetBackend(be)
            cl.Recover(api.UpdateCapability)
        }
        if terr := fileutil.TouchDirAll(cfg.MemberDir()); terr != nil {
            return nil, fmt.Errorf("cannot access member directory: %v", terr)
        }
        sstats := &stats.ServerStats{
            Name: cfg.Name,
            ID:   id.String(),
        }
        sstats.Initialize()
        lstats := stats.NewLeaderStats(id.String())
        heartbeat := time.Duration(cfg.TickMs) * time.Millisecond
        srv = &EtcdServer{
            readych:   make(chan struct{}),
            Cfg:       cfg,
            snapCount: cfg.SnapCount,
            errorc:    make(chan error, 1),
            store:     st,
            r: *newRaftNode(
                raftNodeConfig{
                    isIDRemoved: func(id uint64) bool {
                           return cl.IsIDRemoved(types.ID(id))
                    },
                    Node:        n,
                    heartbeat:   heartbeat,
                    raftStorage: s,
                    storage:     NewStorage(w, ss),
                },
            ),
            id:            id,
            attributes:    membership.Attributes{Name: cfg.Name, ClientURLs: cfg.ClientURLs.StringSlice()},
            cluster:       cl,
            stats:         sstats,
            lstats:        lstats,
            SyncTicker:    time.NewTicker(500 * time.Millisecond),
            peerRt:        prt,
            reqIDGen:      idutil.NewGenerator(uint16(id), time.Now()),
            forceVersionC: make(chan struct{}),
        }
        srv.applyV2 = &applierV2store{store: srv.store, cluster: srv.cluster}
        srv.be = be
        minTTL := time.Duration((3*cfg.ElectionTicks)/2) * heartbeat
        srv.lessor = lease.NewLessor(srv.be, int64(math.Ceil(minTTL.Seconds())))
        srv.kv = mvcc.New(srv.be, srv.lessor, &srv.consistIndex)
        srv.consistIndex.setConsistentIndex(srv.kv.ConsistentIndex())
        tp, err := auth.NewTokenProvider(cfg.AuthToken,
            func(index uint64) <-chan struct{} {
                return srv.applyWait.Wait(index)
            },
        )
        tr := &rafthttp.Transport{
            TLSInfo:     cfg.PeerTLSInfo,
            DialTimeout: cfg.peerDialTimeout(),
            ID:          id,
            URLs:        cfg.PeerURLs,
            ClusterID:   cl.ID(),
            Raft:        srv,
            Snapshotter: ss,
            ServerStats: sstats,
            LeaderStats: lstats,
            ErrorC:      srv.errorc,
        }
        if err = tr.Start(); err != nil {
            return nil, err
        }
        // add all remotes into transport
        for _, m := range remotes {
            if m.ID != id {
                tr.AddRemote(m.ID, m.PeerURLs)
            }
        }
        for _, m := range cl.Members() {
            if m.ID != id {
                tr.AddPeer(m.ID, m.PeerURLs)
            }
        }
        srv.r.transport = tr
        return srv, nil
    }
    
    

这段代码是EtcdServer的初始化主代码，为了简化分析这里做了精简，要想看完整的代码请自行浏览源码。这里我们只简单讨论haveWAL的情况。

1.  **初始化数据库**
    
        ss := snap.New(cfg.SnapDir())
        snapshot, err = ss.Load()
        
        
    
    以上两句是加载snapshot数据，恢复上次的数据库状态。
    
        st.Recovery(snapshot.Data)
        
        
    
    这里st是什么？我们来看看它的代码：
    
        st := store.New(StoreClusterPrefix, StoreKeysPrefix)
        
        
    
    原来这里st是store，从这个逻辑可以看出来，store本身是不保存磁盘的，所有保存磁盘操作都是通过snap或WAL来操作的。
    
    Snap和WAL的区别：
    
    Snap是etcd数据库的完整快照，只有一份。
    
    WAL是事务日志：不光保存事务快照，还保存状态快照，校验快照等。而且WAL事务的记录不光是单条事务，甚至多条事务也可以组织成一个事务记录，事务记录用Entry来表示。
    
2.  **启动raftNode**
    
        if !cfg.ForceNewCluster {
            id, cl, n, s, w = restartNode(cfg, snapshot)
        } else {
            id, cl, n, s, w = restartAsStandaloneNode(cfg, snapshot)
        }
        // 设置cl：
        cl.SetStore(st)
        cl.SetBackend(be)
        cl.Recover(api.UpdateCapability)
        
    
3.  **创建server**
    
        srv = &EtcdServer{
                readych:   make(chan struct{}),
                Cfg:       cfg,
                snapCount: cfg.SnapCount,
                errorc:    make(chan error, 1),
                store:     st,
                r: *newRaftNode(
                    raftNodeConfig{
                        isIDRemoved: func(id uint64) bool { return cl.IsIDRemoved(types.ID(id)) },
                        Node:        n,
                        heartbeat:   heartbeat,
                        raftStorage: s,
                        storage:     NewStorage(w, ss),
                    },
                ),
                id:            id,
                attributes:    membership.Attributes{Name: cfg.Name, ClientURLs: cfg.ClientURLs.StringSlice()},
                cluster:       cl,
                stats:         sstats,
                lstats:        lstats,
                SyncTicker:    time.NewTicker(500 * time.Millisecond),
                peerRt:        prt,
                reqIDGen:      idutil.NewGenerator(uint16(id), time.Now()),
                forceVersionC: make(chan struct{}),
            }
        
    
4.  **启动http处理handler，等待peer的连接**
    
            tr := &rafthttp.Transport{
                TLSInfo:     cfg.PeerTLSInfo,
                DialTimeout: cfg.peerDialTimeout(),
                ID:          id,
                URLs:        cfg.PeerURLs,
                ClusterID:   cl.ID(),
                Raft:        srv,
                Snapshotter: ss,
                ServerStats: sstats,
                LeaderStats: lstats,
                ErrorC:      srv.errorc,
            }
            if err = tr.Start(); err != nil {
                return nil, err
           }
        
        
    

### 7.2.5　主循环

EtcdServer从Start启动，执行start方法后再执行run方法，在run方法中循环处理etcd事件。

下面主要看看run方法的处理流程。run方法的定义：

    func (s \*EtcdServer) run()
    
    

实现逻辑分成两大部分，首先执行一些初始化代码，然后进入for循环等待处理各种etcd事件。

#### 7.2.5.1　初始化代码

通过snapshot恢复上次快照数据：

        sn, err := s.r.raftStorage.Snapshot()
        if err != nil {
            plog.Panicf("get snapshot from raft storage error: %v", err)
        }
        // asynchronously accept apply packets, dispatch progress in-order
        sched := schedule.NewFIFOScheduler()
    
    

初始化raftReadyHandler，其中waitForApply表示等待最先完成的apply操作，updateCommittedIndex更新committedIndex值，updateLeadership表示更换leader后执行的操作。

    rh := &raftReadyHandler{
        updateLeadership: func(newLeader bool) {
            if !s.isLeader() {
                if s.lessor != nil {
                    s.lessor.Demote()
                }
                if s.compactor != nil {
                    s.compactor.Pause()
                }
                setSyncC(nil)
            } else {
                if newLeader {
                    t := time.Now()
                    s.leadTimeMu.Lock()
                    s.leadElectedTime = t
                    s.leadTimeMu.Unlock()
                }
                setSyncC(s.SyncTicker.C)
                if s.compactor != nil {
                    s.compactor.Resume()
                }
            }
            if s.stats != nil {
                s.stats.BecomeLeader()
            }
        },
        updateCommittedIndex: func(ci uint64) {
            cci := s.getCommittedIndex()
            if ci > cci {
                s.setCommittedIndex(ci)
            }
        },
        waitForApply: func() {
            sched.WaitFinish(0)
        },
    }
    
    

启动raftNode：

    s.r.start(rh)
    
    

初始化etcdProcess参数，主要包括Term，Index等参数值。

    ep := etcdProgress{
        confState: sn.Metadata.ConfState,
        snapi:     sn.Metadata.Index,
        appliedt:  sn.Metadata.Term,
        appliedi:  sn.Metadata.Index,
    }
    
    

#### 7.2.5.2　for循环

执行完初始化后就开始for循环，其中我们主要关注的是apply信道，执行applyAll方法。

    for {
        select {
        case ap := <-s.r.apply():
            f := func(context.Context) { s.applyAll(&ep, &ap) }
            sched.Schedule(f)
            ...
        }
    }
    
    

**apply信道**

KV键值命令的真正提交执行在etcdserver/apply_v2.go中定义，修改store数据库。

EtcdServer主循环等待apply信道消息：

    case ap := <-s.r.apply():
        f := func(context.Context) { s.applyAll(&ep, &ap) }
        sched.Schedule(f)
    
    

当写事务已经被集群中超过半数的raft确认过并且保存到WAL日志后，raftNode会触发apply()信道被选择，进而执行applyAll方法进行处理，applyAll方法中会执行真正的写操作修改Etcd数据库内容。

![{%}](http://www.ituring.com.cn/figures/2017/Etcd/024.png)

**图7.3　apply调用流程**

applyAll方法定义：

    func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply) {
        s.applySnapshot(ep, apply)
        st := time.Now()
        s.applyEntries(ep, apply)
        d := time.Since(st)
        entriesNum := len(apply.entries)
        proposalsApplied.Set(float64(ep.appliedi))
        s.applyWait.Trigger(ep.appliedi)
        <-apply.raftDone
        s.triggerSnapshot(ep)
        select {
        // snapshot requested via send()
        case m := <-s.r.msgSnapC:
            merged := s.createMergedSnapshotMessage(m, ep.appliedt, ep.appliedi, ep.confState)
            s.sendMergedSnap(merged)
        default:
        }
    }
    
    

1.  **applySnapshot**
    
    s.applySnapshot会判断apply中的数据是否包含snapshot数据，如果有则用snapshot恢复store中完整状态，此时主要是执行：
    
        s.store.Recovery(apply.snapshot.Data);
        
        
    
    将数据库恢复成snapshot中的数据。
    
2.  **applyEntries**
    
    然后开始执行写事务：
    
        s.applyEntries(ep, apply)
        
        
    
    applyEntries为每个写事务命令调用apply方法。
    
        func (s *EtcdServer) applyEntries(ep *etcdProgress, apply *apply) {
            if len(apply.entries) == 0 {
                return
            }
            firsti := apply.entries[0].Index
            if firsti > ep.appliedi+1 {
                plog.Panicf("first index of committed entry[%d] should <= appliedi[%d] + 1", firsti, ep.appliedi)
            }
            var ents []raftpb.Entry
            if ep.appliedi+1-firsti < uint64(len(apply.entries)) {
                ents = apply.entries[ep.appliedi+1-firsti:]
            }
            if len(ents) == 0 {
                return
            }
            var shouldstop bool
            if ep.appliedt, ep.appliedi, shouldstop = s.apply(ents, &ep.confState); shouldstop {
                go s.stopWithDelay(10*100*time.Millisecond, fmt.Errorf("the member has been permanently removed from the cluster"))
            }
        }
        
        
    
    而apply方法对于常规写操作调用applyEntryNormal处理，对于配置修改请求调用applyConfChange处理：
    
        func (s *EtcdServer) apply(es []raftpb.Entry, confState *raftpb.ConfState) (appliedt uint64, appliedi uint64, shouldStop bool) {
            for i := range es {
                e := es[i]
                switch e.Type {
                case raftpb.EntryNormal:
                    s.applyEntryNormal(&e)
                case raftpb.EntryConfChange:
                    var cc raftpb.ConfChange
                    pbutil.MustUnmarshal(&cc, e.Data)
                    removedSelf, err := s.applyConfChange(cc, confState)
                    shouldStop = shouldStop || removedSelf
                    s.w.Trigger(cc.ID, &confChangeResponse{s.cluster.Members(), err})
                default:
                    plog.Panicf("entry type should be either EntryNormal or EntryConfChange")
                }
                atomic.StoreUint64(&s.r.index, e.Index)
                atomic.StoreUint64(&s.r.term, e.Term)
                appliedt = e.Term
                appliedi = e.Index
            }
            return appliedt, appliedi, shouldStop
        }
        
        
    
    applyEntryNormal中的关键代码如下：
    
        func (s *EtcdServer) applyEntryNormal(e *raftpb.Entry) {
            ……
            s.w.Trigger(r.ID, s.applyV2Request(&r))
            ……
        }
        
        
    
    最后applyV2Request方法调用apply_v2.go中的Post，Put，Delete等方法完成最终的写操作：
    
        func (s *EtcdServer) applyV2Request(r *pb.Request) Response {
            toTTLOptions(r)
            switch r.Method {
            case "POST":
                return s.applyV2.Post(r)
            case "PUT":
                return s.applyV2.Put(r)
            case "DELETE":
                return s.applyV2.Delete(r)
            case "QGET":
                return s.applyV2.QGet(r)
            case "SYNC":
                return s.applyV2.Sync(r)
            default:
                return Response{err: ErrUnknownMethod}
            }
        }
        
        
    

下面具体看看这些Post，Put，Delete等方法的实现。

*   **Delete**
    
    删除指定path，对应delete命令。调用store的方法来删除，返回Response数据结构，可见store源码部分。
    
        func (a *applierV2store) Delete(r *pb.Request) Response {
            switch {
            case r.PrevIndex > 0 || r.PrevValue != "":
                return toResponse(a.store.CompareAndDelete(r.Path, r.PrevValue, r.PrevIndex))
            default:
                return toResponse(a.store.Delete(r.Path, r.Dir, r.Recursive))
            }
        }
        
    
*   **Post**
    
    创建指定path，path可能是目录也可能是叶子节点，对应mk或mkdir命令。调用store的Create方法，返回Response数据结构，可见store源码部分。
    
        func (a *applierV2store) Post(r *pb.Request) Response {
            return toResponse(a.store.Create(r.Path, r.Dir, r.Val, true, toTTLOptions(r)))
        }
        
    
*   **Put**
    
    设置指定path的值，对应set命令，设置path的值。这里会加入一些预处理，如果path不存在则首先创建path，然后再设置path的值。核心代码：
    
        return toResponse(a.store.Set(r.Path, r.Dir, r.Val, ttlOptions))
        
        
    
    最终也是调用的store的Set方法。
    

### 7.2.6　AddMember

**添加集群成员**

EtcdServer的AddMember方法定义是这样的：

    func (s *EtcdServer) AddMember(ctx context.Context, memb membership.Member) ([]*membership.Member, error) {
        b, err := json.Marshal(memb)
        if err != nil {
            return nil, err
        }
        cc := raftpb.ConfChange{
            Type:    raftpb.ConfChangeAddNode,
            NodeID:  uint64(memb.ID),
            Context: b,
        }
        return s.configure(ctx, cc)
    }
    
    

这里只是列出了主要的代码，方便理解。这里我们看到EtcdServer会生成ConfChangeAddNode类型的raft消息，然后调用configure方法来处理ConfChangeAddNode消息。

继续，看看configure的实现逻辑，主要代码如下：

    func (s *EtcdServer) configure(ctx context.Context, cc raftpb.ConfChange) ([]*membership.Member, error) {
        cc.ID = s.reqIDGen.Next()
        ch := s.w.Register(cc.ID)
        start := time.Now()
        if err := s.r.ProposeConfChange(ctx, cc); err != nil {
            s.w.Trigger(cc.ID, nil)
            return nil, err
        }
    }
    
    

看到这里是调用raftNode的ProposeConfChange方法来发送MspProp消息给Raft处理，告诉Raft这是一条配置命令，等待Raft集群确认后提交给EtcdServer执行。

ProposeConfChange的主要代码如下：

    func (n *node) ProposeConfChange(ctx context.Context, cc pb.ConfChange) error {
        data, err := cc.Marshal()
        if err != nil {
            return err
        }
        return n.Step(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Type: pb.EntryConfChange, Data: data}}})
    }
    
    

添加成员命令经过集群中超过半数机器确认后提交给EtcdServer，最后EtcdServer调用applyConfChange方法执行添加成员的实际动作，将该member加入到cluster，并启动该member。

applyConfChange方法的定义如下：

    func (s *EtcdServer) applyConfChange(cc raftpb.ConfChange, confState *raftpb.ConfState) (bool, error)
    
    

新增成员的ConfChange值为raftpb.ConfChangeAddNode，主要代码：

    m := new(membership.Member)
    s.cluster.AddMember(m)
    if m.ID != s.id {
        s.r.transport.AddPeer(m.ID, m.PeerURLs)
    }
    
    

将新增的成员信息添加到cluster中，同时启动AddPeer过程将新成员信息广播到集群让其他机器知道，也执行添加成员的动作。

    func (t *Transport) AddPeer(id types.ID, us []string) {
        urls, err := types.NewURLs(us)
        fs := t.LeaderStats.Follower(id.String())
        t.peers[id] = startPeer(t, urls, id, fs)
        addPeerToProber(t.prober, id.String(), us)
    }
    
    

startPeer启动新增的成员，新成员启动后就会加入到集群并准备接收集群Raft消息，开始正常工作。

![{%}](http://www.ituring.com.cn/figures/2017/Etcd/025.png)

**图7.4　添加成员流程示意图**

### 7.2.7　DeleteMember

**删除集群成员。**

![{%}](http://www.ituring.com.cn/figures/2017/Etcd/026.png)

**图7.5　删除成员流程示意图**

EtcdServer的`RemoveMember`方法定义是这样的：

    func (s *EtcdServer) RemoveMember(ctx context.Context, id uint64) ([]*membership.Member, error) {
        if err := s.checkMembershipOperationPermission(ctx); err != nil {
            return nil, err
        }
        if err := s.mayRemoveMember(types.ID(id)); err != nil {
            return nil, err
        }
        cc := raftpb.ConfChange{
            Type:   raftpb.ConfChangeRemoveNode,
            NodeID: id,
        }
        return s.configure(ctx, cc)
    }
    
    

这里只是列出了主要的代码，方便理解。这里我们看到EtcdServer会生成`ConfChangeRemoveNode`类型的raft消息，然后调用configure方法来处理`ConfChangeRemoveNode`消息。

继续，看看configure的实现逻辑，主要代码如下：

    func (s *EtcdServer) configure(ctx context.Context, cc raftpb.ConfChange) ([]*membership.Member, error) {
        cc.ID = s.reqIDGen.Next()
        ch := s.w.Register(cc.ID)
        start := time.Now()
        if err := s.r.ProposeConfChange(ctx, cc); err != nil {
            s.w.Trigger(cc.ID, nil)
            return nil, err
        }
    }
    
    

看到这里是调用raftNode的ProposeConfChange方法来发送MspProp消息给Raft处理，告诉Raft这是一条配置命令，等待Raft集群确认后提交给EtcdServer执行，这里流程和添加成员的流程是一样的。

ProposeConfChange的主要代码如下：

    func (n *node) ProposeConfChange(ctx context.Context, cc pb.ConfChange) error {
        data, err := cc.Marshal()
        if err != nil {
            return err
        }
        return n.Step(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{
                   {Type: pb.EntryConfChange, Data: data}}})
    }
    
    

添加成员命令经过集群中超过半数机器确认后提交给EtcdServer，最后EtcdServer调用applyConfChange方法执行删除成员的实际动作，将该member从cluster中删除。

applyConfChange方法的定义如下：

    func (s *EtcdServer) applyConfChange(cc raftpb.ConfChange, confState *raftpb.ConfState) (bool, error)
    
    

删除成员的ConfChange值为raftpb. `ConfChangeRemoveNode`，主要代码：

    case raftpb.ConfChangeRemoveNode:
        id := types.ID(cc.NodID)
        s.cluster.RemoveMember(id)
        if id == s.id {
            return true, nil
        }
        s.r.transport.RemovePeer(id)
    
    

将成员从cluster删除，同时启动`RemovePeer`过程将删除成员消息广播到集群让其他机器知道，也其他peer也执行响应的动作。

    func (t *Transport) removePeer(id types.ID) {
        if peer, ok := t.peers[id]; ok {
            peer.stop()
        } else {
            plog.Panicf("unexpected removal of unknown peer '%d'", id)
        }
        delete(t.peers, id)
        delete(t.LeaderStats.Followers, id.String())
        t.prober.Remove(id.String())
        plog.Infof("removed peer %s", id)
    }
    
    

RemovePeer停止待删除的成员，并从peers列表中删除该成员。

### 7.2.8　UpdateMember

**更新集群成员**

EtcdServer的`UpdateMember`方法定义是这样的：

    func (s *EtcdServer) UpdateMember(ctx context.Context, memb membership.Member) ([]*membership.Member, error) {
        b, merr := json.Marshal(memb)
        if merr != nil {
            return nil, merr
        }
        if err := s.checkMembershipOperationPermission(ctx); err != nil {
            return nil, err
        }
        cc := raftpb.ConfChange{
            Type:    raftpb.ConfChangeUpdateNode,
            NodeID:  uint64(memb.ID),
            Context: b,
        }
        return s.configure(ctx, cc)
    }
    
    

这里只是列出了主要的代码，方便理解。这里我们看到EtcdServer会生成`ConfChangeUpdateNode`类型的raft消息，然后调用configure方法来处理`ConfChangeUpdateNode`消息。

继续，看看configure的实现逻辑，主要代码如下：

    func (s *EtcdServer) configure(ctx context.Context, cc raftpb.ConfChange) ([]*membership.Member, error) {
        cc.ID = s.reqIDGen.Next()
        ch := s.w.Register(cc.ID)
        start := time.Now()
        if err := s.r.ProposeConfChange(ctx, cc); err != nil {
            s.w.Trigger(cc.ID, nil)
            return nil, err
        }
    }
    
    

看到这里是调用raftNode的ProposeConfChange方法来发送MspProp消息给Raft处理，告诉Raft这是一条配置命令，等待Raft集群确认后提交给EtcdServer执行，这里流程和添加成员的流程是一样的。

ProposeConfChange的主要代码如下：

    func (n *node) ProposeConfChange(ctx context.Context, cc pb.ConfChange) error {
    data, err := cc.Marshal()
    if err != nil {
        return err
    }
    return n.Step(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Type: pb.EntryConfChange, Data: data}}})
    }
    
    

添加成员命令经过集群中超过半数机器确认后提交给EtcdServer，最后EtcdServer调用applyConfChange方法执行删除成员的实际动作，将该member从cluster中删除。

applyConfChange方法的定义如下：

    func (s \*EtcdServer) applyConfChange(cc raftpb.ConfChange, confState \*raftpb.ConfState) (bool, error)
    
    

删除成员的ConfChange值为raftpb.ConfChangeUpdateNode，主要代码：

    case raftpb.ConfChangeUpdateNode:
        m := new(membership.Member)
        if err := json.Unmarshal(cc.Context, m); err != nil {
            plog.Panicf("unmarshal member should never fail: %v", err)
        }
        if cc.NodeID != uint64(m.ID) {
            plog.Panicf("nodeID should always be equal to member ID")
        }
        s.cluster.UpdateRaftAttributes(m.ID, m.RaftAttributes)
        if m.ID != s.id {
            s.r.transport.UpdatePeer(m.ID, m.PeerURLs)
        }
    
    

修改成员属性，同时启动UpdatePeer过程将修改成员消息广播到集群让其他机器知道，也其他peer也执行响应的动作。

    func (t *Transport) UpdatePeer(id types.ID, us []string) {
        urls, err := types.NewURLs(us)
        t.peers[id].update(urls)
        t.prober.Remove(id.String())
        addPeerToProber(t.prober, id.String(), us)
        plog.Infof("updated peer %s", id)
    }
    
    

UpdatePeer修改成员属性，并重新设置peers列表中该成员属性。

### 7.2.9　Leader

获取Leader编号给客户端：

    func (s \*EtcdServer) Leader() types.ID {　　
        return types.ID(s.Lead())
    }
    
    

7.3　Etcd状态机
-----------

在etcd代码中，node作为Raft状态机的具体实现，是整个分布式算法的核心所在。

raftNode和node一起来维护etcd的状态机，raftNode在etcdserver/raft.go中，node在raft/node.go中。

raftNode对EtcdServer提供了node的上层封装，使得EtcdServer只要处理start，apply，processMessages等少数几个接口即可，raftNode通过start方法来启动。而node提供了众多的Raft交互接口，node通过run方法来启动。

### 7.3.1　raftNode

raftNode继承了node接口，是EtcdServer和Raft之间的桥梁。

每个EtcdServer会启动一个相应的raftNode，raftNode可能在leader，follower，candidate三种模式之间切换。

raftNode的定义在server/raft.go中。raftNode具体是通过raft/node.go来与集群中其他server进行raft协议交互。

数据结构：

    type raftNode struct {
        index uint64
        term  uint64
        lead  uint64
        raftNodeConfig
        msgSnapC chan raftpb.Message
        applyc chan apply
        readStateC chan raft.ReadState
        ticker *time.Ticker
        td *contention.TimeoutDetector
        stopped chan struct{}
        done    chan struct{}
    }
    
    

其中term是选举轮次，index是最大日志计数，lead是leader的id。比较关键的是applyc，它是要执行apply操作的信道。当有提交的事务或者snap要执行时，applyc信道会被激活。

raftNode中定义了一个raftNodeConfig结构，raftNodeConfig包含raft.Node结构，raft.Node中定义了raft节点处理各类消息的方法以及对外的接口。

    type raftNodeConfig struct {
        isIDRemoved func(id uint64) bool
        raft.Node
        raftStorage *raft.MemoryStorage
        storage     Storage
        heartbeat   time.Duration
        transport rafthttp.Transporter
    }
    
    

源码中我们看到有的地方会调用raftNode.Propose方法，但raftNode对象中并没有看到Propose方法的定义啊。原来，这是Go语言的一种语法，raftNode中的 raftNodeConfig属性定义：

**raftNodeConfig**

raftNodeConfig和其他属性最大的区别是它只有类型而没有定义变量，这种语法可以说是Go语言的特色，说明可以直接用raftNode来引用raftNodeConfig的方法，比如raftNode.storage这种用法就是合法的。

同样的raftNode.Propose就是调用的raft.Node.Propose方法。

理解了这一点对Etcd源码中的很多疑惑就会迎刃而解了。

#### 7.3.1.1　启动

raftNode的启动流程在start方法中定义。

raftNode启动方法是：

    func (r *raftNode) start(rh *raftReadyHandler)
    
    

服务端启动server等待客户端请求Request，收到Request后唤醒raftNode的start方法。start方法中等待处理Ready()信道，一旦Ready信道有数据则raftNode的start方法被唤醒。

    func (r *raftNode) start(rh *raftReadyHandler) {
        for {
            select {
            case rd := <-r.Ready():
            //读取node提供的Ready()信道
            ...
        }
    }
    
    

具体逻辑在后面讲到node中的Ready()信道接口时再详细论述。

#### 7.3.1.2　apply信道

当raftNode有新事务要提交给EtcdServer时，就通过apply信道，EtcdServer在raftNode的apply()信道等待新的apply事务。

    func (r *raftNode) apply() chan apply {
        return r.applyc
    }
    
    

![](http://www.ituring.com.cn/figures/2017/Etcd/027.png)

**图7.5　apply信道数据流**

那么applyc在何时被赋值呢，在raftNode的start方法中会等待Ready()信道，收到Ready()信道后再从中抽取出apply数据，并赋值给applyc信道。相关代码：

    func (r *raftNode) start(rh *raftReadyHandler) {
        for {
            select {
                case rd := <-r.Ready():
                    raftDone := make(chan struct{}, 1)
                    ap := apply{
                        entries:  rd.CommittedEntries,
                        snapshot: rd.Snapshot,
                        raftDone: raftDone,
                    }
                    updateCommittedIndex(&ap, rh)
                    select {
                    case r.applyc <- ap:
                    case <-r.stopped:
                        return
                     }
        ...
        }
    
    

apply信道的处理过程：

#### 7.3.1.3　Ready信道

在start方法中，存在以下代码处理ready信道：

    case rd := <-r.Ready():
        if rd.SoftState != nil {
            newLeader := rd.SoftState.Lead != raft.None && atomic.LoadUint64(&r.lead) != rd.SoftState.Lead
            if newLeader {
                leaderChanges.Inc()
            }
            if rd.SoftState.Lead == raft.None {
                hasLeader.Set(0)
            } else {
                hasLeader.Set(1)
            }
            atomic.StoreUint64(&r.lead, rd.SoftState.Lead)
            islead = rd.RaftState == raft.StateLeader
            rh.updateLeadership(newLeader)
            r.td.Reset()
        }
        if len(rd.ReadStates) != 0 {
            select {
            case r.readStateC <- rd.ReadStates[len(rd.ReadStates)-1]:
            case <-time.After(internalTimeout):
                plog.Warningf("timed out sending read state")
            case <-r.stopped:
                return
            }
        }
        raftDone := make(chan struct{}, 1)
        ap := apply{
            entries:  rd.CommittedEntries,
            snapshot: rd.Snapshot,
            raftDone: raftDone,
        }
        updateCommittedIndex(&ap, rh)
        select {
            case r.applyc <- ap:
            case <-r.stopped:
                return
        }
        // the leader can write to its disk in parallel with replicating to the followers and them
        // writing to their disks.
        // For more details, check raft thesis 10.2.1
        if islead {
            // gofail: var raftBeforeLeaderSend struct{}
            r.transport.Send(r.processMessages(rd.Messages))
        }
        // gofail: var raftBeforeSave struct{}
        if err := r.storage.Save(rd.HardState, rd.Entries); err != nil {
            plog.Fatalf("raft save state and entries error: %v", err)
        }
        if !raft.IsEmptyHardState(rd.HardState) {
            proposalsCommitted.Set(float64(rd.HardState.Commit))
        }
        // gofail: var raftAfterSave struct{}
        if !raft.IsEmptySnap(rd.Snapshot) {
            if err := r.storage.SaveSnap(rd.Snapshot); err != nil {
                plog.Fatalf("raft save snapshot error: %v", err)
            }
            // gofail: var raftAfterSaveSnap struct{}
            r.raftStorage.ApplySnapshot(rd.Snapshot)
            plog.Infof("raft applied incoming snapshot at index %d", rd.Snapshot.Metadata.Index)
            // gofail: var raftAfterApplySnap struct{}
        }
        r.raftStorage.Append(rd.Entries)
            if !islead {
                // finish processing incoming messages before we signal raftdone chan
                msgs := r.processMessages(rd.Messages)
                raftDone <- struct{}{}
                waitApply := false
                for _, ent := range rd.CommittedEntries {
                    if ent.Type == raftpb.EntryConfChange {
                        waitApply = true
                        break
                    }
                }
                if waitApply {
                    rh.waitForApply()
                }
                // gofail: var raftBeforeFollowerSend struct{}
                r.transport.Send(msgs)
            } else {
                // leader already processed 'MsgSnap' and signaled
                raftDone <- struct{}{}
            }
        r.Advance()
    
    

这段代码较长，但十分关键，因此我们多花点时间研究一下。

这段代码的触发条件是node的Ready信息有数据：

    rd := <-r.Ready()
    
    

其中r.Ready()是要找到node的Ready方法的，这是Go语言特性决定的，尤其要注意。前面对于`SoftState`和·ReadStates·的处理这里先忽略，重点看看apply数据的处理过程：

    ap := apply{　　
            entries:  rd.CommittedEntries,　　
            snapshot: rd.Snapshot,　　
            raftDone: raftDone,
    }
    
    

entries：已提交的事务，等待apply。

snapshot：快照事务，等待apply

    select {　　
        case r.applyc <- ap:　　
        case <-r.stopped:　　　　
            return
    }
    
    

将apply数据写入applyc信息，也就是raftNode的apply()信道：

    func (r *raftNode) apply() chan apply {
        return r.applyc
    }
    
    

apply()信道在EtcdServer的run方法中等待，触发后调用applyAll方法处理。

然后处理Raft消息，如果Raft是leader，则可以同步将Messages发送出去，follower不可以同步发送：

    if islead {
        r.transport.Send(r.processMessages(rd.Messages))
    }
    
    

接下来处理snapshot,将预写式日志写到WAL和Raft：

    r.storage.Save(rd.HardState, rd.Entries);
    r.raftStorage.Append(rd.Entries)
    
    

其中raftStorage是MemoryStorage类型，只保存在内存中以提高速度，raftStorage和storage数据保持一致：

最后将snapshot保存到磁盘和Raft中：

    r.storage.SaveSnap(rd.Snapshot);
    r.raftStorage.ApplySnapshot(rd.Snapshot);
    
    

其中raftStorage同时也更新snapshot，使得raft运行时snapshot和磁盘snap保持一致：

### 7.3.2　node

在etcd中，对Raft算法的调用如下，你可以在etcdserver/raft.go中的startNode找到：

    storage := raft.NewMemoryStorage()
    n := raft.StartNode(0x01, \[]int64{0x02, 0x03}, 3, 1, storage)
    
    

通过这段代码可以了解到，Raft在运行过程记录数据和状态都是保存在内存中，而代码中raft.StartNode启动的Node就是Raft状态机Node。启动了一个Node节点后，Raft会做如下事项。

首先，你需要把从集群的其他机器上收到的信息推送到Node节点，你可以在etcdserver/server.go中的Process函数看到。

    func (s *EtcdServer) Process(ctx context.Context, m raftpb.Message) error {
        if m.Type == raftpb.MsgApp {
           s.stats.RecvAppendReq(types.ID(m.From).String(), m.Size())
        }
        return s.node.Step(ctx, m)
    }
    
    

在检测发来请求的机器是否是集群中的节点，自身节点是否是Follower，把发来请求的机器作为Leader，具体对Node节点信息的推送和处理则通过node.Step()函数实现。

其次，你需要把日志项存储起来，在你的应用中执行提交的日志项，然后把完成信号发送给集群中的其它节点，再通过node.Ready()监听等待下一次任务执行。有一点非常重要，你必须确保在你发送完成消息给其他节点之前，你的日志项内容已经确切稳定的存储下来了。

最后，你需要保持一个心跳信号Tick()。Raft有两个很重要的地方用到超时机制：心跳保持和Leader竞选。需要用户在其raft的Node节点上周期性的调用Tick()函数，以便为超时机制服务。

综上所述，整个raft节点的状态机循环类似如下所示：

    for {
        select {
        case <-s.Ticker:
            n.Tick()
        case rd := <-s.Ready():
            saveToStorage(rd.State, rd.Entries)
            send(rd.Messages)
            process(rd.CommittedEntries)
            s.Node.Advance()
        case <-s.done:
            return
        }
    }
    
    

而这个状态机真实存在的代码位置为etcdserver/server.go中的run函数。

对状态机进行状态变更（如用户数据更新等）则是调用n.Propose(ctx, data)函数，在存储数据时，会先进行序列化操作。获得大多数其他节点的确认后，数据会被提交，存为已提交状态。

之前提到etcd集群的启动需要借助别的etcd集群或者DNS，而启动完毕后这些外力就不需要了，etcd会把自身集群的信息作为状态存储起来。所以要变更自身集群节点数量实际上也需要像用户数据变更那样添加数据条目到Raft状态机中。这一切由n.ProposeConfChange(ctx, cc)实现。当集群配置信息变更的请求同样得到大多数节点的确认反馈后，再进行配置变更的正式操作，代码如下。

    var cc raftpb.ConfChangecc.Unmarshal(data)n.ApplyConfChange(cc)
    
    

> **注意**：一个ID唯一性的表示了一个集群，所以为了避免不同etcd集群消息混乱，ID需要确保唯一性，不能重复使用旧的token数据作为ID。

#### 7.3.2.1　接口定义

Node中定义了Raft之间的主要交互接口，定义在raft/node.go文件中。Node接口定义了以下主要的接口：

    type Node interface {
        Tick()
        Campaign(ctx context.Context) error
        Propose(ctx context.Context, data []byte) error
        ProposeConfChange(ctx context.Context, cc pb.ConfChange) error
        Step(ctx context.Context, msg pb.Message) error
        Ready() <-chan Ready
        Advance()
        ApplyConfChange(cc pb.ConfChange) *pb.ConfState
        TransferLeadership(ctx context.Context, lead, transferee uint64)
        ReadIndex(ctx context.Context, rctx []byte) error
        Status() Status
        ReportUnreachable(id uint64)
        ReportSnapshot(id uint64, status SnapshotStatus)
        Stop()
    }
    
    

*   Tick
    
    Node的时钟处理函数，在每个tick触发。
    
*   Stop
    
    Node的停止处理函数，停止时执行的代码定义在Stop方法中。
    
*   Step
    
    Raft消息处理函数，收到Raft消息时调用Step接口处理，不同状态模式定义了不同的Step方法实现。有leaderStep，followerStep，candicateStep几种。
    
*   Ready
    
    node状态发生变化的channel。当有预写式日志（Entries）、提交日志（CommittedEntries）或者snapshot数据到来时触发ready channel。Node确保事务的执行严格遵守时间次序。
    
*   Advance
    
    当Ready要执行snap操作时，可能要耗时相当长时间，这时候调用Advance使得Node能继续执行后续代码而无需等待Ready全部执行完。
    
*   Status
    
    返回Node当前状态属性
    
*   TransferLeadership
    
    当leader发生变化时调用该接口
    
*   Propose
    
    写事务提议，客户端发起写请求时调用该接口。Propose会在WAL中添加记录。
    
*   Campaign
    
    当node没有收到leader的心跳，调用该方法变成candidate状态，开始新一轮选举。
    
*   ProposeConfChange
    
    Etcd集群配置信息发生变化时调用。
    
*   ReadIndex
    
    获取读操作状态，读状态设置在ready中，每个读操作指令都会按顺序执行，读状态会保存在一个数组中并关联了一个索引。
    

node struct是Node接口的实现，它的启动方法是run。

    type node struct {
        propc      chan pb.Message
        recvc      chan pb.Message
        confc      chan pb.ConfChange
        confstatec chan pb.ConfState
        readyc     chan Ready
        advancec   chan struct{}
        tickc      chan struct{}
        done       chan struct{}
        stop       chan struct{}
        status     chan chan Status
        logger Logger
    }
    
    

Node代表一个Etcd运行节点，工作在leader，follower，candidate状态。Propose方法接收客户端写事务请求，广播MsgApp消息给follower，控制和管理各个Node同步操作该写事务，达到集群写事务状态的一致性。

#### 7.3.2.2　启动

run是node的主线程，它的参数是raft，说明它是与raft进行交互发送Raft消息以及处理Raft接收消息。

EtcdServer启动时调用run方法使得node开始开始处理Raft消息，从而启动node。run方法很重要，因此这里将完整代码贴出来：

    func (n *node) run(r *raft) {
        var propc chan pb.Message
        var readyc chan Ready
        var advancec chan struct{}
        var prevLastUnstablei, prevLastUnstablet uint64
        var havePrevLastUnstablei bool
        var prevSnapi uint64
        var rd Ready
        lead := None
        prevSoftSt := r.softState()
        prevHardSt := emptyState
    
        for {
            if advancec != nil {
                readyc = nil
            } else {
                rd = newReady(r, prevSoftSt, prevHardSt)
                if rd.containsUpdates() {
                    readyc = n.readyc
                } else {
                    readyc = nil
                }
            }
    
            if lead != r.lead {
                if r.hasLeader() {
                    if lead == None {
                        r.logger.Infof("raft.node: %x elected leader %x at term %d", r.id, r.lead, r.Term)
                    } else {
                        r.logger.Infof("raft.node: %x changed leader from %x to %x at term %d", r.id, lead, r.lead, r.Term)
                    }
                    propc = n.propc
                } else {
                    r.logger.Infof("raft.node: %x lost leader %x at term %d", r.id, lead, r.Term)
                    propc = nil
                }
                lead = r.lead
            }
    
            select {
            case m := <-propc:
                m.From = r.id
                r.Step(m)
            case m := <-n.recvc:
                if _, ok := r.prs[m.From]; ok || !IsResponseMsg(m.Type) {
                    r.Step(m)
                }
            case cc := <-n.confc:
                if cc.NodeID == None {
                    r.resetPendingConf()
                    select {
                    case n.confstatec <- pb.ConfState{Nodes: r.nodes()}:
                    case <-n.done:
                    }
                    break
                }
                switch cc.Type {
                case pb.ConfChangeAddNode:
                    r.addNode(cc.NodeID)
                case pb.ConfChangeRemoveNode:
                    if cc.NodeID == r.id {
                        propc = nil
                    }
                    r.removeNode(cc.NodeID)
                case pb.ConfChangeUpdateNode:
                    r.resetPendingConf()
                default:
                    panic("unexpected conf type")
                }
                select {
                case n.confstatec <- pb.ConfState{Nodes: r.nodes()}:
                case <-n.done:
                }
            case <-n.tickc:
                r.tick()
            case readyc <- rd:
                if rd.SoftState != nil {
                    prevSoftSt = rd.SoftState
                }
                if len(rd.Entries) > 0 {
                    prevLastUnstablei = rd.Entries[len(rd.Entries)-1].Index
                    prevLastUnstablet = rd.Entries[len(rd.Entries)-1].Term
                    havePrevLastUnstablei = true
                }
                if !IsEmptyHardState(rd.HardState) {
                    prevHardSt = rd.HardState
                }
                if !IsEmptySnap(rd.Snapshot) {
                    prevSnapi = rd.Snapshot.Metadata.Index
                }
    
                r.msgs = nil
                r.readStates = nil
                advancec = n.advancec
            case <-advancec:
                if prevHardSt.Commit != 0 {
                    r.raftLog.appliedTo(prevHardSt.Commit)
                }
                if havePrevLastUnstablei {
                    r.raftLog.stableTo(prevLastUnstablei, prevLastUnstablet)
                    havePrevLastUnstablei = false
                }
                r.raftLog.stableSnapTo(prevSnapi)
                advancec = nil
            case c := <-n.status:
                c <- getStatus(r)
            case <-n.stop:
                close(n.done)
                return
            }
        }
    }
    
    

好好分析一下run方法，发现它主要处理propc，recvc，ready等信道数据。

#### 7.3.2.3　propc信道

当EtcdServer调用Propose方法时会往propc信息扔一条MsgProp消息，表示要处理一条事务提议。此时调用raft.step方法将MsgProp发送出去。

    case m := <-propc:
        m.From = r.id
        r.Step(m)
    
    

其中的r是raft/raft.go中的raft，而不是etcdserver/raft.go中的raftNode，这点要特别注意。

将MsgProp消息交给Raft的Step方法处理，Raft如果是leader的话则会将MsgProp携带的Entries写入本地raftLog，然后给其他follower发送MspApp消息携带同样的Entries。leader等待绝大多数follower回复MsgAppResp消息后修改raftLog的committedIndex，表示为可提交数据库执行的Entries，此时再唤醒node中等待在Ready channel上的代码。

#### 7.3.2.4　recvc信道

除了MsgProp类别之外的其他Raft消息，处理逻辑也是交给Raft的Step方法处理。同样Raft根据自己是leader还是follower调用不同的处理代码。不同类别消息的处理细节请参考Raft一节。

    case m := <-n.recvc:
        // filter out response message from unknown From.
        if _, ok := r.prs[m.From]; ok || !IsResponseMsg(m.Type) {
            r.Step(m) // raft never returns an error
        }
    
    

**ready信道**

调用newReady生成Ready对象，检查是否存在待处理事务，如果Ready中包含更新项，则赋值给readyc信道。

    rd = newReady(r, prevSoftSt, prevHardSt)
        case readyc <- rd:
            if rd.SoftState != nil {
                prevSoftSt = rd.SoftState
            }
            if len(rd.Entries) > 0 {
                prevLastUnstablei = rd.Entries[len(rd.Entries)-1].Index
                prevLastUnstablet = rd.Entries[len(rd.Entries)-1].Term
                havePrevLastUnstablei = true
            }
            if !IsEmptyHardState(rd.HardState) {
                prevHardSt = rd.HardState
            }
            if !IsEmptySnap(rd.Snapshot) {
                prevSnapi = rd.Snapshot.Metadata.Index
            }
            r.msgs = nil
            r.readStates = nil
            advancec = n.advancec
    
    

生成Ready对象给readyc信道，触发raftNode与apply相关代码执行，见raftNode的apply信道一节的分析。

#### 7.3.2.5　Propose

当EtcdServer的Do接口接收到客户端写请求时，会调用到node的Propose方法处理客户端请求。

    func (n *node) Propose(ctx context.Context, data []byte) error {
        return n.step(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
        }
    
    

Propose生成对应的MsgProp消息，然后调用step方法。step方法是node中处理pb.Message的子方法，它包含两个信道：

（1）recvc

表示接收消息信道。

（2）propc

专门表示MsgProp消息信道。

对于MsgProp，赋值给propc信道，而对于其他消息类型，赋值给recvc信道处理。

    func (n *node) step(ctx context.Context, m pb.Message) error {
        ch := n.recvc
        if m.Type == pb.MsgProp {
            ch = n.propc
        }
        select {
        case ch <- m:
            return nil
        case <-ctx.Done():
            return ctx.Err()
        case <-n.done:
            return ErrStopped
        }
    }
    
    

从而唤醒run方法中propc或recvc两个channel中的代码。

#### 7.3.2.6　Ready

当node准备好数据给raftNode处理时，会通过Ready信道提供出来：

    func (n *node) Ready() <-chan Ready {
        return n.readyc
    }
    
    

readyc的定义：

    readyc　 　chan Ready
    
    

是Ready类型的channel，Ready：

    type Ready struct {
        *SoftState
        pb.HardState
        ReadStates []ReadState
        Entries []pb.Entry
        Snapshot pb.Snapshot
        CommittedEntries []pb.Entry
        Messages []pb.Message
        MustSync bool
    }
    
    

其中主要的是Entries、CommittedEntries、Snapshot、Messages这几种。

Snapshot：当前数据库的快照。

CommittedEntries：已提交的事务

Entries：WAL记录，提前写日志（write ahead log），Propose会生成Entries。

Message：Raft消息，raftNode需要处理这些消息并发送给接收方。

raftNode对Ready信道的处理在raftNode的Ready信道一节中可以看到。

### 目录

*   [内容摘要](/book/tupubarticle/16517)
*   [作者简介](/book/tupubarticle/16518)
*   [1　Etcd简介](/book/tupubarticle/16499)
*   [2　Raft协议](/book/tupubarticle/16505)
*   3　操作实践
*   4　源码分析
*   5　客户端源码
*   6　Http服务端
*   [7　Etcd服务端实现](/book/tupubarticle/16510)
*   8　集群间通信
*   9　数据存储
*   10　参考文献
