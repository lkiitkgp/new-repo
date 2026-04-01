## POC Ticket: Hybrid Worker Pod Scaling — Full Scope

---

### Title

**[POC] Hybrid Worker Pod Scaling: KEDA Composite Triggers + Resource-Aware Back-Pressure**

---

### Description

**Background:**
Worker pods currently scale via KEDA using only Redis Stream `pendingEntriesCount`. This creates two compounding failure modes: KEDA does not scale up when pods are CPU/memory-exhausted (queue looks manageable), and individual overloaded pods continue pulling messages they cannot process efficiently. The result is blocked requests, cascading latency, and wasted scale-up events — as documented in hybrid-worker-scaling-architecture.md.

**Goal:**
Implement and validate the full hybrid scaling architecture end-to-end across both the infrastructure layer (KEDA) and the application layer (resource-aware back-pressure), including all six complementary mechanisms described in the plan.

**What will be built:**

**1. KEDA Composite Triggers (Infrastructure Layer)**
Extend the existing KEDA `ScaledObject` in deployment manifest  with two additional triggers alongside the existing Redis Streams trigger: a `cpu` trigger (threshold: 75%) and a `memory` trigger (threshold: 80%). KEDA will use the MAX replica count across all three triggers, ensuring scale-up occurs when either queue depth grows or pods are resource-exhausted.

**2. Resource Monitor — Unified OS-Thread Monitor (`utils/resource_monitor.py`)**
A new daemon OS thread that samples CPU and memory from `/proc` on a configurable interval (`RESOURCE_MONITOR_INTERVAL_SECONDS`, default 10s). This is the sole source of resource truth for the worker process. It classifies the pod into a zone (`HEALTHY` / `PRESSURE` / `CRITICAL`) using configurable thresholds, maintains an atomic `resource_critical` flag, and exposes `resource_adjusted_max_slots` via a thread-safe data structure. A dedicated OS thread is used (not an async task) because CPU-intensive agent executions can block the Python event loop for seconds.

**3. Adaptive Slot Calculator**
Enhance [`_calculate_available_slots()`](utils/messaging/redis_streams_trigger_handler.py:1048) to apply:
```
available_slots = min(
    batch_size - active_task_count,   # existing task-based limit
    resource_adjusted_max_slots,       # from Resource Monitor
    slow_start_max_slots               # from Slow Start controller
)
```
When `RESOURCE_MONITOR_ENABLED=False`, behaviour is identical to the current implementation.

**4. Recovery with Hysteresis**
Prevent oscillation between zones using a 10% hysteresis band:
- Enter PRESSURE: CPU > 75% or Memory > 80%
- Exit PRESSURE: CPU < 65% or Memory < 70%
- Enter CRITICAL: CPU > 90% or Memory > 90%
- Exit CRITICAL: CPU < 80% or Memory < 80%

**5. Slow Start — Cold-Start Burst Protection**
New pods begin with `slow_start_slots = 1` (configurable via `SLOW_START_INITIAL_SLOTS`) and increment by `SLOW_START_INCREMENT` after each successful execution completion, ramping up to `batch_size`. If resource pressure is detected during ramp-up, the slot count is frozen. If CRITICAL pressure is detected, slots are overridden to 0 and slow start resets on recovery.

**6. Pre-Execution Headroom Gate (Layer A)**
Before each agent execution starts inside [`_run_agent_execution_with_setup()`](utils/messaging/redis_streams_trigger_handler.py:329), perform a synchronous resource check. If in CRITICAL zone, NACK the message back to the Redis Stream and publish a user-facing warning event. If in PRESSURE zone, proceed but log a warning.

**7. Unified Monitoring Thread as Safety Net (Layer B)**
The same OS thread from item 2 maintains the atomic `resource_critical` flag. If the pod remains in CRITICAL zone for longer than `WATCHDOG_GRACE_PERIOD_SECONDS` (default 15s), the flag is set and no new executions are started. The event loop checks this flag before starting each execution. The thread does NOT cancel running executions.

**8. Kubernetes OOM-Kill as Final Safety Net (Layer C)**
No new code required. The existing [`_pending_messages_monitor()`](utils/messaging/redis_streams_trigger_handler.py:804) already detects orphaned messages after pod restart and publishes a `"failed"` event via [`_publish_delete_message_event()`](utils/messaging/redis_streams_trigger_handler.py:542). The OOM error code `WORKER_ERR_OOM` is added to this existing event payload.

**9. User Notification via Event API**
Publish user-facing warning events via the existing [`enqueue_payload()`](utils/fifo_queue_handler.py:18) → [`process_and_notify_event()`](utils/events.py:50) → [`stream_event()`](utils/events.py:70) pipeline for the following conditions:
- Pre-start rejection (CRITICAL zone): `"warning"` — *"Agent execution temporarily delayed due to high system load. Your request will be retried automatically."*
- Worker enters PRESSURE zone (once per transition): `"warning"` — *"Agent execution may experience slower response times due to current system load. Processing will continue but may take longer than usual."*
- Worker enters CRITICAL zone (once per transition): `"warning"` — *"Agent execution is experiencing high load. Your request may be delayed if conditions do not improve."*
- OOM-kill recovery: already handled by existing `_pending_messages_monitor` — no new code needed.

All events include a machine-readable `errorCode` field (e.g., `WORKER_WARN_HIGH_LOAD`, `WORKER_WARN_SLOW_RESPONSE`, `WORKER_WARN_REQUEUED`) alongside the user-facing message. Event messages must not reference internal class names, queue operations, or implementation details.

**10. Configuration**
Add all new settings to [`config/settings/infra.py`](config/settings/infra.py) as fields on `InfraSettingsMixin`, readable from environment variables at startup and overridable at runtime via the existing [`ConfigManager`](config/config_service.py:13):

| Setting | Default |
|---|---|
| `RESOURCE_MONITOR_ENABLED` | `True` |
| `RESOURCE_MONITOR_INTERVAL_SECONDS` | `10` |
| `CPU_PRESSURE_THRESHOLD_PERCENT` | `75` |
| `CPU_CRITICAL_THRESHOLD_PERCENT` | `90` |
| `MEMORY_PRESSURE_THRESHOLD_PERCENT` | `80` |
| `MEMORY_CRITICAL_THRESHOLD_PERCENT` | `90` |
| `RESOURCE_PRESSURE_SLOT_REDUCTION_FACTOR` | `0.5` |
| `RESOURCE_HYSTERESIS_BAND_PERCENT` | `10` |
| `SLOW_START_ENABLED` | `True` |
| `SLOW_START_INITIAL_SLOTS` | `1` |
| `SLOW_START_INCREMENT` | `1` |
| `PRE_EXECUTION_RESOURCE_CHECK_ENABLED` | `True` |
| `WATCHDOG_GRACE_PERIOD_SECONDS` | `15` |
| `RESOURCE_PRESSURE_EVENTS_ENABLED` | `True` |

**Files to modify/create:**

| File | Change |
|---|---|
| [`config/settings/infra.py`](config/settings/infra.py) | Add all new settings |
| [`config/settings/__init__.py`](config/settings/__init__.py) | Include new settings in settings class |
| `utils/resource_monitor.py` | **New** — unified OS-thread resource monitor |
| [`utils/messaging/redis_streams_trigger_handler.py`](utils/messaging/redis_streams_trigger_handler.py) | Adaptive slot calculator, headroom gate, event publishing |
| [`utils/api-graph-depolyment-sample.yaml`](utils/api-graph-depolyment-sample.yaml) | Add CPU + memory KEDA triggers |
| [`src/worker_main.py`](src/worker_main.py) | Start resource monitor thread on boot |

---

### Acceptance Criteria

**Infrastructure Layer**
- [ ] Deployment manifest contains three KEDA triggers: existing `redis-streams` (unchanged), new `cpu` trigger at 75% utilization, and new `memory` trigger at 80% utilization.

**Resource Monitor**
- [ ] `utils/resource_monitor.py` exists and starts as a daemon OS thread from [`src/worker_main.py`](src/worker_main.py) when `RESOURCE_MONITOR_ENABLED=True`; thread does not start when `False`.
- [ ] Monitor reads CPU from `/proc/stat` and memory from `/proc/meminfo` (or `/proc/self/status`) — no external dependencies.
- [ ] Monitor classifies zone as `HEALTHY`, `PRESSURE`, or `CRITICAL` using configurable thresholds with hysteresis band applied on recovery.
- [ ] Zone and `resource_adjusted_max_slots` are exposed via a thread-safe data structure readable from the async event loop.
- [ ] Atomic `resource_critical` flag is set when pod remains in CRITICAL zone for longer than `WATCHDOG_GRACE_PERIOD_SECONDS`; cleared when pod recovers below exit-critical threshold.
- [ ] CPU readings use a rolling average over the last 3 samples (not instantaneous) to prevent false positives from transient spikes.

**Adaptive Slot Calculator**
- [ ] [`_calculate_available_slots()`](utils/messaging/redis_streams_trigger_handler.py:1048) returns `min(batch_size - active_task_count, resource_adjusted_max_slots, slow_start_max_slots)` when both `RESOURCE_MONITOR_ENABLED=True` and `SLOW_START_ENABLED=True`.
- [ ] When `RESOURCE_MONITOR_ENABLED=False`, slot calculation is identical to the current implementation with no behavioural change.

**Hysteresis**
- [ ] Zone transitions use hysteresis: PRESSURE entered at CPU > 75% / Memory > 80%, exited at CPU < 65% / Memory < 70%; CRITICAL entered at CPU > 90% / Memory > 90%, exited at CPU < 80% / Memory < 80%.
- [ ] Pod does not oscillate between zones on borderline utilization readings.

**Slow Start**
- [ ] On pod startup, `slow_start_slots` begins at `SLOW_START_INITIAL_SLOTS` (default 1) when `SLOW_START_ENABLED=True`.
- [ ] `slow_start_slots` increments by `SLOW_START_INCREMENT` only after a successful execution completion AND resources remain in HEALTHY zone.
- [ ] If resource pressure is detected during ramp-up, `slow_start_slots` is frozen at its current value.
- [ ] If CRITICAL pressure is detected, `slow_start_slots` is overridden to 0; slow start resets when pod recovers from CRITICAL.
- [ ] Once `slow_start_slots >= batch_size`, slow start is complete and the constraint is removed.

**Pre-Execution Headroom Gate**
- [ ] Before each execution in [`_run_agent_execution_with_setup()`](utils/messaging/redis_streams_trigger_handler.py:329), a synchronous resource check is performed when `PRE_EXECUTION_RESOURCE_CHECK_ENABLED=True`.
- [ ] If in CRITICAL zone at execution start: message is NACKed back to the Redis Stream; a `"warning"` event with error code `WORKER_WARN_HIGH_LOAD` is published via `enqueue_payload()`.
- [ ] If in PRESSURE zone at execution start: execution proceeds; a warning is logged.
- [ ] Gate is bypassed entirely when `PRE_EXECUTION_RESOURCE_CHECK_ENABLED=False`.

**Watchdog / Safety Net**
- [ ] The unified monitoring thread sets `resource_critical=True` after the pod has been in CRITICAL zone for longer than `WATCHDOG_GRACE_PERIOD_SECONDS`; the event loop checks this flag before starting new executions.
- [ ] The monitoring thread does NOT cancel or interrupt running executions.
- [ ] `resource_critical` flag is cleared when the pod recovers below the exit-critical threshold.

**User-Facing Event Notifications**
- [ ] Pre-start rejection publishes a `"warning"` event with `agentStatus: "warning"`, `errorCode: "WORKER_WARN_HIGH_LOAD"`, and message: *"Agent execution temporarily delayed due to high system load. Your request will be retried automatically."*
- [ ] HEALTHY → PRESSURE transition publishes a `"warning"` event (once per transition, not on every sample) with `errorCode: "WORKER_WARN_SLOW_RESPONSE"` for each active execution.
- [ ] PRESSURE → CRITICAL transition publishes a `"warning"` event (once per transition) for each active execution.
- [ ] All events use the existing `enqueue_payload()` pattern — no new payload fields beyond `errorCode` are introduced.
- [ ] Event messages contain no references to internal class names, thread names, queue operations, or metric variable names.
- [ ] All event publishing is disabled when `RESOURCE_PRESSURE_EVENTS_ENABLED=False`.
- [ ] OOM-kill recovery event (`WORKER_ERR_OOM`) is added to the existing [`_publish_delete_message_event()`](utils/messaging/redis_streams_trigger_handler.py:542) payload — no new publishing path required.

**Configuration**
- [ ] All 14 settings listed above are added to [`config/settings/infra.py`](config/settings/infra.py) with correct types and defaults.
- [ ] Settings are included in [`config/settings/__init__.py`](config/settings/__init__.py).
- [ ] All threshold settings (`CPU_PRESSURE_THRESHOLD_PERCENT`, `CPU_CRITICAL_THRESHOLD_PERCENT`, `MEMORY_PRESSURE_THRESHOLD_PERCENT`, `MEMORY_CRITICAL_THRESHOLD_PERCENT`, `RESOURCE_PRESSURE_SLOT_REDUCTION_FACTOR`, `WORKER_POD_MESSAGES_BATCH_SIZE`, `WATCHDOG_GRACE_PERIOD_SECONDS`, `RESOURCE_HYSTERESIS_BAND_PERCENT`) are overridable at runtime via [`ConfigManager`](config/config_service.py:13) without pod restart.

**Observability**
- [ ] Structured log entries are emitted on every zone transition with before/after slot values.
- [ ] Periodic resource utilization is logged every `RESOURCE_MONITOR_INTERVAL_SECONDS`.
- [ ] Log entries when consumption is paused (CRITICAL) and resumed (recovery).

**Tests**
- [ ] All existing tests pass unmodified when `RESOURCE_MONITOR_ENABLED=False` and `SLOW_START_ENABLED=False`.
