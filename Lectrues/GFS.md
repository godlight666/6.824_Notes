# Google File System

## 分布存储的问题

1. 目标是Performance，可能通过将任务切片（sharding）来完成
2. 多台机器就会出现错误（Faults），就需要建立容错机制（Fault Tolerance）
3. 容错机制一般就需要利用到复制技术（Replication）
4. 复制技术会导致读取到的数据不一致，所以需要保持一致性（consistency）
5. 为了维护一致性又会需要各种机制，就可能会降低性能，影响最初的目标。

### 强一致性

相当于 只有一个机器，一个数据副本，机器按照自己的顺序依次处理各个请求。

但是为了性能的妥协，还有最终一致性，版本一致性。

### 系统设计的问题

1. 当多台机器的时候，两个机器同时接收到了多个请求，能否保证这两个机器按照相同的顺序执行这多个请求？

## GFS的设计目标

1. Big, fast
2. 通用的，为各个应用程序服务
3. 能够将文件分片Sharding，这样能够读的更快，也能够读比单个磁盘更大的文件
4. 能够自动恢复错误。
5. GFS运行在单个机房（副本之间不会异地）
6. 内部使用
7. 为了大数据的顺序访问

## 整体结构

一个master，多个chunkserver。master相当于目录，存储file到chunk handle的映射。

每个文件被分成多个chunk，chunk大小固定，64MB。

## 读

1. Client把filename和（读的起始位置）offset发送给master**。这里只会读到一个chunk中的内容，如果涉及多个chunk，Client会调用GFS的lib，将一个读操作转化为多个，一个读操作只涉及一个chunk，**然后该lib再把各个不同chunk中的数据拼起来。
2. Master根据map，把对应的chunk handle，list of chunk servers（存了该chunk的servers（多个replicas），client可以选一个最近的获取数据）发送给client。**Client会将该chunk的位置缓存起来，**以后还要读chunk中的数据就不用再去问master。
3. Client根据获得的chunk handle直接访问目标chunkserver，获取目标数据。
4. chunkserver把目标数据传递给Client

## 写（Append）

Client会询问Master要写的文件的最后一个chunk的chunk handle，并且获取该chunk 对应的primary。**写操作必须通过primary完成**。

### No Primary情况

1. Master根据本地记录的该chunk最新的version，找到该chunk的所有replicas，如果其中有符合最新version的，就认定为primary，并给定一个固定时间的租约，其他的作为seconds。同时，会让最新的version的内容与seconds中的进行同步。如果没有一个符合，就让client等待。
2. 增加Master上的最新version num
3. 通知primary和seconds最新的version num并写入磁盘
4. Master把最新的version num写入磁盘

### 有primary后

1. primary根据本地情况，找出追加位置的offset（文件最末尾的offset）
2. 所有replicas将新的数据追加写入对应的offset位置
3. 每个replicas完成操作后，回复yes给primary。如果primary收到了**所有replicas**的yes回复，就给client回复success，否则回复failed（client收到fail后会重试，重新尝试append）

### 不一致情况

根据上面的步骤可以看到，步骤2中，可能有部分replicas追加成功，部分replicas追加失败。重试追加后，**只会根据primary的offset追加**，所以追加失败的replica那部分就会空着。（图片可以看noteability）

比如：

1. A1，A2, A3都append b，在offset1位置，但是A3没收到该数据失败了，A1和A2成功了。
2. client收到失败后重试append b，现在primary的offset在位置2，导致A1和A2有两个b，但是A3只在offset2位置有。

这是GFS所容忍的不一致。需要应用程序自己设计来处理这些情况。

### Split Brain解决方式

每一个Primary有一个租期lease，Primary自己知道，Master也知道。Master找不到Primary，**会等待一个租约时间**（60s），然后再重新分配一个primary。这样永远不会存在两个primary。