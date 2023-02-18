# Lab2 Raft

Raft的实现

## 注意点

1. 当收到requestVote并且同意投票给该请求者后，要将本地的election_time_out重置。
2. 当leader接受到command时，**并不是立刻把change发送给followers，而是等到下一个heartbeat再在heartbeat中附加信息使follower同步**。
3. append entries 中，reply的term用的是最一开始的term， 没有改动过的，不知道要不要修改。
