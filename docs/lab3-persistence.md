# MIT 6.5840(2024) Lab 3C (Persistence)

## 1. 前言
在进行Lab 3C之前，一定要检查需要持久化的状态量是否有**被正确地赋值**（比如，有些变量不做赋值操作默认为`0`，但在无操作情况下应当是`-1[votedFor]`）

如果检查多次没有发现问题，但是test就是有概率 **FAILED** 的话——最好还是重写代码

## 2. 任务
Complete the functions persist() and readPersist() in raft.go by adding code to save and restore persistent state. You will need to encode (or "serialize") the state as an array of bytes in order to pass it to the Persister. Use the labgob encoder; see the comments in persist() and readPersist(). labgob is like Go's gob encoder but prints error messages if you try to encode structures with lower-case field names. For now, pass nil as the second argument to persister.Save(). Insert calls to persist() at the points where your implementation changes persistent state. Once you've done this, and if the rest of your implementation is correct, you should pass all of the 3C tests.  

> [!tip]
> <ol><strong>
> <li>
> Run git pull to get the latest lab software.
> </li>
> <li>
> The 3C tests are more demanding than those for 3A or 3B, and failures may be caused by problems in your code for 3A or 3B.
> </li>
> </strong></ol>

## 3. Code-Rewriting
Follower缺少日志时，Lab 3B中的"回退到commitIndex"不严谨，在此处并不适用；同时，单纯返回日志长度也有问题。这也就出现了在restore情况下的非共识日志提交现象。

Lab 3C 给了一个思路

> [!tip]
> You will probably need the optimization that backs up nextIndex by more than one entry at a time. Look at the <a href="https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf">extended Raft paper</a> starting at the bottom of page 7 and top of page 8 (marked by a gray line). The paper is vague about the details; you will need to fill in the gaps. One possibility is to have a rejection message include:
>
>   <ul><strong>
>   <li>
>   XTerm: term in the conflicting entry (if any)
>   </li>
>   <li>
>   XIndex: index of first entry with that term (if any)
>   </li>
>   <li>
>   XLen: log length
>   </li></strong>
>  </ul>
>   Then the leader's logic can be something like:
> <ul><strong>
>  <li>
>  Case 1: leader doesn't have XTerm:
>    <p style="text-indent: 1rem;">nextIndex = XIndex</p>
>  </li>
>  <li>
>  Case 2: leader has XTerm:
>    <p style="text-indent: 1rem;">nextIndex = leader's last entry for XTerm</p>
>  </li>
>  <li>
>  Case 3: follower's log is too short:
>    <p style="text-indent: 1rem;">nextIndex = XLen</p>
>  </li></strong>
>  </ul>

至少我们知道该怎么改了，重写的时候，最好多看Paper（注释已经写好了，就不再说明）

### 3.1 AppendEntriesRPC

``` go
// AppendEntries RPC
// Arguments:
type AppendEntriesArgs struct {
    Term         int // leader’s term
    LeaderId     int // so follower can redirect clients
    LeaderCommit int // leader’s commitIndex

    Log []Entry // to store (empty for heartbeat; may send more than one for efficiency)

    PrevLogIndex int // index of log entry immediately preceding new ones
    PrevLogTerm  int // term of prevLogIndex entry
}

// Results:
type AppendEntriesReply struct {
    Term    int  // currentTerm, for leader to update itself
    Success bool // true if follower contained entry matching prevLogIndex and prevLogTerm

    XTerm  int // term in the conflicting entry (if any)
    XIndex int // index of first entry with that term (if any)
    XLen   int // log length
    // what should three 'X' do ?
    // Case 1: leader doesn't have XTerm:
    //  nextIndex = XIndex
    // Case 2: leader has XTerm:
    //  nextIndex = leader's last entry for XTerm
    // Case 3: follower's log is too short:
    //  nextIndex = XLen
}
```

### 3.2 AppendEntries

``` go
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
    rf.mu.Lock()
    defer rf.mu.Unlock()
    defer DRPCPrint(rf, reply)

    // Receiver implementation:

    // Reply false if term < currentTerm (§5.1)
    if args.Term < rf.currentTerm {
        reply.Term = rf.currentTerm
        reply.Success = false
        return
    }

    // 确认是当前Leader发的,推迟选举
    rf.delayElection()

    // 任期不一致,立即同步
    // If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)
    if args.Term > rf.currentTerm {
        rf.ConvertToFollower(args.Term)
        rf.persist()
        // 不要马上return
    }

    // Lab 3C: XTerm, XIndex, XLen
    conflict := false
    // Log Matching: if two logs contain an entry with the same index and term, then the logs are identical in all entries
    // up through the given index. §5.3
    // term与index相同的log不冲突(!conflict)
    if args.PrevLogIndex >= len(rf.log) {
        // Append any new entries not already in the log

        reply.XTerm = -1
        // XLen: log length
        reply.XLen = len(rf.log)
        conflict = true
    } else if rf.log[args.PrevLogIndex].Term != args.PrevLogTerm {
        // Reply false if log doesn’t contain an entry at prevLogIndex whose term matches prevLogTerm (§5.3)

        // XTerm: term in the conflicting entry (if any)
        reply.XTerm = rf.log[args.PrevLogIndex].Term

        // XIndex: index of first entry with that term (if any)
        rollbackPoint := args.PrevLogIndex
        for rf.log[rollbackPoint].Term == reply.XTerm {
            rollbackPoint--
        }
        reply.XIndex = rollbackPoint + 1
        conflict = true
    }

    // 出现日志冲突
    if conflict {
        reply.Term = rf.currentTerm
        reply.Success = false
        DXConflictPrint(rf, args, reply)
        return
    }

    // PrevLog匹配,则同步日志
    // Leader的任何日志都是"可信任的"
    // 所以认为PrevLogIndex之后的日志均为冲突日志也是正确的
    // 无条件从PrevLogIndex开始替换
    if len(args.Log) != 0 && len(rf.log) > args.PrevLogIndex+1 {
        // If an existing entry conflicts with a new one (same index but different terms),
        // delete the existing entry and all that follow it (§5.3)
        rf.log = rf.log[:args.PrevLogIndex+1]
    }
    // Append any new entries not already in the log
    rf.log = append(rf.log, args.Log...)
    rf.persist()

    reply.Success = true
    reply.Term = rf.currentTerm

    // 存在可提交日志
    if args.LeaderCommit > rf.commitIndex {
        // If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry)
        rf.commitIndex = min(len(rf.log)-1, args.LeaderCommit)
        rf.applyCond.Signal()
    }
}
```

### 3.3 postAppendEntries/sendAppendEntries

``` go
func (rf *Raft) sendAppendEntries(id int, args *AppendEntriesArgs, reply *AppendEntriesReply) bool {
    defer DRPCPrint(rf, args)
    return rf.peers[id].Call("Raft.AppendEntries", args, reply)
}


func (rf *Raft) postAppendEntries(id int, args *AppendEntriesArgs) {
    reply := &AppendEntriesReply{}
    if ok := rf.sendAppendEntries(id, args, reply); !ok {
        return
    }

    rf.mu.Lock()
    defer rf.mu.Unlock()

    if args.Term != rf.currentTerm {
        return
    }

    // If successful: update nextIndex and matchIndex for follower (§5.3)
    if reply.Success {
        rf.matchIndex[id] = args.PrevLogIndex + len(args.Log)
        rf.nextIndex[id] = rf.matchIndex[id] + 1

        // If there exists an N such that N > commitIndex, a majority of matchIndex[i] ≥ N, and log[N].term == currentTerm:
        // set commitIndex = N (§5.3, §5.4).
        rf.commitIndex = rf.seekSynchronizedIndex()
        rf.applyCond.Signal()
        return
    }

    // !reply.Success
    // If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)
    if reply.Term > rf.currentTerm {
        rf.ConvertToFollower(reply.Term)
        rf.delayElection()
        rf.persist()
        return
    }

    if reply.Term == rf.currentTerm && rf.role == Leader {
        // If AppendEntries fails because of log inconsistency: decrement nextIndex and retry (§5.3)

        DXConflictPrint(rf, args, reply)

        // 这里我们用Lab 3C提到的三个'X'
        // Case 3: follower's log is too short:
        //  nextIndex = XLen
        if reply.XTerm == -1 {
            rf.nextIndex[id] = reply.XLen
            return
        }

        // try to find XTerm
        rollbackPoint := rf.nextIndex[id] - 1
        for rollbackPoint > 0 && rf.log[rollbackPoint].Term > reply.XTerm {
            rollbackPoint--
        }

        if rf.log[rollbackPoint].Term != reply.XTerm {
            // Case 1: leader doesn't have XTerm:
            //  nextIndex = XIndex
            rf.nextIndex[id] = reply.XIndex
        } else {
            // Case 2: leader has XTerm:
            //  nextIndex = leader's last entry for XTerm
            rf.nextIndex[id] = rollbackPoint + 1
        }
    }
}

// 定向到match过半数的最近index
func (rf *Raft) seekSynchronizedIndex() int {
    synchronizedIndex := len(rf.log) - 1
    for synchronizedIndex > rf.commitIndex {
        synchronizedPeers := 1
        for i := range rf.peers {
            if i == rf.me {
                continue
            }
            if rf.matchIndex[i] >= synchronizedIndex && rf.log[synchronizedIndex].Term == rf.currentTerm {
                synchronizedPeers++
            }
        }
        if synchronizedPeers > len(rf.peers)/2 {
            break
        }
        synchronizedIndex--
    }
    return synchronizedIndex
}
```

### 3.4 applyCh
对于日志的提交，我们可以独立一个协程维护（Leader、Follower、Candidate的提交逻辑一致）。所以对于提交日志来说，AppendEntries相关的处理只需要改变commitIndex。

``` go
type Raft struct {
    
    ...

    applyCh   chan ApplyMsg // commit log entries (the log you're sure to apply in the state machine)
    applyCond *sync.Cond    // wake up goroutine to commit log entries
    // applyCh单独一个协程
    // 这样AppendEntries相关部分只需要处理commitIndex即可
}

func Make(peers []*labrpc.ClientEnd, me int, persister *Persister, applyCh chan ApplyMsg) *Raft {
    ...
    rf.applyCh = applyCh
    rf.applyCond = sync.NewCond(&rf.mu)
    ...
    go rf.commit()

    return rf
}


func (rf *Raft) commit() {
    for !rf.killed() {
        rf.mu.Lock()
        for {
            if rf.commitIndex > rf.lastApplied {
                break
            }
            rf.applyCond.Wait()
            // DPrintf("Commiter Wake Up: {id: %v,rf.commitIndex: %v,rf.lastApplied: %v}", rf.me, rf.commitIndex, rf.lastApplied)
        }
        /*
            sync.Cond.Wait():
                c.checker.check()
                t := runtime_notifyListAdd(&c.notify)
                c.L.Unlock()
                runtime_notifyListWait(&c.notify, t)
                c.L.Lock()
            因此会自动释放锁，并等待信号
        */

        // 如果有需要提交的日志
        // If commitIndex > lastApplied: increment lastApplied, apply log[lastApplied] to state machine (§5.3)

        // 注意,可提交的日志必须是"consense".
        // Leader Completeness: if a log entry is committed in a given term, then that entry will be present in the logs
        // of the leaders for all higher-numbered terms. §5.4

        // State Machine Safety: if a server has applied a log entry at a given index to its state machine, no other server
        // will ever apply a different log entry for the same index. §5.4.3
        for rf.commitIndex > rf.lastApplied {
            rf.lastApplied++
            // Leader: If command received from client, append entry to local log, respond after entry applied to state machine (§5.3)
            // Follower: entry applied to state machine when leader allowing
            rf.applyCh <- ApplyMsg{
                CommandValid: true,
                Command:      rf.log[rf.lastApplied].Command,
                CommandIndex: rf.lastApplied,
            }
        }
        DApplierPrint(rf, rf.log[rf.commitIndex-1])
        rf.mu.Unlock()
    }
}
```

### 3.5 election
可以说没什么变动。但我还是调整了一下election的代码，使其更符合实验注释ticker()。

``` go
func (rf *Raft) ticker() {
    for !rf.killed() {
        // Your code here (3A)
        // Check if a leader election should be started.

        <-rf.electionTicker.C
        // If election timeout elapses: start new election
        rf.mu.Lock()

        // 说明Leader没有在下一次选举之前发送任何心跳表示存活,此时必须开始选举
        if rf.role != Leader {
            // if election timeout elapses without receiving AppendEntries
            // RPC from current leader or granting vote to candidate: convert to candidate
            go rf.election()
        }
        // On conversion to candidate, Reset election timer
        rf.delayElection()
        rf.mu.Unlock()

        // pause for a random amount of time between 50 and 350
        // milliseconds.
        // time.Sleep(time.Duration(50+rand.Int63n()%300) * time.Millsecond)
    }
}

func (rf *Raft) election() {
    rf.mu.Lock()
    defer rf.mu.Unlock()

    rf.ConvertToCandidate()

    args := &RequestVoteArgs{
        Term:         rf.currentTerm,
        CandidateId:  rf.me,
        LastLogIndex: len(rf.log) - 1,
        LastLogTerm:  rf.log[len(rf.log)-1].Term,
    }

    // On conversion to candidate, Send RequestVote RPCs to all other servers
    for i := range rf.peers {
        if i == rf.me {
            continue
        }
        go rf.checkVote(i, args)
    }
}

func (rf *Raft) checkVote(id int, args *RequestVoteArgs) {
    if accept := rf.askForVote(id, args); !accept {
        return
    }

    // 访问、修改voteCount应该加锁(只用rf.mu也可以)
    rf.voteMutex.Lock()
    defer rf.voteMutex.Unlock()

    // 已经选上就直接return
    // 不要访问rf.role,因为不是用rf.mu加锁
    if rf.voteCount > len(rf.peers)/2 {
        return
    }

    rf.voteCount++

    // Election Safety: at most one leader can be elected in a given term. §5.2
    if rf.voteCount > len(rf.peers)/2 {
        rf.mu.Lock()
        defer rf.mu.Unlock()
        if rf.role == Follower {
            return
        }

        // If votes received from majority of servers: become leader
        rf.ConvertToLeader()
        for i := range rf.nextIndex {
            rf.nextIndex[i] = len(rf.log)
            rf.matchIndex[i] = 0
        }
        go rf.heartbeat()
    }
}

func (rf *Raft) askForVote(id int, args *RequestVoteArgs) bool {
    reply := &RequestVoteReply{}
    if ok := rf.sendRequestVote(id, args, reply); !ok {
        return false
    }

    rf.mu.Lock()
    defer rf.mu.Unlock()

    // 存在将一个RPC请求延迟到下一个任期的情况(network lag),尤其是Lab 3C
    if args.Term != rf.currentTerm {
        return false
    }

    // If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)
    if reply.Term > rf.currentTerm {
        rf.ConvertToFollower(reply.Term)
        rf.persist()
    }

    return reply.VoteGranted
}
```

### 3.6 RequestVote

``` go
// example RequestVote RPC handler.
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
    // Your code here (3A, 3B).
    rf.mu.Lock()
    defer rf.mu.Unlock()
    defer DRPCPrint(rf, reply)

    // Reply false if term < currentTerm (§5.1)
    if args.Term < rf.currentTerm {
        reply.Term = rf.currentTerm
        reply.VoteGranted = false
        return
    }

    // 接下来会投票
    // If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)
    if args.Term > rf.currentTerm {
        rf.ConvertToFollower(args.Term)
        rf.persist()
    }

    // 有票的情况:           没投票或已投给此RPC候选人
    // 候选人可信任的情况:    日志一样或更"新"
    // If votedFor is null or candidateId (haveVoteTicket),
    // and candidate’s log is at least as up-to-date as receiver’s log(isCandidateLogReliable),
    // grant vote (§5.2, §5.4)
    if rf.haveVoteTicket(args) && rf.isCandidateLogReliable(args) {
        rf.ConvertToFollower(args.Term).VoteTo(args.CandidateId)
        rf.delayElection()
        rf.persist()

        reply.VoteGranted = true
        reply.Term = rf.currentTerm
        return
    }

    // 没投票
    reply.Term = rf.currentTerm
    reply.VoteGranted = false
}
```

### 3.7 Heartbeat

``` go
func (rf *Raft) heartbeat() {
    for !rf.killed() {
        rf.mu.Lock()
        // defer rf.mu.Unlock()
        if rf.role != Leader {
            rf.mu.Unlock()
            return
        }

        // Leader Append-Only: a leader never overwrites or deletes entries in its log; it only appends new entries. §5.3
        // 所以,Leader只管发信息和append(append在Start中实现)
        for i := range rf.peers {
            if i == rf.me {
                continue
            }
            args := &AppendEntriesArgs{
                Term:         rf.currentTerm,
                LeaderId:     rf.me,
                LeaderCommit: rf.commitIndex,
                PrevLogIndex: rf.nextIndex[i] - 1,
            }

            // If last log index ≥ nextIndex for a follower: send AppendEntries RPC with log entries starting at nextIndex
            if len(rf.log)-1 > args.PrevLogIndex {
                args.Log = rf.log[args.PrevLogIndex+1:]
            } else {
                args.Log = make([]Entry, 0)
            }
            args.PrevLogTerm = rf.log[args.PrevLogIndex].Term

            // if len(args.Log) == 0
            // Upon election: send initial empty AppendEntries RPCs
            // (heartbeat) to each server; repeat during idle periods to
            // prevent election timeouts (§5.2)

            go rf.postAppendEntries(i, args)
        }

        rf.mu.Unlock()
        time.Sleep(time.Duration(HeartbeatInterval) * time.Millisecond)
    }
}
```

### 3.8 Note
<s>瞪着屏幕看之前的代码还真不如重写！</s>

#### 3.8.1 rf.log[len(rf.log)-1]
你肯定会认为这段代码会报个数组越界，因为空数组的情况下会访问rf.log[-1]。但如果在初始化的时候这么写就没事，而且还能简化代码：

``` go
rf.log = []Entry{
    {
        Term:    0,
        Index:   -1,
        Command: "孩子们，这不好笑",
    },
}
```

#### 3.8.2 Three 'X' 
AppendEntries中
- 如果Follower发现日志不匹配，只需要传回三个参数
  - XTerm：冲突日志的任期
  - XIndex：任期同冲突日志，但索引最小的日志
  - XLen：日志长度
- 如果Leader发现结果为日志不匹配，会出现三种情形
  - Leader没有保存与不匹配日志任期相同的日志：Follower需删除该任期的全部日志、追加Leader在该索引后的全部日志
  - Leader有保存与不匹配日志任期相同的日志：Follower只需追加Leader在该任期及其之后的日志
  - Follower日志过短：Follower需迅速追加Leader在该索引后的全部日志

以这个逻辑而言
  1. If args.PrevLogIndex >= len(Follower.log) [conflict]
     1. 如果Follower的最后一个日志与Leader对应索引的日志不匹配
        - Follower直接返回XLen，双方不能确定该日志是否匹配
        - 双方显然会发现最后一个日志不匹配，此时为条件b.
     2. 如果Follower的最后一个日志与Leader对应索引的日志匹配
        - Follower直接返回XLen，双方不能确定该日志是否匹配
        - Follower显然会发现最后一个日志匹配，追加日志，此时已同步
  2. !a. && Follower.log[args.PrevLogIndex].Term != args.PrevLogTerm [conflict]
     1. Follower
        1. 返回与冲突日志任期相同的最早日志索引
     2. Leader
        1. 检查是否存在该任期日志
           - 存在，返回该任期及其之后的全部日志
           - 不存在，返回该索引及其之后的全部日志

不可能出现有多个 不同任期 的 日志冲突的情况：
  1. (Assumption)假设Follower存在两个 不同任期 的日志A、B（Term<sub>A</sub><Term<sub>B</sub>），且均与Leader对应索引的日志冲突
  2. (Condition)只有集群所共识的日志才能保存下来，简单来说，是不会在其他任期被删除
  3. (Condition)Leader必然有前任Leader中所保存的日志
  4. (Condition)Leader想要追加日志，Follower必须检查是否有冲突日志
  5. (Condition)不冲突的日志必然是集群所共识的（由Leader保证）
  6. (Condition)Follower只有确认无冲突，才能追加日志
  7. 这说明，A在其之后的日志追加时，未发现冲突
  8. 这说明，A在某任期下的集群所共识
  9. 这说明，A是某个前任Leader保存的日志
  10. 这说明，Leader也保存着这份日志，且索引、任期一致
  11. 这说明，A与Leader对应索引的日志不冲突，与假设矛盾  

简单来说，在这种条件下，Leader负责当前任期日志的更新以及在当前任期之前的日志的拷贝（日志只分为当前任期和非当前任期）；Follower出现日志任期不一致的情况会尽可能少，且Follower中Leader不存在的日志会被一次性替换。

#### 3.8.3 Network lagging?
实验里有个小坑，尽管RPC请求是当前任期发出去的，但并不代表一定会在当前任期接收到响应。

也就是说，如果Leader的请求在T任期内发出，Follower也在T任期内处理并返回，而Follower的响应废了好长时间才到T任期的Leader手上，而此时已经是U任期了。很明显，我们应该不处理这种响应，这也是在使用Call()时必须注意的。

``` go
 func (rf *Raft) ...(args *..., reply *...) {
     if ok := rf.peers[...].Call(xxx, args, reply); !ok { // Call需要时间
         return
     }
     
     if args.Term != rf.currentTerm { // 必须检查
        return
     }
 }
```

## 4. 分析&实现
### 4.1 persist
persist只需针对论文Figure 2提到的需要持久化的三个信息即可（term，log，votedFor）

``` go
// save Raft's persistent state to stable storage,
// where it can later be retrieved after a crash and restart.
// see paper's Figure 2 for a description of what should be persistent.
// before you've implemented snapshots, you should pass nil as the
// second argument to persister.Save().
// after you've implemented snapshots, pass the current snapshot
// (or nil if there's not yet a snapshot).
func (rf *Raft) persist() {
    // Your code here (3C).
    // Example:
    // w := new(bytes.Buffer)
    // e := labgob.NewEncoder(w)
    // e.Encode(rf.xxx)
    // e.Encode(rf.yyy)
    // raftstate := w.Bytes()
    // rf.persister.Save(raftstate, nil)

    w := new(bytes.Buffer)
    e := labgob.NewEncoder(w)
    e.Encode(rf.votedFor)
    e.Encode(rf.currentTerm)
    e.Encode(rf.log)
    raftstate := w.Bytes()
    rf.persister.Save(raftstate, nil)
}
```

### 4.2 readPersist
对着example抄就行了（临时变量最好不要用海象表达式）

``` go
// restore previously persisted state.
func (rf *Raft) readPersist(data []byte) {
    if data == nil || len(data) < 1 { // bootstrap without any state?
        return
    }
    // Your code here (3C).
    // Example:
    // r := bytes.NewBuffer(data)
    // d := labgob.NewDecoder(r)
    // var xxx
    // var yyy
    // if d.Decode(&xxx) != nil ||
    //    d.Decode(&yyy) != nil {
    //   error...
    // } else {
    //   rf.xxx = xxx
    //   rf.yyy = yyy
    // }    
    r := bytes.NewBuffer(data)
    d := labgob.NewDecoder(r)

    var votedFor int
    var currentTerm int
    var log []Entry
    if d.Decode(&votedFor) != nil ||
        d.Decode(&currentTerm) != nil ||
        d.Decode(&log) != nil {
        DPrintf("unexpect error: server[%v] read persist failed.", rf.me)
    } else {
        rf.votedFor = votedFor
        rf.currentTerm = currentTerm
        rf.log = log
    }
}
```

### 4.3 How to use persist()?
当然是三个状态改变的时候放个persist()，failed的主要问题大多是Lab 3A/3B的细微bug。

## 5. Debug Func
相比于之前较为简陋的输出，此次实验需要更为细致的信息，否则容易卡一整天。不过还是重写了。

``` go
package raft

import (
    "crypto/md5"
    "encoding/hex"
    "fmt"
    "log"
    "math/rand"
)

// Debugging
const Debug = false
const IgnoreDLogDetails = true
const IgnoreRPC = true
const RandomIgnoreRPC = true
const ShowApplier = false
const ShowConflictX = true

func DPrintf(format string, a ...interface{}) {
    if Debug {
        log.Printf(format, a...)
    }
}

type DLogFMT struct {
    term    int
    payload string
}

type DLogSummary struct {
    length  int
    payload string
}

func DLogSprint(rf *Raft) (result string) {
    if Debug {
        DLogs := []DLogFMT{}
        LogSummary := DLogSummary{}
        if rf.mu.TryLock() {
            defer rf.mu.Unlock()
        }
        for _, log := range rf.log {
            dlog := DLogFMT{term: log.Term}
            if IgnoreDLogDetails {
                dlog.payload = fmt.Sprint(log.Command)
            } else {
                dlog.payload = fmt.Sprintf("%c", fmt.Sprint(log.Command)[0])
            }
            DLogs = append(DLogs, dlog)
        }
        if IgnoreDLogDetails {
            logDetails := fmt.Sprint(DLogs)
            hash := md5.Sum([]byte(logDetails))

            LogSummary.length = len(rf.log)
            LogSummary.payload = hex.EncodeToString(hash[:])

        } else {
            LogSummary.length = len(rf.log)
            LogSummary.payload = fmt.Sprint(DLogs)
        }
        result = fmt.Sprintf("%#v", LogSummary)
    }
    return
}

type DServerFMT struct {
    id         int
    term        int
    votedFor    int
    commitIndex int
    lastApplied int
    voteCount   int
    role        string
}

func DServerSprint(rf *Raft) (result string) {
    if Debug {
        if rf.mu.TryLock() {
            defer rf.mu.Unlock()
        }
        dserver := DServerFMT{
            id:         rf.me,
            term:        rf.currentTerm,
            votedFor:    rf.votedFor,
            commitIndex: rf.commitIndex,
            lastApplied: rf.lastApplied,
            voteCount:   rf.voteCount,
        }

        dserver.role = getRoleString(rf)

        result = fmt.Sprintf("%#v", dserver)
    }
    return
}

func DServerPrint(rf *Raft) {
    debugID := rand.Int63() % 1000
    DPrintf("DEBUG[%03d]: %v", debugID, DServerSprint(rf))
    DPrintf("DEBUG[%03d]: %v", debugID, DLogSprint(rf))
}

type DApplier struct {
    id          int
    index       int
    term        int
    commitIndex int
    lastApplied int
}

func DApplierPrint(rf *Raft, lastApplyLog Entry) {
    if !Debug {
        return
    }

    if !ShowApplier {
        return
    }

    if rf.mu.TryLock() {
        defer rf.mu.Unlock()
    }

    dapplier := DApplier{
        id:          rf.me,
        index:       lastApplyLog.Index,
        term:        lastApplyLog.Term,
        commitIndex: rf.commitIndex,
        lastApplied: rf.lastApplied,
    }
    debugID := rand.Int63() % 1000
    DPrintf("DEBUG[%03d]: %#v", debugID, dapplier)
}

type DRPC struct {
    id      int
    details string
}

func DRPCPrint(rf *Raft, rpc interface{}) {
    if !Debug {
        return
    }

    if IgnoreRPC {
        return
    }

    if RandomIgnoreRPC {
        if rand.Int31()&1 == 0 {
            return
        }
    }

    if rf.mu.TryLock() {
        defer rf.mu.Unlock()
    }

    drpc := DRPC{
        id:      rf.me,
        details: fmt.Sprintf("%#v", rpc),
    }
    debugID := rand.Int63() % 1000
    DPrintf("DEBUG[%03d]: %#v", debugID, drpc)
}

// lab 3C
type DXConflict struct {
    id   int
    role string

    XTerm  int
    XIndex int
    XLen   int

    PrevLogIndex int
    PrevLogTerm  int

    ConflictLogMD5 string
}

func DXConflictPrint(rf *Raft, args *AppendEntriesArgs, reply *AppendEntriesReply) {
    if Debug {
        if !ShowConflictX {
            return
        }

        if rf.mu.TryLock() {
            rf.mu.Unlock()
        }

        logDetails := fmt.Sprint(rf.log[reply.XIndex].Command)
        hash := md5.Sum([]byte(logDetails))
        logSnapshot := hex.EncodeToString(hash[:])
        debugID := rand.Int63() % 1000

        dXConflict := DXConflict{
            id:     rf.me,
            XTerm:  reply.Term,
            XIndex: reply.XIndex,
            XLen:   reply.XLen,

            PrevLogIndex: args.PrevLogIndex,
            PrevLogTerm:  args.PrevLogTerm,

            ConflictLogMD5: logSnapshot,
        }

        dXConflict.role = getRoleString(rf)

        DPrintf("DEBUG[%03d]: %#v", debugID, dXConflict)
    }
}

// 默认已上锁
func getRoleString(rf *Raft) string {
    switch rf.role {
    case Leader:
        return "Leader"
    case Candidate:
        return "Candidate"
    case Follower:
        return "Follower"
    default:
        return "Null"
    }
}
```

## 6. 测试结果
<s>一轮测试基本上需要两分钟以上，越是难以看见的小bug越是容易测上一天</s>

选举周期: [300 ms, 400 ms)

心跳间隔: 100ms

Lab 3A/3B同样通过测试

### 6.1 Debug = false
测试指令：

``` bash
for i in {0..9}; do go test -run 3C; done > ${file-path}
```

50次测试，50次pass：

[3C.test.log](./pictures/lab3-persistence/3C.test.log)

[3C0.test.log](./pictures/lab3-persistence/3C0.test.log)

[3C1.test.log](./pictures/lab3-persistence/3C1.test.log)

[3C2.test.log](./pictures/lab3-persistence/3C2.test.log)

[3C3.test.log](./pictures/lab3-persistence/3C3.test.log)


### 6.2 Debug = true

Server、Log Entries、Conflict[XTerm、XIndex、XLen]

[debug.test.log](./pictures/lab3-persistence/debug.test.log)