# Raft

## Log Replication

### 要保证的特性

1. 两个日志中，如果某个entry的index和term相同，则这两个entry一定是相同的command。这是因为leader为一个entry分配唯一一个index，而follower会和leader保持一致。
2. 两个日志中，如果某两个entry的index和term相同，则在这两个logs中这个entry前面的entries都是相同的。这是因为每个append entry会有检查机制，将上一个entry的index和term发送给follower，如果follower中没有这个entry，就回拒绝给leader。

### 基本操作（正常情况下）

1. leader收到client的请求后，生成entry，发送给所有followers。
2. 当这个entry在大部分followers中都复制成功后，就可以进行commit（运行），并将运行结果返回给client。
3. 如果某个follower没有复制成功，就一直尝试直到成功（即使这个entry已经返回给client了）。
4. 保证commit的entry最终在所有follower中都被执行，并且index前面的entry也都被提交了。

### Append Entry 检查

每个append entry会有检查机制，将上一个entry的index和term发送给follower，如果follower中没有这个entry，就回拒绝给leader。

Raft中Leader强制让别的follower都和自己一样，所以如果有冲突，就会只听leader的。

Leader为每个follower保存nextIndex，表示下一个要发送给这个follower的entry的index。最一开始每个nextIndex都是leader的log中最后index+1，如果收到拒绝，则将nextIndex-1并重新发送appendentries，重复这个过程直到follower同意。通过这种方式将follower中少的或者与自己不一致的entry全部统一了。

体现了**Leader永远不会修改或者删除自己的log，只会append自己的log。（Leader Append-Only）**