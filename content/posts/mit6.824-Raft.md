---
title: "Raft总结"
date: "2020-04-22"
draft: false
tags: ["MIT6.824", "分布式系统", "Golang"]
---

## 前言

这里是对MIT6.824课程中有关Raft部分的一些总结（包括一些个人的理解以及Lab2的实现思路等）。首先还是很感谢MIT的大牛们能把这门神课开源出来，让更多的人能更加系统的学习分布式系统（搭配《DDIA》效果更佳！）。



## 简介

> 因为需要遵守 Collaboration Policy 不能公布全部代码，主要是思路和伪代码。

Raft是一个用于管理副本log的共识算法，它的功能类似于Paxos，但是比Paxos更简单（相对而言），更容易让人去理解学习，也容易在现实中实现。Raft主要有这几部分：领导选举、日志复制、集群变更、日志持久化、快照，本文只讨论与Lab2有关的部分。

### 共识算法---详细可以去看书

> 在6.824课程和《Raft》论文中对共识的阐释没有系统具体的阐释，这里主要是引用《DDIA》中的关于共识的部分内容：

共识问题是分布式计算中最重要也是最基本的问题之一。共识其实就是让几个节点就某件事情达成一致。这样表面上看去不是太难，但是因为分布式系统本身就是不可靠的，存在各种问题、失败、故障（网络故障、进程暂停、时钟问题等），想让多个节点到达共识其实是存在很大的难度。

共识算法需要满足一些性质：

* 协商一致性（Uniform agreement）：所有节点都接受相同的决议
* 诚实性（Integrity）：所有节点不能反悔，即对一项提议不能有两次决定
* 合法性（Validity）：如果决定了值v，则v一定由某个节点所提议
* 可终止性（Termination）：节点如果不崩溃则最终一定可以达成协议

协商一致性和诚实性决定了共识算法的核心思想：**决定一致的结果，一旦决定，就不能改变**。合法性（有效性，书里面会混用，属于翻译问题）则是排除一些无意义的方案，例如值为空（NULL）。前三个主要是保障了安全性（Safety），而可终止性则是引入了容错的思想。

> 2PC，两阶段提交也是共识算法但是不具有容错性，因为它的leader（独裁者、协调者）是人为指定的，也就没有可终止性，如果协调者故障则系统会一直原地空转

最著名的容错共识算法包括：VSR、Paxos、Raft和Zab（Zookeeper），他们大都不是直接使用上面的形式化描述。相反，他们是决定了一系列值，然后采用全序关系广播算法。



## 具体实现（思路和伪代码）

### 领导选举（Lab2A）

领导选举其实就是一次共识的过程，需要一个Candidate节点发起election让其与的节点来进行投票（vote）。

1. 首先这里先介绍一下Raft中每个节点能够扮演的角色：

   * Follower：只负责接收Leader、Candidate的请求来进行响应，自身无法主动发起任何请求，但是会有一个election time，如果到期则会成为Candidate节点
   * Candidate：自身的term+1，重置election time，启动一个election vote活动，向集群中的所有节点发送vote请求，如果能在下一次election time到期之前获得大多数（Majority）节点的投票则会成为Leader（Candidate自己会投给自己），否则会开启新一轮的election
   * Leader：每个term最多只有一个Leader，Leader会定时向集群中的节点发送心跳包来阻止新一轮的选举，同时会接受client的请求并复制到Follower节点

2. 领导选举的整个流程：

   ![](https://images-1253801505.cos.ap-nanjing.myqcloud.com/blog/image-20200419130504004.png)

   * 首先所有节点初始化为Follower状态，并且设置election time（注意为了防止出现多个Candidate 的情况，需要在指定区间内随机time的值）

   * 当time out时某个Follower会成为Candidate，开启一轮选举，如果获得大多数投票则选举成功，成为Leader，否则等待下一次time out开启新一轮选举。

   * Leader会以更短（比election time）的时间间隔向其他节点发送心跳包防止进行新选举

基本上需要按照论文中Figure2中的表示进行编写代码，不然容易出bug。这里需要注意的一点：

代码主要分为三部分：Election time超时机制，Candidate发起选举，`RequestVote` RPC处理

#### Election time超时机制。

首先每个Raft实例都会启动一个`attemptElection`的goroutine

```go
func (rf *Raft) attemptElection() {
	for !rf.killed() {
		timeout := getRandTime() // 预定超时时间
		time.Sleep(timeout)
		if time.Since(rf.lastHeartMsg) > timeout && rf.state != Leader {
			rf.mu.Unlock()
			// 进行选举
		go rf.kickOffElection()
		} 
	}
}
```

`timeout`是在一个指定时间区间里一个随机值。

由于Lab2的讲义中建议不要使用`time.Timer` 或者 `time.Ticker`（因为容易出错），所以这里我使用用`time.Sleep`，并且会用`lastHeartMsg`来记录上一次更新的时间，用`time.Now() - lastHeartMsg`再和预定的超时时间（`timeout`）来进行比较。

这里有个小问题就是当该Raft节点成为Leader后这个goroutine还是会继续执行，但是不影响整体的结果，对于效率而言该goroutine大部分时间处理休眠状态不占用CPU，所以为了代码逻辑的简洁就这样实现了（其实是懒）



#### Candidate发起选举。

关于开启一轮新的选举的实现思路：

* 首先需要更新节点的信息和状态（term、Candidate和votefor）
* 并发的向除自己外的所有节点发送请求
* 当获得大多数投票时停止其他请求（未完成），成为Leader

这里的难点主要是并发发送请求，我的实现方式是遍历所有节点peer，对每个peer启动一个goroutine进行发送请求以及投票计数和leader转化

```go
for i := range rf.peers {
	if i == rf.me {
		continue
	}
	go func(x int) {
		accept, acceptTerm := rf.callRequestVote(x, term, lastLogIndex, lastLogTerm)
		if acceptTerm == -1 { // call rpc fail
			return
		}
		rf.mu.Lock()
		defer rf.mu.Unlock()
		if !accept {
			if acceptTerm > term {
				rf.convertToFollower(acceptTerm)
			}
			return
		}
		voteCnt++
		if voteCnt <= len(rf.peers)/2 {
			return
		}
   // 需要二次检查
		if rf.state == Candidate && rf.currentTerm == term {
			rf.convertToLeader()
		}
	}(i)
}
```

这里需要注意的是在发送请求进行rpc调用的时候不能加锁，不然可能这个rpc一直阻塞那么整个节点都会一直处于加锁状态。这里关于投票计数有两种方式（我了解到），一种是上述直接在每个goroutine中利用闭包的特性进行计数（不过需要加锁），另一种是使用条件变量`sync.Cond`在外部进行统计（个人觉得第一种方式更加简单）。

在转化成Leader时需要特判一下该节点是否还是Candidate以及currentTerm是否是发起这轮选举时记录的那个term，导致这种情况可能是因为这个Candidate已经成为了Follower或者因为网络延迟开启了新的一轮选举（接收到了上一轮请求的回复）



#### `RequestVote`RPC的处理。

主要就是确认是否要投票给这个Candidate。首先可以明确拒绝的情况有：

* `args.Term <= currentTerm`
* 如果Candidate的日志没有比自己的日志**up-to-date**的话，即：比较最后一个log的term，如果相等则比较log的长度

```go
if lastLog.Term > args.LastLogTerm || (lastLog.Term == args.LastLogTerm && len(rf.log)-1 > args.LastLogIndex) {
		reply.VoteGranted = false
		return
}
```

这里有个需要注意的点就是关于该Raft节点的Election time的重置时机，需要时确定要投票给这个Candidate才要去重置，不然会导致日志比较落后的节点一直发起选举，而拥有最新日志的节点则一直被重置，最终无法选择出一个Leader

关于`RequestVote`RPC的处理中，不能投票给`args.Term == rf.currentTerm`的Candidate，不然可能会出现两个不断交替的Leader（由于某条网络连接故障）

```go
 func (rf *Raft) RequestVote(args *RequestVoteArgs, reply 	*RequestVoteReply) {
 // 检查term
 if rf.currentTerm >= args.Term {
   // 拒绝投票
   return
 }
 // 检查candidate的log是否比自己的更 up-to-date
 lastLog := rf.log[len(rf.log)-1]
 if lastLog.Term > args.LastLogTerm || (lastLog.Term == args.LastLogTerm && len(rf.log)-1 > args.LastLogIndex) {
   // 拒绝投票
   return
 }
 // 投票给Candidate
 }
```



### 日志复制（Lab2B）

日志复制主要可以分为这么几个部分：

1. 接收上层的command命令；

2. Leader的心跳包和日志同步；

3. `AppendEntries`RPC的处理；

4. 应用日志结果（通知上层可以应用该command）

   <!--上层程序可以是很多：kv数据库、GFS、MapReduce等等，Raft其实相当于一个Lib库-->

日志复制需要实现：

1.  只有`commited`的日志项才能被上层程序应用；
2. 只有被大多数节点复制备份的日志项才能被标记为`commited`
3. 需要保证所有节点的日志项被上层程序应用的顺序相同（这样才能保证所有节点最终会达到相同的state）

日志复制示意图：

![](https://images-1253801505.cos.ap-nanjing.myqcloud.com/blog/image-20200420154327676.png)



#### 接收上层的command命令 

这部分主要是完善Lab中已经给出的`Start`方法，需要定义一下日志（log）的数据结构，然后要注意只有Leader的节点才能执行`Start`方法，然后将上层给出的`command`构造成一个日志项加入到该节点的日志中

```go
if rf.state == Leader {
			index = len(rf.log)
			term = rf.currentTerm
			isLeader = true

			rf.log = append(rf.log, LogEntry{
				Term:    term,
				Command: command,
			})
}
```



#### Leader的心跳包和日志同步（复制）

1. 心跳包主要是为了及时的重置其他节点（非Leader节点）的election time，防止发生新一轮的选举
2. 日志同步是夹杂在心跳包之中（换句话说如果`AppendEntries`RPC的`args.Entries`属性为空就是心跳包，否则就是日志同步）

>  关于什么时候应该进行日志同步？

在每一次心跳time到期的时候需要向其他所有节点并发发送请求（开goroutine进行RPC调用），这里在对于某个节点（编号`x`）发送之前需要检查对比一下`nextIndex[x]`和`log`的最后一项下标的大小，如果`log`最后一项下标`>= nextIndex[x]`则需要进行日志同步

这里有一个坑：就是当节点`x`日志同步成功时，关于更新`nextIndex[x]`和`matchIndex[x]`，不能直接将`nextIndex[x]`的值直接更新为`len(log)-1`因为在rpc调用的过程中这个节点是没有加锁的，那么`log`的长度可能会增加，所以应该用`len(args.Entries) + args.PrevLogIndex`来更新`nextIndex[x]`和`matchIndex[x]`

当节点`x`日志同步失败（如果当前Leader节点已经**退位**的话就直接成为Follower，这里不讨论这种情况）的时候需要回滚`nextIndex[x]`，有很多方法，本文主要说两种：一项一项的回滚，快速回滚

* 一项一项的回滚就是如果节点`x`调用失败，直接`nextIndex[x]--`

* 快速回滚，在论文中只是稍微提及（作者似乎觉得不必要），但是**Robert Tappan Morris**教授（传奇大牛）课上说如果不进行快速回滚效率会很低（特别是当某个节点宕机很长时间后再进行日志同步），而且会过不了lab test。这里才用了[TA的blog](https://thesquareplanet.com/blog/students-guide-to-raft/)中提起的方法：

  * `AppendEntries`当失败时会在`reply`中设置两个变量`ConflictTerm`和`ConflictIndex`
  * 首先在本节点`log`中查找是否有`log的term == ConflictTerm`，如果存在则将该`log中属于该term`的最后一项下标设置为`nextIndex[x]`
  * 如果没有找到则设置`nextIndex[x] = ConflictIndex`

  关于`ConflictTerm`和`ConflictIndex`的设置在`AppendEntries`的处理会说明

如果当日志同步到大多数节点时，Leader节点会更新自己的`commitIndex`，其值根据`matchIndex`数组决定，然后下一次心跳包发送的时候就带上这个`commitIndex`

*这部分是lab中的核心，很有意思就不贴代码了，每个人可以自己去设计自己的方案*

#### `AppendEntries`RPC的处理

`AppendEntries`的处理主要分为三个部分：`log`一致性检查；`args.Entries`处理；更新`commitIndex`

* `log`一致性检查。主要是根据`PrevLogIndex`和`PrevLogTerm`检查自己和Leader的`log`是否一致（`args.Entries`之前的部分）
  * 如果本节点的`len(log) <= PrevLogIndex`则说明不一致，并且设置`ConflictTerm = null，ConflictIndex = len(log)`
  * 如果本节点在`PrevLogIndex`位置上的`term != ConflictTerm `，则将`ConflictTerm = term`并且将`ConflictIndex`设置为`ConflictTerm`的第一项的索引下表
* `args.Entries`处理。通过一致性检查的话，需要对`Entries`做两件事情：将本节点`log`中与`Entries`不一致的项给截断，同时需要过滤掉本节点`log`中已经存在的项（过滤重复项，主要是因为网络延迟等问题可能会接收到多个相同的请求）
* 更新`commitIndex`。通过一致性检查的话，如果`LeaderCommitIndex > 本节点的commitIndex`，那么需要更新本节点的`commitIndex`，这里需要注意不能直接更新成`LeaderCommitIndex`因为可能其值会大于本节点`len(log)`即`args.PrevLogIndex + len(args.Entries)`，所以需要更新为其中较小的那个值

*这部分是lab中的核心，很有意思就不贴代码了，每个人可以自己去设计自己的方案*

#### 应用日志结果

应用日志结果在Lab2中就是将已经标记为`commited`的日志项通过`channel applyCh`进行传输。这里有一个属性`lastApplied`记录已经应用的项（防止重复传输）

`lastApplied`的更新是通过比较`commitIndex`来更新，如果`rf.commitIndex > rf.lastApplied`则`rf.lastApplied += 1`，将`log[rf.lastApplied]`处的项传入`applyCh`通道中

这个和election time倒计时以及Leader的定时心跳包一样都是需要定时运行（在我的实现中）

```go
for !rf.killed() {
	time.Sleep(50 * time.Millisecond)
	rf.mu.Lock()
	for rf.commitIndex > rf.lastApplied {
		rf.lastApplied += 1
		applyLog := ApplyMsg{
			CommandValid: true,
			Command:      rf.log[rf.lastApplied].Command,
			CommandIndex: rf.lastApplied,
		}
		rf.mu.Unlock()
	// 这里最好不要加锁
		rf.applyCh <- applyLog
		rf.mu.Lock()
	}
	rf.mu.Unlock()
}
```

### 日志持久化（Lab2C）

> 这里的思路和实现都比较简单，如果在2C test失败的话可能是由于2B的部分逻辑问题（比如没有考虑到一些网络阻塞的情况）

Lab2C要求实现`persist`和`readPersist`两个方法，而且给出了例子这里就不多说了。在Lab中并没有真正的持久化到硬盘存储中，是在外部使用`Persister`结构来存储（说白了就是存在内存中，在测试的过程中只是重启了Raft程序的进程，没有重启整个test程序）

那么2C中只需要考虑`persist`的使用时机问题，可以分析需要持久化的属性，再根据代码实现中这些值作出变更的时候进行持久化。但是这里就需要注意加锁的问题，因为可能再Raft对应属性变更的时候已经加锁了，而`persist`又在内部需要加锁，这时候就要记得释放锁

```go
// persist的实现
func (rf *Raft) persist() {
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	rf.mu.Lock() // 这里进行了加锁
	e.Encode(rf.currentTerm)
	e.Encode(rf.votedFor)
	e.Encode(rf.log)
	rf.mu.Unlock()
	data := w.Bytes()
	rf.persister.SaveRaftState(data)
}
// persist的应用
// 比如在RequestVote的实现中，可能需要更改本节点的currentTerm
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	// 一些操作
	rf.state = Follower
	rf.currentTerm = args.Term
	rf.mu.Unlock()
	rf.persist()
	rf.mu.Lock()
  // 后续操作
}
```

## 总结

这是我第二次做6.824，第一次的时候是三个月以前，感觉自己确实进步了很多，实际写代码的时候可能总共就三天吧，但是学到这里却用了将近一个月（自己也比较懒），主要的时间都在读论文、看视频、看相关的资料，因为大部分都是英文（自己英语比较差）所以花了很多时间，但是成效也很明显，英文文献阅读能力提高了很多（虽然还是需要借助翻译工具），这里推荐曹大的一篇[blog](https://xargin.com/how-to-learn/)

## 参考资料

* [mit6.824](https://pdos.csail.mit.edu/6.824/schedule.html) 
* [Students' Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/) 
* [MIT 6.824 2020 Raft 实现细节备忘](https://www.qtmuniao.com/2020/01/18/raft-implement-points/)
* [Raft总结](https://mr-dai.github.io/raft/)

