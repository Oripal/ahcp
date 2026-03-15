# AHCP v0.2 评审委员会汇总报告

**评审日期**: 2026-03-15
**评审对象**: AHCP Specification v0.2 + Proto Definition
**评审委员**: 5 位专家（协议设计 / 分布式系统 / AI/ML 平台 / 安全 / 开发者体验）

---

## 1. 共识意见（5/5 专家一致同意）

以下问题被**所有专家**独立提出：

| # | 问题 | 提出者 | 严重度 |
|---|------|--------|:------:|
| **U-1** | **Spec 与 Proto 的 `ack_deadline` 类型不一致**：Spec 写 `int64 unix_ms`，Proto 用 `google.protobuf.Timestamp` | 全部 5 人 | P0 |
| **U-2** | **SuspendJob/ResumeJob 调用方向矛盾**：Spec 说 Scheduler→Worker，Proto 放在 JobService（Worker→Scheduler） | 全部 5 人 | P0 |
| **U-3** | **SuspendJob/ResumeJob 缺少 `worker_id` 和 `assignment_epoch`** | 协议/分布式/AI/DX 4 人 | P0 |
| **U-4** | **`state` 字段用 string 而非 enum** | 协议/DX 2 人 | P1 |
| **U-5** | **`drain_reason` 是 string 而非 enum** | 协议/DX 2 人 | P1 |

---

## 2. 高频意见（3+ 专家提出）

| # | 问题 | 提出者 | 建议修复 |
|---|------|--------|---------|
| **H-1** | `CompleteJobResponse` 缺 `success` 字段 | 协议/AI/DX | 加 `success` bool，对齐其他 Response |
| **H-2** | 时钟偏移（clock skew）未考虑 | 协议/分布式/DX | 要求 NTP 或改用相对时间 `next_heartbeat_within_ms` |
| **H-3** | 缺少端到端 sequence diagram / 示例 | DX（强烈）/ AI / 协议 | 新增 Quick Start + 故障恢复 + HITL 三个 worked example |
| **H-4** | 心跳批量化（BatchJobHeartbeat）缺失 | 协议/分布式/DX | 至少在 Proto 中预留 RPC |
| **H-5** | fencing token（assignment_epoch）可预测性 | 安全/分布式 | 改为随机值或加 worker_id 校验 |

---

## 3. 按专家分类的独特意见

### 3.1 协议设计专家（独有）

| # | 问题 | 说明 |
|---|------|------|
| P-1 | **Job 状态机缺 `RUNNING→QUEUED` 重试转换** | FailJob 且未耗尽重试时的路径未定义 |
| P-2 | **`SUBMITTED` 状态无对应 RPC** | 无 SubmitJob RPC，应声明为内部状态或移除 |
| P-3 | **`DEAD_LETTER` 未出现在状态图中** | 终态遗漏 |
| P-4 | **Proto field 编号有跳跃（3→10, 6→15）** | 应添加 `reserved` 声明或注释说明编号规约 |
| P-5 | **HTTP error response body 格式未定义** | 需定义标准错误响应结构 |
| P-6 | **版本协商单向**：只有 Response 有 `protocol_version`，Request 没有 | 应双向，实现互操作 |
| P-7 | **Conformance level 无 wire-level 协商机制** | Worker 应在注册时声明支持的一致性等级 |
| P-8 | **Semver 在 0.x 阶段误用 minor/major 规则** | 0.x 无稳定性保证，当前规则应标注"1.0 起适用" |
| P-9 | **`budget_utilization_pct` 冗余** | 可由 consumed/limit 推导，携带冗余数据有一致性风险 |
| P-10 | **`OperationCategory` enum 定义了但 Spec 无行为关联** | 纯 metadata 无 scheduler 动作，需明确其作用 |

### 3.2 分布式系统专家（独有）

| # | 问题 | 说明 |
|---|------|------|
| D-1 | **Fencing token 不保护外部 side effect** | Worker A 被判死但已写入数据库，Worker B 重复写入。需 side effect safety 指导 |
| D-2 | **Grace period 恢复路径矛盾** | Worker 被判 DEAD 后如何 CompleteJob？需定义 grace period 期间 epoch 保持不变 |
| D-3 | **§7.2 CompleteJob-before-JobHeartbeat 排序规则不可实现** | 分布式系统中"同时到达"无意义，改为状态机规则（终态优先） |
| D-4 | **CancelledJob-after-TIMEOUT 语义有歧义** | Job 已进入重试队列后再收到 CancelledJob 的处理未定义 |
| D-5 | **Scheduler HA 缺关键不变量** | 未列举哪些状态必须一致复制，未定义启动时死亡检测宽限期 |
| D-6 | **网络分区时 Worker 行为未定义** | 应规定连续 N 次心跳失败后 Worker 停止接新 Job |
| D-7 | **epoch=0（Proto 默认值）未显式拒绝** | 应声明 epoch=0 非法 |

### 3.3 AI/ML 平台专家（独有）

| # | 问题 | 说明 |
|---|------|------|
| A-1 | **LLM streaming 超时分类错误** | `LLM_CALL` 标记为"无心跳"，但 streaming 调用应继续心跳并报告 token 消耗 |
| A-2 | **TokenBudgetStatus 缺 input/output token 拆分** | 不同模型 input/output 单价差 3-5 倍，单一 `tokens_consumed` 不够 |
| A-3 | **TokenBudgetStatus 缺 `model_id`** | 不知道哪个模型消耗的 token，预算数据可用性大降 |
| A-4 | **HITL 缺 `hitl_request_id`** | 单个 Job 多次 HITL 请求时无法区分哪个被批准 |
| A-5 | **HITL 缺拒绝语义** | 只有 approve(resume) 和 timeout，人工 reject 的处理未定义 |
| A-6 | **多 Agent 取消级联未要求传递闭包** | 未显式要求孙级 Job 也被取消 |
| A-7 | **缺少 Agent 循环迭代追踪** | ReAct loop 的 `steps_total` 未知，需 `loop_iteration` 字段或指导 |
| A-8 | **FailJobRequest 缺 `partial_result`** | CancelledJob 有但 FailJob 没有，失败任务的部分结果丢失 |
| A-9 | **缺少 LLM provider 限流/宕机状态** | 429 Rate Limit 不是 fail 也不是 timeout，是独立状态 |
| A-10 | **checkpoint_payload 无大小指导** | Agent 状态可能很大，需建议上限 |
| A-11 | **suspend 时间是否算入 execution_timeout 未说明** | HITL 30min 等待不应消耗 5min 执行预算 |

### 3.4 安全专家（独有）

| # | 问题 | 说明 |
|---|------|------|
| S-1 | **§12 体量不足**（当前 ~15 行规范文本） | 需扩展 3 倍：增加威胁模型、信任边界、传输加密要求 |
| S-2 | **TLS 应为 MUST 而非 RECOMMEND** | 无 TLS 的 AHCP 无法防中间人篡改 cancel 信号 |
| S-3 | **worker_id 绑定认证身份应为 MUST 而非 SHOULD** | SHOULD 允许不绑定，导致 worker 可伪造他人身份 |
| S-4 | **缺少多租户隔离要求** | 无 `tenant_id`，企业部署中跨租户可见性无防护 |
| S-5 | **Checkpoint payload 投毒攻击** | 恶意 Worker 写入有毒 checkpoint，新 Worker 反序列化时触发 RCE |
| S-6 | **缺少数据机密性要求** | checkpoint_payload/result_payload 可能含 PII/对话记录，静态加密未要求 |
| S-7 | **缺少审计日志要求** | 安全关键事件（cancel、worker death、epoch reject）应有审计记录 |
| S-8 | **自由文本字段（error_message 等）的数据泄露风险** | 应禁止在 metadata/error 中放凭证和 PII |
| S-9 | **Heartbeat 回放攻击** | 无 nonce/sequence number，捕获的心跳响应可回放 |
| S-10 | **DoS 防护措施从 SHOULD 升为 MUST** | 注册限速、心跳限速应为强制 |

### 3.5 开发者体验专家（独有）

| # | 问题 | 说明 |
|---|------|------|
| X-1 | **`CancelledJob` RPC 命名不一致** | 其他 RPC 都是动词（StartJob/CompleteJob/FailJob），这个是过去分词。建议改为 `AcknowledgeCancel` |
| X-2 | **`success` 字段语义重载** | Worker 心跳的 success=false 是"重新注册"，Job 心跳的 success=false 是"epoch 过期"，同名不同义 |
| X-3 | **§6 JobTypePolicy 无对应 Proto message** | 配置模型只在 Spec 中，Proto 无法体现 |
| X-4 | **缺少 SDK 线程模型指导** | 开发者需知道 Worker 进程内的心跳并发模式 |
| X-5 | **无 streaming heartbeat RPC 选项** | 高并发场景下 unary heartbeat 开销大，应提供 stream 变体 |
| X-6 | **错误处理表缺 "是否重试" 列** | 开发者需对每个 status code 判断重试策略 |
| X-7 | **HTTP binding 未定义请求/响应 body 格式** | JSON 字段命名（camelCase vs snake_case）未规定 |
| X-8 | **未说明 AHCP 对外部队列的假设** | "Worker 从队列拉取 Job" 但队列协议 out of scope 需显式声明 |
| X-9 | **缺少推荐的默认心跳间隔值** | 应给出"用户可见 AI 任务 3-5s，后台 30s"等参考值 |

---

## 4. 修复优先级总表

### P0（阻断实现）

| # | 问题 | 来源 |
|---|------|------|
| 1 | Spec/Proto `ack_deadline` 类型对齐 | U-1 |
| 2 | SuspendJob/ResumeJob 方向重设计 | U-2 |
| 3 | SuspendJob/ResumeJob 加 worker_id + epoch | U-3 |
| 4 | Grace period 恢复路径定义 | D-2 |
| 5 | 安全章节扩展：TLS MUST、身份绑定 MUST、多租户 | S-1/2/3/4 |

### P1（正确性）

| # | 问题 | 来源 |
|---|------|------|
| 6 | `state` / `drain_reason` 改为 enum | U-4/5 |
| 7 | Job 状态机补全（RUNNING→QUEUED 重试、SUSPENDED→CANCELLED） | P-1, D-4 |
| 8 | Fencing token 安全加固（随机化或 worker_id 校验） | H-5 |
| 9 | CompleteJobResponse 加 success | H-1 |
| 10 | 终态优先规则替代排序规则 | D-3 |
| 11 | Token budget: input/output 拆分 + model_id | A-2/3 |
| 12 | HITL: hitl_request_id + 拒绝语义 | A-4/5 |
| 13 | Streaming LLM 心跳指导修正 | A-1 |
| 14 | Side effect safety 指导 | D-1 |
| 15 | Scheduler HA 不变量 + 启动宽限期 | D-5 |
| 16 | Checkpoint payload 完整性（签名/HMAC）+ 大小指导 | S-5, A-10 |

### P2（完善性）

| # | 问题 | 来源 |
|---|------|------|
| 17 | 新增 Quick Start + 3 个 worked example | H-3 |
| 18 | 时钟偏移处理 | H-2 |
| 19 | 心跳批量化 RPC 预留 | H-4 |
| 20 | 版本协商双向化 + conformance level 声明 | P-6/7 |
| 21 | CancelledJob 重命名 | X-1 |
| 22 | 错误表加"是否重试"列 | X-6 |
| 23 | FailJobRequest 加 partial_result | A-8 |
| 24 | 网络分区 Worker 行为规定 | D-6 |
| 25 | 审计日志要求 | S-7 |
| 26 | 数据泄露防护（自由文本字段） | S-8 |
| 27 | suspend 时间与 execution_timeout 的关系 | A-11 |
| 28 | 多 Agent 取消传递闭包 + 深度限制 | A-6 |

---

## 5. 专家总评

| 专家 | 总评 |
|------|------|
| **协议设计** | "协议设计根基扎实。双层分离有充分动机，fencing token 正确，AI 扩展范围合理。问题集中在边缘场景和 spec/proto 对齐。预计 v0.3 修正后可进入实现者反馈阶段。" |
| **分布式系统** | "双层模型和整体架构很强。AI 原生扩展构思好，填补了真实空白。主要弱点在崩溃恢复边缘场景、Scheduler HA、以及 spec/proto 不一致。" |
| **AI/ML 平台** | "核心心跳/fencing/取消机制达到生产级。主要差距在 AI 原生扩展——方向正确，但需更多深度才能应对多模型、多 Agent、streaming LLM 的真实运维复杂度。" |
| **安全** | "协议骨架安全（fencing、协作取消、双层分离），但安全章节假设了可信网络和可信 Worker，这不是安全的假设。§12 需要扩展约 3 倍。" |
| **开发者体验** | "协议设计水平高于同类 v0.2 草案。差距不在思考而在呈现——没有示例、没有 Quick Start、spec/proto 不一致。修复 10 项建议后，这将成为 AI Agent 生态中最好的协议规范之一。" |

---

## 6. 逐项讨论决议（2026-03-16）

### P0（5 项，全部确认）

| # | 决议 |
|---|------|
| P0-1 | **方案 C**：同时提供 `ack_deadline`(Timestamp) + `next_heartbeat_within_ms`(int64 相对时间)。Worker 用相对时间执行等待，消除时钟偏移 |
| P0-2 | **方案 B（心跳搭载）**：删除 SuspendJob/ResumeJob RPC。控制信号统一走心跳响应（`should_suspend` + `HITLResolution`）。Worker 确认挂起发 `ReportSuspended` RPC。所有 Scheduler→Worker 控制信号走同一通道（cancel/drain/suspend），Worker 永远是 caller |
| P0-3 | **随 P0-2 解决**：心跳请求天然携带 worker_id + assignment_epoch |
| P0-4 | **Grace period 期间 epoch 不递增**。Worker 被判 DEAD 后，在 grace_period_ms 内保持原 epoch。CompleteJob(原 epoch) 可被接受。过期后递增 epoch 重新入队 |
| P0-5 | **安全章节三项升级**：TLS 1.2+ MUST；worker_id 绑定认证身份 MUST；`tenant_id` 加为 OPTIONAL 字段（present 时 MUST 隔离，absent 时单租户模式） |

### P1（11 项，全部确认）

| # | 决议 |
|---|------|
| P1-6 | 新增 `JobState` enum + `DrainReason` enum，替代 string |
| P1-7 | 状态机补全：`RUNNING→QUEUED`(重试)、`SUSPENDED→CANCELLED`、`SUSPENDED→TIMEOUT`。`SUBMITTED` 标注为 Scheduler 内部状态。`DEAD_LETTER` 补入状态图 |
| P1-8 | **顺序 epoch + worker_id 双校验**。协议定义约束（唯一性、epoch=0 非法、worker_id 必须匹配），不规定生成策略（顺序/随机由实现决定） |
| P1-9 | `CompleteJobResponse` + `CancelledJobResponse`（现改为 `AcknowledgeCancelResponse`）补 `success` 字段 |
| P1-10 | 删除"先处理 CompleteJob"排序规则，改为状态机规则：终态 Job 的 heartbeat 静默丢弃，非错误 |
| P1-11 | `TokenBudgetStatus` 改为 `input_tokens_consumed` + `output_tokens_consumed` + `model_id`，删除冗余的 `budget_utilization_pct` |
| P1-12 | HITL 加 `hitl_request_id` + `HITLResolution`(含 `HITLOutcome` enum: APPROVED/REJECTED/TIMEOUT)。Worker 对 REJECTED 的处理为业务逻辑，Spec 加非规范性提示 |
| P1-13 | Streaming LLM 心跳指导修正：不改 enum，加非规范性文本说明 streaming 调用 SHOULD 继续心跳 |
| P1-14 | 新增 §9.4 Side Effect Safety：声明 fencing token 只保护控制面，RECOMMEND idempotency key / CAS / journaling 模式 |
| P1-15 | §9.3 Scheduler HA 补充：列举必须一致复制的状态、epoch 生成串行化、启动宽限期（≥ 1 个 heartbeat timeout），不规定 HA 拓扑 |
| P1-16 | Checkpoint payload：完整性用 HMAC SHOULD（非 MUST），大小由 job type policy 的 `max_checkpoint_size_bytes` 控制，协议给非规范性参考值 |

### P2（12 项，全部确认）

| # | 决议 |
|---|------|
| P2-17 | 新增 Appendix C：Quick Start sequence diagram + 故障恢复 + HITL 三个 worked example |
| P2-18 | 随 P0-1 方案 C 解决 |
| P2-19 | Proto 预留 `BatchJobHeartbeat` RPC，标注 OPTIONAL / Reserved for future use |
| P2-20 | `RegisterWorkerRequest` 加 `protocol_version` + `conformance_level` 字段，双向协商 |
| P2-21 | `CancelledJob` 重命名为 `AcknowledgeCancel`。语义更精确：确认收到并执行了取消指令 |
| P2-22 | §15 错误表新增 Retry 列（是/否/条件重试） |
| P2-23 | `FailJobRequest` 加 `bytes partial_result`，与 AcknowledgeCancel 对称 |
| P2-24 | 新增规范性文本：连续 N 次心跳失败后 Worker MUST 停止接新 Job（N 可配置，RECOMMENDED default 3） |
| P2-25 | §12 新增 §12.5 Audit Logging，列举 MUST 记录的安全关键事件 |
| P2-26 | §12 加规范性文本：MUST NOT 在 error_message / metadata / current_step 中放凭证和 PII |
| P2-27 | 规范性文本："Time spent in SUSPENDED state MUST NOT count against `execution_timeout_ms`" |
| P2-28 | 取消级联 MUST 传递闭包（transitive）。Job type policy SHOULD 指定 `max_child_depth`，Scheduler MUST 拒绝超深嵌套。非规范性指导：depth=1 适合大多数场景 |
