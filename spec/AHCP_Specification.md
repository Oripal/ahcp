# AHCP — Agent Heartbeat & Control Protocol Specification

**Version**: 1.0-rc1
**Date**: 2026-03-16
**Status**: Release Candidate
**Authors**: Hydree (VoborSpark)

---

## Abstract

AHCP (Agent Heartbeat & Control Protocol) defines a dual-layer heartbeat and control signal protocol for AI-native agent platforms. It provides runtime liveness detection, cancellation propagation, progress reporting, and crash recovery for distributed agent task execution.

AHCP sits between the communication layer (MCP, A2A, ACP) and the observability layer (OpenTelemetry), filling a gap that none of these protocols address.

## Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 1. Introduction

### 1.1 Problem Statement

AI agent platforms execute long-running, stateful, resource-intensive tasks across distributed workers. These tasks involve unpredictable LLM calls, sandboxed code execution, multi-agent coordination, and human-in-the-loop (HITL) approvals. The current protocol ecosystem lacks a standard for:

- Detecting whether a worker process or an individual task is alive
- Propagating cancellation from user to the deepest executing component
- Recovering from crashes with progress preservation
- Adapting timeout and heartbeat strategies to the diverse execution profiles of AI workloads

Existing protocols address adjacent concerns:

| Protocol | Scope | Heartbeat/Liveness |
|----------|-------|--------------------|
| MCP (Anthropic) | Agent ↔ Tool connectivity | Connection keepalive only |
| A2A (Google) | Agent ↔ Agent interoperability | Task states defined, no heartbeat spec |
| ACP (IBM) | Agent communication | Agent lifecycle states, no heartbeat protocol detail |
| OpenTelemetry | Observability | Trace/span tracking, not liveness detection |
| Temporal | Durable workflow execution | Comprehensive heartbeat, but not AI-native |

### 1.2 Scope

AHCP defines:

- Worker process registration, heartbeat, drain, and deregistration
- Job-level heartbeat with progress reporting and cancel signal propagation
- Job type policies governing heartbeat interval, timeout, and crash recovery
- AI-native extensions for LLM timeout classification, token budget, HITL coordination, and multi-agent topology

AHCP does NOT define:

- Job queue implementation or dispatch mechanism (out of scope; platform-specific). AHCP assumes the worker obtains `job_id` from an external queue mechanism.
- Job payload format or execution semantics (opaque to AHCP)
- Authentication and authorization protocols (delegated to transport layer; see §13)
- Observability data format (delegated to OpenTelemetry; see §11.3)

### 1.3 Design Principles

| # | Principle | Description |
|---|-----------|-------------|
| **P-1** | Dual-layer separation | Worker liveness and job health are independent concerns with independent protocols, independent timeouts, and independent failure handling |
| **P-2** | Heartbeat as control channel | Heartbeat responses carry control signals (cancel, drain, suspend). No separate control plane is REQUIRED for the common case |
| **P-3** | Cooperative cancellation | Cancellation is a request, not a kill. Workers and tasks MUST be given the opportunity to clean up gracefully |
| **P-4** | Policy over code | Heartbeat behavior (interval, timeout, crash recovery) is configured per job type, not hardcoded per worker |
| **P-5** | AI-native timeout taxonomy | LLM calls, tool executions, sandbox sessions, and HITL waits have fundamentally different timeout characteristics and SHOULD receive distinct treatment |
| **P-6** | Zero-overhead default | Jobs that do not need heartbeat MUST incur no heartbeat overhead. Heartbeat is opt-in per job type |
| **P-7** | Transport agnostic | The protocol defines messages and semantics. gRPC is the primary transport binding; HTTP/REST is an alternative binding |
| **P-8** | Unidirectional control | All RPCs flow from worker to scheduler. All scheduler-to-worker control signals are delivered via heartbeat responses. Workers never expose server endpoints |

### 1.4 Relationship to Other Protocols

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Agent Platform                         │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │   MCP    │  │   A2A    │  │   ACP    │  │ AGENTS.md │  │
│  │ Tool     │  │ Agent ↔  │  │ Agent    │  │ Behavior  │  │
│  │ Access   │  │ Agent    │  │ Comms    │  │ Guidance  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────────┘  │
│       │              │              │                        │
│       └──────────────┼──────────────┘                        │
│                      │                                       │
│              ┌───────┴───────┐                               │
│              │     AHCP      │  ← Liveness & Control Layer  │
│              │  Heartbeat &  │                               │
│              │   Control     │                               │
│              └───────┬───────┘                               │
│                      │                                       │
│              ┌───────┴───────┐                               │
│              │ OpenTelemetry │  ← Observability Layer        │
│              │  AI SemConv   │                               │
│              └───────────────┘                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Terminology

| Term | Definition |
|------|-----------|
| **Worker** | An OS process that consumes jobs from a queue and executes them. A worker MAY execute multiple jobs concurrently or sequentially. |
| **Job** | A unit of work with a defined lifecycle (see §5.1). A job runs on exactly one worker at a time. |
| **Scheduler** | The central component that manages job lifecycle, assigns jobs to workers, and monitors health via heartbeats. |
| **Worker Heartbeat** | A periodic signal from a worker process to the scheduler proving the process is alive. |
| **Job Heartbeat** | A periodic signal from a worker to the scheduler proving a specific job is making progress. |
| **Control Signal** | A directive from the scheduler to a worker, delivered via heartbeat response (e.g., cancel, drain, suspend). Per Principle P-8, all control signals flow through heartbeat responses. |
| **Job Type** | A named category that determines heartbeat policy, timeout, retry, and crash recovery behavior (see §6). |
| **Grace Period** | A configurable waiting interval after a worker death detection before rescheduling background jobs, to reduce false positives from transient issues. During the grace period, the assignment epoch is NOT incremented. |
| **Drain** | A graceful shutdown signal: the worker MUST stop accepting new jobs, finish current ones, then exit. |
| **Fencing Token** | The `assignment_epoch` value assigned to each job assignment, used to prevent split-brain duplicate execution (see §10.2). |

---

## 3. Architecture

### 3.1 Dual-Layer Model

AHCP defines two independent heartbeat layers:

```
┌─ Layer 1: Worker Heartbeat ─────────────────────────────────┐
│                                                              │
│  Question: "Is this worker process alive?"                   │
│  Scope: Per-worker (one heartbeat stream per process)        │
│  Failure impact: ALL jobs on this worker are affected         │
│                                                              │
│  Worker ──[WorkerHeartbeat]──→ Scheduler                     │
│         ←──{ack, drain_signal}──                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌─ Layer 2: Job Heartbeat ────────────────────────────────────┐
│                                                              │
│  Question: "Is this specific job making progress?"           │
│  Scope: Per-job (one heartbeat stream per active job)        │
│  Failure impact: Only THIS job is affected                   │
│                                                              │
│  Worker ──[JobHeartbeat(job_id, progress)]──→ Scheduler      │
│         ←──{ack, cancel, suspend, hitl_resolution}──         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

The two layers MUST operate independently. A Layer 2 timeout (job hang) MUST NOT cause a Layer 1 state change (worker death), and vice versa.

### 3.2 Failure Detection Matrix

| Scenario | Worker Heartbeat | Job Heartbeat | Detection |
|----------|:----------------:|:-------------:|-----------|
| Worker crashes (OOM, SIGKILL) | Stops | All stop | Layer 1 detects → batch recovery |
| Single job hangs (deadlock) | Continues | Stops for this job | Layer 2 detects → single job recovery |
| Worker alive, all jobs healthy | Continues | All continue | Normal operation |
| Network partition | Both delayed | Both delayed | Grace period before action |
| Background job (heartbeat_interval=0) | Continues | N/A | Layer 1 provides baseline |

### 3.3 Heartbeat as Control Channel

Per Principle P-8, all scheduler-to-worker control signals piggyback on heartbeat responses:

| Control Signal | Carried In | Field |
|---------------|------------|-------|
| Graceful shutdown | `WorkerHeartbeatResponse` | `should_drain` |
| Job cancellation | `JobHeartbeatResponse` | `should_cancel` |
| Job suspension | `JobHeartbeatResponse` | `should_suspend` |
| HITL resolution | `JobHeartbeatResponse` | `hitl_resolution` |
| Stale job notification | `WorkerHeartbeatResponse` | `stale_job_ids` |

For **urgent** control signals (e.g., user clicks cancel), platforms MAY implement an additional push channel (e.g., pub/sub). The heartbeat response then serves as a **reliable fallback** ensuring the control signal (e.g., cancel) is eventually delivered even if the push channel fails.

---

## 4. Worker Lifecycle Protocol

### 4.1 State Machine

```
REGISTERING → ACTIVE → DRAINING → DEREGISTERED
                ↓
              DEAD (detected by scheduler)
```

**Valid transitions:**

| From | To | Trigger |
|------|----|---------|
| `REGISTERING` | `ACTIVE` | Scheduler accepts `RegisterWorker`. Initial state is entered when the worker process starts and calls `RegisterWorker`. |
| `ACTIVE` | `DRAINING` | Scheduler sets `should_drain` to a value other than `DRAIN_REASON_NONE` in `WorkerHeartbeatResponse`; OR worker initiates graceful shutdown |
| `DRAINING` | `DEREGISTERED` | Worker completes all jobs and calls `DeregisterWorker` |
| `ACTIVE` | `DEAD` | Worker heartbeat timeout exceeded |
| `DRAINING` | `DEAD` | Worker heartbeat timeout exceeded during drain |

All other transitions are invalid. Implementations MUST reject them.

### 4.2 Registration

When a worker process starts, it MUST register with the scheduler before consuming any jobs.

**RegisterWorkerRequest fields:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `worker_id` | string | REQUIRED | Globally unique worker identifier. Implementations SHOULD use the format `{hostname}-{pid}-{uuid}`. |
| `capabilities` | string[] | REQUIRED | Job types this worker can execute. |
| `queues` | string[] | REQUIRED | Queue names this worker will consume from. |
| `max_concurrent_jobs` | int32 | OPTIONAL | Maximum concurrent jobs. Default: 1. |
| `metadata` | map | OPTIONAL | Key-value metadata (version, region, GPU, etc.). |
| `tenant_id` | string | OPTIONAL | Tenant isolation scope. See §13.6. |
| `protocol_version` | string | OPTIONAL | AHCP protocol version supported by this worker (e.g., "0.3"). |
| `conformance_level` | ConformanceLevel | OPTIONAL | AHCP conformance level: `CORE`, `STANDARD`, or `FULL`. See §15. Default: `CONFORMANCE_LEVEL_CORE`. |

**RegisterWorkerResponse fields:**

| Field | Type | Description |
|-------|------|-------------|
| `success` | bool | Whether registration was accepted. |
| `heartbeat_interval_ms` | int64 | Scheduler-assigned worker heartbeat interval. Worker MUST send heartbeats at this interval. |
| `worker_heartbeat_timeout_ms` | int64 | If no heartbeat is received within this duration, the scheduler MUST consider the worker dead. |
| `protocol_version` | string | AHCP protocol version supported by the scheduler. |
| `message` | string | Human-readable status message. |

If `success` is false, the worker MUST NOT consume jobs and SHOULD retry registration with exponential backoff.

The scheduler MUST reject `RegisterWorker` if a worker with the same `worker_id` is already in `ACTIVE` state.

### 4.3 Worker Heartbeat

Workers MUST send `WorkerHeartbeat` RPCs at the interval specified in `RegisterWorkerResponse.heartbeat_interval_ms`.

**WorkerHeartbeatRequest fields:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `worker_id` | string | REQUIRED | Worker identifier (same as registered). |
| `active_job_count` | int32 | REQUIRED | Number of currently executing jobs. |
| `cpu_usage_pct` | float | OPTIONAL | CPU utilization (0.0–1.0). |
| `memory_usage_pct` | float | OPTIONAL | Memory utilization (0.0–1.0). |
| `available_capacity` | int32 | OPTIONAL | Remaining job slots. |
| `status_metadata` | map | OPTIONAL | Extensible metadata (e.g., MCP server health, GPU status). |
| `trace_id` | string | OPTIONAL | OpenTelemetry trace ID for correlation. |

**WorkerHeartbeatResponse fields:**

| Field | Type | Description |
|-------|------|-------------|
| `success` | bool | Heartbeat acknowledged. If false, worker is unknown (e.g., declared DEAD during partition); worker MUST stop accepting new jobs and MAY complete in-flight jobs if within grace period. |
| `ack_deadline` | Timestamp | Absolute deadline for next heartbeat. |
| `next_heartbeat_within_ms` | int64 | Relative duration (milliseconds) within which the next heartbeat MUST arrive. Workers SHOULD use this value for scheduling, as it is immune to clock skew. |
| `should_drain` | DrainReason | If not `DRAIN_REASON_NONE`, worker MUST enter DRAINING state. |
| `stale_job_ids` | string[] | Job IDs that the scheduler considers timed out on this worker (informational). |

**DrainReason enum:**

| Value | Description |
|-------|-------------|
| `DRAIN_REASON_NONE` | No drain requested (default). |
| `DRAIN_REASON_ROLLING_UPDATE` | Rolling deployment in progress. |
| `DRAIN_REASON_SCALE_DOWN` | Cluster scale-down. |
| `DRAIN_REASON_MAINTENANCE` | Scheduled maintenance. |
| `DRAIN_REASON_RESOURCE_PRESSURE` | Resource limits approaching. |

Implementations MAY define additional values. Workers that encounter an unknown value MUST treat it as `DRAIN_REASON_MAINTENANCE` and proceed with drain.

**Non-normative note:** `DRAIN_REASON_NONE` uses "NONE" rather than "UNSPECIFIED" because absence of drain is a meaningful signal (no drain requested), not an unknown state.

**Enum zero-value behavior (applies to all AHCP enums):**

| Enum | Zero Value | Behavior When Received |
|------|-----------|----------------------|
| `JobState` | `JOB_STATE_UNSPECIFIED` | Implementation error; receiver SHOULD log a warning |
| `CancelReason` | `CANCEL_REASON_UNSPECIFIED` | Treat as `CANCEL_REASON_SYSTEM` |
| `DrainReason` | `DRAIN_REASON_NONE` | No drain requested (normal) |
| `RetryBackoff` | `RETRY_BACKOFF_UNSPECIFIED` | Treat as `RETRY_BACKOFF_EXPONENTIAL` |
| `CrashRecovery` | `CRASH_RECOVERY_UNSPECIFIED` | Treat as `CRASH_RECOVERY_IMMEDIATE` |
| `CompletionStatus` | `COMPLETION_STATUS_UNSPECIFIED` | Treat as `COMPLETION_STATUS_COMPLETE` |
| `HITLRiskLevel` | `HITL_RISK_LEVEL_UNSPECIFIED` | Workers MUST NOT send this; schedulers MUST treat as `HIGH` (conservative) |
| `HITLOutcome` | `HITL_OUTCOME_UNSPECIFIED` | Implementation error; worker SHOULD treat as `TIMEOUT` |
| `OperationCategory` | `OPERATION_CATEGORY_UNSPECIFIED` | No category declared; scheduler applies no category-specific logic |
| `ConformanceLevel` | `CONFORMANCE_LEVEL_UNSPECIFIED` | Treat as `CONFORMANCE_LEVEL_CORE` |

### 4.4 Deregistration

Before shutting down, workers SHOULD call `DeregisterWorker`. If a worker terminates without deregistering, the scheduler detects this via heartbeat timeout.

### 4.5 Worker Death Detection

When no `WorkerHeartbeat` is received before `ack_deadline`:

1. Scheduler MUST mark the worker state as `DEAD`.
2. Scheduler MUST identify all `RUNNING` jobs assigned to this worker.
3. For each job, the scheduler MUST apply the crash recovery strategy defined by the job's type (see §6.3).
4. Scheduler SHOULD emit a `worker.dead` observability event (see §11.3).

### 4.6 Worker Partition Behavior

If a worker fails to deliver N consecutive `WorkerHeartbeat` RPCs (where N is configurable; RECOMMENDED default is 3), it MUST assume it may be partitioned and MUST stop accepting new jobs from the queue. It MAY continue executing in-flight jobs, but MUST NOT start new ones. Upon successful heartbeat delivery, the worker MAY resume accepting jobs.

---

## 5. Job Lifecycle Protocol

### 5.1 State Machine

```
SUBMITTED → QUEUED → RUNNING → COMPLETED
                        │         ↗
                        ├→ FAILED
                        ├→ CANCELLED
                        ├→ SUSPENDED → RUNNING (resumed)
                        │      ├→ CANCELLED
                        │      └→ TIMEOUT
                        └→ TIMEOUT → QUEUED (retry) or DEAD_LETTER
```

`SUBMITTED` is a scheduler-internal state representing a job that has been accepted but not yet enqueued. It is not directly observable via AHCP RPCs.

**Valid transitions:**

| From | To | Trigger | Terminal? |
|------|----|---------|:---------:|
| `SUBMITTED` | `QUEUED` | Scheduler enqueues job | |
| `QUEUED` | `RUNNING` | Worker calls `StartJob` | |
| `RUNNING` | `COMPLETED` | Worker calls `CompleteJob` | Yes |
| `RUNNING` | `QUEUED` | Worker calls `FailJob` AND `retry_count < max_retries` | |
| `RUNNING` | `FAILED` | Worker calls `FailJob` AND `retry_count >= max_retries` | Yes |
| `RUNNING` | `CANCELLED` | Worker calls `AcknowledgeCancel` in response to `should_cancel` | Yes |
| `RUNNING` | `TIMEOUT` | Scheduler detects job heartbeat timeout OR execution timeout | |
| `RUNNING` | `SUSPENDED` | Worker calls `ReportSuspended` in response to `should_suspend` | |
| `SUSPENDED` | `RUNNING` | Scheduler delivers `hitl_resolution` via `JobHeartbeatResponse`; the scheduler MUST transition the job to `RUNNING` upon delivery, and the worker MUST resume execution upon receipt | |
| `SUSPENDED` | `CANCELLED` | User cancels during suspension; delivered via heartbeat `should_cancel` | Yes |
| `SUSPENDED` | `TIMEOUT` | Suspend timeout expires | |
| `TIMEOUT` | `QUEUED` | `retry_count < max_retries`; scheduler re-enqueues with incremented epoch | |
| `TIMEOUT` | `DEAD_LETTER` | `retry_count >= max_retries` | Yes |
| `QUEUED` | `CANCELLED` | Scheduler transitions to `CANCELLED` upon receiving a user cancellation request for a `QUEUED` job | Yes |
| `SUBMITTED` | `CANCELLED` | Scheduler transitions to `CANCELLED` upon receiving a user cancellation request for a `SUBMITTED` job | Yes |

All other transitions are invalid. Implementations MUST reject them.

The execution timeout clock MUST be paused when the scheduler transitions the job to `SUSPENDED` state (upon receiving `ReportSuspended`) and MUST be resumed when the scheduler transitions the job back to `RUNNING` state.

### 5.2 Job Start

When a worker picks up a job from a queue, it MUST call `StartJob` to acknowledge receipt.

**StartJobRequest fields:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `job_id` | string | REQUIRED | Job identifier. |
| `worker_id` | string | REQUIRED | Worker claiming this job. |
| `job_type` | string | OPTIONAL | Job type identifier (for logging; scheduler already knows). |
| `parent_job_id` | string | OPTIONAL | Parent job for cascading cancel (see §9.4). |
| `tenant_id` | string | OPTIONAL | Tenant scope. When present, MUST match the worker's `tenant_id`. |

**StartJobResponse fields:**

| Field | Type | Description |
|-------|------|-------------|
| `success` | bool | Whether the acknowledgment was accepted. |
| `message` | string | Human-readable status. |
| `heartbeat_interval_ms` | int64 | Job-level heartbeat interval. 0 = no heartbeat REQUIRED. |
| `job_heartbeat_timeout_ms` | int64 | Duration after which the scheduler considers the job stale. |
| `execution_timeout_ms` | int64 | Maximum total execution time for this job. |
| `assignment_epoch` | int64 | Fencing token (see §10.2). Worker MUST include this in all subsequent RPCs for this job. |

`StartJob` MUST be idempotent. If called for a job already in `RUNNING` state with the same `worker_id`, the scheduler MUST return success. If called by a different `worker_id` for a job already in `RUNNING` state, the scheduler MUST reject with `FAILED_PRECONDITION`. If called for a job in a terminal state, the scheduler MUST return success and log the duplicate.

### 5.3 Job Heartbeat

If `heartbeat_interval_ms > 0`, the worker MUST send `JobHeartbeat` RPCs at the specified interval while the job is running or suspended.

**JobHeartbeatRequest fields:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `job_id` | string | REQUIRED | Job identifier. |
| `worker_id` | string | REQUIRED | Worker executing this job. |
| `assignment_epoch` | int64 | REQUIRED | MUST match the epoch returned by `StartJob`. If stale, the scheduler MUST reject. |
| `progress` | JobProgress | OPTIONAL | Progress information (see §5.4). |
| `trace_id` | string | OPTIONAL | OpenTelemetry trace ID for correlation. |
| `span_id` | string | OPTIONAL | OpenTelemetry span ID for correlation. |

**JobHeartbeatResponse fields:**

| Field | Type | Description |
|-------|------|-------------|
| `success` | bool | Heartbeat acknowledged. If false, `assignment_epoch` is stale; worker MUST stop executing this job. |
| `ack_deadline` | Timestamp | Absolute deadline for next heartbeat. |
| `next_heartbeat_within_ms` | int64 | Relative duration for next heartbeat (immune to clock skew). |
| `should_cancel` | bool | If true, worker MUST begin graceful termination of this job and MUST call `AcknowledgeCancel` within a platform-defined deadline. If the worker does not acknowledge within the deadline, the scheduler MUST transition the job to `TIMEOUT`. |
| `cancel_reason` | CancelReason | Why the job is being cancelled. |
| `should_suspend` | bool | If true, worker MUST pause execution and MUST call `ReportSuspended`. |
| `suspend_reason` | string | Why the job is being suspended (e.g., `hitl_wait`, `resource_pressure`). |
| `hitl_resolution` | HITLResolution | If present, delivers the result of a pending HITL request. Worker SHOULD resume execution based on this resolution. |

**CancelReason enum:**

| Value | Description |
|-------|-------------|
| `CANCEL_REASON_UNSPECIFIED` | Default / unknown. |
| `CANCEL_REASON_USER` | User explicitly cancelled. |
| `CANCEL_REASON_TIMEOUT` | Execution timeout exceeded. |
| `CANCEL_REASON_BUDGET_EXCEEDED` | Token or cost budget exceeded. |
| `CANCEL_REASON_SYSTEM` | System-initiated (e.g., scheduler shutdown). |
| `CANCEL_REASON_SUPERSEDED` | Job superseded by a newer request. |

Implementations MAY extend this enum. Workers that encounter an unknown value MUST treat it as `CANCEL_REASON_SYSTEM`.

### 5.4 Job Progress

`JobProgress` is an OPTIONAL structured payload carried by `JobHeartbeat`. It enables frontend progress display and checkpoint-based crash recovery.

**Fields:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `percent` | int32 | OPTIONAL | Completion percentage (0–100). |
| `current_step` | string | OPTIONAL | Human-readable description of current step. |
| `steps_completed` | int32 | OPTIONAL | Number of completed steps. |
| `steps_total` | int32 | OPTIONAL | Total expected steps (0 = unknown, e.g., agent loops). |
| `checkpoint_payload` | bytes | OPTIONAL | Opaque checkpoint data. Scheduler MUST persist this. On retry, the scheduler MUST pass the latest checkpoint to the new worker via job payload. See §13.5 for integrity requirements. |
| `custom_metadata` | map | OPTIONAL | Application-specific key-value metadata. |
| `token_budget` | TokenBudgetStatus | OPTIONAL | AI extension: token consumption (see §9.2). |
| `hitl_state` | HITLState | OPTIONAL | AI extension: HITL interrupt state (see §9.3). |
| `current_operation` | OperationCategory | OPTIONAL | AI extension: current operation type (see §9.1). |

### 5.5 Job Completion

Workers MUST report job completion via one of three RPCs:

**CompleteJobRequest fields:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `job_id` | string | REQUIRED | Job identifier. |
| `worker_id` | string | REQUIRED | Worker that executed this job. |
| `assignment_epoch` | int64 | REQUIRED | MUST match the current epoch. |
| `result_payload` | bytes | OPTIONAL | Job result (opaque to AHCP). |
| `completion_status` | CompletionStatus | OPTIONAL | `COMPLETE` (default) or `PARTIAL`. `PARTIAL` indicates checkpoint-based partial completion; the scheduler SHOULD re-enqueue the job with the latest checkpoint so a new worker can continue. |

**CompleteJobResponse fields:**

| Field | Type | Description |
|-------|------|-------------|
| `success` | bool | Whether the completion was accepted. False if epoch is stale or worker_id mismatches. |
| `job_id` | string | Job identifier. |
| `state` | JobState | Final state confirmed by scheduler (e.g., `JOB_STATE_COMPLETED`). |

**FailJobRequest fields:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `job_id` | string | REQUIRED | Job identifier. |
| `worker_id` | string | REQUIRED | Worker that executed this job. |
| `assignment_epoch` | int64 | REQUIRED | MUST match the current epoch. |
| `error_code` | string | OPTIONAL | Application-defined error code. |
| `error_message` | string | OPTIONAL | Human-readable error description. MUST NOT contain credentials, secrets, or PII (see §13.7). |
| `retryable` | bool | OPTIONAL | Worker's advisory on retryability. The scheduler SHOULD consider this hint but MAY override it based on job type policy (e.g., `max_retries`). |
| `partial_result` | bytes | OPTIONAL | Partial results to preserve. |

**FailJobResponse fields:**

| Field | Type | Description |
|-------|------|-------------|
| `success` | bool | Whether the failure report was accepted. |
| `job_id` | string | Job identifier. |
| `state` | JobState | Resulting state: `JOB_STATE_QUEUED` (will retry), `JOB_STATE_FAILED` (retries exhausted), or `JOB_STATE_DEAD_LETTER` (max retries exceeded). |
| `retry_count` | int32 | Current retry count after this failure. |

**AcknowledgeCancelRequest fields:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `job_id` | string | REQUIRED | Job identifier. |
| `worker_id` | string | REQUIRED | Worker that executed this job. |
| `assignment_epoch` | int64 | REQUIRED | MUST match the current epoch. |
| `cancel_reason` | CancelReason | OPTIONAL | Echo back the reason received in `JobHeartbeatResponse`. |
| `partial_result` | bytes | OPTIONAL | Any partial results produced before cancellation. |

**AcknowledgeCancelResponse fields:**

| Field | Type | Description |
|-------|------|-------------|
| `success` | bool | Whether the acknowledgment was accepted. |
| `job_id` | string | Job identifier. |
| `state` | JobState | Terminal state: `JOB_STATE_CANCELLED`. |

All three RPCs MUST be idempotent. Duplicate calls with matching `assignment_epoch` MUST succeed without changing state. Calls with stale `assignment_epoch` MUST be rejected with `FAILED_PRECONDITION`.

A job in a terminal state (`COMPLETED`, `FAILED`, `CANCELLED`, `DEAD_LETTER`) MUST silently return success for any lifecycle RPC that arrives with a matching or stale `assignment_epoch`. The scheduler MUST NOT change the job's state.

### 5.6 Job Suspension

When a worker receives `should_suspend = true` in a `JobHeartbeatResponse`:

1. Worker MUST pause execution.
2. Worker MUST call `ReportSuspended` RPC to confirm suspension.
3. Worker MUST continue sending `JobHeartbeat` RPCs during suspension. During `SUSPENDED` state, these heartbeats serve exclusively as a control signal delivery channel (to receive cancel or HITL resolution signals). The scheduler MUST NOT treat missing heartbeats during suspension as a timeout event.
4. The scheduler MUST suspend the job heartbeat timeout while the job is in `SUSPENDED` state.
5. A separate suspend timeout MAY apply (e.g., HITL timeout from `HITLState.hitl_timeout_ms`).

When the scheduler delivers a `hitl_resolution` via `JobHeartbeatResponse`, the worker MUST resume execution based on the resolution outcome.

### 5.7 Job Heartbeat Timeout

When a job's heartbeat is not received before `ack_deadline` but the worker is still alive (worker heartbeat is current):

1. Scheduler MUST mark the job as `TIMEOUT`.
2. Scheduler MUST apply the job type's retry policy.
3. Scheduler SHOULD notify the worker via `stale_job_ids` in the next `WorkerHeartbeatResponse`. Upon receiving `stale_job_ids`, the worker SHOULD stop execution of the listed jobs and release their resources. The worker MAY call `FailJob` for these jobs; the scheduler has already transitioned them to `TIMEOUT` and MUST accept the RPC per §7.2.
4. If `retry_count >= max_retries`, the job MUST move to `DEAD_LETTER`.

A job in a terminal state (`COMPLETED`, `FAILED`, `CANCELLED`, `DEAD_LETTER`) MUST silently discard any subsequent `JobHeartbeat`. This is not an error.

---

## 6. Job Type Policy

### 6.1 Configuration Model

Each job type declares a heartbeat policy. The scheduler MUST use this policy to determine heartbeat requirements, timeout behavior, and crash recovery strategy.

**Policy fields:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `job_type` | string | REQUIRED | Unique job type identifier. |
| `heartbeat_interval_ms` | int64 | REQUIRED | Job heartbeat interval. 0 = no heartbeat REQUIRED. |
| `job_heartbeat_timeout_ms` | int64 | REQUIRED if interval > 0 | RECOMMENDED: 2–3x heartbeat interval. |
| `execution_timeout_ms` | int64 | REQUIRED | Maximum total execution time. |
| `max_retries` | int32 | REQUIRED | Maximum retry count before DEAD_LETTER. |
| `retry_backoff` | RetryBackoff | OPTIONAL | `LINEAR`, `EXPONENTIAL`, `FIXED`. Default: `EXPONENTIAL`. |
| `crash_recovery` | CrashRecovery | REQUIRED | `IMMEDIATE` or `GRACE_PERIOD`. |
| `grace_period_ms` | int64 | REQUIRED if GRACE_PERIOD | Wait duration before rescheduling. |
| `user_visible` | bool | OPTIONAL | Hint: is this job visible to end users? Default: false. |
| `cancellable` | bool | OPTIONAL | Whether cancel signals are supported. Default: true. When false, the scheduler MUST NOT set `should_cancel` in `JobHeartbeatResponse` and SHOULD reject user cancellation requests for this job type. |
| `progress_reporting` | bool | OPTIONAL | Whether workers SHOULD send progress in `JobHeartbeat`. Default: false. When false, workers MAY still send progress, but the scheduler is not required to surface it. |
| `max_checkpoint_size_bytes` | int64 | OPTIONAL | Maximum allowed checkpoint payload size. Default: platform-defined (RECOMMENDED: 1048576 / 1MB). |
| `max_child_depth` | int32 | OPTIONAL | Maximum parent-child nesting depth for multi-agent jobs. Default: 1 (flat parent-children only). When absent, the scheduler SHOULD treat as 1. |

**Non-normative note:** Job type policies are configured at the platform level. AHCP does not define a wire-level protocol for managing job type policies; the `StartJobResponse` conveys the relevant policy parameters for each specific job.

**Non-normative recommendation:** The following defaults are reasonable starting points for common AI workloads:

| Job Type | heartbeat_interval_ms | execution_timeout_ms | crash_recovery |
|----------|:---------------------:|:--------------------:|:--------------:|
| Interactive AI agent task | 5000–10000 | 600000 | IMMEDIATE |
| Document parsing | 30000–60000 | 900000 | IMMEDIATE |
| Background ETL | 0 | 300000 | GRACE_PERIOD |
| Notification delivery | 0 | 30000 | GRACE_PERIOD |

### 6.2 Heartbeat Interval = 0

When `heartbeat_interval_ms = 0`:

- The worker MUST NOT send `JobHeartbeat` RPCs for this job type.
- The scheduler MUST NOT expect job heartbeats.
- Job health is monitored ONLY via worker heartbeat (Layer 1) and execution timeout.
- This is the **zero-overhead default** (Principle P-6).

### 6.3 Crash Recovery Strategies

When a worker dies (detected via Layer 1 heartbeat timeout), the scheduler MUST handle each affected job according to its type's `crash_recovery` policy:

| Strategy | Behavior | Intended Use |
|----------|----------|--------------|
| `IMMEDIATE` | Re-enqueue affected jobs immediately with incremented `assignment_epoch`. | User-visible tasks: minimize wait time. |
| `GRACE_PERIOD` | Wait `grace_period_ms` before re-enqueuing. During this period, the `assignment_epoch` is NOT incremented. If the original worker recovers and calls `CompleteJob` with the original epoch, the scheduler MUST accept it. If the grace period expires without completion, the scheduler MUST increment the epoch and re-enqueue. | Background tasks: avoid false positives from transient network issues. |

---

## 7. Concurrency and Ordering

### 7.1 Concurrent RPCs

Multiple `JobHeartbeat` RPCs (for different jobs on the same worker) MAY be in-flight concurrently. The scheduler MUST handle them independently.

### 7.2 Terminal State Precedence

AHCP does not require strict ordering of RPCs. Instead, the following state-machine rule applies:

A job in a terminal state (`COMPLETED`, `FAILED`, `CANCELLED`, `DEAD_LETTER`) MUST silently discard any subsequent `JobHeartbeat`, `AcknowledgeCancel`, or `ReportSuspended` RPC. This is not an error. The scheduler MUST return success for discarded RPCs to maintain idempotency.

If `AcknowledgeCancel` arrives for a job in `TIMEOUT` state, the scheduler MUST accept it and transition to `CANCELLED` only if the `assignment_epoch` matches the current epoch and the job has not yet been re-enqueued with a new epoch.

### 7.3 Idempotency

All job lifecycle RPCs (`StartJob`, `CompleteJob`, `FailJob`, `AcknowledgeCancel`, `ReportSuspended`) MUST be idempotent when called with the same `assignment_epoch`. The scheduler MUST NOT change state on duplicate calls.

`WorkerHeartbeat` and `JobHeartbeat` are naturally idempotent: processing a duplicate heartbeat has no adverse effect. Implementations SHOULD persist checkpoint data from `JobHeartbeat` idempotently (e.g., last-write-wins keyed by `job_id`).

`RegisterWorker` with the same `worker_id` while the worker is in `ACTIVE` state MUST be rejected. After a worker transitions to `DEAD` or `DEREGISTERED`, re-registration with the same `worker_id` format is allowed (but a new UUID component is RECOMMENDED).

---

## 8. AI-Native Extensions: Overview

These extensions are REQUIRED for AHCP Full conformance (see §15) and OPTIONAL for AHCP Core and Standard.

---

## 9. AI-Native Extensions: Specification

### 9.1 Operation Category Taxonomy

AI agent tasks involve heterogeneous operations with fundamentally different timeout and heartbeat profiles. AHCP defines the following categories:

| Category | Typical Duration | Heartbeat | Cancel Propagation |
|----------|:----------------:|:---------:|:------------------:|
| `LLM_CALL` | 2s–60s | See note below | Not propagated (third-party API) |
| `TOOL_SHORT` | < 10s | None | Not propagated (not cost-effective) |
| `TOOL_LONG` | 10s–10min | Job heartbeat | Cooperative cancel (signal → grace → force) |
| `HITL_WAIT` | Seconds to hours | Continues (for control signal delivery) | Via `should_cancel` in heartbeat |
| `MULTI_AGENT` | Minutes to hours | Parent job heartbeat | Cascading cancel to descendants |
| `RATE_LIMITED` | Variable | Continues | Via `should_cancel` in heartbeat |

Workers SHOULD report their current operation via `JobProgress.current_operation`. The scheduler MAY use this information for adaptive timeout behavior. The specific adaptation logic is a platform concern, not mandated by AHCP.

**Non-normative note:** Workers executing streaming LLM calls SHOULD continue sending job heartbeats with updated `token_budget` as tokens arrive. The `LLM_CALL` category does not imply heartbeat suspension — heartbeat behavior is governed by the job type policy (§6), not by the current operation. Workers SHOULD report LLM provider health and rate-limit status in `WorkerHeartbeatRequest.status_metadata`.

### 9.2 Token Budget Association

AHCP carries token budget data as a transport mechanism. The decision to cancel based on budget is a platform concern.

`TokenBudgetStatus` is an OPTIONAL extension to `JobProgress`.

**Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `input_tokens_consumed` | int64 | Input tokens consumed so far. |
| `output_tokens_consumed` | int64 | Output tokens consumed so far. |
| `token_budget_limit` | int64 | Budget ceiling for this job. 0 = unlimited. |
| `model_id` | string | Identifier of the model that consumed these tokens. |

The scheduler MAY set `should_cancel = true` with `cancel_reason = CANCEL_REASON_BUDGET_EXCEEDED`, but this is not mandated by the protocol.

### 9.3 HITL Interrupt Coordination

When a job requires human approval:

1. Worker MUST send a `JobHeartbeat` with `hitl_state` populated and `current_operation = HITL_WAIT`.
2. Scheduler MUST set `should_suspend = true` in the next `JobHeartbeatResponse`.
3. Worker receives `should_suspend`, pauses execution, calls `ReportSuspended`.
4. Scheduler transitions job to `SUSPENDED` and suspends job heartbeat timeout.
5. Scheduler applies a separate HITL timeout (from `hitl_state.hitl_timeout_ms`).
6. Worker continues sending `JobHeartbeat` during suspension (to receive control signals).
7. When the user responds, the scheduler delivers `hitl_resolution` via `JobHeartbeatResponse`.
8. If the user cancels during HITL wait, the scheduler sets `should_cancel = true`.
9. If `hitl_timeout_ms` expires, the scheduler SHOULD transition the job to `TIMEOUT`.

**HITLState fields:**

| Field | Type | Description |
|-------|------|-------------|
| `hitl_request_id` | string | Unique identifier for this HITL request within the job. Enables multiple distinct HITL requests during a single job's lifecycle. |
| `hitl_action` | string | What approval is being requested. |
| `hitl_risk_level` | HITLRiskLevel | `LOW`, `MEDIUM`, `HIGH`. |
| `hitl_requested_at` | Timestamp | When HITL was requested. |
| `hitl_timeout_ms` | int64 | How long to wait for user response. |

**HITLResolution fields:**

| Field | Type | Description |
|-------|------|-------------|
| `hitl_request_id` | string | Correlates to `HITLState.hitl_request_id`. |
| `outcome` | HITLOutcome | `APPROVED`, `REJECTED`, or `TIMEOUT`. |
| `resolution_payload` | bytes | Approval data, rejection reason, or modified parameters. |

**HITLOutcome enum:**

| Value | Description |
|-------|-------------|
| `HITL_OUTCOME_UNSPECIFIED` | Default. |
| `HITL_OUTCOME_APPROVED` | Human approved the action. |
| `HITL_OUTCOME_REJECTED` | Human explicitly rejected the action. |
| `HITL_OUTCOME_TIMEOUT` | No response within `hitl_timeout_ms`. |

**Non-normative note:** Upon receiving `HITL_OUTCOME_REJECTED`, the worker's behavior is application-specific. Common patterns include:
- Fail the job with an application-defined error code
- Retry with a modified approach (e.g., alternative tool, lower-risk operation)
- Request a new HITL approval with adjusted parameters

Upon receiving `HITL_OUTCOME_TIMEOUT`, the worker MAY treat it as rejection or proceed with a default action, depending on the job type policy.

### 9.4 Multi-Agent Heartbeat Topology

AHCP defines heartbeat semantics for common multi-agent execution patterns:

**Solo**: Standard single-job heartbeat. No special semantics.

**Pipeline**: One job with progress tracking. `JobProgress.steps_completed` / `steps_total` tracks pipeline stage advancement.

**Team (DAG Parallel)**: Parent job and child jobs heartbeat independently. The scheduler MUST support a `parent_job_id` field in `StartJobRequest`. Cancellation MUST cascade transitively: when a parent job is cancelled, the scheduler MUST set `should_cancel` for all descendant jobs (children, grandchildren, etc.), not only direct children.

**Simulation**: Runs as a single long job. `JobProgress.current_step` reports tick/round advancement. In-process archetype agents do NOT heartbeat separately.

Job type policy SHOULD specify `max_child_depth`. The scheduler MUST reject `StartJob` with `parent_job_id` when the resulting depth exceeds this limit.

**Non-normative note:** A depth limit of 1 (flat parent-children, no grandchildren) is sufficient for most AI agent workloads including DAG-parallel team execution. Deeper nesting is needed only for recursive agent spawning scenarios and SHOULD be enabled with caution.

### 9.5 Sandbox Session Heartbeat

Long-running compute sandboxes (Jupyter kernels, browser sessions) exist below the AHCP layer. AHCP does not directly manage sandbox liveness, but platforms implementing AHCP Full SHOULD:

- For long-lived sandboxes (10min+): implement cooperative cancel via a platform-specific `CancelExecution` mechanism with signal → grace period → force-destroy semantics.
- For short-lived sandboxes (< 60s): rely on TTL-based expiry.

---

## 10. Failure Scenarios and Recovery

### 10.1 Failure Matrix

| Failure | Detection | Latency | Recovery |
|---------|-----------|---------|----------|
| Worker crash | Worker heartbeat timeout | 2–3 heartbeat cycles | Batch reschedule per job type policy |
| Job hang | Job heartbeat timeout | 2–3 heartbeat cycles | Single job retry or DEAD_LETTER |
| Network partition | Both heartbeats delayed | Grace period | Wait, then reschedule if persistent |
| Scheduler crash | External health check | Platform-specific | Restart scheduler; workers retry heartbeat |

### 10.2 Split-Brain Prevention

If a worker is falsely declared dead and its jobs rescheduled, the original worker may still be executing. AHCP prevents duplicate execution via **fencing tokens**:

1. **Assignment epoch**: Each `StartJobResponse` includes an `assignment_epoch`. The scheduler MUST generate epoch values. An `assignment_epoch` of 0 is invalid and MUST be rejected.

2. **Epoch validation**: Workers MUST include `assignment_epoch` in `JobHeartbeat`, `CompleteJob`, `FailJob`, `AcknowledgeCancel`, and `ReportSuspended`. The scheduler MUST reject RPCs where `assignment_epoch` does not match the current epoch. The scheduler MUST also reject RPCs where `assignment_epoch` matches but `worker_id` does not match the currently assigned worker.

3. **Epoch generation**: The protocol does not mandate a specific generation strategy. Implementations MAY use sequential integers (simple, debuggable), cryptographic random values (resistant to prediction), or any other strategy. The scheduler MUST ensure uniqueness per `(job_id, assignment)` pair.

4. **Worker self-check**: If `WorkerHeartbeatResponse.success` is false, the worker MUST stop accepting new jobs. It MAY complete in-flight jobs whose `assignment_epoch` has not been invalidated (grace period scenario).

### 10.3 Side Effect Safety

AHCP fencing tokens protect only the AHCP control plane. They do NOT prevent a zombie worker from executing external side effects (database writes, API calls, message sends) after its assignment has been revoked.

Preventing duplicate side effects is a platform responsibility. AHCP RECOMMENDS the following patterns:

- **Idempotency keys**: Derive idempotency keys from `(job_id, assignment_epoch)`. External systems that accept this key can deduplicate writes.
- **Conditional writes**: Use `assignment_epoch` as a compare-and-swap token when writing to external stores.
- **Side effect journaling**: Record side effects in the checkpoint payload so that retried executions can skip already-completed effects.

### 10.4 Scheduler High Availability

AHCP does not mandate a specific HA topology (active-standby, active-active, Raft-based). Implementations MUST ensure:

1. **Consistent state replication**: Scheduler instances MUST share a consistent view of worker registry (states, heartbeat deadlines), job assignments (`job_id` → `worker_id` → `assignment_epoch`), and epoch values.
2. **Serialized epoch generation**: Epoch generation MUST be serialized (e.g., via database sequence or consensus) to prevent duplicate epochs.
3. **Startup grace period**: On startup or failover, the scheduler MUST NOT declare any worker dead until at least one full `worker_heartbeat_timeout` has elapsed since the scheduler's startup or failover completion. This prevents mass false-positive death declarations.

---

## 11. Integration Points

### 11.1 With A2A (Agent-to-Agent Protocol)

| A2A State | AHCP Mechanism |
|-----------|---------------|
| `submitted` | Job enters queue (`SUBMITTED` → `QUEUED`) |
| `working` | Worker calls `StartJob`; job heartbeats begin |
| `input-required` | HITL interrupt; heartbeat carries `hitl_state` |
| `completed` | Worker calls `CompleteJob` |
| `failed` | Worker calls `FailJob` or heartbeat timeout → retry/DEAD_LETTER |
| `canceled` | `should_cancel` delivered via heartbeat response |

### 11.2 With MCP (Model Context Protocol)

- **MCP server health**: Workers MAY include MCP connection status in `WorkerHeartbeatRequest.status_metadata`.
- **Tool call timeout**: AHCP's operation category taxonomy (§9.1) applies to MCP tool invocations.

### 11.3 With OpenTelemetry

AHCP events SHOULD be emitted as OpenTelemetry signals:

| AHCP Event | OTel Signal |
|------------|------------|
| `WorkerHeartbeat` | Gauge: `ahcp.worker.active_jobs` |
| `JobHeartbeat` | Span event on job trace |
| Worker death detected | Counter: `ahcp.worker.deaths.total` |
| Job heartbeat timeout | Counter: `ahcp.job.timeouts.total` |
| Cancel signal delivered | Counter: `ahcp.job.cancellations.total` |
| Token budget update | Gauge: `ahcp.job.tokens.consumed` |

Heartbeat requests SHOULD carry `trace_id` and `span_id` for correlation with business traces. These SHOULD be encoded as lowercase hex strings (32 characters for trace ID, 16 characters for span ID) per the W3C Trace Context specification.

---

## 12. Transport Bindings

### 12.1 gRPC (Primary Binding)

See [proto/ahcp/v1/ahcp.proto](../../proto/ahcp/v1/ahcp.proto).

gRPC RPCs SHOULD set deadlines shorter than the heartbeat interval to ensure timely timeout detection.

### 12.2 HTTP/REST (Alternative Binding)

| gRPC RPC | HTTP Method | Path |
|----------|-------------|------|
| `RegisterWorker` | POST | `/ahcp/v1/workers` |
| `WorkerHeartbeat` | PUT | `/ahcp/v1/workers/{worker_id}/heartbeat` |
| `DeregisterWorker` | DELETE | `/ahcp/v1/workers/{worker_id}` |
| `StartJob` | POST | `/ahcp/v1/jobs/{job_id}/start` |
| `JobHeartbeat` | PUT | `/ahcp/v1/jobs/{job_id}/heartbeat` |
| `CompleteJob` | POST | `/ahcp/v1/jobs/{job_id}/complete` |
| `FailJob` | POST | `/ahcp/v1/jobs/{job_id}/fail` |
| `AcknowledgeCancel` | POST | `/ahcp/v1/jobs/{job_id}/acknowledge-cancel` |
| `ReportSuspended` | POST | `/ahcp/v1/jobs/{job_id}/report-suspended` |

Request and response bodies MUST be JSON-encoded using the proto3 JSON mapping (camelCase field names). Error responses MUST use the following structure:

```json
{
  "code": "FAILED_PRECONDITION",
  "message": "stale assignment_epoch: expected 3, got 2"
}
```

### 12.3 Urgent Control Channel (OPTIONAL)

For sub-second cancel delivery, platforms MAY implement a push channel (e.g., pub/sub, WebSocket) alongside heartbeat-based delivery. The heartbeat response serves as a reliable fallback.

### 12.4 Heartbeat Batching (Reserved)

For high-throughput workers executing many concurrent jobs, a `BatchJobHeartbeat` RPC is reserved for future specification. Implementations MAY implement this RPC ahead of formal specification, provided it maintains the same per-job semantics as individual `JobHeartbeat` calls.

---

## 13. Security Considerations

### 13.1 Transport Security

All AHCP communication MUST use TLS 1.2 or later. Plaintext communication MUST NOT be used in any deployment.

### 13.2 Authentication

All AHCP RPCs MUST be authenticated. Implementations MUST support at least one of:

- mTLS with X.509 client certificates
- Per-RPC bearer token authentication (OAuth 2.0 / JWT)
- Platform-specific authentication that binds to a verifiable identity

### 13.3 Worker Identity

Implementations MUST bind `worker_id` to the authenticated identity. The scheduler MUST reject any RPC where the `worker_id` does not match the identity presented in the authentication credential. A worker MUST NOT be able to send heartbeats or complete jobs on behalf of another worker.

The scheduler MUST reject `RegisterWorker` if a worker with the same `worker_id` is already in `ACTIVE` state.

### 13.4 Denial of Service

Schedulers MUST rate-limit heartbeat RPCs per worker. Heartbeats arriving faster than `heartbeat_interval_ms * 0.5` MUST be throttled.

Schedulers MUST limit the number of registered workers per authenticated identity.

Schedulers MUST enforce a maximum `max_concurrent_jobs` ceiling regardless of the value requested by the worker.

### 13.5 Checkpoint Payload Integrity

Implementations MUST treat `checkpoint_payload` as untrusted input. Deserialization SHOULD use safe parsers that reject malformed data.

Implementations SHOULD sign checkpoint payloads (e.g., HMAC with a per-job key) so the receiving worker can verify the checkpoint was produced by a legitimate prior execution.

Job type policy SHOULD specify `max_checkpoint_size_bytes`. The scheduler MUST reject `JobHeartbeat` where `checkpoint_payload` exceeds this limit. When no limit is configured, implementations SHOULD use a platform-defined default.

**Non-normative note:** Typical AI agent checkpoints (conversation history + tool results) range from 10KB to 1MB. Checkpoints exceeding 1MB SHOULD be stored via an external object store, with `checkpoint_payload` carrying only a reference (URI).

### 13.6 Multi-Tenancy

`tenant_id` is OPTIONAL. When present:

- The scheduler MUST ensure workers only receive jobs matching their `tenant_id`.
- The scheduler MUST reject `StartJob` if the job's `tenant_id` does not match the worker's `tenant_id`.
- Tenant identity SHOULD be verified against the authenticated credential, not solely trusted from the client-supplied field.

When absent, the scheduler MAY operate in single-tenant mode with no isolation enforcement.

### 13.7 Data Sanitization

Implementations MUST NOT include credentials, secrets, or personally identifiable information in `error_message`, `custom_metadata`, `status_metadata`, or `current_step` fields. Implementations SHOULD sanitize these fields before persisting or forwarding them.

### 13.8 Audit Logging

Implementations MUST log the following security-relevant events:

- Worker registration and deregistration
- Worker death detection
- Job cancellation signal delivery
- Fencing token rejection (stale epoch or worker_id mismatch)
- Authentication failures
- Rate limiting activations

---

## 14. Versioning and Evolution

### 14.1 Protocol Version

AHCP uses [Semantic Versioning 2.0.0](https://semver.org/). The current version is `0.3`.

During the 0.x development phase, any change may be breaking. The minor/major bump rules described below apply starting from version 1.0:

- **Minor version bump** (1.0 → 1.1): Adding OPTIONAL fields to existing messages, adding new RPCs, adding new enum values. Backward compatible.
- **Major version bump** (1.x → 2.0): Removing fields, changing field semantics, changing REQUIRED fields. Not backward compatible.

### 14.2 Version Negotiation

Both `RegisterWorkerRequest` and `RegisterWorkerResponse` include a `protocol_version` field, enabling bidirectional version negotiation.

**Compatibility rules:**
- Starting from version 1.0: if the worker's `protocol_version` major version differs from the scheduler's supported major version, the scheduler MUST reject registration with `FAILED_PRECONDITION`.
- During the 0.x development phase: the scheduler SHOULD reject registration if the worker's minor version is lower than the scheduler's minimum supported minor version.
- If the worker omits `protocol_version`, the scheduler SHOULD assume the oldest supported version.

### 14.3 Forward Compatibility

Implementations MUST ignore unknown fields in messages (standard protobuf behavior). Implementations MUST treat unknown enum values as the default/unspecified value.

---

## 15. Conformance Levels

| Level | Requirements |
|-------|-------------|
| **AHCP Core** | Worker lifecycle (§4) + Job lifecycle (§5.1–5.5) + Idempotency (§7.3) + Fencing tokens (§10.2) + TLS (§13.1) + Authentication (§13.2) |
| **AHCP Standard** | Core + Job heartbeat with cancel signal (§5.3) + Job suspension (§5.6) + Job type policies (§6) + Crash recovery (§6.3) + Concurrency rules (§7) + Worker identity (§13.3) |
| **AHCP Full** | Standard + AI-native extensions (§8–§9) + OTel integration (§11.3) + Full security (§13) + Audit logging (§13.8) |

Implementations MUST declare their conformance level via `RegisterWorkerRequest.conformance_level`.

---

## 16. Error Handling

### 16.1 gRPC Status Codes

| Scenario | gRPC Status | Retry? | Description |
|----------|-------------|:------:|-------------|
| Success | `OK` | — | Normal response. |
| Unknown worker_id | `NOT_FOUND` | Yes, after re-register | Worker not registered. Worker MUST re-register. |
| Unknown job_id | `NOT_FOUND` | No | Job does not exist. |
| Stale assignment_epoch | `FAILED_PRECONDITION` | No | Epoch mismatch. Worker MUST stop executing this job. |
| Worker_id mismatch on epoch | `FAILED_PRECONDITION` | No | Epoch matches but worker_id does not match current assignee. |
| Invalid state transition | `FAILED_PRECONDITION` | No | E.g., `CompleteJob` on an already-cancelled job. |
| Duplicate active worker_id | `ALREADY_EXISTS` | No | Worker with this ID is already ACTIVE. |
| Rate limited | `RESOURCE_EXHAUSTED` | Yes, with backoff | Heartbeat too frequent. Worker SHOULD back off. |
| Scheduler unavailable | `UNAVAILABLE` | Yes, with exponential backoff | Worker SHOULD retry. |
| Authentication failure | `UNAUTHENTICATED` | No | Invalid or missing credentials. |
| Permission denied | `PERMISSION_DENIED` | No | Worker not authorized for this operation. |
| Tenant mismatch | `PERMISSION_DENIED` | No | Job's tenant_id does not match worker's tenant_id. |

### 16.2 HTTP Status Codes

| gRPC Status | HTTP Status |
|-------------|:-----------:|
| `OK` | 200 |
| `NOT_FOUND` | 404 |
| `FAILED_PRECONDITION` | 409 |
| `ALREADY_EXISTS` | 409 |
| `RESOURCE_EXHAUSTED` | 429 |
| `UNAVAILABLE` | 503 |
| `UNAUTHENTICATED` | 401 |
| `PERMISSION_DENIED` | 403 |

---

## 17. Future Work

| Item | Description |
|------|-------------|
| **Heartbeat batching** | Formal specification of `BatchJobHeartbeat` RPC for high-throughput workers. |
| **Phi Accrual failure detection** | Probabilistic detection based on heartbeat arrival time distribution, replacing fixed thresholds. |
| **Cross-platform federation** | AHCP gateways bridging heartbeat domains across organizational boundaries. |
| **Formal verification** | TLA+ specification of the dual-layer protocol's safety and liveness properties. |
| **Conformance test suite** | Reference test suite for validating AHCP Core/Standard/Full implementations. |
| **Streaming heartbeat RPC** | Bidirectional streaming variant for reduced per-heartbeat overhead. |

---

## Appendix A: References

- [RFC 2119: Key words for use in RFCs](https://www.rfc-editor.org/rfc/rfc2119)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Temporal: Activity Heartbeat & Timeouts](https://temporal.io/blog/activity-timeouts)
- [Google A2A Protocol Specification](https://a2a-protocol.org/latest/specification/)
- [IBM Agent Communication Protocol](https://agentcommunicationprotocol.dev/)
- [Anthropic Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25)
- [OpenTelemetry AI Agent Observability](https://opentelemetry.io/blog/2025/ai-agent-observability/)
- [Martin Fowler: HeartBeat Pattern](https://martinfowler.com/articles/patterns-of-distributed-systems/heartbeat.html)
- [gRPC Cancellation Propagation](https://grpc.io/docs/guides/cancellation/)
- [Kubernetes Graceful Termination](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace)

## Appendix B: Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-15 | Initial draft. |
| 0.2 | 2026-03-15 | Added: Security (§12→§13), Versioning (§13→§14), Error handling (§15→§16), Concurrency (§7), RFC 2119 conventions. Formalized state machines. Added SuspendJob/ResumeJob. Added CancelReason enum. |
| 0.3 | 2026-03-16 | Expert review incorporation (28 items). Key changes: (1) Dual deadline fields — absolute Timestamp + relative `next_heartbeat_within_ms` for clock skew immunity. (2) Unidirectional control — all scheduler→worker signals via heartbeat responses (P-8); removed SuspendJob/ResumeJob RPCs, added `should_suspend` + `ReportSuspended` + `HITLResolution`. (3) Grace period epoch preservation — epoch not incremented during grace period. (4) Security hardened — TLS MUST, worker identity binding MUST, `tenant_id` OPTIONAL, checkpoint integrity, audit logging. (5) `CancelledJob` renamed to `AcknowledgeCancel`. (6) All string state/reason fields converted to enums (`JobState`, `DrainReason`, `CancelReason`, `HITLOutcome`). (7) Token budget split into `input_tokens_consumed` + `output_tokens_consumed` + `model_id`. (8) Job state machine completed — added RUNNING→QUEUED retry, SUSPENDED→CANCELLED, SUSPENDED→TIMEOUT. (9) Side effect safety guidance (§10.3). (10) Scheduler HA invariants (§10.4). |
| 1.0-rc1 | 2026-03-16 | Language precision review. Key changes: (1) `should_cancel` and `should_suspend` handling elevated from SHOULD to MUST with fallback behavior defined. (2) `should_drain` references corrected from bool to DrainReason enum. (3) `conformance_level` type corrected from string to ConformanceLevel enum. (4) Complete field tables added for CompleteJobRequest, FailJobRequest, AcknowledgeCancelRequest and their responses. (5) Enum zero-value behavior table added for all 10 enums. (6) SUSPENDED→RUNNING actor explicitly assigned (scheduler transitions, worker resumes). (7) Version incompatibility criteria defined. (8) Default values added for all OPTIONAL policy fields. (9) Worker behavior for `stale_job_ids` specified. (10) Execution timeout clock pause/resume semantics for SUSPENDED state clarified. |
