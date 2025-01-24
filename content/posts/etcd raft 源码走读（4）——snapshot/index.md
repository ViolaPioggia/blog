---
title: "etcd raft 源码走读（4）——snapshot"
date: "2025-01-23"
---
snapshot（快照）顾名思义就是帮助压缩日志，那在 etcd-raft 中又是怎么样实现的呢？Leader 什么时候会主动发送带 snapshot 的日志，Follower 又是怎么处理接收到的 snapshot？在这一章节中会得到解答

# 发送 snapshot

首先来看 Leader 什么时候会主动发送带 snapshot 的日志：

可以看出在 maybeSendAppend 函数中如果日志被截断在 pr.Next，所以我们无法获取 follower 的日志，这时候就需要主动发送 snapshot；另外一种情况是尝试获取将要发送给 follower 的 entries，如果获取失败则使用 snapshot

```go
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

然后来看一下生成 snapshot 的逻辑

```go
// maybeSendSnapshot fetches a snapshot from Storage, and sends it to the given
// node. Returns true iff the snapshot message has been emitted successfully.
func (r *raft) maybeSendSnapshot(to uint64, pr *tracker.Progress) bool {
	if !pr.RecentActive {
		r.logger.Debugf("ignore sending snapshot to %x since it is not recently active", to)
		return false
	}

	// 获取 snapshot
	snapshot, err := r.raftLog.snapshot()
	if err != nil {
		if err == ErrSnapshotTemporarilyUnavailable {
			r.logger.Debugf("%x failed to send snapshot to %x because snapshot is temporarily unavailable", r.id, to)
			return false
		}
		panic(err) // TODO(bdarnell)
	}
	if IsEmptySnap(snapshot) {
		panic("need non-empty snapshot")
	}
	sindex, sterm := snapshot.Metadata.Index, snapshot.Metadata.Term
	r.logger.Debugf("%x [firstindex: %d, commit: %d] sent snapshot[index: %d, term: %d] to %x [%s]",
		r.id, r.raftLog.firstIndex(), r.raftLog.committed, sindex, sterm, to, pr)
	// 状态更新为 snapshot
	pr.BecomeSnapshot(sindex)
	r.logger.Debugf("%x paused sending replication messages to %x [%s]", r.id, to, pr)

	// 发送 snapshot 类型的日志消息
	r.send(pb.Message{To: to, Type: pb.MsgSnap, Snapshot: &snapshot})
	return true
}
```

获取 snapshot 是从 storage 中获取，通常会保存在内存中

```go
type Storage interface {
	//...
	Snapshot() (pb.Snapshot, error)
}

// MemoryStorage implements the Storage interface backed by an
// in-memory array.
type MemoryStorage struct {
	// Protects access to all fields. Most methods of MemoryStorage are
	// run on the raft goroutine, but Append() is run on an application
	// goroutine.
	sync.Mutex

	hardState pb.HardState
	snapshot  pb.Snapshot
	// ents[i] has raft log position i+snapshot.Metadata.Index
	ents []pb.Entry

	callStats inMemStorageCallStats
}

// Snapshot implements the Storage interface.
func (ms *MemoryStorage) Snapshot() (pb.Snapshot, error) {
	ms.Lock()
	defer ms.Unlock()
	ms.callStats.snapshot++
	return ms.snapshot, nil
}
```

而 snapshot 类型也很简单，storage.go 里同样给出了 `ApplySnapshot` 和 `CreateSnapshot` 的调用方法，但这两个也是需要在用户层帮助封装实现的，可能 CreateSnapshot 在 bootstrap 新加入节点的时候会用到然后后续定时对。

```go
type Snapshot struct {
	Data     []byte           `protobuf:"bytes,1,opt,name=data" json:"data,omitempty"`
	Metadata SnapshotMetadata `protobuf:"bytes,2,opt,name=metadata" json:"metadata"`
}

// ApplySnapshot overwrites the contents of this Storage object with
// those of the given snapshot.
func (ms *MemoryStorage) ApplySnapshot(snap pb.Snapshot) error {
	ms.Lock()
	defer ms.Unlock()

	//handle check for old snapshot being applied
	msIndex := ms.snapshot.Metadata.Index
	snapIndex := snap.Metadata.Index
	if msIndex >= snapIndex {
		return ErrSnapOutOfDate
	}

	ms.snapshot = snap
	ms.ents = []pb.Entry{{Term: snap.Metadata.Term, Index: snap.Metadata.Index}}
	return nil
}

// CreateSnapshot makes a snapshot which can be retrieved with Snapshot() and
// can be used to reconstruct the state at that point.
// If any configuration changes have been made since the last compaction,
// the result of the last ApplyConfChange must be passed in.
func (ms *MemoryStorage) CreateSnapshot(i uint64, cs *pb.ConfState, data []byte) (pb.Snapshot, error) {
	ms.Lock()
	defer ms.Unlock()
	if i <= ms.snapshot.Metadata.Index {
		return pb.Snapshot{}, ErrSnapOutOfDate
	}

	offset := ms.ents[0].Index
	if i > ms.lastIndex() {
		getLogger().Panicf("snapshot %d is out of bound lastindex(%d)", i, ms.lastIndex())
	}

	ms.snapshot.Metadata.Index = i
	ms.snapshot.Metadata.Term = ms.ents[i-offset].Term
	if cs != nil {
		ms.snapshot.Metadata.ConfState = *cs
	}
	ms.snapshot.Data = data
	return ms.snapshot, nil
}
```

除了 snapshot 之外还有 compact 帮助合并日志，通过传入 compactIndex 合并在这之前的 entries

```go
// Compact discards all log entries prior to compactIndex.
// It is the application's responsibility to not attempt to compact an index
// greater than raftLog.applied.
func (ms *MemoryStorage) Compact(compactIndex uint64) error {
	ms.Lock()
	defer ms.Unlock()
	offset := ms.ents[0].Index
	if compactIndex <= offset {
		return ErrCompacted
	}
	if compactIndex > ms.lastIndex() {
		getLogger().Panicf("compact %d is out of bound lastindex(%d)", compactIndex, ms.lastIndex())
	}

	i := compactIndex - offset
	// NB: allocate a new slice instead of reusing the old ms.ents. Entries in
	// ms.ents are immutable, and can be referenced from outside MemoryStorage
	// through slices returned by ms.Entries().
	ents := make([]pb.Entry, 1, uint64(len(ms.ents))-i)
	ents[0].Index = ms.ents[i].Index
	ents[0].Term = ms.ents[i].Term
	ents = append(ents, ms.ents[i+1:]...)
	ms.ents = ents
	return nil
}
```

# 接收 snapshot

然后再来看 follower 接收到 snapshot 之后是怎么一个处理逻辑

follower 在接收到 MsgSnap 类型的消息之后调用 r.handleSnapshot

```go
	case pb.MsgSnap:
		r.electionElapsed = 0
		r.lead = m.From
		r.handleSnapshot(m)
```

获取 msg 里的 snapshot 进行处理，尝试使用 restore 从 snapshot 里应用到状态机，如果成功则返回最新的 index，如果失败则返回已提交的 index

```go
func (r *raft) handleSnapshot(m pb.Message) {
	// MsgSnap messages should always carry a non-nil Snapshot, but err on the
	// side of safety and treat a nil Snapshot as a zero-valued Snapshot.
	var s pb.Snapshot
	if m.Snapshot != nil {
		s = *m.Snapshot
	}
	sindex, sterm := s.Metadata.Index, s.Metadata.Term
	if r.restore(s) {
		r.logger.Infof("%x [commit: %d] restored snapshot [index: %d, term: %d]",
			r.id, r.raftLog.committed, sindex, sterm)
		r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.lastIndex()})
	} else {
		r.logger.Infof("%x [commit: %d] ignored snapshot [index: %d, term: %d]",
			r.id, r.raftLog.committed, sindex, sterm)
		r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.committed})
	}
}
```

restore 代码

```go
// restore recovers the state machine from a snapshot. It restores the log and the
// configuration of state machine. If this method returns false, the snapshot was
// ignored, either because it was obsolete or because of an error.
func (r *raft) restore(s pb.Snapshot) bool {
	// 如果比当前节点的 commit 日志 index 小则直接返回 false
	if s.Metadata.Index <= r.raftLog.committed {
		return false
	}
	// 保证当前节点一定是 follower
	if r.state != StateFollower {
		// This is defense-in-depth: if the leader somehow ended up applying a
		// snapshot, it could move into a new term without moving into a
		// follower state. This should never fire, but if it did, we'd have
		// prevented damage by returning early, so log only a loud warning.
		//
		// At the time of writing, the instance is guaranteed to be in follower
		// state when this method is called.
		r.logger.Warningf("%x attempted to restore snapshot as leader; should never happen", r.id)
		r.becomeFollower(r.Term+1, None)
		return false
	}

	// More defense-in-depth: throw away snapshot if recipient is not in the
	// config. This shouldn't ever happen (at the time of writing) but lots of
	// code here and there assumes that r.id is in the progress tracker.
	found := false
	cs := s.Metadata.ConfState

	for _, set := range [][]uint64{
		cs.Voters,
		cs.Learners,
		cs.VotersOutgoing,
		// `LearnersNext` doesn't need to be checked. According to the rules, if a peer in
		// `LearnersNext`, it has to be in `VotersOutgoing`.
	} {
		for _, id := range set {
			if id == r.id {
				found = true
				break
			}
		}
		if found {
			break
		}
	}
	// 如果发送来的 snapshot 没在节点集群里则 warning 并返回 false
	if !found {
		r.logger.Warningf(
			"%x attempted to restore snapshot but it is not in the ConfState %v; should never happen",
			r.id, cs,
		)
		return false
	}

	// Now go ahead and actually restore.
	// 现在进行真正的存储实现
	id := entryID{term: s.Metadata.Term, index: s.Metadata.Index}
	// 判断节点当前任期和 snapshot 任期是否相等
	if r.raftLog.matchTerm(id) {
		// TODO(pav-kv): can print %+v of the id, but it will change the format.
		last := r.raftLog.lastEntryID()
		r.logger.Infof("%x [commit: %d, lastindex: %d, lastterm: %d] fast-forwarded commit to snapshot [index: %d, term: %d]",
			r.id, r.raftLog.committed, last.index, last.term, id.index, id.term)
		// 如果日志的 term 和快照的 term 匹配，意味着当前节点的日志与快照数据一致
		// 或者至少已经过了与快照一致的某个点。此时，可以直接将日志 提交到快照的索引
		// 这样就相当于 跳过了日志的部分复制，提高了效率，也被称为快进提交
		r.raftLog.commitTo(s.Metadata.Index)
		return false
	}

	// 从 snapshot 中恢复到 unstable 的 raftLog 中
	r.raftLog.restore(s)

	// Reset the configuration and add the (potentially updated) peers in anew.
	r.trk = tracker.MakeProgressTracker(r.trk.MaxInflight, r.trk.MaxInflightBytes)
	cfg, trk, err := confchange.Restore(confchange.Changer{
		Tracker:   r.trk,
		LastIndex: r.raftLog.lastIndex(),
	}, cs)

	if err != nil {
		// This should never happen. Either there's a bug in our config change
		// handling or the client corrupted the conf change.
		panic(fmt.Sprintf("unable to restore config %+v: %s", cs, err))
	}

	assertConfStatesEquivalent(r.logger, cs, r.switchToConfig(cfg, trk))

	last := r.raftLog.lastEntryID()
	r.logger.Infof("%x [commit: %d, lastindex: %d, lastterm: %d] restored snapshot [index: %d, term: %d]",
		r.id, r.raftLog.committed, last.index, last.term, id.index, id.term)
	return true
}
```

另外还有一种特殊情况，如果 step 处理到 MsgStorageAppendResp 类型的 msg，即使 msg 的term 小于节点现在的 term，也会去 applySnap，因为 snapshot 里的信息仍然是有效的

```go
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
		}
```