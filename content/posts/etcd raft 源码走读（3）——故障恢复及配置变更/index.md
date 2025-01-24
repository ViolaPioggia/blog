---
title: "etcd raft 源码走读（3）——故障恢复及配置变更"
date: "2025-01-21"
---

# 结构

```go
// Changer facilitates configuration changes. It exposes methods to handle
// simple and joint consensus while performing the proper validation that allows
// refusing invalid configuration changes before they affect the active
// configuration.
type Changer struct {
	Tracker   tracker.ProgressTracker
	LastIndex uint64
}

// ProgressTracker tracks the currently active configuration and the information
// known about the nodes and learners in it. In particular, it tracks the match
// index for each peer which in turn allows reasoning about the committed index.
type ProgressTracker struct {
	Config

	// leader 视角的 follower 的状态
	Progress ProgressMap

	Votes map[uint64]bool

	MaxInflight      int
	MaxInflightBytes uint64
}

// Config reflects the configuration tracked in a ProgressTracker.
type Config struct {
	Voters quorum.JointConfig
	// AutoLeave is true if the configuration is joint and a transition to the
	// incoming configuration should be carried out automatically by Raft when
	// this is possible. If false, the configuration will be joint until the
	// application initiates the transition manually.
	AutoLeave bool
	// Learners is a set of IDs corresponding to the learners active in the
	// current configuration.
	//
	// Invariant: Learners and Voters does not intersect, i.e. if a peer is in
	// either half of the joint config, it can't be a learner; if it is a
	// learner it can't be in either half of the joint config. This invariant
	// simplifies the implementation since it allows peers to have clarity about
	// its current role without taking into account joint consensus.
	Learners map[uint64]struct{}
	// When we turn a voter into a learner during a joint consensus transition,
	// we cannot add the learner directly when entering the joint state. This is
	// because this would violate the invariant that the intersection of
	// voters and learners is empty. For example, assume a Voter is removed and
	// immediately re-added as a learner (or in other words, it is demoted):
	//
	// Initially, the configuration will be
	//
	//   voters:   {1 2 3}
	//   learners: {}
	//
	// and we want to demote 3. Entering the joint configuration, we naively get
	//
	//   voters:   {1 2} & {1 2 3}
	//   learners: {3}
	//
	// but this violates the invariant (3 is both voter and learner). Instead,
	// we get
	//
	//   voters:   {1 2} & {1 2 3}
	//   learners: {}
	//   next_learners: {3}
	//
	// Where 3 is now still purely a voter, but we are remembering the intention
	// to make it a learner upon transitioning into the final configuration:
	//
	//   voters:   {1 2}
	//   learners: {3}
	//   next_learners: {}
	//
	// Note that next_learners is not used while adding a learner that is not
	// also a voter in the joint config. In this case, the learner is added
	// right away when entering the joint configuration, so that it is caught up
	// as soon as possible.
	LearnersNext map[uint64]struct{}
}

```

# 配置变更

Joint Configuration 是实现 multi-raft 的关键，因为只有保障在配置动态变更过程中旧节点能正常接收日志的同时新节点也能同步日志，才能保障 raft 集群服务的高可用

在 etcd-raft 的实现中状态分为 incoming（新）和 outgoing（旧）

首先来看 EnterJoint 这个方法

```go
// EnterJoint verifies that the outgoing (=right) majority config of the joint
// config is empty and initializes it with a copy of the incoming (=left)
// majority config. That is, it transitions from
//
//	(1 2 3)&&()
//
// to
//
//	(1 2 3)&&(1 2 3).
//
// The supplied changes are then applied to the incoming majority config,
// resulting in a joint configuration that in terms of the Raft thesis[1]
// (Section 4.3) corresponds to `C_{new,old}`.
//
// [1]: https://github.com/ongardie/dissertation/blob/master/online-trim.pdf
func (c Changer) EnterJoint(autoLeave bool, ccs ...pb.ConfChangeSingle) (tracker.Config, tracker.ProgressMap, error) {
	// 拷贝复制配置和任务状态
	cfg, trk, err := c.checkAndCopy()
	if err != nil {
		return c.err(err)
	}
	// 只能进入一次 joint 状态
	if joint(cfg) {
		err := errors.New("config is already joint")
		return c.err(err)
	}
	// 如果进入 joint 状态变更后的节点数为空直接返回报错
	if len(incoming(cfg.Voters)) == 0 {
		// We allow adding nodes to an empty config for convenience (testing and
		// bootstrap), but you can't enter a joint state.
		err := errors.New("can't make a zero-voter config joint")
		return c.err(err)
	}
	// Clear the outgoing config.
	// 清除旧配置
	*outgoingPtr(&cfg.Voters) = quorum.MajorityConfig{}
	// Copy incoming to outgoing.
	// 更新新配置
	for id := range incoming(cfg.Voters) {
		outgoing(cfg.Voters)[id] = struct{}{}
	}
	// 应用配置变更
	if err := c.apply(&cfg, trk, ccs...); err != nil {
		return c.err(err)
	}
	// autoleave 参数控制旧节点自动移除
	cfg.AutoLeave = autoLeave
	return checkAndReturn(cfg, trk)
}

// apply a change to the configuration. By convention, changes to voters are
// always made to the incoming majority config Voters[0]. Voters[1] is either
// empty or preserves the outgoing majority configuration while in a joint state.
func (c Changer) apply(cfg *tracker.Config, trk tracker.ProgressMap, ccs ...pb.ConfChangeSingle) error {
	for _, cc := range ccs {
		if cc.NodeID == 0 {
			// etcd replaces the NodeID with zero if it decides (downstream of
			// raft) to not apply a change, so we have to have explicit code
			// here to ignore these.
			continue
		}
		switch cc.Type {
		// 增加节点
		case pb.ConfChangeAddNode:
			c.makeVoter(cfg, trk, cc.NodeID)
	  // 增加 Learner 节点
		case pb.ConfChangeAddLearnerNode:
			c.makeLearner(cfg, trk, cc.NodeID)
		// 移除节点	
		case pb.ConfChangeRemoveNode:
			c.remove(cfg, trk, cc.NodeID)
		// 更新节点	
		case pb.ConfChangeUpdateNode:
		default:
			return fmt.Errorf("unexpected conf type %d", cc.Type)
		}
	}
	if len(incoming(cfg.Voters)) == 0 {
		return errors.New("removed all voters")
	}
	return nil
}

// makeVoter adds or promotes the given ID to be a voter in the incoming
// majority config.
func (c Changer) makeVoter(cfg *tracker.Config, trk tracker.ProgressMap, id uint64) {
	pr := trk[id]
	if pr == nil {
		c.initProgress(cfg, trk, id, false /* isLearner */)
		return
	}

	pr.IsLearner = false
	nilAwareDelete(&cfg.Learners, id)
	nilAwareDelete(&cfg.LearnersNext, id)
	incoming(cfg.Voters)[id] = struct{}{}
}

func (c Changer) makeLearner(cfg *tracker.Config, trk tracker.ProgressMap, id uint64) {
	pr := trk[id]
	if pr == nil {
		c.initProgress(cfg, trk, id, true /* isLearner */)
		return
	}
	if pr.IsLearner {
		return
	}
	// Remove any existing voter in the incoming config...
	c.remove(cfg, trk, id)
	// ... but save the Progress.
	trk[id] = pr
	// Use LearnersNext if we can't add the learner to Learners directly, i.e.
	// if the peer is still tracked as a voter in the outgoing config. It will
	// be turned into a learner in LeaveJoint().
	//
	// Otherwise, add a regular learner right away.
	if _, onRight := outgoing(cfg.Voters)[id]; onRight {
		nilAwareAdd(&cfg.LearnersNext, id)
	} else {
		pr.IsLearner = true
		nilAwareAdd(&cfg.Learners, id)
	}
}
```

新加入的节点不会立刻参与投票，而是作为 Learner 的身份进行日志同步，依然是通过 AppendEntries 函数，在同步完成后，由集群通过 confChange 事件将其升级为 Voter

在 raft-etcd 中没有提供自动的动态配置变更，用户如果想加入节点必须手动调用`ApplyConfChange` 进行节点 Learner 和 Voter 状态的转换

LeaveJoint 也是同理，就不再赘述了

# 故障恢复

这里不会涉及 snapshot，snapshot 会留到下一节再探讨

在这里梳理一些我能够想到的有意思的边界 case，然后到代码中找寻一下对应的实现

## candidate 收到 leader log

如果是日志同步 MsgProp 请求则会返回报错；如果是 commit 请求则变为 follower 然后正常地 appendEntries。

至于为什么不都报错或者不都转换为 follower 状态，我个人的想法是如果网络延迟的提案信息可能会干扰正常的节点选举，所以应该直接 drop 掉；而如果是提交请求，那么此刻日志至少已经被初始化了，操作应该是幂等的。这里我也还是比较迷惑，不敢肯定。

```go
switch m.Type {
	case pb.MsgProp:
		r.logger.Infof("%x no leader at term %d; dropping proposal", r.id, r.Term)
		return ErrProposalDropped
	case pb.MsgApp:
		r.becomeFollower(m.Term, m.From) // always m.Term == r.Term
		r.handleAppendEntries(m)
```

## Leader 只能提交当前任期的 log

这也是一个记录在原论文中很经典的问题了，在下面这段代码中就能看到必须 matchTerm 才能满足 commit 的条件

```go
func (l *raftLog) maybeCommit(at entryID) bool {
	// NB: term should never be 0 on a commit because the leader campaigned at
	// least at term 1. But if it is 0 for some reason, we don't consider this a
	// term match.
	if at.term != 0 && at.index > l.committed && l.matchTerm(at) {
		l.commitTo(at.index)
		return true
	}
	return false
}
```

## 未提交最新 log 的节点能成为 leader 吗

这个问题好像也在论文中提到过，如果在 raft 集群中 leader 只成功向一半的节点同步并提交了最新的 log，这时候 leader 挂掉了，经过新一轮选举恰好未同步成功的节点能成为新的 leader 吗？

答案当然是不能啦， r.raftLog.isUpToDate(candLastID) 表明了一切

```go
	case pb.MsgVote, pb.MsgPreVote:
		// We can vote if this is a repeat of a vote we've already cast...
		canVote := r.Vote == m.From ||
			// ...we haven't voted and we don't think there's a leader yet in this term...
			(r.Vote == None && r.lead == None) ||
			// ...or this is a PreVote for a future term...
			(m.Type == pb.MsgPreVote && m.Term > r.Term)
		// ...and we believe the candidate is up to date.
		lastID := r.raftLog.lastEntryID()
		candLastID := entryID{term: m.LogTerm, index: m.Index}
		if canVote && r.raftLog.isUpToDate(candLastID) {
```

## 接收到比自己节点任期小的 msg 会发生什么

这个问题一拍脑门可能就会觉得是直接 ignore，但是事实往往没有这么简单

- 如果 msg 来自上一任 leader 的心跳检测或者提交请求，那么会返回一个 `*MsgAppResp`，目的是帮助快速 fresh 掉源节点可能出现的 candidate 状态，降级成 follower*
- 如果是预投票则返回 reject 的 `*MsgPreVoteResp`，因为如果直接 ignore 在存在高 term，少 log 的集群中可能会发生死锁*
- 如果是`*MsgStorageAppendResp` 且 snapshot 不为空则应用，因为依然是有效的，这个我们在后面的章节再说*
- 其他情况直接 ignore 掉

```go
case m.Term < r.Term:
		if (r.checkQuorum || r.preVote) && (m.Type == pb.MsgHeartbeat || m.Type == pb.MsgApp) {
			// We have received messages from a leader at a lower term. It is possible
			// that these messages were simply delayed in the network, but this could
			// also mean that this node has advanced its term number during a network
			// partition, and it is now unable to either win an election or to rejoin
			// the majority on the old term. If checkQuorum is false, this will be
			// handled by incrementing term numbers in response to MsgVote with a
			// higher term, but if checkQuorum is true we may not advance the term on
			// MsgVote and must generate other messages to advance the term. The net
			// result of these two features is to minimize the disruption caused by
			// nodes that have been removed from the cluster's configuration: a
			// removed node will send MsgVotes (or MsgPreVotes) which will be ignored,
			// but it will not receive MsgApp or MsgHeartbeat, so it will not create
			// disruptive term increases, by notifying leader of this node's activeness.
			// The above comments also true for Pre-Vote
			//
			// When follower gets isolated, it soon starts an election ending
			// up with a higher term than leader, although it won't receive enough
			// votes to win the election. When it regains connectivity, this response
			// with "pb.MsgAppResp" of higher term would force leader to step down.
			// However, this disruption is inevitable to free this stuck node with
			// fresh election. This can be prevented with Pre-Vote phase.
			r.send(pb.Message{To: m.From, Type: pb.MsgAppResp})
		} else if m.Type == pb.MsgPreVote {
			// Before Pre-Vote enable, there may have candidate with higher term,
			// but less log. After update to Pre-Vote, the cluster may deadlock if
			// we drop messages with a lower term.
			last := r.raftLog.lastEntryID()
			// TODO(pav-kv): it should be ok to simply print %+v of the lastEntryID.
			r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] rejected %s from %x [logterm: %d, index: %d] at term %d",
				r.id, last.term, last.index, r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term)
			r.send(pb.Message{To: m.From, Term: r.Term, Type: pb.MsgPreVoteResp, Reject: true})
		} else if m.Type == pb.MsgStorageAppendResp {
			if m.Index != 0 {
				// Don't consider the appended log entries to be stable because
				// they may have been overwritten in the unstable log during a
				// later term. See the comment in newStorageAppendResp for more
				// about this race.
				r.logger.Infof("%x [term: %d] ignored entry appends from a %s message with lower term [term: %d]",
					r.id, r.Term, m.Type, m.Term)
			}
			if m.Snapshot != nil {
				// Even if the snapshot applied under a different term, its
				// application is still valid. Snapshots carry committed
				// (term-independent) state.
				r.appliedSnap(m.Snapshot)
			}
		} else {
			// ignore other cases
			r.logger.Infof("%x [term: %d] ignored a %s message with lower term from %x [term: %d]",
				r.id, r.Term, m.Type, m.From, m.Term)
		}
		return nil
```