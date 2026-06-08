# Dynamo Router 调用链与关键信息

从 `prefill_op` 的 forward 入口追踪到 router 核心（`KvRouter` / `KvPushRouter`）的完整调用过程与决策逻辑。

> **核心结论**
> `KvRouter` 只负责**选 worker**，不转发；真正的端到端路由（选 worker + 写回 indexer + 分发 + 生命周期）由 `KvPushRouter` 完成。分离式服务下，从 `prefill_op` 出发会先后经过 **prefill 侧**和 **decode 侧**两个 router 核心。

---

## 代码分两层

| 层 | 路径 | 内容 |
|---|---|---|
| **算法层** | `lib/kv-router/`（crate `dynamo-kv-router`） | 纯算法 / 状态层，不依赖 runtime 的具体类型。`indexer/`（radix tree 前缀匹配）、`scheduling/`（选择器、队列、本地调度器）、`sequences/`（活跃序列 / active block 跟踪） |
| **集成层** | `lib/llm/src/kv_router/` | 把算法层与 Dynamo runtime（discovery、NATS、pipeline）粘合。对外提供 `KvRouter`、`KvPushRouter`、`Indexer`（含远端/旁路）、`publisher`、`PrefillRouter`、`sticky_sessions` 等 |

---

## 核心组件关系

```text
请求 tokens
   │
   ▼
KvPushRouter (端到端，含分发 / 生命周期)
   │  调用
   ▼
KvRouter::find_best_match_details          ← 顶层协调
   ├─► Indexer.find_matches_by_tier()       ← 缓存命中（device / host / disk 分层）
   ├─► (可选) SharedKvCache.check_blocks()   ← 外部共享缓存（HiCache）
   └─► KvScheduler.schedule()
          └─► LocalScheduler → SchedulerQueue → WorkerSelector(cost function)
                                    │
                                    └─► ActiveSequencesMultiWorker(active block 跟踪)
```

> 缓存块由 `KvIndexer` 的全局 radix tree 维护；活跃块由 router 本地 slot manager 跟踪。

---

## 两个核心负载指标

router 为每个 worker 跟踪这两类负载，共同喂给成本函数。

- **Potential Prefill Blocks（预填充成本）**
  新请求需从头计算的 token =（总输入 − 重叠块 × block_size）/ block_size。
  来源：`KvIndexer` 的全局前缀树（基于 worker 上报的 KV 事件）。重叠越多 → 改善 TTFT。

- **Potential Decode Blocks（解码负载）**
  worker 当前活跃序列占用的 block，反映其繁忙程度。
  来源：router 本地 slot manager（`ActiveSequencesMultiWorker`）。加入 +1、prefill 完成更新、释放 −1。影响 ITL。

---

## 算子流水线位置

```text
frontend → preprocessor → migration → backend → [prefill_op] → [service_backend]
```

KV 模式下 `service_backend` = decode 侧 `KvPushRouter`。
来源：`lib/llm/src/entrypoint/input/common.rs:369–379`

---

## 调用链（从 prefill_op forward）

缩进表示嵌套调用；标 ★ 的为进入 router 核心的步骤。

1. **`PrefillRouter::generate`**
   `prefill_op` 的 forward 入口。未激活 prefill → 直接走 `next`（聚合模式）；否则 `set_phase(Prefill)`，克隆 `prefill_req`（`max_tokens=1`）。
   `lib/llm/src/kv_router/prefill_router/mod.rs:82`

   2. **`resolve_prefill_worker(prefill_req, preselected)`**
      为 Bootstrap 优化提前选 worker、拿 bootstrap（host/port/room）。无预选 worker 时调 `query_prefill_worker`。
      `lib/llm/src/kv_router/prefill_router/execution.rs:24`

      3. ★ **`KvRouter::find_best_match(update_states=false)`**
         peek 行为：只窥探最佳 worker，不预订调度槽。非 KV 模式则走 `SimpleRouter` 轮询/随机。
         `execution.rs:281 → kv_router.rs:632`

   4. **`execute_prefill` / `spawn_prefill_task`**
      真正执行 prefill。Bootstrap 路径后台 spawn，原始路径同步等待完成。
      `lib/llm/src/kv_router/prefill_router/execution.rs:128`

      5. ★ **`InnerPrefillRouter::generate_to_worker → KvPushRouter::generate`**
         prefill 侧端到端路由：请求已带 `prefill_worker_id`，走精确 pin 路径（`update_states=true` 预订槽）+ `inner.direct()` 分发。
         `inner.rs:33 → push_router.rs:490`

   6. ★ **`next.generate(decode_req) → KvPushRouter::generate`**
      prefill 完成后 `set_phase(Decode)`，注入 decode override（`overlap_score_credit=0`、`assume_kv_reuse=false`、`track_prefill_tokens=false`），路由到 decode worker。
      `mod.rs:249 → service_backend (decode KvPushRouter)`

---

## Router 核心内部：find_best_match_details

上面 3 处核心入口最终都收敛到 `KvRouter::find_best_match_details`（`kv_router.rs:464`），内部三步：

- **a. `compute_block_hash_for_seq(tokens)`** — 按 `block_size` 切块并对每块 token 内容做 hash；有 LoRA 时把 adapter 名混入 hash。`kv_router.rs:490`
- **b. `Indexer::find_matches_by_tier(block_hashes)`** — 在 radix tree 上做分层前缀匹配（device / host_pinned / disk），返回 per-worker 命中块数；可并行查询外部 `SharedKvCache`（HiCache）。`kv_router/indexer/mod.rs:337`
- **c. `KvScheduler::schedule → SchedulerQueue → DefaultWorkerSelector`** — 队列准入（容量阈值背压）→ 成本函数为每个 worker 打分 → 选最低 cost（或 softmax 采样）→ `ActiveSequences` 记账。`scheduler.rs / scheduling/queue.rs / scheduling/selector.rs`

---

## 成本函数（选 worker 的核心）

```text
adjusted_prefill = max(
    raw_prefill_blocks
    - overlap_score_credit   * device_overlap_blocks
    - host_cache_hit_weight  * host_overlap_blocks
    - disk_cache_hit_weight  * disk_overlap_blocks
    - shared_cache_multiplier* shared_beyond_blocks,
    0)

cost = prefill_load_scale * adjusted_prefill + decode_blocks   // 越低越好
```

来源：`lib/kv-router/src/scheduling/selector.rs:160–168`
`temperature=0` 取最低 cost（并列随机）；`temperature>0` 对 cost 做 softmax 采样以分散负载。

---

## 三个核心入口对照

| 阶段 | 入口 | 进入的核心 | update_states | 作用 |
|---|---|---|:---:|---|
| 解析 prefill | `query_prefill_worker` | `find_best_match` | false (peek) | 提前选 worker 拿 bootstrap |
| 执行 prefill | `generate_to_worker` | `KvPushRouter::generate` | true (pin) | 路由 + 分发到 prefill worker |
| 执行 decode | `next.generate` | `KvPushRouter::generate` | true | 路由到 decode worker 生成 |

---

## 事件传输模式

- **NATS Core / Event Plane + Local Indexer（默认）**
  worker 各自维护 local radix tree，通过事件面（NATS Core 或 ZMQ）发布；事件带单调递增 id。router 检测序号 gap 主动向 worker 补查；worker 上线 dump 全量、下线移除其块。延迟低、部署简单。

- **JetStream（`--durable-kv-events`，opt-in）**
  KV 事件进持久化 JetStream，各 router replica 作为 durable consumer 消费，状态经 object store 快照跨重启持久化。适合需要多副本一致性的生产环境；前端与 worker 都需开启。已标记 deprecated。

---

## 关键配置（KvRouterConfig）

| 参数 | 默认 | 作用 |
|---|:---:|---|
| `overlap_score_credit` | 1.0 | device 前缀命中折扣（0~1）；0 关闭前缀匹配与事件订阅 |
| `prefill_load_scale` | 1.0 | 扣除缓存 credit 后对 prefill 负载的缩放 |
| `host/disk_cache_hit_weight` | 0.75 / 0.25 | 下层缓存命中折算权重 |
| `router_temperature` | 0.0 | >0 启用 softmax 采样 |
| `use_kv_events` | true | 用引擎 KV 事件；false=approximate TTL 模式 |
| `router_queue_threshold` | 4.0 | 队列容量阈值；None 关闭队列背压 |
| `router_event_threads` | 4 | >1 用并发 RadixTree 线程池 |
| `router_queue_policy` | fcfs | fcfs（尾部 TTFT）/ lcfs / wspt（平均 TTFT） |
| `router_replica_sync` | false | 多 router 活跃块状态同步 |

来源：`lib/kv-router/src/scheduling/config.rs` · 多数参数还可通过 `RouterConfigOverride` 逐请求覆盖。
