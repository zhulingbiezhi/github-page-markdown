---
title: "【原】etcd3-server-API"
date: 2018-08-06T11:18:15+08:00
weight: 170
keywords: ["etcd"]
description: "etcd3-server-API"
tags: ["etcd", "分布式"]
categories: ["etcd"]
author: "去去"
---  
> etcd3项目分为两部分，一是etcd clientv3，一是etcd3 server。etcd3与etcd2是不同的代码实现，同时etcd3 server支持etcd2 api，但是数据不是共用的，V2和V3 server分别独立维护自己的数据。
> 
> etcd clientv3是将对V3 server的grpc请求封装成go packge供go程序员调用的库。
> 
> 本文旨在概述etcd3 的grpc API，所有代码协议均是grpc。

* * *

(一)    etcd3  gRPC API概述
------------------------

* * *

发送到etcd服务器的每个API请求都是gRPC远程过程调用。etcd3中的RPC根据功能分类为：

1、处理etcd key空间的API包括：

> KV - 创建，更新，提取和删除key-value对。
> 
> Watch - 监视对key的更改。
> 
> Lease - 指定时间（秒）的TTL，用于关联多个key。

2、管理集群本身的API包括：

> Auth - 基于角色的身份验证机制，用于验证用户身份。
> 
> Cluster - 提供集群成员信息和配置方法。
> 
> Maintenance - 获取恢复快照，维护人员用于恢复。

* * *

#### _1.1     Request和Response_

* * *

etcd3中的所有gRPC都遵循相同的格式。每个gRPC都有一个函数Name，该函数NameRequest作为参数并NameResponse作为响应返回。详细gRPC API描述：

> **service KV  {**
> 
>           _  /\* Range gets the keys in the range from the key-value store. */_
> 
>             **Range (RangeRequest)  returns  (RangeResponse)**
> 
>         _    /\* Put puts the given key into the key-value store.  A put request increments the revision of the key-value store  and generates one event in the event history.*/_
> 
>            ** Put (PutRequest)  returns  (PutResponse) **
> 
> _            /\* DeleteRange deletes the given range from the key-value store.  A delete request increments the revision of the key-value store  and generates a delete event in the event history for every deleted key.*/_
> 
>             **DeleteRange (DeleteRangeRequest)  returns  (DeleteRangeResponse) **
> 
>         _/* Txn processes multiple requests in a single transaction. A txn request increments the revision of the key-value store and generates events with the same revision for every completed request.  It is not allowed to modify the same key several times within one txn.*/_
> 
>             **Txn (TxnRequest)  returns  (TxnResponse)**
> 
>         _    /\* Compact compacts the event history in the etcd key-value store. The key-value  store should be periodically compacted or the event history will continue to grow  indefinitely.*/_
> 
>            ** Compact (CompactionRequest)  returns  (CompactionResponse)**
> 
> **}**
> 
> **service Watch {**
> 
>            _ /\* Watch watches for events happening or that have happened. Both input and output  are streams; the input stream is for creating and canceling watchers and the output  stream sends events. One watch RPC can watch on multiple key ranges, streaming events  for several watches at once. The entire event history can be watched starting from the  last compaction revision.*/_
> 
>            ** Watch (stream WatchRequest)  returns  (stream WatchResponse)**
> 
> **}**
> 
> **service Lease {**
> 
>           _/*  LeaseGrant creates a lease which expires if the server does not receive a keepAlive  within a given time to live period. All keys attached to the lease will be expired and deleted if the lease expires. Each expired key generates a delete event in the event history.*/_
> 
>          **   LeaseGrant (LeaseGrantRequest)  returns  (LeaseGrantResponse)**
> 
>         _    /\* LeaseRevoke revokes a lease. All keys attached to the lease will expire and be deleted. */_
> 
>            ** LeaseRevoke (LeaseRevokeRequest)  returns  (LeaseRevokeResponse)**
> 
>            _ /\* LeaseKeepAlive keeps the lease alive by streaming keep alive requests from the client  to the server and streaming keep alive responses from the server to the client. */_
> 
>             **LeaseKeepAlive (stream  LeaseKeepAliveRequest)  returns  (stream  LeaseKeepAliveResponse)**
> 
>           _/\* LeaseTimeToLive retrieves lease information. */_
> 
>             **LeaseTimeToLive (LeaseTimeToLiveRequest)  returns  (LeaseTimeToLiveResponse)**
> 
>            _/\* LeaseLeases lists all existing leases. */_
> 
>            ** LeaseLeases (LeaseLeasesRequest)  returns  (LeaseLeasesResponse)**
> 
> **}**
> 
> **service Auth {**
> 
>             _/*  AuthEnable enables authentication. */_
> 
>             **AuthEnable (AuthEnableRequest)  returns  (AuthEnableResponse)**
> 
>             _/* AuthDisable disables authentication. */_
> 
>             **AuthDisable (AuthDisableRequest)  returns  (AuthDisableResponse)**
> 
>             _/*  Authenticate processes an authenticate request.  */_
> 
>            ** Authenticate (AuthenticateRequest)  returns  (AuthenticateResponse)**
> 
>             _/*  UserAdd adds a new user. */_
> 
>             **UserAdd(AuthUserAddRequest) returns (AuthUserAddResponse)**
> 
>           _  /*  UserGet gets detailed user information.*/_
> 
>             **UserGet(AuthUserGetRequest) returns (AuthUserGetResponse)**
> 
>           _  /*  UserList gets a list of all users.*/_
> 
>             **UserList(AuthUserListRequest) returns (AuthUserListResponse)**
> 
>             _/* UserDelete deletes a specified user. */_
> 
>            ** UserDelete(AuthUserDeleteRequest) returns (AuthUserDeleteResponse)     **
> 
>             _/*  UserChangePassword changes the password of a specified user.*/ _      
> 
>             **UserChangePassword(AuthUserChangePasswordRequest) returns (AuthUserChangePasswordResponse)            **
> 
>             _/* UserGrant grants a role to a specified user.  */_
> 
>             **UserGrantRole(AuthUserGrantRoleRequest) returns (AuthUserGrantRoleResponse)  **
> 
>           _  /*  UserRevokeRole revokes a role of specified user.*/          _
> 
>             **UserRevokeRole(AuthUserRevokeRoleRequest) returns (AuthUserRevokeRoleResponse)     **
> 
>           _/\* RoleAdd adds a new role.  */_
> 
>           **  RoleAdd(AuthRoleAddRequest) returns (AuthRoleAddResponse)            **
> 
>             _/* RoleGet gets detailed role information. */_
> 
>             **RoleGet(AuthRoleGetRequest) returns (AuthRoleGetResponse)            **
> 
>             _/*  RoleList gets lists of all roles.*/_
> 
>            ** RoleList(AuthRoleListRequest) returns (AuthRoleListResponse)        **
> 
>             _/*  RoleDelete deletes a specified role.*/    _
> 
>             **RoleDelete(AuthRoleDeleteRequest) returns (AuthRoleDeleteResponse)         **
> 
> _            /*  RoleGrantPermission grants a permission of a specified key or range to a specified role.*/   _
> 
>             **RoleGrantPermission(AuthRoleGrantPermissionRequest) returns (AuthRoleGrantPermissionResponse) **
> 
>     _/* RoleRevokePermission revokes a key or range permission of a specified role. */_
> 
>             **RoleRevokePermission(AuthRoleRevokePermissionRequest) returns (AuthRoleRevokePermissionResponse)**
> 
> **}**
> 
> **service Cluster {**
> 
>             _/\* MemberAdd adds a member into the cluster.*/_
> 
>             **MemberAdd(MemberAddRequest) returns (MemberAddResponse)**
> 
>             _/\* MemberRemove removes an existing member from the cluster. */_
> 
>             **MemberRemove(MemberRemoveRequest) returns (MemberRemoveResponse)**
> 
>             _/\* MemberUpdate updates the member configuration. */_
> 
>             **MemberUpdate(MemberUpdateRequest) returns (MemberUpdateResponse)**
> 
>             _/\* MemberList lists all the members in the cluster. */_
> 
>           **  MemberList(MemberListRequest) returns (MemberListResponse)**
> 
> **}**
> 
> **service Maintenance {**
> 
>             _/*  Alarm activates, deactivates, and queries alarms regarding cluster health. */_
> 
>           **  Alarm(AlarmRequest) returns (AlarmResponse)**
> 
>             _/* Status gets the status of the member. */_
> 
>            ** Status(StatusRequest) returns (StatusResponse)**
> 
>             _/\* Defragment defragments a member's backend database to recover storage space. */_
> 
>         **    Defragment(DefragmentRequest) returns (DefragmentResponse)**
> 
>             _/* Hash computes the hash of whole backend keyspace,  including key, lease, and other buckets in storage.  This is designed for testing ONLY!  Do not rely on this in production with ongoing transactions, since Hash operation does not hold MVCC locks. Use "HashKV" API instead for "key" bucket consistency checks. */_
> 
>            ** Hash(HashRequest) returns (HashResponse)**
> 
>             _/*  HashKV computes the hash of all MVCC keys up to a given revision. It only iterates "key" bucket in backend storage.*/_
> 
>             **HashKV(HashKVRequest) returns (HashKVResponse)**
> 
>           _  /\* Snapshot sends a snapshot of the entire backend from a member over a stream to a client. */_
> 
>             **Snapshot(SnapshotRequest) returns (stream SnapshotResponse)**
> 
>           _/*  MoveLeader requests current leader node to transfer its leadership to transferee. */_
> 
>             **MoveLeader(MoveLeaderRequest) returns (MoveLeaderResponse)**
> 
> **}**

* * *

#### _1.2    Response Header_

* * *

来自etcd API的所有响应都有一个附加的响应头，其中包含响应的集群元数据：

> **message    ResponseHeader  {**
> 
> _/*  cluster_id is the ID of the cluster which sent the response. */_
> 
> **        uint64  cluster_id=1;**
> 
> _   /* member_id is the ID of the member which sent the response. */_
> 
> **        uint64  member_id=2;**
> 
>  _/* revision is the key-value store revision when the request was applied.  For watch progress responses, the header.revision indicates progress. All future events  recieved in this stream are guaranteed to have a higher revision number than the  header.revision number. */_
> 
> **        int64    revision=3;**
> 
>  _/\* raft_term is the raft term when the request was applied. */_
> 
> **        uint64  raft_term=4;**
> 
> **}**

Cluster_ID - 生成响应的集群的ID。

Member_ID - 生成响应的成员的ID。

Reversion - 生成响应时key-value存储的修订版。

Raft_Term - 生成响应时成员的Raft术语。

* * *

(二)    KV API
-------------

* * *

_**2.1**_    _**Key-Value**_

* * *

_Key-Value_是KV API可以操作的最小单位。每个key-value对都有许多以[protobuf格式](https://github.com/coreos/etcd/blob/master/mvcc/mvccpb/kv.proto)定义的字段：

> **message     ****KeyValue  {**
> 
>        _  /\* key is the key in bytes. An empty key is not allowed.  */_
> 
> **        bytes    key=1;**
> 
>        _ /*  create_revision is the revision of last creation on this key.  */_
> 
> **        int64    create_revision=2;**
> 
>         _/* mod_revision is the revision of last modification on this key.  */_
> 
> **        int64    mod_revision=3;**
> 
>          _/* version is the version of the key. A deletion resets  the version to zero and any modification of the key  increases its version. */_
> 
> **        int64    version=4;**
> 
>        _  /* value is the value held by the key, in bytes.  */_
> 
> **        bytes    value=5;**
> 
>          _/\* lease is the ID of the lease that attached to key.  When the attached lease expires, the key will be deleted.  If lease is 0, then no lease is attached to the key.  */_
> 
> **        int64    lease=6;**
> 
> **}**

Key --- 以字节为单位的key， 不允许使用空ke'y。

Value --- 以字节为单位的value。

Version --- 版本是key的版本。删除version将重置为零，并且对key的任何修改都会增加其版本。

Create_Revision --- 最后一次创建key的版本号。

Mod_Revision --- 最后一次修改key的版本号。

Lease --- 附加到key的Lease的ID。如果Lease为0，表明没有将任何Lease附加到key。

* * *

**_2.2_**    **_Range_** **_Request_**

* * *

使用Range API调用从KV存储中获取key，其中包含RangeRequest：

> **message RangeRequest {**
> 
> **  enum SortOrder {**
> 
>         NONE = 0; // _default, no sorting_
> 
>         ASCEND = 1; // _lowest target value first_
> 
>         DESCEND = 2; // _highest target value first_
> 
>   **}**
> 
>   **enum SortTarget {**
> 
>         KEY = 0;
> 
>         VERSION = 1;
> 
>         CREATE = 2;
> 
>         MOD = 3;
> 
>         VALUE = 4;
> 
>   **}**
> 
> _  /*  key is the first key for the range.  If range_end is not given, the request only looks up key. */_
> 
>   **bytes   key = 1;**
> 
> _  /*  range\_end is the upper bound on the requested range \[key, range\_end). If range\_end is '\\0', the range is all keys >= key. If range\_end is key plus one (e.g., "aa"+1 == "ab", "a\\xff"+1 == "b"), then the range request gets all keys prefixed with key. If both key and range_end are '\\0', then the range request returns all keys. */_
> 
> **  bytes     range_end = 2;**
> 
>   /* _limit is a limit on the number of keys returned for the request. When limit is set to 0, it is treated as no limit. */_
> 
> **  int64     limit = 3;**
> 
>   /*  _revision is the point-in-time of the key-value store to use for the range.  If revision is less or equal to zero, the range is over the newest key-value store. If the revision has been compacted, ErrCompacted is returned as a response. */_
> 
>   **int64     revision = 4;**
> 
>   _/*  sort_order is the order for returned sorted results. */_
> 
>  **SortOrder     sort_order = 5;**
> 
> _  /*  sort_target is the key-value field to use for sorting. */_
> 
> **  SortTarget     sort_target = 6;**
> 
> _  /*  serializable sets the range request to use serializable member-local reads.  Range requests are linearizable by default; linearizable requests have higher  latency and lower throughput than serializable requests but reflect the current consensus of the cluster. For better performance, in exchange for possible stale reads, a serializable range request is served locally without needing to reach consensus with other nodes in the cluster. */_
> 
> **  bool     serializable = 7;**
> 
> _  /*  keys_only when set returns only the keys and not the values. */_
> 
>   **bool     keys_only = 8;**
> 
> _  /*  count_only when set returns only the count of the keys in the range. */_
> 
>   **bool     count_only = 9;**
> 
> _  /*  min\_mod\_revision is the lower bound for returned key mod revisions; all keys with lesser mod revisions will be filtered away. */_
> 
>   **int64     min\_mod\_revision = 10; **
> 
> _  /*  max\_mod\_revision is the upper bound for returned key mod revisions; all keys with  greater mod revisions will be filtered away. */_
> 
>   **int64     max\_mod\_revision = 11;**
> 
> _  /\* min\_create\_revision is the lower bound for returned key create revisions; all keys with  lesser create revisions will be filtered away. */_
> 
> **  int64     min\_create\_revision = 12;**
> 
> _  /\* max\_create\_revision is the upper bound for returned key create revisions; all keys with  greater create revisions will be filtered away. */_
> 
>   **int64     max\_create\_revision = 13;**
> 
> **}**

Key，Range_End --- 要获取的key范围。

Limit ---限制请求返回的最大key数量。当limit设置为0时，将其视为无限制。

Revision --- 用于key Range内的KV存储的版本范围限制。如果revision小于或等于零，则范围超过最新的key-value存储如果压缩修订，则返回ErrCompacted作为响应。

Sort_Order --- 请求倒序还是顺序。

Sort\_Target --- 要排序的类型（key, value, version, create\_version, mod_version）。

Serializable --- 设置范围请求以使用可序列化的成员本地读取。默认情况下，Range是可线性化的; 它反映了该集群目前的共识。为了获得更好的性能和可用性，为了换取可能的过时读取，可在本地提供可序列化范围请求，而无需与群集中的其他节点达成共识。

Keys_Only --- 仅返回key而不返回value。

Count_Only --- 仅返回range中key的计数。

Min\_Mod\_Revision --- key mod version的下限; 

Max\_Mod\_Revision --- key mod version的上限; 

Min\_Create\_Revision --- key create version的下限; 

Max\_Create\_Revision --- key create version的上限; 

* * *

**_2.3_**    **_Range_** **_Response_：**

* * *

> **message    RangeResponse {**
> 
>     **ResponseHeader    header=1;**
> 
>     /\* kvs is the list of key-value pairs matched by the range request.  kvs is empty when count is requested. */
> 
>    _repeated_**   KeyValue    kvs=2;**
> 
>     /\* more indicates if there are more keys to return in the requested range. */
> 
> **    bool    more=3;**
> 
>    /*  count is set to the number of keys within the range when requested. */
> 
>     **int64    count=4;**
> 
> **}**

Header ---见ResponseHeader

Kvs --- 范围请求匹配的key-value对列表。当Count_Only设置为true时，Kvs为空。

More --- 表明RangeRequest中的limit为true。

Count --- 满足范围请求的key总数。

* * *

_**2.4**_    _**Put**_ _**Request**_

* * *

通过发出Put Request将key-value保存到KV存储中：

> **message    PutRequest {**
> 
>     _/\* key is the key, in bytes, to put into the key-value store. */_
> 
>  **   bytes    key=1;**
> 
>     _/\* value is the value, in bytes, to associate with the key in the key-value store. */_
> 
>    ** bytes    value=2;**
> 
>     _/\* lease is the lease ID to associate with the key in the key-value store. A lease  value of 0 indicates no lease. */_
> 
>     **int64    lease=3;**
> 
>    _/*  If prev_kv is set, etcd gets the previous key-value pair before changing it. The previous key-value pair will be returned in the put response. */_
> 
>      **bool    prev_kv=4;**
> 
>    /\* If ignore_value is set, etcd updates the key using its current value. Returns an error if the key does not exist. */
> 
>    ** bool    ignore_value=5;**
> 
>     _/\* If ignore_lease is set, etcd updates the key using its current lease.  Returns an error if the key does not exist. */_
> 
>    ** bool    ignore_lease=6;**
> 
> }

Key --- 放入key-value存储区的key的名称。

Value --- 与key-value存储中的key关联的值（以字节为单位）。

Lease ---与key-value存储中的key关联的Lease ID。Lease值为0表示没有Lease。

Prev_Kv --- 设置时，在从此Put请求更新之前使用key-value对数据进行响应。

Ignore_Value --- 设置后，更新key而不更改其当前值。如果key不存在，则返回错误。

Ignore_Lease --- 设置后，更新key而不更改其当前Lease。如果key不存在，则返回错误。

* * *

_**2.5**_    _**Put**_ _**Response**_

* * *

> **message   ****PutResponse    {**
> 
> **    ResponseHeader    header=1;**
> 
>     _/*  if prev_kv is set in the request, the previous key-value pair will be returned. */_
> 
> **    KeyValue    prev_kv=2;**
> 
> **}**

Header ---见ResponseHeader

Prev\_Kv --- 由Putif Prev\_Kv设置的key-value对PutRequest。

* * *

_**2.6**_    _**Delete Range Request**_

* * *

使用该DeleteRange呼叫删除key范围，其中DeleteRangeRequest：

> **message    DeleteRangeRequest {**
> 
>     _/\* key is the first key to delete in the range. */_
> 
> **    byteskey=1;**
> 
>   _/\* range\_end is the key following the last key to delete for the range \[key, range\_end).  If range\_end is not given, the range is defined to contain only the key argument.  If range\_end is one bit larger than the given key, then the range is all the keys  with the prefix (the given key).  If range_end is '\\0', the range is all keys greater than or equal to the key argument. */_
> 
> **    bytesrange_end=2;**
> 
> _    /*  If prev_kv is set, etcd gets the previous key-value pairs before deleting it.  The previous key-value pairs will be returned in the delete response. */_
> 
> **    boolprev_kv=3;**
> 
> **}**

Key，Range_End - 要删除的key范围。

Prev_Kv - 设置后，返回已删除key-value对的内容。

* * *

**_2.6    Delete Range Request_**

* * *

> **message    DeleteRangeResponse {**
> 
> **    ResponseHeader    header=1;**
> 
>   _  /* deleted is the number of keys deleted by the delete range request. */_
> 
> **    int64    deleted=2;**
> 
>    _/*  if prev_kv is set in the request, the previous key-value pairs will be returned. */_
> 
> _repeated_**    KeyValue    prev_kvs=3;**
> 
> **}**

Deleted --- 已删除的key数。

Prev_Kv --- DeleteRange操作删除的所有key-value对的列表。

* * *

**_2.7_**    **_Transaction_**

* * *

事务是KV存储上的原子If / Then / Else构造。它提供了一个原语，用于将请求分组在原子块（即then / else）中，这些原子块的执行基于key-value存储的内容被保护（即if）。事务可用于保护key免受意外的并发更新，构建比较和交换操作以及开发更高级别的并发控制。

事务可以在单个请求中以原子方式处理多个请求。对于key-value存储的修改，这意味着商店的修订仅针对事务递增一次，并且事务生成的所有事件将具有相同的修订。但是，禁止在单个事务中多次修改同一个key。

所有交易都通过比较结合来保护，类似于“如果”声明。每次比较都会检查商店中的单个key。它可以检查值的缺失或存在，与给定值进行比较，或检查key的修订版本或版本。两种不同的比较可适用于相同或不同的key。所有的比较都是原子地应用的; 如果所有比较都为真，则说事务成功并且etcd应用事务的then / successrequest块，否则称其失败并应用else / failurerequest块。

每个比较都编码为一条Compare消息：

> _message_**    Compare {**
> 
>         _enum_    **CompareResult {**
> 
>             EQUAL=0;
> 
>             GREATER=1;
> 
>             LESS=2;
> 
>             NOT_EQUAL=3;
> 
>         **}**
> 
>         _enum_    **CompareTarget {**
> 
>             VERSION=0;
> 
>             CREATE=1;
> 
>             MOD=2;
> 
>             VALUE=3;
> 
>         **}**
> 
>         **CompareResult    result=1;       ** // target is the key-value field to inspect for the comparison.
> 
>         **CompareTarget    target=2;   **     // key is the subject key for the comparison operation.
> 
>         **bytes    key=3;**
> 
>         _oneof_    **target_union** {
> 
>  _  /*  version is the version of the given key. */_
> 
> **                int64    version=4;**
> 
> _                 /*  create_revision is the creation revision of the given key */_
> 
> **                int64    create_revision=5;**
> 
> _/*   mod_revision is the last modified revision of the given key.  */_
> 
> **                int64    mod_revision=6;**
> 
> _ /*  value is the value of the given key, in bytes. */_
> 
> **                bytes    value=7;**
> 
> _/*   lease is the lease id of the given key. */_
> 
> **                int64 lease = 8;**
> 
>         }
> 
> }

Result --- 逻辑比较操作的类型（例如，等于，小于等）。

Target --- 要比较的key-value字段。key的版本，创建修订版，修改版本或值。

Key --- 比较的key。

Target_Union --- 用于比较的用户指定数据。

在处理比较块之后，事务应用一个请求块。块是RequestOp消息：

> _message_ **RequestOp {**
> 
>  _/\* request is a union of request types accepted by a transaction. */_
> 
>  _oneof_**  request {**
> 
> **        RangeRequest **    **request_range =1;**
> 
> **        PutRequest **    **request_put =2;**
> 
> **        DeleteRangeRequest **    **request\_delete\_range =3;**
> 
> **        TxnRequest **    **request_txn =4;**
> 
> **    }**
> 
> **}**

Request_Range --- a RangeRequest。

Request_Put --- a PutRequest。钥匙必须是唯一的。它可能不与任何其他Puts或Deletes共享key。

Request\_Delete\_Range --- a DeleteRangeRequest。它可能不会与任何Puts或Deletes请求共享key。

总之，一个事务是通过TxnAPI调用发出的，它需要TxnRequest：

> _message_    **TxnRequest  {**
> 
>         _/\* compare is a list of predicates representing a conjunction of terms. If the comparisons succeed, then the success requests will be processed in order, and the response will contain their respective responses in order. If the comparisons fail, then the failure requests will be processed in order,  and the response will contain their respective responses in order. */_
> 
>         _repeated_ **Compare   compare =1;**
> 
>         _/\* success is a list of requests which will be applied when compare evaluates to true. */_
> 
>         _repeated_**   RequestOp  success =2;**
> 
>         _/\* failure is a list of requests which will be applied when compare evaluates to false. */_
> 
>          _repeated_**   RequestOp  failure =3;**
> 
> **}**

Compare --- 表示保护事务的术语组合的谓词列表。

Success --- 如果所有比较测试评估为true，则处理请求列表。

Failure --- 如果任何比较测试评估为false，则处理的请求列表。

客户端收到TxnResponse来自Txn呼叫的消息：

> _message_    **TxnResponse{**
> 
> **          ResponseHeader header =1;**
> 
> _ /\* succeeded is set to true if the compare evaluated to true or false otherwise. */_
> 
> **          bool **    **succeeded =2;**
> 
> _  /\* responses is a list of responses corresponding to the results from applying success if succeeded is true or failure if succeeded is false. */_
> 
> _repeated_**   ResponseOp   responses =3;**
> 
> **}**

Success --- 无论是Compare评估为真还是假。

Responses --- Success如果成功，则应用块的结果对应的响应列表为true或Failure if成功为false。

该Responses列表对应于应用RequestOp列表的结果，每个响应编码为ResponseOp：

> _message_ **ResponseOp {**
> 
> _/\* response is a union of response types returned by a transaction.  */_
> 
> _oneof_**   response {**
> 
> **                    RangeResponse   response_range =1;**
> 
> **                    PutResponse  response_put =2;**
> 
> **                    DeleteRangeResponse   response\_delete\_range =3;**
> 
> **                     TxnResponse   response_txn =4;**
> 
> **    }**
> 
> **}**

* * *

(三)    Watch   API
==================

* * *

* * *

Watch API提供基于事件的接口，用于异步监视key更改。etcd3 watch通过持续watch当前或历史的给定修订，等待key的更改，并将key更新流回客户端。

* * *

**_3.1    Events_**

* * *

每个key的每次更改都用Event消息表示。一个Event消息同时提供更新的数据和更新的类型：

> _message_ **Event{**
> 
> _enum_ **EventType {**
> 
> **            PUT =0;**
> 
> **            DELETE =1;**
> 
> **    }**
> 
> _/\* type is the kind of event. If type is a PUT, it indicates new data has been stored to the key. If type is a DELETE, it indicates the key was deleted. */_
> 
> **      EventType type =1;**
> 
> _ /*  kv holds the KeyValue for the event. A PUT event contains current kv pair. A PUT event with kv.Version=1 indicates the creation of a key. A DELETE/EXPIRE event contains the deleted key with  its modification revision set to the revision of deletion. */_
> 
> **      KeyValue kv =2;**
> 
>  _ /*  prev_kv holds the key-value pair before the event happens. */_
> 
> **      KeyValue prev_kv =3;**
> 
> **}**

Type --- 事件的类型。PUT类型表示新数据已存储到Key中。DELETE表示Key已删除。

KV --- 与事件关联的KeyValue。PUT事件包含当前的kv对。kv.Version = 1的PUT事件表示创建key。DELETE事件包含已删除的key，其修改修订版设置为删除修订版。

Prev_KV --- 紧接事件之前修订的key的key-value对。为了节省带宽，只有在watch明确启用它时才会填写。

* * *

_**3.2    Watch streams**_

* * *

Watch是长时间运行的请求，并使用gRPC流来传输事件数据。_Watch streams_是双向的; 客户端写入流以建立监视和读取以接收监视事件。单个监视流可以通过使用每个监视标识符标记事件来复用许多不同的监视。这种多路复用有助于减少核心etcd集群的内存占用和连接开销。

Watch对事件做出三点保证：

有序 \-\-\- 事件按修订排序; 如果事件发生在已经发布的事件之前，则该事件将永远不会出现在watch上。

可靠 \-\-\- 一系列事件永远不会丢失任何事件的后续序列; 如果有事件按时间顺序排列为

原子 \-\-\- 事件清单保证包含完整的修订; 多个key上相同修订版的更新不会分成几个事件列表。

* * *

**_3.3    WatchCreateRequest_**

* * *

> _message_ **WatchCreateRequest {**
> 
>  _/\* key is the key to register for watching. */_
> 
> **  bytes    key =1;**
> 
> _/\* range\_end is the end of the range \[key, range\_end) to watch. If range\_end is not given, only the key argument is watched. If range\_end is equal to '\\0', all keys greater than or equal to the key argument are watched. If the range_end is one bit larger than the given key,  then all keys with the prefix (the given key) will be watched. */_
> 
> **  bytes    range_end =2;**
> 
> _/\* start\_revision is an optional revision to watch from (inclusive). No start\_revision is "now". */_
> 
> **  int64   start_revision =3;**
> 
> _/\* progress_notify is set so that the etcd server will periodically send a WatchResponse with no events to the new watcher if there are no recent events. It is useful when clients wish to recover a disconnected watcher starting from a recent known revision. The etcd server may decide how often it will send notifications based on current load. */_
> 
> **  bool   progress_notify =4;**
> 
> _enum _ **FilterType{**
> 
> **    NOPUT =0;   **_// filter out put event._
> 
> **    NODELETE =1;  **_// filter out delete event._
> 
> **}**
> 
>  _/* filters filter the events at server side before it sends back to the watcher. */_
> 
> _  repeated_**   FilterType  filters =5;**
> 
> _/* If prev_kv is set, created watcher gets the previous KV before the event happens.  If the previous KV is already compacted, nothing will be returned. */_
> 
> **  bool   prev_kv =6;**
> 
> _/\* If watch_id is provided and non-zero, it will be assigned to this watcher. Since creating a watcher in etcd is not a synchronous operation,  this can be used ensure that ordering is correct when creating multiple watchers on the same stream. Creating a watcher with an ID already in  use on the stream will cause an error to be returned. */_
> 
> **  int64   watch_id =7;**
> 
> _/\* fragment enables splitting large revisions into multiple watch responses. */_
> 
> **  bool   fragment =8;**
> 
> **}**

Key，Range_End --- 要watch的key范围。

Start_Revision --- 包含开始watch的可选修订版。如果没有给出，它将在修改监视创建响应标题修订版之后流式传输事件。可以从最后一次压缩修订开始watch整个可用事件历史记录。

Progress_Notify --- 设置后，如果没有最近的事件，watch将定期收到没有事件的WatchResponse。当客户希望从最近的已知修订版本开始恢复断开连接的watcher时，它非常有用。etcd服务器根据当前服务器负载决定发送通知的频率。

Fliters --- 要在服务器端过滤掉的事件类型列表。

Prev_Kv --- 设置后，watch会在事件发生之前接收key-value数据。这对于了解已覆盖的数据非常有用。

* * *

_**3.5    WatchCreateResponse：**_

* * *

> _message_ **WatchResponse {**
> 
> **  ResponseHeader header =1;**
> 
> _/\* watch_id is the ID of the watcher that corresponds to the response. */_
> 
> **  int64    watch_id =2;**
> 
> _/\* created is set to true if the response is for a create watch request.  The client should record the watch\_id and expect to receive events for the created watcher from the same stream.  All events sent to the created watcher will attach with the same watch\_id.  */_
> 
> **  bool    created =3;**
> 
> _/\* canceled is set to true if the response is for a cancel watch request.  No further events will be sent to the canceled watcher.  */_
> 
> **  bool   canceled =4;**
> 
> _/\* compact\_revision is set to the minimum index if a watcher tries to watch  at a compacted index.  This happens when creating a watcher at a compacted revision or the watcher cannot  catch up with the progress of the key-value store. The client should treat the watcher as canceled and should not try to create any  watcher with the same start\_revision again.  */_
> 
> **  int64   compact_revision =5;**
> 
> _/\* cancel_reason indicates the reason for canceling the watcher.  */_
> 
> **  string   cancel_reason =6;**
> 
> _/\* framgment is true if large watch response was split over multiple responses.  */_
> 
> **  bool   fragment =7;**
> 
> _repeated_**   Event events =11;**
> 
> **}**

Watch_ID --- 与响应对应的监视的ID。

Created --- 如果响应是针对创建监视请求，则设置为true。客户端应记录ID并期望在流上接收监视事件。发送给创建的watcher的所有事件都将具有相同的watch_id。

Canceled --- 如果响应是取消watch请求，则设置为true。不会向已取消的watcher发送更多事件。

Compact\_Revision --- 如果watcher尝试watch压缩版本，则设置为etcd可用的最小历史版本。在压缩版本中创建watch程序或watcher无法跟上key-value存储的进度时会发生这种情况。watcher将被取消; 使用相同的start\_revision创建新watch将失败。

Events - 与给定监视ID对应的顺序新事件列表。

* * *

**_3.6    WatchCancelRequest：_**

* * *

> _message_ **WatchCancelRequest {**
> 
>  _/\* watch_id is the watcher id to cancel so that no more events are transmitted. */_
> 
> **  int64 watch_id =1;**
> 
> **}**

Watch_ID - 要取消的watch的ID，以便不再传输任何事件。

* * *

(四)    Lease API
================

* * *

Lease是一种检测客户活跃度的机制。集群授予Lease生存时间。如果etcd集群在给定的TTL周期内没有收到keepAlive，则Lease到期。

为了将Lease绑定到key-value存储中，每个key可以附加到最多一个Lease。当Lease到期或被撤销时，附加到该Lease的所有key都将被删除。每个过期的key在事件历史记录中生成删除事件。

* * *

**_4.1   LeaseGrantRequest （ 获得Lease）_**

* * *

> _message_**   LeaseGrantRequest{**
> 
>  _/\* TTL is the advisory time-to-live in seconds. Expired lease will return -1. */_
> 
> **  int64    TTL =1;**
> 
> _/\* ID is the requested ID for the lease. If ID is set to 0, the lessor chooses an ID. */_
> 
> **  int64    ID =2;**
> 
> **}**

TTL --- 咨询生存时间，以秒为单位。

ID --- 请求的Lease ID。如果ID设置为0，则etcd将选择一个ID。

* * *

**_4.2    LeaseGrantResponse_**

* * *

> _message_ **LeaseGrantResponse{**
> 
> **  ResponseHeader header =1;**
> 
> _/\* ID is the lease ID for the granted lease. */_
> 
> **  int64 ID =2;**
> 
> _/\* TTL is the server chosen lease time-to-live in seconds. */_
> 
> **  int64 TTL =3;**
> 
> **string error =4;**
> 
> **}**

ID - 授予的LeaseID。

TTL - 服务器为Lease选择的生存时间（以秒为单位）。

* * *

**_4.3    LeaseRevokeRequest_**

* * *

> _message_**   LeaseRevokeRequest{**
> 
> _  /\* ID is the lease ID to revoke. When the ID is revoked, all associated keys will be deleted. */_
> 
> **  int64    ID =1;**
> 
> **}**

ID --- 要撤消的LeaseID。撤销Lease后，将删除所有附加的key。

* * *

**_4.4    LeaseRevokeResponse_**

* * *

> **_message_**   **LeaseRevokeResponse{**
> 
> **  ResponseHeader header =1;**
> 
> **}**

* * *

**_4.5    LeaseKeepAliveRequest_**

* * *

使用通过LeaseKeepAliveAPI调用创建的双向流刷新Lease。当客户希望刷新Lease时，它会通过LeaseKeepAliveRequest流发送：

> _message_ **LeaseKeepAliveRequest{**
> 
> _  /\* ID is the lease ID for the lease to keep alive. */_
> 
> **  int64 ID =1;**
> 
> **}**

ID - 要保持活动的Lease的ID。

* * *

**_4.6    LeaseKeepAliveResponse_**

* * *

> _message_ **LeaseKeepAliveResponse{**
> 
> **  ResponseHeader header =1;**
> 
> _/\* ID is the lease ID from the keep alive request. */_
> 
> **  int64   ID =2;**
> 
> _/\* TTL is the new time-to-live for the lease.*/_
> 
> **  int64   TTL =3;**
> 
> **}**

ID - 使用新TTL刷新的Lease。

TTL - Lease剩余的新生存时间（以秒为单位）。

