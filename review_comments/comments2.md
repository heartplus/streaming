
**“可实现的系统设计” → “可长期运行的生产级存储系统”**

重点补齐几个核心部分：

1. **Append 幂等与 offset 分配**
2. **Commit offset 算法**
3. **读一致性**
4. **Recovery 算法**
5. **副本同步机制**
6. **写入 durability 策略**
7. **客户端 pipeline 与流控**
8. **MetaServer 扩展**
9. **可运维性（监控/限流/诊断）**

我会给出 **可以直接加入文档的新章节设计**，并指出哪些地方需要修改原设计。
（引用原设计：）

---

# 一、核心设计升级（最关键）

## 1 Append 幂等机制（Production 必须）

原设计：

```
AppendRequest {
    segment_id
    epoch
    offset
    data
}
```

问题：

* retry 会导致重复写
* offset 冲突
* 无法实现 exactly-once

### 改进

引入 **append_id + sequence**

```
AppendRequest {
    uint64 segment_id
    uint64 epoch
    uint64 writer_id
    uint64 sequence
    bytes data
}
```

writer 维护：

```
sequence++
```

DataServer 维护：

```
map<writer_id, last_sequence>
```

规则：

```
if sequence <= last_sequence:
    return DUPLICATE_OK

if sequence == last_sequence + 1:
    append()

else:
    return SEQUENCE_GAP
```

这样：

```
retry append
```

不会写两次。

这也是 **Kafka idempotent producer 的核心思想**。

---

# 二、offset 分配方式

当前 offset 由 client 指定（不安全）。

建议改为：

**DataServer 分配 offset**

append RPC：

```
AppendRequest {
    segment_id
    epoch
    writer_id
    sequence
    data
}
```

返回：

```
AppendResponse {
    offset
    length
}
```

流程：

```
client -> replicas
replica allocate offset
```

offset 规则：

```
offset = local_tail
local_tail += len
```

然后 client 等 quorum。

---

# 三、Commit Offset 算法（核心）

当前文档：

> Client 本地推进 committed_offset 

这是 **生产系统不能接受的设计**。

### 正确方式

每个 replica 维护：

```
replica_offset
```

MetaServer 或 client 计算：

```
commit_offset = quorum_min(replica_offsets)
```

例子：

```
A: 100
B: 80
C: 60
quorum = 2
```

排序：

```
100
80
60
```

commit_offset = **80**

因为：

```
至少2副本持久化
```

---

### DataServer API 扩展

```
rpc get_replica_offset
```

返回：

```
ReplicaOffset {
    replica_offset
    epoch
}
```

---

# 四、读一致性设计

当前 read：

```
read from any replica
```

问题：

可能读到：

```
未提交数据
```

### 改进

read 限制：

```
read_offset <= commit_offset
```

流程：

```
Reader -> MetaServer
get commit_offset
```

或者：

```
Reader -> DataServer
get replica_offset
```

选择：

```
max replica_offset
```

---

# 五、Recovery 算法（必须修改）

原文：

> 取最大 offset 作为恢复目标 

这是错误的。

---

### 正确算法

```
replica_offsets = {o1, o2, o3}
sort(desc)
recover_offset = offsets[write_quorum-1]
```

例如：

```
A:100
B:80
C:60
quorum=2
```

recover_offset = **80**

---

### Recovery 流程

```
1 MetaServer 收集 replica_offsets
2 计算 recover_offset
3 truncate > recover_offset
4 sync missing logs
5 mark segment sealed
```

---

# 六、副本同步机制

系统必须支持：

```
replica catch-up
```

新增 RPC：

```
rpc fetch_data
```

```
FetchRequest {
    segment_id
    offset
    max_bytes
}
```

流程：

```
slow replica -> leader replica
fetch missing logs
```

类似：

```
Kafka follower replication
```

---

# 七、DataServer 写入 durability

必须明确：

```
fsync policy
```

新增配置：

```
flush_policy = sync | async | interval
```

### sync

```
write
fsync
ack
```

### async

```
write
ack
background fsync
```

### interval

```
fsync every N ms
```

生产系统一般：

```
5ms flush
```

---

# 八、客户端 Pipeline

现在：

```
append
wait ack
append
```

吞吐非常低。

### 改进

允许：

```
inflight append
```

例如：

```
max_inflight = 64
```

client：

```
append1
append2
append3
...
```

并行等待。

吞吐可提升：

```
10x
```

---

# 九、Batch Append

现在：

```
1 record / RPC
```

生产系统必须：

```
batch append
```

```
AppendRequest {
    repeated records
}
```

优点：

```
减少 RPC
提高吞吐
```

Kafka 每个 batch：

```
1MB
```

---

# 十、Segment 生命周期

增加状态：

```
SegmentState

CREATING
WRITING
SEALING
SEALED
RECOVERING
DELETED
```

防止：

```
并发 seal
```

---

# 十一、MetaServer 扩展

当前：

```
1 raft group
```

未来会成为瓶颈。

建议：

```
partitioned metadata
```

例如：

```
meta_group = hash(stream_id) % N
```

每组：

```
独立 raft group
```

类似：

```
TiKV
Kafka KRaft
```

---

# 十二、Replica Placement 改进

增加：

```
rack awareness
```

副本分布：

```
rack1
rack2
rack3
```

避免：

```
同机架故障
```

---

# 十三、限流与流控

新增：

```
Backpressure
```

DataServer 返回：

```
THROTTLED
```

client：

```
retry with backoff
```

---

# 十四、监控指标（Production 必须）

每个 DataServer 输出：

```
append_qps
append_latency
disk_write_bw
replica_lag
segment_count
```

MetaServer：

```
stream_count
segment_count
recovery_count
```

---

# 十五、数据校验

每条 append：

```
crc32
```

存储：

```
[length][crc][data]
```

读取：

```
verify crc
```

---

# 十六、日志结构优化

data file：

```
|len|crc|data|
|len|crc|data|
```

index：

```
offset -> file_pos
```

每：

```
64KB
```

一个 index。

---

# 十七、Segment Size 建议

当前：

```
64MB
```

生产建议：

```
512MB
或
1GB
```

否则 meta 膨胀。

---

# 十八、生产系统完整写入流程

最终写流程：

```
Writer

append(batch)

-> replicas

replica write
replica fsync

ack

client 收到 quorum ack

update commit_offset
```

---

# 十九、升级后的系统架构

整体架构会变成类似：

```
        Client
          │
          │
     MetaServer
          │
  ┌───────┼────────┐
  │       │        │
DataServer DataServer DataServer
   │          │          │
segment log  segment log segment log
```

Replica 之间：

```
catchup replication
```

---

# 二十、最终 Production 能力

升级后系统可以达到：

| 能力   | 状态 |
| ---- | -- |
| 高写可用 | ✅  |
| 线性日志 | ✅  |
| 幂等写  | ✅  |
| 读一致  | ✅  |
| 副本恢复 | ✅  |
| 节点扩展 | ✅  |
| 高吞吐  | ✅  |

---

# 二十一、系统定位

优化后系统非常接近：

| 系统         | 相似度   |
| ---------- | ----- |
| Kafka      | ⭐⭐⭐⭐⭐ |
| BookKeeper | ⭐⭐⭐⭐  |
| Pravega    | ⭐⭐⭐⭐  |

可以看成：

**Kafka-like Distributed Log Storage**

