---
displayed_sidebar: "Chinese"
keywords: ['Fuben']
---

# 管理副本

本文介绍如何管理 StarRocks 中的数据副本 (replica)。

> **注意**
>
> 为保证元数据一致，您需要在 Leader FE 节点进行副本管理操作。

## 修复副本

TabletChecker 作为常驻的后台进程，会定期检查所有分片的状态。对于非健康状态的分片，将会交给 TabletScheduler 进行调度和修复。修复的实际操作，都由 BE 节点上的 Clone 任务完成。FE 节点只负责生成这些 Clone 任务。

> 注意：
>
> * 副本修复的主要思想是先通过创建或补齐使得分片的副本数达到期望值，然后再删除多余的副本。
> * 一个 Clone 任务就是完成从一个指定远端 BE 拷贝指定数据到指定目的端 BE 的过程。

### 查看副本状态

您可以查看副本的健康状态。

1. **检查集群中所有副本的状态。**

    ```sql
    SHOW PROC '/statistic';
    ```

    示例：

    ```plain text
    +----------+-----------------------------+----------+--------------+----------+-----------+------------+--------------------+-----------------------+
    | DbId     | DbName                      | TableNum | PartitionNum | IndexNum | TabletNum | ReplicaNum | UnhealthyTabletNum | InconsistentTabletNum |
    +----------+-----------------------------+----------+--------------+----------+-----------+------------+--------------------+-----------------------+
    | 35153636 | default_cluster:DF_Newrisk  | 3        | 3            | 3        | 96        | 288        | 0                  | 0                     |
    | 48297972 | default_cluster:PaperData   | 0        | 0            | 0        | 0         | 0          | 0                  | 0                     |
    | 5909381  | default_cluster:UM_TEST     | 7        | 7            | 10       | 320       | 960        | 1                  | 0                     |
    | Total    | 240                         | 10       | 10           | 13       | 416       | 1248       | 1                  | 0                     |
    +----------+-----------------------------+----------+--------------+----------+-----------+------------+--------------------+-----------------------+
    ```

    * `UnhealthyTabletNum` 列显示了对应的 Database 中，有多少 Tablet 处于非健康状态。
    * `InconsistentTabletNum` 列显示了对应的 Database 中，有多少 Tablet 处于副本不一致的状态。

    `Total` 行对整个集群进行了统计。正常情况下 `UnhealthyTabletNum` 和 `InconsistentTabletNum` 应为 0。如果不为零，可以进一步查看具体有哪些 Tablet。如上图中，UM_TEST 数据库有 1 个 Tablet 状态不健康。

    您可以通过一下命令查询状态不健康的 Tablet。

    ```sql
    SHOW PROC '/statistic/DbId';
    ```

    `DbId` 为数据库所对应的 ID。

    示例：

    ```plain text
    +------------------+---------------------+
    | UnhealthyTablets | InconsistentTablets |
    +------------------+---------------------+
    | [40467980]       | []                  |
    +------------------+---------------------+
    ```

    查询返回状态不健康的 Tablet ID 为 40467980。
2. **检查表或分区中的副本状态。**
    您可以通过以下命令查看指定表或分区的副本状态，并可以通过 WHERE 语句对状态进行过滤。

    ```sql
    ADMIN SHOW REPLICA STATUS FROM tbl PARTITION (p1, p2, ...) WHERE STATUS = "STATUS";
    ```

    示例：

    ```plain text
    mysql> ADMIN SHOW REPLICA STATUS FROM tbl PARTITION (p1, p2) WHERE STATUS = "OK";
    +----------+-----------+-----------+---------+-------------------+--------------------+------------------+------------+------------+-------+--------+--------+
    | TabletId | ReplicaId | BackendId | Version | LastFailedVersion | LastSuccessVersion | CommittedVersion | SchemaHash | VersionNum | IsBad | State  | Status |
    +----------+-----------+-----------+---------+-------------------+--------------------+------------------+------------+------------+-------+--------+--------+
    | 29502429 | 29502432  | 10006     | 2       | -1                | 2                  | 1                | -1         | 2          | false | NORMAL | OK     |
    | 29502429 | 36885996  | 10002     | 2       | -1                | -1                 | 1                | -1         | 2          | false | NORMAL | OK     |
    | 29502429 | 48100551  | 10007     | 2       | -1                | -1                 | 1                | -1         | 2          | false | NORMAL | OK     |
    | 29502433 | 29502434  | 10001     | 2       | -1                | 2                  | 1                | -1         | 2          | false | NORMAL | OK     |
    | 29502433 | 44900737  | 10004     | 2       | -1                | -1                 | 1                | -1         | 2          | false | NORMAL | OK     |
    | 29502433 | 48369135  | 10006     | 2       | -1                | -1                 | 1                | -1         | 2          | false | NORMAL | OK     |
    +----------+-----------+-----------+---------+-------------------+--------------------+------------------+------------+------------+-------+--------+--------+
    ```

    结果展示所有副本的状态。其中 `IsBad` 列为 `true` 则表示副本已经损坏。而 `Status` 列则会显示另外的其他状态。更多信息，请参考 [ADMIN SHOW REPLICA STATUS](../sql-reference/sql-statements/Administration/ADMIN_SHOW_REPLICA_STATUS.md)。

    您还可以通过以下命令查看指定表中副本的一些额外信息。

    ```sql
    SHOW TABLET FROM tbl1;
    ```

    示例：

    ```plain text
    +----------+-----------+-----------+------------+---------+-------------+-------------------+-----------------------+------------------+----------------------+---------------+----------+----------+--------+-------------------------+--------------+----------------------+--------------+----------------------+----------------------+----------------------+
    | TabletId | ReplicaId | BackendId | SchemaHash | Version | VersionHash | LstSuccessVersion | LstSuccessVersionHash | LstFailedVersion | LstFailedVersionHash | LstFailedTime | DataSize | RowCount | State  | LstConsistencyCheckTime | CheckVersion |     CheckVersionHash | VersionCount | PathHash             | MetaUrl              | CompactionStatus     |
    +----------+-----------+-----------+------------+---------+-------------+-------------------+-----------------------+------------------+----------------------+---------------+----------+----------+--------+-------------------------+--------------+----------------------+--------------+----------------------+----------------------+----------------------+
    | 29502429 | 29502432  | 10006     | 1421156361 | 2       | 0           | 2                 | 0                     | -1               | 0                    | N/A           | 784      | 0        | NORMAL | N/A                     | -1           |     -1               | 2            | -5822326203532286804 | url                  | url                  |
    | 29502429 | 36885996  | 10002     | 1421156361 | 2       | 0           | -1                | 0                     | -1               | 0                    | N/A           | 784      | 0        | NORMAL | N/A                     | -1           |     -1               | 2            | -1441285706148429853 | url                  | url                  |
    | 29502429 | 48100551  | 10007     | 1421156361 | 2       | 0           | -1                | 0                     | -1               | 0                    | N/A           | 784      | 0        | NORMAL | N/A                     | -1           |     -1               | 2            | -4784691547051455525 | url                  | url                  |
    +----------+-----------+-----------+------------+---------+-------------+-------------------+-----------------------+------------------+----------------------+---------------+----------+----------+--------+-------------------------+--------------+----------------------+--------------+----------------------+----------------------+----------------------+
    ```

    结果展示了包括副本大小、行数、版本数量、所在数据路径等一些额外的信息。

    > 注意：这里显示的 `State` 列的内容不代表副本的健康状态，而是副本处于某种任务下的状态，比如 CLONE、SCHEMA_CHANGE、ROLLUP 等。

    此外，您也可以通过以下命令，查看指定表或分区的副本分布情况，来检查副本分布是否均匀。

    ```sql
    ADMIN SHOW REPLICA DISTRIBUTION FROM tbl1;
    ```

    示例：

    ```plain text
    +-----------+------------+-------+---------+
    | BackendId | ReplicaNum | Graph | Percent |
    +-----------+------------+-------+---------+
    | 10000     | 7          |       | 7.29 %  |
    | 10001     | 9          |       | 9.38 %  |
    | 10002     | 7          |       | 7.29 %  |
    | 10003     | 7          |       | 7.29 %  |
    | 10004     | 9          |       | 9.38 %  |
    | 10005     | 11         | >     | 11.46 % |
    | 10006     | 18         | >     | 18.75 % |
    | 10007     | 15         | >     | 15.62 % |
    | 10008     | 13         | >     | 13.54 % |
    +-----------+------------+-------+---------+
    ```

    结果展示了表 `tbl1` 的副本在各个 BE 节点上的个数、百分比，以及一个简单的图形化显示。
3. **检查 Tablet 中的副本状态。**
    当定位至具体的 Tablet 时，您可以使用以下命令来查看特定 Tablet 的状态。

    ```sql
    SHOW TABLET tablet_id;
    ```

    ```plain text
    mysql> SHOW TABLET 29502553;
    +------------------------+-----------+---------------+-----------+----------+----------+-------------+----------+--------+---------------------------------------------------------------------------+
    | DbName                 | TableName | PartitionName | IndexName | DbId     | TableId  | PartitionId | IndexId  | IsSync | DetailCmd                                                                 |
    +------------------------+-----------+---------------+-----------+----------+----------+-------------+----------+--------+---------------------------------------------------------------------------+
    | default_cluster:test   | test      | test          | test      | 29502391 | 29502428 | 29502427    | 29502428 | true   | SHOW PROC '/dbs/29502391/29502428/partitions/29502427/29502428/29502553'; |
    +------------------------+-----------+---------------+-----------+----------+----------+-------------+----------+--------+---------------------------------------------------------------------------+
    ```

    结果展示了当前 Tablet 所对应的数据库、表、分区、上卷表等信息。

    您可以复制并执行 `DetailCmd` 列中的命令以查看详细信息。

    ```plain text
    mysql> SHOW PROC '/dbs/29502391/29502428/partitions/29502427/29502428/29502553';
    +-----------+-----------+---------+-------------+-------------------+-----------------------+------------------+----------------------+---------------+------------+----------+----------+--------+-------+--------------+----------------------+----------+------------------+
    | ReplicaId | BackendId | Version | VersionHash | LstSuccessVersion | LstSuccessVersionHash | LstFailedVersion | LstFailedVersionHash | LstFailedTime | SchemaHash | DataSize | RowCount | State  | IsBad | VersionCount | PathHash             | MetaUrl  | CompactionStatus |
    +-----------+-----------+---------+-------------+-------------------+-----------------------+------------------+----------------------+---------------+------------+----------+----------+--------+-------+--------------+----------------------+----------+------------------+
    | 43734060  | 10004     | 2       | 0           | -1                | 0                     | -1               | 0                    | N/A           | -1         | 784      | 0        | NORMAL | false | 2            | -8566523878520798656 | url      | url              |
    | 29502555  | 10002     | 2       | 0           | 2                 | 0                     | -1               | 0                    | N/A           | -1         | 784      | 0        | NORMAL | false | 2            | 1885826196444191611  | url      | url              |
    | 39279319  | 10007     | 2       | 0           | -1                | 0                     | -1               | 0                    | N/A           | -1         | 784      | 0        | NORMAL | false | 2            | 1656508631294397870  | url      | url              |
    +-----------+-----------+---------+-------------+-------------------+-----------------------+------------------+----------------------+---------------+------------+----------+----------+--------+-------+--------------+----------------------+----------+------------------+
    ```

    结果展示了对应 Tablet 的所有副本情况。

### 调度副本优先级

TabletScheduler 里等待被调度的分片会根据状态不同，被赋予不同的优先级。

通常情况下，系统会自动判断并调度副本优先级。但如果您希望某些表或分区的分片能够更快的被修复，您可以手动调度副本优先级。

通过以下命令调度副本优先级，对需要优先修复的表或分区中的有问题的 Tablet，给予 VERY_HIGH 的优先级。

```sql
ADMIN REPAIR TABLE tbl [PARTITION (p1, p2, ...)];
```

> 注意：此命令只是一个 Hint，并不能保证一定能修复成功，且副本的优先级仍会随 TabletScheduler 的调度而发生变化。当 Leader FE 节点切换或重启后，该命令所包含的信息会丢失。

您也可以通过以下命令取消优先级调度。

```sql
ADMIN CANCEL REPAIR TABLE tbl [PARTITION (p1, p2, ...)];
```

## 均衡副本

StarRocks 会自动进行集群内的副本均衡。

均衡的主要思想，对于某些分片，首先在低负载的节点上创建一个副本，然后再删除这些分片在高负载节点上的副本。同时，因为不同存储介质的存在，在同一个集群内的不同 BE 节点上，可能存在一种或两种存储介质。StarRocks 要求存储介质为 A 的分片在均衡后，尽量依然存储在存储介质 A 中。所以 StarRocks 根据存储介质，对集群的 BE 节点进行划分。然后针对不同的存储介质的 BE 节点集合，进行负载均衡调度。同样，副本均衡会保证不会将同一个 Tablet 的副本部署在同一个 host 的 BE 上。

### BE 节点负载

StarRocks 使用 ClusterLoadStatistics（CLS）表示一个集群中各个 BE 的负载均衡情况。TabletScheduler 根据此统计值，来触发集群均衡。StarRocks 当前通过 **磁盘使用率** 和 **副本数量** 两个指标，为每个 BE 计算得出一个 loadScore，作为 BE 的负载分数。分数越高，表示该 BE 的负载越重。TabletScheduler 会每隔 1 分钟更新一次 CLS。

磁盘使用率和副本数量各有一个权重系数，分别为 `capacityCoefficient` 和 `replicaNumCoefficient`，其和衡为 `1`。其中 `capacityCoefficient` 会根据实际磁盘使用率动态调整。当一个 BE 的总体磁盘使用率在 50% 以下，则 `capacityCoefficient` 值为 `0.5`，如果磁盘使用率在 75% 以上，则值为 `1`（您可以通过 FE 配置项 `capacity_used_percent_high_water` 配置）。如果使用率介于 50% ~ 75% 之间，则该权重系数平滑增加，公式为：`capacityCoefficient = 2 * 磁盘使用率 - 0.5`。该权重系数保证当磁盘使用率过高时，该 BE 的负载分数会更高，以使系统尽快降低这个 BE 的负载。

### 均衡策略

TabletScheduler 在每轮调度时，都会通过 LoadBalancer 来选择一定数目的健康分片作为 Balance 的候选分片。在下一次调度时，会尝试根据这些候选分片，进行均衡调度。

### 查看副本调度任务

您可以查看以下副本调度任务。

1. 查看等待被执行的调度任务。

    ```sql
    SHOW PROC '/cluster_balance/pending_tablets';
    ```

    示例：

    ```plain text
    +----------+--------+-----------------+---------+----------+----------+-------+---------+--------+----------+---------+---------------------+---------------------+---------------------+----------+------+-------------+---------------+---------------------+------------+---------------------+--------+---------------------+-------------------------------+
    | TabletId | Type   | Status          | State   | OrigPrio | DynmPrio | SrcBe | SrcPath | DestBe | DestPath | Timeout | Create              | LstSched            | LstVisit            | Finished | Rate | FailedSched | FailedRunning | LstAdjPrio          | VisibleVer | VisibleVerHash      | CmtVer | CmtVerHash          | ErrMsg                        |
    +----------+--------+-----------------+---------+----------+----------+-------+---------+--------+----------+---------+---------------------+---------------------+---------------------+----------+------+-------------+---------------+---------------------+------------+---------------------+--------+---------------------+-------------------------------+
    | 4203036  | REPAIR | REPLICA_MISSING | PENDING | HIGH     | LOW      | -1    | -1      | -1     | -1       | 0       | 2019-02-21 15:00:20 | 2019-02-24 11:18:41 | 2019-02-24 11:18:41 | N/A      | N/A  | 2           | 0             | 2019-02-21 15:00:43 | 1          | 0                   | 2      | 0                   | unable to find source replica |
    +----------+--------+-----------------+---------+----------+----------+-------+---------+--------+----------+---------+---------------------+---------------------+---------------------+----------+------+-------------+---------------+---------------------+------------+---------------------+--------+---------------------+-------------------------------+
    ```

    各列的具体含义如下：

    * `TabletId`：等待调度的 Tablet 的 ID。一个调度任务只针对一个 Tablet。
    * `Type`：任务类型，可以是 REPAIR（修复） 或 BALANCE（均衡）。
    * `Status`：该 Tablet 当前的状态，如 REPLICA_MISSING（副本缺失）。
    * `State`：该调度任务的状态，可能为 PENDING/RUNNING/FINISHED/CANCELLED/TIMEOUT/UNEXPECTED。
    * `OrigPrio`：初始的优先级。
    * `DynmPrio`：当前动态调整后的优先级。
    * `SrcBe`：源端 BE 节点的 ID。
    * `SrcPath`：源端 BE 节点的路径的 hash 值。
    * `DestBe`：目的端 BE 节点的 ID。
    * `DestPath`：目的端 BE 节点的路径的 hash 值。
    * `Timeout`：当任务被调度成功后，这里会显示任务的超时时间，单位秒。
    * `Create`：任务被创建的时间。
    * `LstSched`：上一次任务被调度的时间。
    * `LstVisit`：上一次任务被访问的时间。这里“被访问”指包括被调度，任务执行汇报等和这个任务相关的被处理的时间点。
    * `Finished`：任务结束时间。
    * `Rate`：clone 任务的数据拷贝速率。
    * `FailedSched`：任务调度失败的次数。
    * `FailedRunning`：任务执行失败的次数。
    * `LstAdjPrio`：上一次优先级调整的时间。
    * `CmtVer/CmtVerHash/VisibleVer/VisibleVerHash`：用于执行 Clone 任务的 version 信息。
    * `ErrMsg`：任务被调度和运行过程中，出现的错误信息。

2. **查看正在运行的调度任务。**

    ```sql
    SHOW PROC '/cluster_balance/running_tablets';
    ```

    其结果中各列的含义和 `pending_tablets` 相同。
3. **查看已结束的调度任务。**

    StarRocks 默认保留最近 1000 个完成的任务。

    ```sql
    SHOW PROC '/cluster_balance/history_tablets';
    ```

    其结果中各列的含义和 `pending_tablets` 相同。如果 `State` 列为 `FINISHED`，则说明任务正常完成。如果为其他，则可以根据 `ErrMsg` 列的错误信息查看具体原因。

## 资源控制

无论是副本修复还是均衡，都是通过副本在各个 BE 之间拷贝完成的。如果同一台 BE 同一时间执行过多的任务，则会带来较大的 IO 压力。因此，StarRocks 在调度时控制了每个节点上能够执行的任务数目。最小的资源控制单位是磁盘（即在 **be.conf** 中指定的一个数据路径）。StarRocks 默认为每块磁盘配置两个 slot 用于副本修复。一个 clone 任务会占用源端和目的端各一个 slot。如果 slot 数目为零，则不会再对这块磁盘分配任务。该 slot 个数可以通过 FE 的 `tablet_sched_slot_num_per_path` 参数动态配置。

另外，StarRocks 默认为每块磁盘提供 2 个单独的 slot 用于均衡任务。目的是防止高负载的节点因为 slot 被修复任务占用，而无法通过均衡释放空间。
