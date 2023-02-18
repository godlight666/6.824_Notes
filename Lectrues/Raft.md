# Raft

copy-on-write 具体了解。

### Log的作用

通过leader和log确定了唯一的一系列command执行的顺序。单机上可以通过锁来确定，多机上就没这么容易。

Log可以记录下一系列的操作，当crash之后可以利用log恢复。各个机器上也会将自己的log写入磁盘进行持久化，宕机后也可以通过log恢复。

问题：当leader的执行速度远远大于follower时，follower的log（放在内存中，因为还没执行）可能会无限变大，最后超出内存的容量。

### 需要Persist的成员

1. Log
2. currentTerm
3. votedFor

前一个是为了记录下所有操作，**后两个是为了确保在一个term中不为两个server投票而导致脑裂**。而且currentTerm也需要记录下来防止有两个不同的term使用同一个term号，即使用一个已经用过的term号。

## 要解决的问题

1. Partition情况下，发生网络分区，导致split brain。即各个网络分区有自己的leader，导致好几个leader同时存在。
2. 通过Majority Vote来解决这个问题。最多只有一个分区能获得majority的votes，就不会有多个leader了。
3. 总共2f+1个server，可以容忍f个server故障时仍然正常运转。
4. Majority还有一个重要的属性，**就是两次选举中总是有至少一个重合的server**。通过这个特性，raft的许多属性得以实现。比如上一个term的term号，下一个term一定会大于它，不然成不了leader。还比如上一个term提交的log，一定在至少一个重合的server中有，就能保证新的leader的log中也有这个entry。

### 强一致性和序列化

多replicas的情况下，也要和只有一份replica时表现一样。

## Leader Election

要保证一个term中，要不没有leader，要不最多只能有一个leader。

election timer的设置时间问题：1. 超时时间最小要大于两倍的heartbeat，最大取决于系统需要在多长时间内恢复。2. 两个server在1中的区间取随机值从而不同时开启投票，**而且他们之间的间隔最好要大于一次通信往返的时间（用来请求所有server的投票**。3. 一个server的随机election time不要在server初始化时就固定并且不变，因为可能还是会导致投票瓜分？。（**为什么**）

### nextIndex更新的优化的方法

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

## Safety

### 要保证的特性

1. 所有commit的entry，都要出现在每一个leader中。也就是说，要成为一个leader，它的log中就必须有之前所有的commit的entry。通过leader election的限制来实现这个特性。

### Leader Election Restriction

请求投票时，candidate会将自己log的信息包含在rpc中，candidate只有比majority的log都更up-to-date才能成为leader。

**如何判断up-to-date**：requestVote rpc中，包含log最后一个entry的index和term，优先比较term，term大的就更up-to-date。如果term相同，就比较index（即日志的长度）

**为什么这样能保证每个commit的entry都在leader中**：因为每个commit的entry都至少在majority中复制成功了，而新leader又比majority的log更新，这两个majority必定会有重合的（至少一个），所以leader一定有commit的entry。

### Committing Entries from previous term

每个leader只提交自己当前term的entries，不管之前term的。一旦提交了一个，之前的entries也都间接提交了。**为什么？**

## 存在的问题

1. 提供n倍的机器，但是并不能获得n倍的性能，因为多余的机器只是在作为replica，回应client都是通过leader，只用上了leader的性能。
   1. 可能的解决方法：读操作放到replica中进行。**但是不可以**。因为没法确保这个replica相比于leader是否是up-to-date的，就导致数据可能过时。这就没法维持强一致性（线性一致性）（这时候就需要学习分布式事务了，通过一些timestamp或者版本号来标识数据，但是也不能保证最新的）**ZooKeeper解决这个的方式是不提供强一致性**。
2. Zookeeper保证的是：
   1. 写请求的顺序严格按照clients指定的来。即**写请求是可线性化的**。
   2. FIFO Client Order
      1. **读请求读到的值一定至少比上一次读到的值一样新**，或者更新，不过读到更旧的
      2. **对于同一个client的操作请求也是可线性化的**。同一个client的命令执行顺序一定和client指定的一样。比如一个client写了x为5，之后该client再读x，x肯定为5，而不会读到旧值。但是不能保证多个客户端之间的有序性，比如client1先发x为5，然后client2读，不一定读到的是x为5，可能为x的旧值。
