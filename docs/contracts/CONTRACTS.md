# Fairlane — External Contracts

**Project:** Fairlane — Fair & Fault-Tolerant GPU Scheduler
**Milestone:** Mini-Project 4 (Interface Contracts)
**Version:** 1.0
**Date:** 24 September 2026
**Domain:** Operating Systems & DBMS | **Team:** OSDBMS-V-2026-T065

This document finalizes all external dependencies (API payloads and storage conventions) shared between the Central Server, Worker Agent, Dashboard, and NFS storage layer. All teams must implement against these contracts exactly as specified. Any change requires a new issue on this repository and sign-off from the affected leads.

---

## 1. Job Submission Payload

**Owner:** Central Server · **Agreed with:** Vansh Sagar
**Endpoint:** `POST /api/jobs`

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `job_name` | string | Yes | Non-empty, ≤ 255 chars | Human-readable name of the job. |
| `script_path` | string | Yes | Must start with `/mnt/nfs/users/` | Absolute NFS path to the training script. |
| `required_gpus` | integer | Yes | ≥ 1 | Number of whole GPUs requested. |
| `memory_mb` | integer | Yes | > 0 | Peak memory requirement in megabytes. |
| `estimated_runtime` | integer | Yes | > 0 | Estimated runtime in seconds (used for scheduling hints). |
| `priority` | integer | Yes | ∈ [1, 5] | Base priority; 5 = highest. Aging increases effective priority over time. |
| `user_id` | string | Yes | Must match an authenticated user | Owning user's identifier. |

**Validation rules:**

- `script_path` must pass the prefix check: value starts with `/mnt/nfs/users/`; requests failing this check are rejected with HTTP `422`.
- `priority` must be in the inclusive range `[1, 5]`; out-of-range values are rejected with HTTP `422`.
- `required_gpus` must be ≥ 1; zero or negative values are rejected with HTTP `422`.
- `job_name` must not be empty after trimming whitespace.
- All integer fields must be whole numbers (no floats, no string-encoded numbers).
- The submitting user must match `user_id` unless the caller holds the `professor` or `admin` role.

---

## 2. Job Status Response

**Owner:** Central Server · **Agreed with:** Vansh Sagar
**Endpoint:** `GET /api/jobs/{id}`

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `job_id` | integer | No | Unique job identifier. |
| `user_id` | string | No | Owning user's identifier. |
| `status` | enum | No | One of `QUEUED` \| `RUNNING` \| `COMPLETED` \| `FAILED` \| `CANCELLED`. |
| `submitted_at` | ISO 8601 datetime | No | Time the job entered the queue (UTC). |
| `started_at` | ISO 8601 datetime | Yes | Time the job began running; `null` while `QUEUED`. |
| `ended_at` | ISO 8601 datetime | Yes | Time the job reached a terminal state; `null` otherwise. |
| `logs_url` | string | Yes | NFS path to the job's log file. |
| `output_url` | string | Yes | NFS path to the job's output artifacts directory. |

**Validation rules:**

- `status` must be exactly one of the five enum values (case-sensitive, uppercase).
- `submitted_at`, `started_at`, and `ended_at` use ISO 8601 format with UTC offset (e.g. `2026-09-24T13:45:00Z`).
- `started_at` must be non-null when `status` is `RUNNING`, `COMPLETED`, or `FAILED`.
- `ended_at` must be non-null when `status` is `COMPLETED`, `FAILED`, or `CANCELLED`.
- `logs_url` and `output_url` are populated once the job has started; both must satisfy the NFS shared-folder convention in Section 4.

---

## 3. Worker Registration Payload

**Owner:** Central Server · **Agreed with:** Suryansh
**Endpoint:** `POST /api/workers/register` (registration) · `POST /api/workers/{id}/heartbeat` (updates)

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `hostname` | string | Unique across the cluster | Worker node's hostname. |
| `gpu_count` | integer | ≥ 1 | Number of GPUs physically present on the node. |
| `gpu_utilization` | float | 0.0–1.0 | Fraction of GPU capacity currently in use (reported each heartbeat). |
| `memory_utilization` | float | 0.0–1.0 | Fraction of node memory currently in use (reported each heartbeat). |
| `state` | enum | One of `ONLINE` \| `DRAINING` \| `OFFLINE` | Current worker lifecycle state. |
| `last_heartbeat` | ISO 8601 datetime | UTC | Timestamp of the most recent heartbeat. |

**Validation rules:**

- `hostname` must be unique; re-registration with an existing hostname updates the existing worker record instead of creating a duplicate.
- `gpu_utilization` and `memory_utilization` are floats in the inclusive range `[0.0, 1.0]`; values outside the range are rejected.
- `state` must be exactly one of `ONLINE`, `DRAINING`, or `OFFLINE` (case-sensitive, uppercase).
- Workers send heartbeats every N seconds (default: 30); a worker missing N consecutive heartbeats is transitioned to `OFFLINE` by the server, and its running jobs are re-queued automatically.
- `last_heartbeat` is server-authoritative: workers report activity, but the server stamps the final value on receipt.

---

## 4. NFS Shared-Folder Convention

**Owner:** NFS/Storage Layer · **Agreed with:** Harshvardhan

All job scripts, logs, and outputs live under a single, predictable NFS tree:

```text
Base path: /mnt/nfs/users/{username}/jobs/{job_id}/
```

**Directory layout:**

```text
/mnt/nfs/users/{username}/jobs/{job_id}/
├── scripts/        # submitted training scripts (e.g. train.py)
├── logs/           # job log files (logs_url points here)
└── output/         # job output artifacts (output_url points here)
```

**Validation:**

- `script_path` must be a subpath under this convention, e.g. `/mnt/nfs/users/alice/jobs/123/train.py`.
- Paths outside `/mnt/nfs/users/{username}/jobs/{job_id}/` are rejected at submission time (HTTP `422`), regardless of prefix — a path under another user's directory is invalid.
- `{username}` must match the submitting user's account name; `{job_id}` must match the job identifier assigned at submission.
- Directory traversal sequences (`..`) are rejected outright.
- Permissions follow standard Unix/NFS ownership: only the owning user (and `admin` role) may read/write a job's subtree.

---

## Readiness Check

✅ All three leads (Vansh Sagar, Harshvardhan, Suryansh) have approved this document via GitHub issue comments.

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-09-24 | Initial release — all four contracts finalized for Mini-Project 4. |
