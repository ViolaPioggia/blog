---
title: "etcd raft 源码走读（2）——日志同步"
date: "2025-01-20"
---

# 前言

在上一节中我们阅读了关于 etcd-raft 选举的流程，在本小节中我们将学习关于日志同步的内容，这也是 raft 作为分布式一致性共识算法最核心的部分，客户端发起请求，Leader 将请求同步到每一个 Follower。

# 实现

etcd-raft 通过 Propose 方法进行日志同步，非常的清晰易懂，由注释可知日志同步过程中还是有数据丢失的风险，需要由用户端封装重试机制。

```go
	// Propose proposes that data be appended to the log. Note that proposals can be lost without
	// notice, therefore it is user's job to ensure proposal retries.
	Propose(ctx context.Context, data []byte) error
```

还是通过 step 方法，使用 MsgProp 类型进行标识日志同步

带 wait 可以通过一个 error 类型的 chan 获取到 result

```go
func (n *node) Propose(ctx context.Context, data []byte) error {
	return n.stepWait(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
}

propc      chan msgWithResult

//...
case pm := <-propc:
	m := pm.m
	m.From = r.id
	err := r.Step(m)
	if pm.result != nil {
		 pm.result <- err
		 close(pm.result)
	}
```

Follower 接收到这条消息之后会进入 stepFollower 的逻辑进行处理：

如果当前 leader 为空 或者当前节点设置为禁止 proposal 那么就回返回错误，

不为空则发送一个 pb.MsgProp 给 Leader

```go
	switch m.Type {
	case pb.MsgProp:
		if r.lead == None {
			r.logger.Infof("%x no leader at term %d; dropping proposal", r.id, r.Term)
			return ErrProposalDropped
		} else if r.disableProposalForwarding {
			r.logger.Infof("%x not forwarding to leader %x at term %d; dropping proposal", r.id, r.lead, r.Term)
			return ErrProposalDropped
		}
		m.To = r.lead
		r.send(m)
	}
```

在 stepLeader 中经过一系列鉴别，最后 appendEntry 和 bcastAppend 是关键：

```go
case pb.MsgProp:
		if len(m.Entries) == 0 {
			r.logger.Panicf("%x stepped empty MsgProp", r.id)
		}
		if r.trk.Progress[r.id] == nil {
			// If we are not currently a member of the range (i.e. this node
			// was removed from the configuration while serving as leader),
			// drop any new proposals.
			return ErrProposalDropped
		}
		if r.leadTransferee != None {
			r.logger.Debugf("%x [term %d] transfer leadership to %x is in progress; dropping proposal", r.id, r.Term, r.leadTransferee)
			return ErrProposalDropped
		}

		for i := range m.Entries {
			e := &m.Entries[i]
			var cc pb.ConfChangeI
			if e.Type == pb.EntryConfChange {
				var ccc pb.ConfChange
				if err := ccc.Unmarshal(e.Data); err != nil {
					panic(err)
				}
				cc = ccc
			} else if e.Type == pb.EntryConfChangeV2 {
				var ccc pb.ConfChangeV2
				if err := ccc.Unmarshal(e.Data); err != nil {
					panic(err)
				}
				cc = ccc
			}
			if cc != nil {
				alreadyPending := r.pendingConfIndex > r.raftLog.applied
				alreadyJoint := len(r.trk.Config.Voters[1]) > 0
				wantsLeaveJoint := len(cc.AsV2().Changes) == 0

				var failedCheck string
				if alreadyPending {
					failedCheck = fmt.Sprintf("possible unapplied conf change at index %d (applied to %d)", r.pendingConfIndex, r.raftLog.applied)
				} else if alreadyJoint && !wantsLeaveJoint {
					failedCheck = "must transition out of joint config first"
				} else if !alreadyJoint && wantsLeaveJoint {
					failedCheck = "not in joint state; refusing empty conf change"
				}

				if failedCheck != "" && !r.disableConfChangeValidation {
					r.logger.Infof("%x ignoring conf change %v at config %s: %s", r.id, cc, r.trk.Config, failedCheck)
					m.Entries[i] = pb.Entry{Type: pb.EntryNormal}
				} else {
					r.pendingConfIndex = r.raftLog.lastIndex() + uint64(i) + 1
					traceChangeConfEvent(cc, r)
				}
			}
		}

		if !r.appendEntry(m.Entries...) {
			return ErrProposalDropped
		}
		r.bcastAppend()
		return nil
```

```go
func (r *raft) appendEntry(es ...pb.Entry) (accepted bool) {
	// 获取最新 index
	li := r.raftLog.lastIndex()
	// 遍历追加的 entry，赋值上 term 和 index
	for i := range es {
		es[i].Term = r.Term
		es[i].Index = li + 1 + uint64(i)
	}
	// Track the size of this uncommitted proposal.
	if !r.increaseUncommittedSize(es) {
		r.logger.Warningf(
			"%x appending new entries to log would exceed uncommitted entry size limit; dropping proposal",
			r.id,
		)
		// Drop the proposal.
		return false
	}

	traceReplicate(r, es...)

	// use latest "last" index after truncate/append
	// 在 raftLog 中追加 entries，但不提交
	li = r.raftLog.append(es...)
	// The leader needs to self-ack the entries just appended once they have
	// been durably persisted (since it doesn't send an MsgApp to itself). This
	// response message will be added to msgsAfterAppend and delivered back to
	// this node after these entries have been written to stable storage. When
	// handled, this is roughly equivalent to:
	//
	//  r.trk.Progress[r.id].MaybeUpdate(e.Index)
	//  if r.maybeCommit() {
	//  	r.bcastAppend()
	//  }
	// 发送 pb.MsgAppResp 类型的通知，直到不稳定状态（例如 log entry 写和投票选举）
	// 已经被持久化到本地磁盘
	r.send(pb.Message{To: r.id, Type: pb.MsgAppResp, Index: li})
	return true
}

// increaseUncommittedSize computes the size of the proposed entries and
// determines whether they would push leader over its maxUncommittedSize limit.
// If the new entries would exceed the limit, the method returns false. If not,
// the increase in uncommitted entry size is recorded and the method returns
// true.
//
// Empty payloads are never refused. This is used both for appending an empty
// entry at a new leader's term, as well as leaving a joint configuration.
func (r *raft) increaseUncommittedSize(ents []pb.Entry) bool {
	s := payloadsSize(ents)
	// 防止未提交日志过多占用过多空间
	if r.uncommittedSize > 0 && s > 0 && r.uncommittedSize+s > r.maxUncommittedSize {
		// If the uncommitted tail of the Raft log is empty, allow any size
		// proposal. Otherwise, limit the size of the uncommitted tail of the
		// log and drop any proposal that would push the size over the limit.
		// Note the added requirement s>0 which is used to make sure that
		// appending single empty entries to the log always succeeds, used both
		// for replicating a new leader's initial empty entry, and for
		// auto-leaving joint configurations.
		return false
	}
	r.uncommittedSize += s
	return true
}

```

然后就是广播给所有节点

```go
// bcastAppend sends RPC, with entries to all peers that are not up-to-date
// according to the progress recorded in r.trk.
func (r *raft) bcastAppend() {
	r.trk.Visit(func(id uint64, _ *tracker.Progress) {
		if id == r.id {
			return
		}
		r.sendAppend(id)
	})
}

// sendAppend sends an append RPC with new entries (if any) and the
// current commit index to the given peer.
func (r *raft) sendAppend(to uint64) {
	r.maybeSendAppend(to, true)
}

func (r *raft) maybeSendAppend(to uint64, sendIfEmpty bool) bool {
	pr := r.trk.Progress[to]
	if pr.IsPaused() {
		return false
	}

	prevIndex := pr.Next - 1
	prevTerm, err := r.raftLog.term(prevIndex)
	if err != nil {
		// The log probably got truncated at >= pr.Next, so we can't catch up the
		// follower log anymore. Send a snapshot instead.
		return r.maybeSendSnapshot(to, pr)
	}

	var ents []pb.Entry
	// In a throttled StateReplicate only send empty MsgApp, to ensure progress.
	// Otherwise, if we had a full Inflights and all inflight messages were in
	// fact dropped, replication to that follower would stall. Instead, an empty
	// MsgApp will eventually reach the follower (heartbeats responses prompt the
	// leader to send an append), allowing it to be acked or rejected, both of
	// which will clear out Inflights.
	if pr.State != tracker.StateReplicate || !pr.Inflights.Full() {
		ents, err = r.raftLog.entries(pr.Next, r.maxMsgSize)
	}
	if len(ents) == 0 && !sendIfEmpty {
		return false
	}
	// TODO(pav-kv): move this check up to where err is returned.
	if err != nil { // send a snapshot if we failed to get the entries
		return r.maybeSendSnapshot(to, pr)
	}

	// Send the actual MsgApp otherwise, and update the progress accordingly.
	r.send(pb.Message{
		To:      to,
		Type:    pb.MsgApp,
		Index:   prevIndex,
		LogTerm: prevTerm,
		Entries: ents,
		Commit:  r.raftLog.committed,
	})
	pr.SentEntries(len(ents), uint64(payloadsSize(ents)))
	pr.SentCommit(r.raftLog.committed)
	return true
}
```

Follower 接收到 Leader 的信息重置计时器并追加 entries

```go
	case pb.MsgApp:
		r.electionElapsed = 0
		r.lead = m.From
		r.handleAppendEntries(m)
	//...	
		
	func (r *raft) handleAppendEntries(m pb.Message) {
	// TODO(pav-kv): construct logSlice up the stack next to receiving the
	// message, and validate it before taking any action (e.g. bumping term).
	a := logSliceFromMsgApp(&m)

	// 如果该 index 已经被提交则直接返回提交的 index msg
	if a.prev.index < r.raftLog.committed {
		r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.committed})
		return
	}
	// 尝试 append entries，如果成功则返回最后的 index msg
	if mlastIndex, ok := r.raftLog.maybeAppend(a, m.Commit); ok {
		r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: mlastIndex})
		return
	}
	r.logger.Debugf("%x [logterm: %d, index: %d] rejected MsgApp [logterm: %d, index: %d] from %x",
		r.id, r.raftLog.zeroTermOnOutOfBounds(r.raftLog.term(m.Index)), m.Index, m.LogTerm, m.Index, m.From)

	// Our log does not match the leader's at index m.Index. Return a hint to the
	// leader - a guess on the maximal (index, term) at which the logs match. Do
	// this by searching through the follower's log for the maximum (index, term)
	// pair with a term <= the MsgApp's LogTerm and an index <= the MsgApp's
	// Index. This can help skip all indexes in the follower's uncommitted tail
	// with terms greater than the MsgApp's LogTerm.
	//
	// See the other caller for findConflictByTerm (in stepLeader) for a much more
	// detailed explanation of this mechanism.

	// NB: m.Index >= raftLog.committed by now (see the early return above), and
	// raftLog.lastIndex() >= raftLog.committed by invariant, so min of the two is
	// also >= raftLog.committed. Hence, the findConflictByTerm argument is within
	// the valid interval, which then will return a valid (index, term) pair with
	// a non-zero term (unless the log is empty). However, it is safe to send a zero
	// LogTerm in this response in any case, so we don't verify it here.
	// 如果没成功则发生了冲突，需要对比 message 和该节点日志，找到最早的未冲突的 index
	hintIndex := min(m.Index, r.raftLog.lastIndex())
	hintIndex, hintTerm := r.raftLog.findConflictByTerm(hintIndex, m.LogTerm)
	r.send(pb.Message{
		To:         m.From,
		Type:       pb.MsgAppResp,
		Index:      m.Index,
		Reject:     true,
		RejectHint: hintIndex,
		LogTerm:    hintTerm,
	})
}
```

- Follower 拒绝追加
  - 当 Follower 拒绝追加请求时，Leader 会根据 `RejectHint` 和 `LogTerm` 来决定如何继续探测（probe）Follower 上的日志状态。
  - `RejectHint` 表示 Follower 接收到拒绝的日志索引，`LogTerm` 是 Follower 在该索引处的日志项的任期。
  - 如果 `LogTerm > 0`，Leader 会执行一种优化，避免线性探测所有日志项，而是跳过一些明显不匹配的日志条目。
  - 如果 Leader 需要减少进度（`MaybeDecrTo`），则进度器的状态会从 `StateReplicate` 转为 `StateProbe`，并再次尝试发送日志追加请求。
- Follower 接收追加
  - 表示 Follower 接受了日志追加请求，Leader 会根据返回的日志索引 (`m.Index`) 更新自己的进度器 `pr`。
  - 如果进度器当前处于 `StateProbe` 状态，并且该响应更新了 `Match` 索引（即日志条目已成功追加），则进度器状态会变为 `StateReplicate`。
  - 如果进度器处于 `StateSnapshot` 状态，并且 `Match` 索引大于等于日志的第一个索引，则说明可以恢复到复制状态，并发送后续的日志追加请求。

Leader 会调用 `r.maybeCommit()`  检测是否可以提交，如果可以提交则提交并释放读索引请求并且广播给所有节点提交

commit 操作只是对 raftLog 状态的修改，并没有涉及到实际数据的持久化。

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

func (l *raftLog) commitTo(tocommit uint64) {
	// never decrease commit
	if l.committed < tocommit {
		if l.lastIndex() < tocommit {
			l.logger.Panicf("tocommit(%d) is out of range [lastIndex(%d)]. Was the raft log corrupted, truncated, or lost?", tocommit, l.lastIndex())
		}
		l.committed = tocommit
	}
}
```

```go
	switch m.Type {
	case pb.MsgAppResp:
		// NB: this code path is also hit from (*raft).advance, where the leader steps
		// an MsgAppResp to acknowledge the appended entries in the last Ready.

		pr.RecentActive = true

		if m.Reject {
			// RejectHint is the suggested next base entry for appending (i.e.
			// we try to append entry RejectHint+1 next), and LogTerm is the
			// term that the follower has at index RejectHint. Older versions
			// of this library did not populate LogTerm for rejections and it
			// is zero for followers with an empty log.
			//
			// Under normal circumstances, the leader's log is longer than the
			// follower's and the follower's log is a prefix of the leader's
			// (i.e. there is no divergent uncommitted suffix of the log on the
			// follower). In that case, the first probe reveals where the
			// follower's log ends (RejectHint=follower's last index) and the
			// subsequent probe succeeds.
			//
			// However, when networks are partitioned or systems overloaded,
			// large divergent log tails can occur. The naive attempt, probing
			// entry by entry in decreasing order, will be the product of the
			// length of the diverging tails and the network round-trip latency,
			// which can easily result in hours of time spent probing and can
			// even cause outright outages. The probes are thus optimized as
			// described below.
			r.logger.Debugf("%x received MsgAppResp(rejected, hint: (index %d, term %d)) from %x for index %d",
				r.id, m.RejectHint, m.LogTerm, m.From, m.Index)
			nextProbeIdx := m.RejectHint
			if m.LogTerm > 0 {
				// If the follower has an uncommitted log tail, we would end up
				// probing one by one until we hit the common prefix.
				//
				// For example, if the leader has:
				//
				//   idx        1 2 3 4 5 6 7 8 9
				//              -----------------
				//   term (L)   1 3 3 3 5 5 5 5 5
				//   term (F)   1 1 1 1 2 2
				//
				// Then, after sending an append anchored at (idx=9,term=5) we
				// would receive a RejectHint of 6 and LogTerm of 2. Without the
				// code below, we would try an append at index 6, which would
				// fail again.
				//
				// However, looking only at what the leader knows about its own
				// log and the rejection hint, it is clear that a probe at index
				// 6, 5, 4, 3, and 2 must fail as well:
				//
				// For all of these indexes, the leader's log term is larger than
				// the rejection's log term. If a probe at one of these indexes
				// succeeded, its log term at that index would match the leader's,
				// i.e. 3 or 5 in this example. But the follower already told the
				// leader that it is still at term 2 at index 6, and since the
				// log term only ever goes up (within a log), this is a contradiction.
				//
				// At index 1, however, the leader can draw no such conclusion,
				// as its term 1 is not larger than the term 2 from the
				// follower's rejection. We thus probe at 1, which will succeed
				// in this example. In general, with this approach we probe at
				// most once per term found in the leader's log.
				//
				// There is a similar mechanism on the follower (implemented in
				// handleAppendEntries via a call to findConflictByTerm) that is
				// useful if the follower has a large divergent uncommitted log
				// tail[1], as in this example:
				//
				//   idx        1 2 3 4 5 6 7 8 9
				//              -----------------
				//   term (L)   1 3 3 3 3 3 3 3 7
				//   term (F)   1 3 3 4 4 5 5 5 6
				//
				// Naively, the leader would probe at idx=9, receive a rejection
				// revealing the log term of 6 at the follower. Since the leader's
				// term at the previous index is already smaller than 6, the leader-
				// side optimization discussed above is ineffective. The leader thus
				// probes at index 8 and, naively, receives a rejection for the same
				// index and log term 5. Again, the leader optimization does not improve
				// over linear probing as term 5 is above the leader's term 3 for that
				// and many preceding indexes; the leader would have to probe linearly
				// until it would finally hit index 3, where the probe would succeed.
				//
				// Instead, we apply a similar optimization on the follower. When the
				// follower receives the probe at index 8 (log term 3), it concludes
				// that all of the leader's log preceding that index has log terms of
				// 3 or below. The largest index in the follower's log with a log term
				// of 3 or below is index 3. The follower will thus return a rejection
				// for index=3, log term=3 instead. The leader's next probe will then
				// succeed at that index.
				//
				// [1]: more precisely, if the log terms in the large uncommitted
				// tail on the follower are larger than the leader's. At first,
				// it may seem unintuitive that a follower could even have such
				// a large tail, but it can happen:
				//
				// 1. Leader appends (but does not commit) entries 2 and 3, crashes.
				//   idx        1 2 3 4 5 6 7 8 9
				//              -----------------
				//   term (L)   1 2 2     [crashes]
				//   term (F)   1
				//   term (F)   1
				//
				// 2. a follower becomes leader and appends entries at term 3.
				//              -----------------
				//   term (x)   1 2 2     [down]
				//   term (F)   1 3 3 3 3
				//   term (F)   1
				//
				// 3. term 3 leader goes down, term 2 leader returns as term 4
				//    leader. It commits the log & entries at term 4.
				//
				//              -----------------
				//   term (L)   1 2 2 2
				//   term (x)   1 3 3 3 3 [down]
				//   term (F)   1
				//              -----------------
				//   term (L)   1 2 2 2 4 4 4
				//   term (F)   1 3 3 3 3 [gets probed]
				//   term (F)   1 2 2 2 4 4 4
				//
				// 4. the leader will now probe the returning follower at index
				//    7, the rejection points it at the end of the follower's log
				//    which is at a higher log term than the actually committed
				//    log.
				nextProbeIdx, _ = r.raftLog.findConflictByTerm(m.RejectHint, m.LogTerm)
			}
			if pr.MaybeDecrTo(m.Index, nextProbeIdx) {
				r.logger.Debugf("%x decreased progress of %x to [%s]", r.id, m.From, pr)
				if pr.State == tracker.StateReplicate {
					pr.BecomeProbe()
				}
				r.sendAppend(m.From)
			}
		} else {
			// We want to update our tracking if the response updates our
			// matched index or if the response can move a probing peer back
			// into StateReplicate (see heartbeat_rep_recovers_from_probing.txt
			// for an example of the latter case).
			// NB: the same does not make sense for StateSnapshot - if `m.Index`
			// equals pr.Match we know we don't m.Index+1 in our log, so moving
			// back to replicating state is not useful; besides pr.PendingSnapshot
			// would prevent it.
			if pr.MaybeUpdate(m.Index) || (pr.Match == m.Index && pr.State == tracker.StateProbe) {
				switch {
				case pr.State == tracker.StateProbe:
					pr.BecomeReplicate()
				case pr.State == tracker.StateSnapshot && pr.Match+1 >= r.raftLog.firstIndex():
					// Note that we don't take into account PendingSnapshot to
					// enter this branch. No matter at which index a snapshot
					// was actually applied, as long as this allows catching up
					// the follower from the log, we will accept it. This gives
					// systems more flexibility in how they implement snapshots;
					// see the comments on PendingSnapshot.
					r.logger.Debugf("%x recovered from needing snapshot, resumed sending replication messages to %x [%s]", r.id, m.From, pr)
					// Transition back to replicating state via probing state
					// (which takes the snapshot into account). If we didn't
					// move to replicating state, that would only happen with
					// the next round of appends (but there may not be a next
					// round for a while, exposing an inconsistent RaftStatus).
					pr.BecomeProbe()
					pr.BecomeReplicate()
				case pr.State == tracker.StateReplicate:
					pr.Inflights.FreeLE(m.Index)
				}

				if r.maybeCommit() {
					// committed index has progressed for the term, so it is safe
					// to respond to pending read index requests
					releasePendingReadIndexMessages(r)
					r.bcastAppend()
				} else if r.id != m.From && pr.CanBumpCommit(r.raftLog.committed) {
					// This node may be missing the latest commit index, so send it.
					// NB: this is not strictly necessary because the periodic heartbeat
					// messages deliver commit indices too. However, a message sent now
					// may arrive earlier than the next heartbeat fires.
					r.sendAppend(m.From)
				}
				// We've updated flow control information above, which may
				// allow us to send multiple (size-limited) in-flight messages
				// at once (such as when transitioning from probe to
				// replicate, or when freeTo() covers multiple messages). If
				// we have more entries to send, send as many messages as we
				// can (without sending empty messages for the commit index)
				if r.id != m.From {
					for r.maybeSendAppend(m.From, false /* sendIfEmpty */) {
					}
				}
				// Transfer leadership is in progress.
				if m.From == r.leadTransferee && pr.Match == r.raftLog.lastIndex() {
					r.logger.Infof("%x sent MsgTimeoutNow to %x after received MsgAppResp", r.id, m.From)
					r.sendTimeoutNow(m.From)
				}
			}
		}
```

Follower 接收到广播信息后，还是调用 `handleAppendEntries` 函数进行日志提交

```go
	if mlastIndex, ok := r.raftLog.maybeAppend(a, m.Commit); ok {
		r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: mlastIndex})
		return
	}
	
// maybeAppend returns (0, false) if the entries cannot be appended. Otherwise,
// it returns (last index of new entries, true).
func (l *raftLog) maybeAppend(a logSlice, committed uint64) (lastnewi uint64, ok bool) {
	if !l.matchTerm(a.prev) {
		return 0, false
	}
	// TODO(pav-kv): propagate logSlice down the stack. It will be used all the
	// way down in unstable, for safety checks, and for useful bookkeeping.

	lastnewi = a.prev.index + uint64(len(a.entries))
	ci := l.findConflict(a.entries)
	switch {
	case ci == 0:
	case ci <= l.committed:
		l.logger.Panicf("entry %d conflict with committed entry [committed(%d)]", ci, l.committed)
	default:
		offset := a.prev.index + 1
		if ci-offset > uint64(len(a.entries)) {
			l.logger.Panicf("index, %d, is out of range [%d]", ci-offset, len(a.entries))
		}
		l.append(a.entries[ci-offset:]...)
	}
	l.commitTo(min(committed, lastnewi))
	return lastnewi, true
}
```

# 总结

核心代码 raft.go 2000行左右的篇幅就让我有些许吃力，这还是在有论文基础上进行的源码阅读，但是硬啃下来的收获还是蛮大的，比只看网上总结的笔记效果要好得多，同时也能发现一些原论文没有的闪光点，可能是在长时间的工程实践和开源社区共同作用下培育出的优质实现。