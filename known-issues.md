# Known Issues

The following issues are tracked for the deepagents workspace template. Each entry describes the symptom, root cause, affected components, and current workaround.

---

## Issue 1 — executor.py timeout not propagated to platform

**Severity:** High  
**First introduced:** v0.3.0  
**Components affected:** `executor.py`

### Symptom

When a sub-agent task exceeds the configured `DEEPAGENTS_TASK_TIMEOUT_SEC`, the executor terminates the subprocess and marks the task as `failed` locally, but the Molecule platform continues to show the task as `running` indefinitely.

### Root cause

`executor.py` raises a `TimeoutError` internally and updates the local `status.yaml` to `failed`, but `adapter.py` does not receive an explicit signal to call the platform's `tasks.complete` or `tasks.cancel` endpoint with the timeout error code. The adapter only surfaces results on normal completion paths.

### Current workaround

Set `DEEPAGENTS_ABORT_ON_TIMEOUT=1` to make the executor emit a synthetic completion event before terminating the subprocess. This is a temporary workaround that can cause duplicate completion events if the subprocess recovers after the signal.

A proper fix requires adding a `timeout_exceeded=True` flag to the result payload that `adapter.py` reads and forwards as `tasks.cancel` with `reason=timeout` to the platform.

### Tracking

- Filed: 2025-03-14
- Ticket: DEEP-412

---

## Issue 2 — providers.py model routing fallback not configured

**Severity:** Medium  
**First introduced:** v0.2.0  
**Components affected:** `providers.py`

### Symptom

When `MODEL_ROUTING` env var is set to a JSON map that omits a role listed in `config.yaml`, the provider falls back to `None` instead of the default model declared in the config. This results in the orchestrator sending requests to a `None`-valued model endpoint, which raises a validation error in the SDK.

### Root cause

`providers.py`'s `resolve_model_for_role()` function reads `MODEL_ROUTING` first, and if the key is absent it returns `None` rather than checking `config.yaml` defaults before returning the hard-coded fallback.

### Current workaround

Always include all roles in `MODEL_ROUTING`, even if the intent is to use the config defaults:

```bash
export MODEL_ROUTING='{"orchestrator": "claude-sonnet-4-20250514", "task": "claude-haiku-4-20250514"}'
```

### Tracking

- Filed: 2025-02-20
- Ticket: DEEP-387

---

## Issue 3 — config.yaml schema version drift

**Severity:** Medium  
**First introduced:** v0.4.0 (config schema introduced)  
**Components affected:** `config.yaml`, `adapter.py`

### Symptom

After updating the template by merging changes from upstream, the workspace fails to start with the error `SchemaVersionMismatch: expected 1.2.0, got 1.1.0`. Agents do not boot.

### Root cause

`config.yaml` carries a top-level `schema_version` field (`1.2.0`). The `adapter.py` validator was updated to require `1.2.0` when the config shipped with `1.1.0`. If the operator's copy of `adapter.py` is older (e.g., from a cached Docker layer), the version check fails. There is no backwards-compatibility shim.

### Current workaround

Pin the `molecule-deepagents` package version in `requirements.txt` and rebuild the image:

```bash
pip install molecule-deepagents==0.4.2
docker build --no-cache -t deepagents-local:latest .
```

Always rebuild (avoid `--cache-from`) when updating the template.

### Tracking

- Filed: 2025-04-01
- Ticket: DEEP-445

---

## Issue 4 — escalation.py loop guard missing

**Severity:** High  
**First introduced:** v0.3.1  
**Components affected:** `escalation.py`

### Symptom

When a task fails and is re-queued for retry, it is possible for the orchestrator to enter an infinite re-escalation loop if the task input is malformed in a way that causes immediate failure on every attempt. The orchestrator never exits and the Molecule platform logs show repeated escalation events.

### Root cause

`escalation.py` has a `max_retries` check but does not guard against the case where a task is re-queued by the executor without incrementing the retry counter. The counter is incremented only at the start of `escalation.py`'s `run()` method. If the executor calls `escalate()` again for the same task after writing a new `status.yaml` without resetting the failure count, the guard never triggers.

### Current workaround

Set `DEEPAGENTS_MAX_RETRIES=1` (minimum viable setting) and set `DEEPAGENTS_CIRCUIT_BREAK_THRESHOLD=1` to force circuit breaking after a single immediate failure. Monitor `/workspace/agent-shared/task-agents/<id>/retry_count.yaml` and alert if it exceeds 2.

### Tracking

- Filed: 2025-03-28
- Ticket: DEEP-439

---
