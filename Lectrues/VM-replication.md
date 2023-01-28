# Primary-Backup Replication in VMware

## Replication能解决的问题

1. 能解决fail-stop fault，就是网络故障、断电，停机等问题。
2. 不能解决软件硬件本身的bug，比如计算错误等。
3. 当多个replicas同时出错，比如在同一个机房但是发生地震了，也没有办法。

## Replication方案

### State Transfer

往replica中发送state，比如当前内存中的数据，备份整个机器当前的状态。

发送的是内存中的数据。

### Replicated State Machine

不发送state，只发送外部事件。因为只有外部事件会使内部状态发生改变。

就是将所有发送的外部事件从primary发送到backup。即两台机器从相同的状态开始，按相同的顺序执行相同的外部事件并得到相同的输入，最后的状态也是一致的。

发送的只是外部输入或者外部事件的操作。

更复杂会面对更多的问题，但是传输的数据更小得多。

文章中，这种方式只用在单核机器上。对于多核机器，VMware用的是state transfer。因为多核会导致一串相同的指令，不同机器的执行顺序会不一样，导致执行后的状态不一致。

## Replicated State Machine要解决的问题

1. What state？就是有哪些state要发送

   不同的方案会有不同的要求。一般的解决方案只会复制应用层的数据。但是VMware会直接复制底层内存中的数据和寄存器中的数据。这种方式就让复制独立于应用程序，完全底层的复制，不管这个应用程序是什么都能复制。但是这种复制的损耗比较大。

2. Primary和Backup的同步要多紧密

3. Cut-Over

4. primary fail后，怎么自动切换到backup

5. 新建一个replica，需要用state transfer

### 流程

client给Primary发送数据包，primary的VMM（虚拟机监视器，在硬件和操作系统之间）收到后，会发送给上层的操作系统，**同时生成一个该数据的副本，发送给backup的VMM**，backup收到了一样的输入，就生成了一样的状态和输出。primary生成的输出通过VMM到网络中发送给client，backup同样也会生成输出，**但是backup的VMM检测到后，知道这个是backup，就会将包丢弃**。只有primary才能回应client。

### 不确定事件及解决方式

1. input是通过网络方式，网络方式中input有data也有中断（就是操作系统的中断）。如果中断的时机不一致，可能会导致结果不一样
2. 一些随机的指令，比如生成随机数，获取当前时间
3. 多核并行。同一系列指令运行在多核上，由于并行，会导致运行的顺序发生变化导致结果不一致。（这种方式这篇文章直接屏蔽了，没有解决）

解决方法：primary会给backup发送log entry，这其中包括instruction num（该指令是第几个指令，这就唯一确定了顺序），type（指令的类型，是网络还是随机指令），data（输入数据或者随机指令产生的，比如随机数）。通过这个entry就能让backup完全是primary的复制。

### 保证backup的滞后性

为了防止backup走在primary前面导致不可控，backup的VMM中有一个instruction buffer，这个buffer用来接收log entry，如果buffer中至少有一个entry backup才会继续执行，这样就能保证backup总是比primary慢。

### Output Rule

为了防止primary fail后，发送的log entry同时被网络中丢包了，导致backup没有形成最新的状态。

规定primary的VMM只有在确认backup收到了该log entry（backup收到会给primary发ACK）后才将输出传输给client。

### Split Brain问题

有一个第三方的服务器test-and-set server，这个server的内存中有一个flag（相当于一个锁），server1和server2要成为primary都需要向这个server获取这个flag，如果是1才能变成primary。

**但是当这个server宕机了，这个系统就崩溃了**。单点故障。