# Lab2 Raft
Raft的实现
## 注意点
1. 当收到requestVote并且同意投票给该请求者后，要将本地的election_time_out重置。