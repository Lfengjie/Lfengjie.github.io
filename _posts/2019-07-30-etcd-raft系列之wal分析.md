最近在优化gift底层存储，选用etcd raft来作为一致性协议raft的实现库,需要了解其内部的运行原理。

# etcd raft介绍
etcd raft是目前使用最广泛的raft库，如果想深入了解raft请直接阅读论文 "In Search of an Understandable Consensus Algorithm"(https://raft.github.io/raft.pdf), etcd raft在etcd, Kubernetes, Docker Swarm, Cloud Foundry Diego, CockroachDB, TiDB, Project Calico, Flannel等分布式系统中都有应用。大部分的raft的实现都是单体设计（实现了存储层、消息序列化、网络层等），而etcd raft只实现了最核心的raft算法，网络、存储都需要用户自己实现，etcd-wal是etcd实现的一种存储层。

# etcd wal介绍
WAL是write ahead log的缩写，etcd使用其实现的wal来完成raft日志的持久化存储，etcd对wal的所有实现都在其wal目录中。

## 文件组织
wal的所有日志都会放在一个指定目录下，日志的文件名以 .wal 作为结尾，格式为<seq>-<index>.wal，seq和index的格式都为%016x,例如：0000000000000001-0000000000000001.wal。index代表这个文件中第一条raft日志的index，seq是这个文件的序列号。每个文件的大小默认为64M，当文件大小大于这个文件时，wal会自动生成新的日志文件用于存储日志。
每个日志文件都会使用flock锁定文件，参数为LOCK_EX，这是一把独有锁，同一时间只能有一个进程可以操作这个日志文件。

## 日志逻辑组织
wal日志可以存储多种类型的数据，具体如下。

类型 | 描述
--------|--------- | ---------
metadataType| 文件元信息，每个wall文件都会有一个元信息日志
entryType | raft日志
stateType | 状态信息
crcType | crc类型
snapshotType | 快照

![日志逻辑架构](/images/posts/wal-log.png)


- crcType
每个新的日志文件的第一条记录都会是crcType类型的记录，crcType也只会在每个日志文件的开始时写入，用于记录上一个文件最后的crc是什么
- metadataType
每个新的日志文件中metadataType紧跟在crcType记录后面，每个日志文件只会出现一次
- stateType
这种日志类型会在两种情况下加入：
1. 自动切分日志文件时，新的日志文件中，紧跟在metadataType后面会存入一条stateType的日志
2. 当raft核心程序ready中返回hard state时也会存储该类型的日志
- snapshotType
wal日志中只会存储snapshot的term和index，具体的数据存储在专门的snapshot中，每次存储快照都会在wal日志中存储一个wal的快照。当存储快照时，会将快照之前index的日志文件都释放掉。wal中存储的snapshot主要用于检查起始的快照是否正确。


## 日志读写
wal通过封装的encoder和decoder模块来实现日志读写。

### 写日志
wal通过encoder实现写日志，在这个模块中会完成crc的计算。wal为了更好的管理数据，日志中的每条数据都会以8字节对齐，wal会自动对齐字节。
日志写入的流程如下。

![encoder写日志流程](/images/posts/encoder-flow.png)

图中的crc是增量计算，以之前的所有日志数据为增量基础。
wal只关注写入日志，不会校验日志的index是否重复，如果将raft节点未提交的日志粗暴的写入到wal中时可能会造成数据的混乱。etcd-raft官方提供的例子就有这个问题(它只是简单的将core传过来的数据粗暴的写入到日志中)

## 日志切分
wal实现了日志自动切分，当日志数据大于默认的64M时就会生成新的文件写入日志，日志的切分通过wal.go文件中的cut方法来实现。cut方法只会在调用wal中的Save方法才会触发调用。

## 日志compact
wal没有实现日志的自动compact，系统只提供了MemoryStorage的日志compact方法（需要用户主动调用）。
