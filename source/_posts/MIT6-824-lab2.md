---
title: "MIT 6.5840(6.824) - lab 2: Key/Value Server"
date: 2024-02-22 12:30:28
tags:
- 分布式
- csdiy
categories: 学点新东西
---

> In this lab you will build a key/value server for a single machine that ensures that each operation is executed exactly once despite network failures and that the operations are [linearizable](https://pdos.csail.mit.edu/6.824/papers/linearizability-faq.txt). Later labs will replicate a server like this one to handle server crashes.

lab 2 是一个比较简单的单机键值服务器。需要实现三种RPC调用：`Put` `Append` `Get`。由于服务器是单机的，所以并不需要考虑一致性问题，只需要保证线性执行即可。

<!--more-->

# 实现过程

lab 2 分为两个子任务，先完成没有网络故障的版本，再修改为可处理网络故障的版本。

## 无故障版

> Your first task is to implement a solution that works when there are no dropped messages. 
>
> You'll need to add RPC-sending code to the Clerk Put/Append/Get methods in `client.go`, and implement `Put`, `Append()` and `Get()` RPC handlers in `server.go`. 
>
> You have completed this task when you pass the first two tests in the test suite: "one client" and "many clients".

非常简单，客户端只需要完成RPC的调用；服务端在执行RPC调用时加入`sync.Mutex`避免数据竞争即可。

## 有故障版

> Now you should modify your solution to continue in the face of dropped messages (e.g., RPC requests and RPC replies). If a message was lost, then the client's `ck.server.Call()` will return `false` (more precisely, `Call()` waits for a reply message for a timeout interval, and returns false if no reply arrives within that time). One problem you'll face is that a `Clerk` may have to send an RPC multiple times until it succeeds. Each call to `Clerk.Put()` or `Clerk.Append()`, however, should result in just a *single* execution, so you will have to ensure that the re-send doesn't result in the server executing the request twice.
>
> Add code to `Clerk` to retry if doesn't receive a reply, and to  `server.go` to filter duplicates if the operation requires it. These notes include guidance on [duplicate detection](https://pdos.csail.mit.edu/6.824/notes/l-raft-QA.txt).

对于有网络故障的版本，需要解决的问题有两个：

- 客户端没收到回复：

  直接重新请求！

- 服务端收到重复请求：

  这是由客户端的重新请求引入的，需要根据请求的函数不同来做出不同的相应。

### 客户端

```go
func (ck *Clerk) Get(key string) (value string) {
	for done := false; !done; {
		args := GetArgs{}
		args.Key = key
		args.ClerkId = ck.id
		args.OperationId = ck.operationId
		reply := GetReply{}
		ok := ck.server.Call("KVServer.Get", &args, &reply)
		if ok {
			value = reply.Value
			ck.operationId++
			done = true
		}
	}
	return
}

func (ck *Clerk) PutAppend(key string, value string, op string) (oldValue string) {
	for done := false; !done; {
		args := PutAppendArgs{}
		args.Key = key
		args.Value = value
		args.ClerkId = ck.id
		args.OperationId = ck.operationId
		reply := PutAppendReply{}
		ok := ck.server.Call("KVServer."+op, &args, &reply)
		if ok {
			oldValue = reply.Value
			ck.operationId++
			done = true
		}
	}
	return
}
```

直接把请求塞进for里即可。

### 服务端

对于三种不同调用，有不同的逻辑：

- Put/Append：由于它们会更改服务端的状态，要避免重复执行，所以需要维护上一次请求的内容。
- Get：不会更改服务端的状态，直接执行即可。

```go
type operationHistory struct {
	operationId int64
	value       string
}

type KVServer struct {
	mu            sync.Mutex
	KvMap         map[string]string
	lastOperation map[int64]operationHistory
}
```

使用一个map来维护对应客户端的上一次（且需要维护的）请求的历史记录。

- 在成功执行Put/Append后新增一条
- 在成功执行任意调用后，删除或覆盖上一次的历史记录

此时即可得出三种调用的修改方法：

```go
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	delete(kv.lastOperation, args.ClerkId)
	value, ok := kv.KvMap[args.Key]
	if ok {
		reply.Value = value
	}
}

func (kv *KVServer) Put(args *PutAppendArgs, reply *PutAppendReply) {
	_ = reply
	kv.mu.Lock()
	defer kv.mu.Unlock()
	_, result := kv.checkLastOperation(args.ClerkId, args.OperationId)
	switch result {
	case lastOperation:
	case newOperation:
		kv.KvMap[args.Key] = args.Value
		kv.lastOperation[args.ClerkId] = operationHistory{
			operationId: args.OperationId,
			value:       "",
		}
	}
}

func (kv *KVServer) Append(args *PutAppendArgs, reply *PutAppendReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	lastValue, result := kv.checkLastOperation(args.ClerkId, args.OperationId)
	switch result {
	case lastOperation:
		reply.Value = lastValue
	case newOperation:
		oldValue, ok := kv.KvMap[args.Key]
		if !ok {
			oldValue = ""
		}
		kv.KvMap[args.Key] = oldValue + args.Value
		reply.Value = oldValue
		kv.lastOperation[args.ClerkId] = operationHistory{
			operationId: args.OperationId,
			value:       oldValue,
		}
	}
}

func (kv *KVServer) checkLastOperation(clerkId int64, operationId int64) (lastValue string, result historyCheckResult) {
	history, ok := kv.lastOperation[clerkId]
	if ok {
		lastValue = history.value
		if operationId == history.operationId {
			result = lastOperation
		} else {
			result = newOperation
		}
	} else {
		result = newOperation
	}
	return
}
```

其中，Get直接删除已有的历史记录（及时释放内存）；Put/Append通过创建新的历史记录，覆盖已有的内容。
