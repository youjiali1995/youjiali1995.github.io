---
title: Raft 笔记(四) -- Leader election
excerpt: 介绍 etcd/raft 的 leader election 相关实现和优化。
layout: post
categories: Raft
---

{% include toc %}

## raft 结构体
`raft` 结构体中和 `leader election` 相关的状态主要如下:
* `id`: 当前节点的 `id`；
* `Term`: 当前 `term`；
* `Vote`: 投票给哪个节点；
* `raftLog`: 用于获取 `log` 最后一个 `entry` 的信息；
* `state`: 当前节点的状态：`leader`、`follower`、`candidate` 等；

## leader election
在集群刚启动时，所有节点的状态都为 `follower`，等待超时触发 `leader election`。

### election timeout
超时时间由 `Config` 设置。`etcd/raft` 没有用真实时间而是使用逻辑时钟，当调用 `tick` 的次数超过指定次数时触发超时事件。
对于 `follower` 和 `cadidate` 而言，`tick` 中会判断是否超时，若超时则会本地生成一个 `MsgHup` 类型的消息触发 `leader election`:
```go
// tickElection is run by followers and candidates after r.electionTimeout.
func (r *raft) tickElection() {
	r.electionElapsed++

	if r.promotable() && r.pastElectionTimeout() {
		r.electionElapsed = 0
		r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
	}
}
```

### 消息处理
`step` 方法是消息处理的入口，不同的 `state` 处理的消息不同且处理方式不同，所以有多个 `step` 方法:
* `raft.Step()`: 消息处理的入口，做一些共性的检查，如 `term`，或处理所有状态都需要处理的消息。若需要更进一步处理，会根据状态
调用下面的方法:
    * `raft.stepLeader()`: `leader` 状态的消息处理方法；
    * `raft.stepFollower()`: `follower` 状态的消息处理方法；
    * `raft.stepCandidate()`: `candidate` 状态的消息处理方法。

`MsgHup` 是本地消息(`term` 为0)，用于触发 `leader election`，非 `leader` 状态的节点会调用 `raft.campaign()`:
1. 成为 `candidate`，增加 `term`，给自己投票，重置状态:
```go
func (r *raft) becomeCandidate() {
	// TODO(xiangli) remove the panic when the raft implementation is stable
	if r.state == StateLeader {
		panic("invalid transition [leader -> candidate]")
	}
	r.step = stepCandidate
	r.reset(r.Term + 1) // 增加 term，重置超时时间，重置每个节点状态等
	r.tick = r.tickElection
	r.Vote = r.id
	r.state = StateCandidate
	r.logger.Infof("%x became candidate at term %d", r.id, r.Term)
}
```
2. 调用 `raft.poll()` 给自己投票，若是单节点的集群，则立刻成为 `leader`；
3. 给集群内其他节点发送 `MsgVote` 消息，这里**只是把需要发送的消息追加到 slice**，之后通过 `Ready` 交给用户处理:
```go
r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
```

当收到其他节点发起的投票消息时，处理过程如下：
1. 成为 `follower`: 消息中包含了发起投票节点的 `term`，一般会比当前节点大。这里为了避免 `split vote` 的发生，会重置 `electionElapsed`；
2. 判断是否能够投票:
```go
// We can vote if this is a repeat of a vote we've already cast...
canVote := r.Vote == m.From ||
    // ...we haven't voted and we don't think there's a leader yet in this term...
    (r.Vote == None && r.lead == None) ||
    // ...or this is a PreVote for a future term...
    (m.Type == pb.MsgPreVote && m.Term > r.Term)
```
3. 根据 `msg.Index` 和 `msg.LogTerm` 比较 `log` 新旧:
```go
func (l *raftLog) isUpToDate(lasti, term uint64) bool {
	return term > l.lastTerm() || (term == l.lastTerm() && lasti >= l.lastIndex())
}
```
4. 返回响应:
    * 同意:
```go
r.send(pb.Message{To: m.From, Term: m.Term, Type: voteRespMsgType(m.Type)})
if m.Type == pb.MsgVote {
    // Only record real votes.
    r.electionElapsed = 0
    r.Vote = m.From // 当前 term 只会给一个节点投票
}
```
    * 拒绝：
```go
r.send(pb.Message{To: m.From, Term: r.Term, Type: voteRespMsgType(m.Type), Reject: true})
```

当 `candidate` 收到投票响应后，会统计收到的票数，收到 `majority` 的投票后会变为 `leader`，并立即发送空的 `msgApp` 给其他节点：
```go
case myVoteRespType:
    // poll() 的实现很简单，使用 map 记录每个节点的投票信息，然后返回 granted 的个数
    gr := r.poll(m.From, m.Type, !m.Reject)
    r.logger.Infof("%x [quorum:%d] has received %d %s votes and %d vote rejections", r.id, r.quorum(), gr, m.Type, len(r.votes)-gr)
    switch r.quorum() {
    case gr: // 收到 quorum 的投票
        if r.state == StatePreCandidate {
            r.campaign(campaignElection)
        } else {
            r.becomeLeader()
            r.bcastAppend()
        }
    case len(r.votes) - gr: // 收到 quorum 的拒绝
        // pb.MsgPreVoteResp contains future term of pre-candidate
        // m.Term > r.Term; reuse r.Term
        r.becomeFollower(r.Term, None)
    }
```

### heartbeat
收到 `majority` 的投票后成为 `leader`，`tick` 变为 `tickHeartbeat`。触发 `heartbeat` 的逻辑和触发 `election` 相同。
超时后会处理一个本地消息 `MsgBeat`，然后给其他节点发送 `MsgHeartbeat`。
其他节点收到 `heartbeat` 后会重置超时时间(`follower`)或成为 `follower`(同时选举的 `candidate`)，然后返回给 `leader` 响应。`leader` 处理 `MsgHeartbeatResp` 的逻辑后面再看。

### 一些细节
* `raft` 会忽略大部分 `term` 比自己小的消息；
* 改变状态的时候会重置超时时间，并随机设置 `election timeout`：包括触发选举、收到 `term` 比自己高的消息时、同时选举的 `candidate` 收到 `leader` 的消息时；
```go
func (r *raft) resetRandomizedElectionTimeout() {
	r.randomizedElectionTimeout = r.electionTimeout + globalRand.Intn(r.electionTimeout)
}
```
* `leader` 发送 `heartbeat` 时，其余节点只会 `check` 消息的 `term`，而不会比较 `log`，这是因为每个 `term` 只会有一个有效 `leader`;
* 当选举成功后，新 `leader` 会立即发送一个空的 `MsgApp` 给其他节点，这是为了及时更新 `commit index`，保证 `linearizablity`(在与客户端交互的时候再详细介绍)；
* 当 `leader` 收到 `heartbeat` 响应后会立即发送滞后的 `log entries` 给对端，此时可以认为对端是就绪的：
```go
if pr.Match < r.raftLog.lastIndex() {
    r.sendAppend(m.From)
}
```

## 优化
现在考虑一种场景，一个节点被 `partition` 了，导致这个节点收不到集群内其他节点的消息，然后一直超时触发 `leader election` 并增加的 `term`。
当网络恢复后，其他节点收到了它的 `MsgVote`，导致原来还在正常工作的 `leader` 变成 `follower`，触发不必要的 `leader election` 过程。

### Pre-Vote
`Pre-Vote` 用于解决上述问题。
