# Overall Design: 分布式 Stream 存储系统

## 1. 系统定位

基于 brpc/braft/本地文件系统构建的分布式 Stream 存储系统。提供 append-only 的流式数据写入和随机读取能力。写入路径采用 Quorum-based 协议，目标写可用性不低于 99.99995%。

## 2. 核心概念

| 概念 | 说明 |
|------|------|
| **Stream** | 逻辑上有序、仅追加的数据流，由唯一 `stream_id` 标识。单写多读，单写由用户逻辑保证 |
| **Segment** | Stream 的物理分片单元，一个 Stream 由有序的 Segment 列表组成 |
| **Replica** | Segment 在某个 DataServer 上的副本，每个 Segment 有 N 个 Replica 分布在不同 DataServer 上 |
| **MetaServer** | 基于 braft/rocksdb 的元数据服务，管理 Stream 和 Segment 的元信息，自身通过 Raft 保证高可用 |
| **DataServer** | 数据存储节点，负责 Segment 副本的实际数据存储（本地 SSD 文件系统），处理写入和读取请求 |

## 3. 整体架构

```
                        ┌──────────┐
                        │  Client  │
                        └────┬─────┘
                             │ brpc
              ┌──────────────┼──────────────┐
              │(meta ops)    │(data IO)     │
              ▼              ▼              ▼
     ┌─────────────┐  ┌───────────┐  ┌───────────┐
     │ MetaServer  │  │DataServer │  │DataServer │  ...
     │ (braft+     │  │ (SSD)     │  │ (SSD)     │
     │  rocksdb)   │  └───────────┘  └───────────┘
     └─────────────┘
      raft peers:
     ┌─────────────┐
     │ MetaServer  │
     └─────────────┘
     ┌─────────────┐
     │ MetaServer  │
     └─────────────┘
```

### 架构要点

- **控制面与数据面分离**：MetaServer 仅管理元数据，不参与数据 IO 路径
- **数据面 Quorum 写入**：Client 直接向多个 DataServer 并发写入，收到 Quorum 数量的确认即视为成功
- **读取直连 DataServer**：Client 从任意持有副本的 DataServer 读取已提交数据
- **MetaServer 可扩展**：元数据存储层抽象为接口，保留后续扩展到分布式 KV 系统的能力

## 4. 数据模型

### 4.1 Stream 元数据（存储在 MetaServer）

```
StreamMeta {
    stream_id     : uint64
    status        : enum { kCreating, kOpen, kSealed, kDeleted }
    segment_list  : vector<SegmentInfo>   // 有序的 Segment 列表
    create_time   : int64
    current_epoch : uint64                // 用于 fencing 旧 writer
}
```

### 4.2 Segment 信息（存储在 MetaServer）

```
SegmentInfo {
    segment_id     : uint64
    stream_id      : uint64
    start_offset   : uint64      // 在 Stream 中的起始偏移
    committed_size : uint64      // 已提交的数据大小
    status         : enum { kCreating, kWriting, kSealing, kSealed, kRecovering, kDeleted }
    epoch          : uint64      // Segment 的 epoch，用于 fencing
    replicas       : vector<ReplicaInfo>
    write_quorum   : uint32      // 写入需要的确认数（如 2/3）
    ack_quorum     : uint32      // 读取需要的确认数（通常 1）
}
```

### 4.3 Replica 信息

```
ReplicaInfo {
    dataserver_id  : uint64
    dataserver_addr: string      // brpc 地址
    replica_status : enum { kActive, kFenced, kFailed, kRecovering }
}
```

### 4.4 DataServer 本地存储布局

DataServer 使用本地 SSD 文件系统存储 Segment 数据：

```
<data_root>/
├── segments/
│   ├── <segment_id>/
│   │   ├── data            # 追加写入的数据文件
│   │   ├── index           # 稀疏索引（offset -> file_position 的映射）
│   │   └── meta            # 本地副本元数据（epoch, 已写入大小等）
│   └── ...
```

#### 数据文件格式

data 文件中每条记录的存储格式：

```
┌────────┬────────┬────────┬──────────────┐
│ length │ crc32  │ sequence│    data      │
│ uint32 │ uint32 │ uint64  │   bytes      │
└────────┴────────┴────────┴──────────────┘
```

- **length**：data 字段的字节数
- **crc32**：对 data 字段的 CRC32C 校验，用于检测静默数据损坏
- **sequence**：Writer 的单调递增序列号，用于幂等去重
- **data**：实际数据内容

#### 索引策略

- **写入期间**：index 仅维护在内存中（`std::vector<IndexEntry>`），每隔固定大小（如 64KB）记录一个 `{offset, file_position}` 索引项
- **Seal 时**：内存索引一次性刷盘写入 index 文件
- **重启恢复**：若 index 文件不存在（非 sealed segment），通过扫描 data 文件重建内存索引

## 5. RPC 接口设计

### 5.1 Client -> MetaServer

```protobuf
service MetaService {
    // 创建 Stream
    rpc create_stream(CreateStreamRequest) returns (CreateStreamResponse);

    // 打开 Stream，获取元数据和路由信息，递增 epoch
    rpc open_stream(OpenStreamRequest) returns (OpenStreamResponse);

    // 获取当前可写 Segment 的路由信息
    rpc get_writable_segment(GetWritableSegmentRequest) returns (GetWritableSegmentResponse);

    // 封存当前 Segment，分配新 Segment（需校验 epoch）
    rpc seal_segment(SealSegmentRequest) returns (SealSegmentResponse);

    // 删除 Stream
    rpc delete_stream(DeleteStreamRequest) returns (DeleteStreamResponse);

    // 获取 Stream 的 committed_offset（供 Reader 查询）
    rpc get_committed_offset(GetCommittedOffsetRequest) returns (GetCommittedOffsetResponse);

    // DataServer 心跳上报
    rpc heartbeat(HeartbeatRequest) returns (HeartbeatResponse);
}
```

### 5.2 Client -> DataServer（数据面）

```protobuf
service DataService {
    // 追加数据到指定 Segment 副本（支持 batch）
    rpc append(AppendRequest) returns (AppendResponse);

    // 按偏移量随机读取
    rpc read(ReadRequest) returns (ReadResponse);

    // 封存本地 Segment 副本
    rpc seal(SealRequest) returns (SealResponse);

    // 获取 Segment 副本的当前偏移量（用于 commit 计算和 recovery）
    rpc get_replica_offset(GetReplicaOffsetRequest) returns (GetReplicaOffsetResponse);

    // 副本间数据同步（落后副本从领先副本拉取数据）
    rpc fetch_data(FetchDataRequest) returns (FetchDataResponse);
}
```

#### append（幂等写入 + batch 支持）

```protobuf
message AppendRecord {
    bytes  data     = 1;
    uint32 crc32    = 2;     // data 的 CRC32C 校验
}

message AppendRequest {
    uint64 segment_id     = 1;
    uint64 epoch          = 2;    // 用于 fencing
    uint64 writer_id      = 3;    // Writer 唯一标识
    uint64 sequence       = 4;    // 单调递增序列号（batch 中首条记录的序列号）
    repeated AppendRecord records = 5;  // 支持 batch 写入
}

message AppendResponse {
    int32  status_code = 1;
    uint64 offset      = 2;    // DataServer 分配的写入偏移
    uint64 length      = 3;    // 写入的总字节数
}
```

#### read

```protobuf
message ReadRequest {
    uint64 segment_id = 1;
    uint64 offset     = 2;    // Segment 内偏移
    uint64 length     = 3;
}

message ReadResponse {
    int32  status_code   = 1;
    bytes  data          = 2;
    uint64 actual_length = 3;
    uint64 next_offset   = 4;   // 下一次读取的起始偏移，简化 Client 逻辑
}
```

#### fetch_data（副本间数据同步）

```protobuf
message FetchDataRequest {
    uint64 segment_id = 1;
    uint64 offset     = 2;    // 从此偏移开始拉取
    uint64 max_bytes  = 3;    // 单次最大拉取字节数
}

message FetchDataResponse {
    int32  status_code = 1;
    bytes  data        = 2;
    uint64 next_offset = 3;
}
```

### 5.3 心跳协议

```protobuf
message HeartbeatRequest {
    uint64 dataserver_id    = 1;
    uint64 disk_total_bytes = 2;
    uint64 disk_used_bytes  = 3;
    bool   readonly         = 4;    // 磁盘满时标记只读
    uint32 segment_count    = 5;
    // 各 segment 的 replica_offset 上报
    repeated SegmentOffsetReport segment_offsets = 6;
}

message HeartbeatResponse {
    int32  status_code = 1;
    // 待删除的 segment 列表（GC 指令）
    repeated uint64 segments_to_delete = 2;
}

message SegmentOffsetReport {
    uint64 segment_id     = 1;
    uint64 replica_offset = 2;
    uint64 epoch          = 3;
}
```

## 6. 核心机制

### 6.1 Append 幂等机制

引入 `writer_id + sequence` 实现 exactly-once 语义，防止网络重试导致的重复写入。

**Writer 端**：

- 每次 open_stream 获得唯一 `writer_id`（由 MetaServer 分配）
- 维护单调递增的 `sequence`，每次 append 后 `sequence += batch_size`

**DataServer 端**：

- 维护 `map<writer_id, last_sequence>` 去重表
- 收到 append 请求时：
  - `sequence <= last_sequence`：返回 `kDuplicate`（幂等成功，返回之前的 offset）
  - `sequence == last_sequence + 1`：正常执行 append
  - `sequence > last_sequence + 1`：返回 `kSequenceGap`，Client 需重试缺失的请求

### 6.2 Offset 分配

Offset 由 DataServer 本地分配，而非 Client 指定：

```
offset = local_tail
local_tail += total_record_bytes
```

不同副本上的 offset 可能不同（因乱序到达或部分失败），但每个副本内部保证连续无空洞。committed_offset 的语义在所有副本间通过 Quorum 算法对齐。

**DataServer 端连续性校验**：

```
if request.sequence != last_sequence + 1:
    reject  // 保证本地数据文件无空洞
```

### 6.3 Committed Offset 算法

每个副本维护自己的 `replica_offset`（已持久化的数据偏移量）。系统通过 Quorum 算法计算 `committed_offset`：

```
committed_offset = quorum_min(replica_offsets)
```

即：将所有副本的 `replica_offset` 降序排列，取第 `write_quorum` 位置的值。

**示例**（3 副本，write_quorum=2）：

```
A: 100, B: 80, C: 60
排序: [100, 80, 60]
committed_offset = 80  // 至少 2 个副本已持久化到 80
```

**更新路径**：

1. DataServer 通过心跳定期上报各 Segment 的 `replica_offset`
2. MetaServer 收集所有副本的 `replica_offset`，计算并存储 `committed_offset`
3. Writer 在收到 Quorum ack 后，也在本地计算 `committed_offset` 用于快速确认

### 6.4 Fencing 机制

使用 epoch 机制防止旧 Writer 的脏写，**所有写操作**（append、seal_segment）均需校验 epoch：

1. 每次 open_stream 时，MetaServer 递增 Stream 的 `current_epoch` 并分配 `writer_id`
2. 新 Segment 继承当前 epoch
3. DataServer 收到 append/seal 请求时，检查 epoch：
   - `request.epoch < local.epoch`：拒绝操作（返回 `kFenced`）
   - `request.epoch >= local.epoch`：更新本地 epoch 并执行操作
4. MetaServer 收到 seal_segment 请求时，校验 `request.epoch == current_epoch`，防止旧 Writer 意外封存新 Writer 正在使用的 Segment
5. 旧 Writer 收到 `kFenced` 后停止所有操作

### 6.5 写入 Durability 策略

DataServer 支持可配置的 fsync 策略：

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `sync` | write → fsync → ack | 最高持久性，延迟较高 |
| `interval` | write → ack，后台每 N ms fsync | 平衡持久性与性能（生产推荐，默认 5ms） |
| `async` | write → ack，依赖 OS page cache | 最高吞吐，掉电可能丢数据 |

### 6.6 数据校验

- **写入时**：Client 计算每条记录的 CRC32C，随 AppendRequest 发送；DataServer 写入前验证 CRC，写入时将 CRC 持久化到 data 文件
- **读取时**：DataServer 读取 data 文件后验证 CRC，校验失败则返回 `kChecksumMismatch`，Client 自动切换到其他副本重试
- **后台巡检**：DataServer 定期扫描 data 文件验证 CRC，发现损坏主动标记副本为 `kFailed`

## 7. 核心流程

### 7.1 写入流程 (Quorum Append with Pipeline)

```
Client (Writer)            MetaServer           DataServer A    DataServer B    DataServer C
  │                            │                     │               │               │
  │── get_writable_segment ──>│                     │               │               │
  │<── SegmentInfo(A,B,C) ────│                     │               │               │
  │                            │                     │               │               │
  │  (pipeline: 多条 append 并发)│                     │               │               │
  │── append(seq=1, batch) ──────────────────────────>│               │               │
  │── append(seq=1, batch) ───────────────────────────────────────────>│               │
  │── append(seq=1, batch) ────────────────────────────────────────────────────────────>│
  │── append(seq=2, batch) ──────────────────────────>│               │               │
  │── append(seq=2, batch) ───────────────────────────────────────────>│               │
  │── append(seq=2, batch) ────────────────────────────────────────────────────────────>│
  │                            │                     │               │               │
  │<── ack(seq=1) ───────────────────────────────────│               │               │
  │<── ack(seq=1) ────────────────────────────────────────────────────│               │
  │  (seq=1 收到 2/3 ack, 已提交) │                     │               │               │
  │<── ack(seq=2) ───────────────────────────────────│               │               │
  │<── ack(seq=2) ────────────────────────────────────────────────────│               │
  │  (seq=2 收到 2/3 ack, 已提交) │                     │               │               │
```

**Pipeline 写入**：

- Client 维护 inflight 窗口（默认 `max_inflight = 64`），无需等待前一条 ack 即可发送下一条
- 每条 append 支持 batch（多条记录合并为一次 RPC），减少 RPC 次数
- 吞吐相比同步写入可提升一个数量级

**写入步骤**：

1. Client 从 MetaServer 获取当前可写 Segment 及其副本列表
2. Client 向所有副本 DataServer **并发**发送 append 请求（附带 writer_id、sequence、CRC）
3. DataServer 校验 epoch → 校验 sequence 连续性 → 校验 CRC → 写入 data 文件 → 按 flush_policy 持久化 → 返回 ack
4. Client 等待 **write_quorum** 个确认，推进本地 committed_offset
5. 若某个 DataServer 持续失败，Client 请求 MetaServer seal 当前 Segment，分配新 Segment

### 7.2 读取流程 (Read Committed)

```
Client (Reader)            MetaServer           DataServer (any replica)
  │                            │                     │
  │── open_stream ────────────>│                     │
  │<── StreamMeta(segments) ──│                     │
  │                            │                     │
  │── get_committed_offset ──>│                     │
  │<── committed_offset ──────│                     │
  │                            │                     │
  │  (根据 offset 定位 segment)  │                     │
  │── read(seg, offset, len) ────────────────────────>│
  │<── data + next_offset ────────────────────────────│
```

**读一致性保证**：

- Reader 从 MetaServer 获取 `committed_offset`，只读取 `offset <= committed_offset` 范围内的数据
- 对于已 sealed 的 Segment，所有数据均已提交，可直接读取
- 对于当前正在写入的 Segment，Reader 需定期刷新 `committed_offset`
- DataServer 读取数据后验证 CRC，校验失败自动返回错误，Client 切换副本重试
- ReadResponse 中包含 `next_offset`，简化跨 Segment 读取和 Client SDK 逻辑

**跨 Segment 读取**：Reader 在客户端侧根据 Segment 列表拆分为多次 read 调用并合并。

### 7.3 Segment Seal 与切换

```
Client (Writer)            MetaServer           DataServer A/B/C
  │                            │                     │
  │  (检测到副本故障或 Segment 满) │                     │
  │── seal_segment(seg, epoch)>│                     │
  │                            │── 校验 epoch          │
  │                            │── seal(seg_id) ────>│  (所有副本)
  │                            │<── ack + offsets ───│
  │                            │                     │
  │                            │── 计算 recover_offset │
  │                            │── 触发 Recovery       │
  │                            │── 分配新 Segment      │
  │                            │   选择新的副本集合     │
  │<── new SegmentInfo ────────│                     │
  │                            │                     │
  │  (继续在新 Segment 写入)     │                     │
```

**Seal 触发条件**：

- Segment 数据量达到阈值（默认 512MB）
- 某个副本 DataServer 持续不可达
- Writer 主动关闭 Stream

**快速切换**：当多数副本故障导致无法满足 write_quorum 时，Writer 请求 MetaServer 强制 seal 当前 Segment 并立即在一组健康节点上分配新 Segment。这种"快速切片"能力是达到 99.99995% 写可用性的关键。

### 7.4 Recovery 流程

Segment seal 后，各副本上的数据可能不一致（Quorum 写不保证所有副本都有全部数据）。Recovery 确定最终一致的提交数据：

```
1. MetaServer 收集所有 Active 副本的 replica_offset
2. 计算 recover_offset = sort(replica_offsets, desc)[write_quorum - 1]
3. 截断超过 recover_offset 的数据（仅存在于少数副本的未提交数据）
4. 通过 fetch_data RPC 将缺失数据从领先副本同步到落后副本
5. 所有副本对齐后，更新 SegmentInfo.committed_size，标记 Segment 为 kSealed
```

**示例**（3 副本，write_quorum=2）：

```
A: 100, B: 80, C: 60
recover_offset = 80
→ A 截断到 80，C 从 A 或 B 拉取 [60, 80) 的数据
→ 最终 A=80, B=80, C=80
```

### 7.5 副本后台同步

落后副本不仅在 Recovery 时补齐，运行期间也通过后台同步保持追赶：

- DataServer 监测本地 `replica_offset` 与其他副本的差距
- 差距超过阈值时，通过 `fetch_data` RPC 从领先副本拉取数据
- 避免在 Seal 瞬间进行大规模数据拷贝造成 IO 尖峰

### 7.6 GC 流程

Stream 删除后的物理数据清理：

1. MetaServer 将 Stream 状态标记为 `kDeleted`，记录其所有 Segment ID
2. 通过心跳响应向 DataServer 下发 `segments_to_delete` 列表
3. DataServer 异步删除本地 Segment 目录及文件
4. DataServer 在后续心跳中确认删除完成
5. MetaServer 清理元数据

## 8. 模块划分

```
src/
├── common/                      # 通用工具
│   ├── config.h/cc              # 配置管理
│   ├── status.h                 # 统一错误码
│   ├── logging.h                # 日志工具（含比例采样）
│   ├── crc32.h/cc               # CRC32C 校验工具
│   └── id_generator.h/cc        # ID 生成器
│
├── proto/                       # protobuf 定义
│   ├── common.proto             # 通用类型定义
│   ├── meta_service.proto       # MetaService RPC 定义
│   └── data_service.proto       # DataService RPC 定义
│
├── metaserver/                  # 元数据服务
│   ├── meta_server.h/cc         # MetaServer 主类
│   ├── meta_service_impl.h/cc   # MetaService brpc 服务实现
│   ├── meta_state_machine.h/cc  # braft 状态机（元数据复制）
│   ├── meta_store.h/cc          # 元数据存储层（rocksdb）
│   ├── meta_store_interface.h   # 存储层抽象接口（可扩展到分布式 KV）
│   ├── stream_manager.h/cc      # Stream 生命周期管理
│   ├── segment_allocator.h/cc   # Segment 分配与副本选择策略
│   └── recovery_manager.h/cc    # Segment Recovery 管理
│
├── dataserver/                  # 数据存储服务
│   ├── data_server.h/cc         # DataServer 主类
│   ├── data_service_impl.h/cc   # DataService brpc 服务实现
│   ├── segment_store.h/cc       # Segment 本地存储引擎（文件系统）
│   ├── segment_writer.h/cc      # Segment 追加写入（含幂等去重）
│   ├── segment_reader.h/cc      # Segment 随机读取（含 CRC 校验）
│   ├── segment_index.h/cc       # 稀疏索引管理（内存 + 刷盘）
│   ├── replica_syncer.h/cc      # 副本后台同步
│   └── disk_monitor.h/cc        # 磁盘空间监控
│
├── client/                      # Client SDK
│   ├── stream_client.h/cc       # 客户端主入口
│   ├── stream_writer.h/cc       # 写入器（Quorum 写 + pipeline + batch）
│   ├── stream_reader.h/cc       # 读取器（read committed + 跨 Segment）
│   └── meta_client.h/cc         # MetaServer 客户端
│
└── server/                      # 服务入口
    ├── meta_main.cc             # MetaServer 启动入口
    └── data_main.cc             # DataServer 启动入口
```

## 9. 关键设计决策

### 9.1 Quorum 写 vs Raft 复制

数据面采用 Quorum 写而非 Raft，原因：
- **更高写可用性**：Quorum 写无需 Leader 选举，任意 write_quorum 个节点可用即可写入，避免 Leader 切换导致的写入中断
- **更低写延迟**：无需 Leader 中转，Client 直接并发写入所有副本
- **更灵活的容错**：配合 Segment seal + 快速切换，单节点故障可在毫秒级恢复写入
- **单写简化一致性**：Stream 单写由用户保证，无需 Raft 的 Leader 仲裁

### 9.2 MetaServer 存储抽象

MetaServer 的存储层通过 `MetaStoreInterface` 抽象：

```cpp
class MetaStoreInterface {
public:
    virtual Status put(const std::string& key, const std::string& value) = 0;
    virtual Status get(const std::string& key, std::string* value) = 0;
    virtual Status del(const std::string& key) = 0;
    virtual Status scan(const std::string& prefix, ScanCallback cb) = 0;
};
```

当前实现基于 rocksdb（单 Raft Group 复制）。Key 设计时预留按 `stream_id` 哈希分片的能力（如 `hash(stream_id) % N`），后续可无缝扩展到多 Raft Group 或分布式 KV 系统。

### 9.3 Segment 副本选择策略

分配新 Segment 时，MetaServer 根据以下因素选择 DataServer：
- **故障域分散**：副本分布在不同机架（rack awareness），避免同机架故障导致多副本同时不可用
- **磁盘利用率**：优先选择剩余空间充足的节点，排除已标记 `readonly` 的节点
- **当前负载**：参考心跳上报的写入 QPS / 带宽
- **故障历史**：排除最近频繁故障的节点

### 9.4 日志采样

每个 RPC 服务结束时打印 INFO 日志，支持采样率控制：

```cpp
// sample_rate: 1 = 100%（每条都打印），10000 = 0.01%
void log_with_sampling(uint32_t sample_rate, const char* fmt, ...);
```

## 10. 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `segment_max_size` | 512 MB | 单个 Segment 最大数据量，达到后触发 seal |
| `replica_count` | 3 | 每个 Segment 的副本数 |
| `write_quorum` | 2 | 写入需要的最少确认数 |
| `index_interval` | 64 KB | 稀疏索引的间隔 |
| `meta_raft_group` | 3 节点 | MetaServer Raft Group 大小 |
| `flush_policy` | interval | 持久化策略：sync / interval / async |
| `flush_interval_ms` | 5 | interval 模式下的 fsync 间隔（毫秒） |
| `max_inflight` | 64 | Client pipeline 最大并发 append 数 |
| `max_batch_size` | 1 MB | 单次 append batch 的最大字节数 |
| `disk_readonly_threshold` | 95% | 磁盘使用率达到此阈值时标记 readonly |
| `replica_sync_lag_threshold` | 16 MB | 副本落后超过此值触发后台同步 |

## 11. 错误处理

```cpp
enum class StatusCode : int32_t {
    kOk = 0,
    kNotFound = 1,
    kAlreadyExists = 2,
    kInvalidArgument = 3,
    kIOError = 4,
    kNotLeader = 5,        // MetaServer 非 Leader
    kStreamSealed = 6,
    kSegmentSealed = 7,
    kSegmentFull = 8,
    kFenced = 9,           // epoch 过期，被 fence
    kTimeout = 10,
    kInternal = 11,
    kUnavailable = 12,     // DataServer 不可用
    kDuplicate = 13,       // 幂等重复写入（成功）
    kSequenceGap = 14,     // sequence 不连续
    kChecksumMismatch = 15,// CRC 校验失败
    kThrottled = 16,       // 被限流
    kDiskFull = 17,        // 磁盘满
};
```

### 错误恢复策略

| 错误 | Client 行为 |
|------|------------|
| `kNotLeader` | 刷新 MetaServer Leader 地址并重试 |
| `kSegmentSealed` | 从 MetaServer 获取新的可写 Segment |
| `kFenced` | Writer 停止所有操作，通知上层应用 |
| `kUnavailable` | 若仍满足 Quorum，继续写入；否则 seal 并切换 Segment |
| `kIOError` / `kDiskFull` | 标记副本为 Failed，seal 并切换 Segment |
| `kDuplicate` | 视为成功（幂等），继续下一条 |
| `kSequenceGap` | Client 重试缺失的 sequence |
| `kChecksumMismatch` | 切换到其他副本重试读取 |
| `kThrottled` | Client 指数退避重试 |

## 12. 监控指标

### DataServer

| 指标 | 说明 |
|------|------|
| `append_qps` | 每秒追加请求数 |
| `append_latency_p99` | 追加延迟 P99 |
| `read_qps` | 每秒读取请求数 |
| `read_latency_p99` | 读取延迟 P99 |
| `disk_write_bandwidth` | 磁盘写入带宽 |
| `disk_usage_ratio` | 磁盘使用率 |
| `replica_lag_bytes` | 副本落后字节数 |
| `segment_count` | 管理的 Segment 数量 |
| `checksum_error_count` | CRC 校验失败次数 |

### MetaServer

| 指标 | 说明 |
|------|------|
| `stream_count` | 管理的 Stream 数量 |
| `segment_count` | 管理的 Segment 数量 |
| `seal_qps` | 每秒 Seal 操作数 |
| `recovery_in_progress` | 正在进行的 Recovery 数量 |
| `raft_apply_latency` | Raft apply 延迟 |

## 13. 后续扩展点

- **Stream Truncation**：按偏移量截断历史数据，释放存储空间
- **Compaction**：对已 sealed 的 Segment 进行合并
- **流控与配额**：基于 Stream 维度的写入速率限制和背压机制
- **快照与备份**：Segment 级别的数据备份与恢复
- **MetaServer 分片**：按 `hash(stream_id) % N` 将元数据分散到多个 Raft Group
- **Proxy 层**：可选的无状态 Proxy 层，收敛 Quorum 写入逻辑到服务端，降低 Client SDK 复杂度
- **安全鉴权**：基于 Stream 维度的 ACL 控制（谁可写、谁可读）
