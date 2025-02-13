# Architecture

## 目标
- 足够的简单，包括使用及可运维性；
- 支持集群；
- 支持多副本；
- 支持多IDC;
- 最终一致性；
- 有一定的自监控能力；
- 有一定的自治理能力；

## Overview

![architecture](../../../assets/images/design/architecture.png)

主要分为如下几个组件：
1. Broker
2. Storage
3. ETCD

### Broker

Broker 是一个无状态的服务，具有水平扩容能力。

Master 是 Broker 中承担特殊职责的节点，因为 LinDB 所有的 Metadata 的变更需要唯一节点来完成，以保证 Metadata 变更的一致性。Master 可以从任一 Broker 节点中产生，但 Master 不会维护或者存储太多的 Metadata，所有的 Metadata 都存储在 ETCD 中，因此 Master 可以自由的从 Broker 进行选举，以保证当当前 Master 出问题的时候可以快速的切换，切换操作需要系统自动来完成，整个 Master 的选举为抢占式。

Broker 的主要职责如下：
1. 所有的读写操作都通过 Broker 暴露给终端用户，用户主要跟 Broker 进行交互；
2. WAL Replica；
3. 执行用户的查询请求，根据具体的查询情况生成不同的执行计划；
4. 作为计算层聚合 Broker/Storage 查询返回的数据；
5. 多IDC查询结果的再聚合；

### Storage

Storage 也是一个无状态的服务，只存储实际的数据，不存储整个 Storage 集群的 Metadata，因此也具有水平扩展的能力。主要职责如下：
1. 存储数据及索引；
2. 存储当前节点自己的 Metadata；
2. 执行数据的过滤及一些简单的聚合操作(最原子的聚合计算)；
3. 执行 Broker 下发的 Metadata 变更任务，如创建数据库，数据治理等；
4. 定期上报自身服务的状态；

### ETCD

ETCD 是整个系统的唯一外部依赖，我们希望对 ETCD 也是一个弱依赖，即在 ETCD 不可用的情况下，整个 LinDB 也是可以对外提供服务。

ETCD 主要职责如下：
1. 存储元数据，如数据库的配置，各分片的信息等；
2. 状态信息，如 Broker/Storage 集群各节点的状态；
3. 协调任务状态管理，即所有的 Metadata 的变更都通过 ETCD 中转下发到 Storage 节点；

既然 ETCD 中存储了整个集群所有 Metadata/Status，那么怎么来做到 ETCD 的弱依赖呢？
- 前提条件；
    1. 当 ETCD 出问题的时候，不存在 Metadata 信息的变更，如不修改数据库的配置等；
    2. 集群状态是健康的，即还是可以使用当前的 Metadata/Status 来协调整个集群；
- Broker/Storage 有 Metadata 上报的能力，即当 ETCD 中的数据完整不可用或者丢失时，Broker/Storage 可以上报 Metadata/Status 到新的 ETCD 集群；