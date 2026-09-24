# Fairlane GPU Scheduler

# Fairlane — Product Requirements Document (PRD)

**Version:** 1.0  

**Status:** Draft  

**Author:** Team OSDBMS-V-2026-T065  

**Date:** 24 September 2026  

**Domain:** Operating Systems & DBMS  

**Academic Session:** 2026–27  

---

## 1. Introduction

### 1.1 Purpose

This document defines the functional and non-functional requirements for **Fairlane**, a fair and fault-tolerant scheduler for distributed machine learning (ML) workloads in shared GPU environments. It is intended for developers, testers, and stakeholders involved in the design, implementation, and evaluation phases.

### 1.2 Scope

Fairlane is a lightweight, self-hosted GPU orchestration system that:

- Accepts job submissions from multiple users.

- Queues and schedules jobs using a priority-with-aging algorithm.

- Allocates GPU resources across worker nodes.

- Tracks job status and resource usage in real time.

- Recovers gracefully from worker failures.

The system is designed for academic labs, research groups, and small teams sharing a handful of GPUs, without the complexity of enterprise schedulers like Slurm or Run:ai.

### 1.3 Definitions & Acronyms

| Term | Definition |

|------|------------|

| Job   | A computational task (e.g., ML training) submitted by a user for execution. |

| Worker Node | A machine with GPU(s) that executes jobs. |

| Central Server | The main control node handling scheduling, API, and coordination. |

| Dashboard | Web interface for users to submit jobs and monitor status. |

| NFS | Network File System — shared storage for scripts and data. |

| ACID | Atomicity, Consistency, Isolation, Durability (database properties). |

| Row-Level Locking | Database locking mechanism to prevent concurrent write conflicts. |

| Aging | Technique that increases a job's priority the longer it waits, preventing starvation. |

---

## 2. Goals & Non-Goals

### 2.1 Goals

- **G1:** Provide a centralized, easy-to-use interface for submitting and monitoring ML jobs.

- **G2:** Ensure **fairness** in GPU allocation among concurrent users, preventing starvation and monopoly.

- **G3:** Implement **fault tolerance** — automatically recover from worker node crashes or overloads.

- **G4:** Integrate OS scheduling concepts and DBMS concurrency control into a single working system.

- **G5:** Deliver a functional prototype that can be deployed in a real lab environment with minimal setup.

### 2.2 Non-Goals

- **NG1:** Build a production-grade, enterprise-scale cluster scheduler (like Kubernetes-based ones).

- **NG2:** Provide per-job dependency isolation (e.g., containerized environments). We assume a shared Python environment.

- **NG3:** Implement advanced security features (e.g., Kerberos, multi-tenant isolation beyond standard Unix/NFS permissions).

- **NG4:** Support checkpointing/resume of jobs (evicted jobs restart from scratch).

- **NG5:** Support heterogeneous GPU types or dynamic GPU partitioning (each job gets whole GPUs).

---

## 3. User Personas & Use Cases

### 3.1 Personas

| Persona | Description | Needs |

|---------|-------------|-------|

| **Student Researcher** | A graduate student running ML experiments. | Submit jobs easily, know when they run, see results. |

| **Professor / PI** | Manages the lab and supervises multiple students. | Oversee resource usage, ensure fair sharing, monitor lab activity. |

| **System Administrator** | Maintains the GPU cluster. | Add/remove worker nodes, monitor health, handle failures. |

### 3.2 Use Cases

- **UC1:** User uploads training script to NFS, submits job with resource request (e.g., 2 GPUs, 8GB RAM).

- **UC2:** User views job status (queued, running, completed, failed).

- **UC3:** User cancels a queued or running job.

- **UC4:** Professor views overall resource utilization across all workers.

- **UC5:** Admin adds a new worker node to the pool (dynamic registration).

- **UC6:** Worker node fails mid-job; job is automatically re-queued.

- **UC7:** System prevents a single user from hogging all GPUs (max concurrent jobs per user).

---

## 4. Functional Requirements

Priority legend: **M** (Must), **S** (Should), **C** (Could)

### 4.1 User Management & Authentication

| ID | Requirement | Priority |

|----|-------------|----------|

| FR-1.1 | Users can register/login with a university account (email/password). | M |

| FR-1.2 | Users have a role: `student`, `professor`, `admin`. | M |

| FR-1.3 | Admin can manage users (add/remove/change roles). | S |

| FR-1.4 | Sessions expire after inactivity (e.g., 30 min). | S |

### 4.2 Job Submission & Management

| ID | Requirement | Priority |

|----|-------------|----------|

| FR-2.1 | User can submit a job by providing: job name, path to script in NFS, required GPUs, memory, runtime estimate, and (optionally) priority. | M |

| FR-2.2 | User can cancel a queued job. | M |

| FR-2.3 | User can view details of their jobs (status, start/end time, logs, output path). | M |

| FR-2.4 | User can list all their jobs with filters (status, date). | S |

| FR-2.5 | Admin/professor can view all jobs across users. | S |

### 4.3 Scheduling

| ID | Requirement | Priority |

|----|-------------|----------|

| FR-3.1 | Implement a priority queue with aging (effective priority = base_priority + waiting_factor). | M |

| FR-3.2 | The scheduler selects the next job that fits available resources on a worker. | M |

| FR-3.3 | Enforce per-user maximum concurrent jobs (configurable). | M |

| FR-3.4 | Provide a command-line interface (CLI) for power users (optional). | C |

| FR-3.5 | Support preemption? (Out of scope – but record as future). | C |

### 4.4 Worker Management

| ID | Requirement | Priority |

|----|-------------|----------|

| FR-4.1 | Workers send heartbeats to the central server every N seconds. | M |

| FR-4.2 | Worker can dynamically register with the server (provides hostname, GPU info). | S |

| FR-4.3 | If a worker misses N heartbeats, it is marked offline and its jobs are re-queued. | M |

| FR-4.4 | Admin can manually decommission a worker (graceful drain). | S |

### 4.5 Fault Tolerance & Recovery

| ID | Requirement | Priority |

|----|-------------|----------|

| FR-5.1 | If a job fails due to worker crash, the job is re-queued automatically (up to `max_retries`). | M |

| FR-5.2 | The system logs all failures and recovery actions for audit. | M |

| FR-5.3 | The central server itself is stateless except for the DB; if it crashes, it can restart and resume managing jobs from DB state. | S |

| FR-5.4 | On restart, incomplete jobs are marked `failed` or re-queued based on policy. | S |

### 4.6 Monitoring & Dashboard

| ID | Requirement | Priority |

|----|-------------|----------|

| FR-6.1 | Dashboard shows real-time queue (waiting jobs with priorities) and running jobs. | M |

| FR-6.2 | Dashboard shows per-worker resource usage (GPU %, memory, CPU). | M |

| FR-6.3 | Users see a personalized view (their jobs only). | M |

| FR-6.4 | Admin sees cluster-wide view with all workers and active jobs. | M |

| FR-6.5 | Dashboard provides job logs (streamed) and output file links. | S |

| FR-6.6 | Alerts (e.g., worker down, job failure) are displayed to admin. | S |

### 4.7 Database & State Management

| ID | Requirement | Priority |

|----|-------------|----------|

| FR-7.1 | PostgreSQL stores all persistent data (users, jobs, workers, logs). | M |

| FR-7.2 | Use row-level locking to atomically claim a job for dispatch (prevent double dispatch). | M |

| FR-7.3 | All state transitions (queued→running→completed/failed) are transactional. | M |

| FR-7.4 | Provide database migration scripts (e.g., using Alembic). | S |

### 4.8 API Endpoints (High-Level)

| Method | Endpoint | Description |

|--------|----------|-------------|

| POST | `/api/auth/login` | Authenticate user. |

| POST | `/api/auth/register` | Create user. |

| POST | `/api/jobs` | Submit new job. |

| GET  | `/api/jobs` | List user's jobs. |

| GET  | `/api/jobs/{id}` | Job details & logs. |

| POST | `/api/jobs/{id}/cancel` | Cancel a job. |

| GET  | `/api/workers` | List workers (admin). |

| GET  | `/api/status` | Cluster status (queue, workers). |

| POST | `/api/workers/register` | Worker registration (internal). |

| POST | `/api/workers/{id}/heartbeat` | Heartbeat (internal). |

---

## 5. Non-Functional Requirements

| ID | Category | Requirement | Priority |

|----|----------|-------------|----------|

| NFR-1 | Performance | System should handle at least 50 concurrent users submitting jobs without noticeable latency (< 1s for UI actions). | M |

| NFR-2 | Performance | Scheduling decision should be made within 100ms. | M |

| NFR-3 | Reliability | Uptime of central server ≥ 99% during working hours (excluding planned maintenance). | S |

| NFR-4 | Fault Tolerance | Loss of a worker node should not affect other running jobs; recovery of its jobs within 2 minutes. | M |

| NFR-5 | Security | Passwords stored hashed (bcrypt). Access control enforced via roles. | M |

| NFR-6 | Security | All API endpoints authenticated except login/register. | M |

| NFR-7 | Data Integrity | No job should be lost or double-dispatched due to race conditions. | M |

| NFR-8 | Usability | Dashboard should be intuitive; a new user can submit a job within 2 minutes of first use. | S |

| NFR-9 | Scalability | Adding new workers should require no code changes; automatic registration. | S |

| NFR-10 | Observability | System logs should be structured (JSON) for easy debugging. | S |

| NFR-11 | Maintainability | Code should be modular (server, worker, dashboard) and unit-tested. | S |

---

## 6. System Architecture

### 6.1 High-Level Design

```

[User Browser] --> [Dashboard (FastAPI + Frontend)] --> [Central Server API]

                                                        |

                                                        | REST/WebSocket

                                                        v

                                                   [Scheduler Logic]

                                                        |

                                                        | DB queries with locks

                                                        v

                                                   [PostgreSQL]

                                                        ^

                                                        |

              [Worker Agent] <---- dispatch/control ---+

              (Python, subprocess, NVML)

                     |

                     +-----> NFS (shared storage)

```

### 6.2 Components

1. **Dashboard/UI**: Web-based interface (HTML/JS + FastAPI static serving or separate frontend).

2. **Central Server**: FastAPI app handling API, scheduling loop, worker coordination.

3. **Worker Agent**: Python process running on each GPU node; listens for commands, executes jobs via `subprocess`, reports telemetry via heartbeat.

4. **Database**: PostgreSQL stores jobs, workers, users, logs.

5. **NFS**: Shared filesystem for scripts, data, and outputs.

### 6.3 Data Flow

- Job submission → stored in DB (status=`queued`).

- Scheduler loop queries DB for pending jobs, matches to free workers using `SELECT ... FOR UPDATE SKIP LOCKED`.

- Dispatches job to worker via HTTP/WebSocket.

- Worker runs job, updates status via DB or server API.

- Heartbeats maintain worker liveness.

---

## 7. Data Model (High-Level)

```sql

-- Users

CREATE TABLE users (

    id SERIAL PRIMARY KEY,

    email VARCHAR(255) UNIQUE NOT NULL,

    password_hash VARCHAR(255) NOT NULL,

    role VARCHAR(20) NOT NULL DEFAULT 'student',  -- student/professor/admin

    created_at TIMESTAMP NOT NULL DEFAULT now()

);

-- Jobs

CREATE TABLE jobs (

    id SERIAL PRIMARY KEY,

    user_id INT REFERENCES users(id),

    name VARCHAR(255),

    script_path TEXT NOT NULL,          -- NFS path

    gpu_count INT DEFAULT 1,

    memory_mb INT,

    status VARCHAR(20) NOT NULL DEFAULT 'queued',  -- queued/running/completed/failed/cancelled

    priority INT DEFAULT 0,

    submitted_at TIMESTAMP NOT NULL DEFAULT now(),

    started_at TIMESTAMP,

    ended_at TIMESTAMP,

    worker_id INT REFERENCES workers(id),

    error_log TEXT,

    max_retries INT DEFAULT 3,

    retry_count INT DEFAULT 0

);

-- Workers

CREATE TABLE workers (

    id SERIAL PRIMARY KEY,

    hostname VARCHAR(255) UNIQUE,

    gpu_info JSONB,                     -- e.g., {gpu_model, total_gpus}

    total_mem_mb INT,

    status VARCHAR(20) DEFAULT 'active', -- active/offline/draining

    last_heartbeat TIMESTAMP

);

-- Job_Logs (optional)

CREATE TABLE job_logs (

    id SERIAL PRIMARY KEY,

    job_id INT REFERENCES jobs(id),

    timestamp TIMESTAMP DEFAULT now(),

    log_line TEXT

);

```

---

## 8. User Interface / Dashboard Mock-up

### 8.1 Screens (Descriptive)

- **Login/Register**: Simple form.

- **Dashboard (User)**: 

  - "Submit New Job" button (form: name, script path, resources).

  - Table of my jobs: ID, Name, Status, Priority, Submitted, Actions (cancel, view logs).

  - Real-time status via WebSocket updates.

- **Admin Dashboard**: 

  - Summary cards (total jobs, active workers, queue length).

  - Workers table with health status, GPU utilization.

  - All jobs view with filters.

- **Job Detail Page**: Shows logs, resource usage graph, output file link.

---

## 9. Testing & Acceptance Criteria

### 9.1 Unit Tests

- Scheduler logic (priority aging, resource matching).

- Database locking behavior (no double dispatch).

- API validation.

### 9.2 Integration Tests

- Worker registration & heartbeat.

- End-to-end job lifecycle (submit → run → complete).

- Failure injection (worker crash) → job re-queued.

### 9.3 Acceptance Criteria (Phase-II completion)

- **AC-1:** User can submit a job and see it move from `queued` to `running` to `completed` in the dashboard.

- **AC-2:** With 2 workers and 4 users submitting jobs, fairness metric (e.g., average wait time per user) shows no user starved.

- **AC-3:** Killing a worker mid-job results in that job being re-queued automatically within 2 minutes.

- **AC-4:** Admin can add a new worker by simply running the worker script; it appears on dashboard without restarting server.

- **AC-5:** All database transactions maintain ACID; no job is lost or duplicated under concurrent submissions.

---

## 10. Milestones & Timeline

| Phase | Duration | Deliverables |

|-------|----------|--------------|

| **Phase-I (Proposal & Design)** | Weeks 1–4 | PRD, architecture, DB schema, scheduler design ✔ |

| **Phase-II (Development)** | Weeks 5–12 | Central server API, worker agent, dashboard prototype, basic scheduling |

| **Phase-III (Final Integration)** | Weeks 13–16 | Full system, stress testing, failure simulation, final report & demo |

---

## 11. Risks & Open Questions

| Risk | Mitigation |

|------|------------|

| Worker heterogeneity (GPU types) | Assume uniform GPUs initially; document extension. |

| NFS latency/bandwidth | Keep jobs lightweight; test with realistic workloads. |

| Concurrency bugs | Extensive use of `SELECT ... FOR UPDATE SKIP LOCKED` and transaction tests. |

| Dashboard complexity | Start with simple server-rendered HTML, then add JS if time permits. |

| Security of user scripts | Run as unprivileged user; rely on OS permissions; document that system is for trusted network. |

**Open Questions:**

- Should jobs support GPU memory isolation? (Currently, allocate whole GPUs.)

- What is the maximum number of workers expected? (Assumption: 1–10.)

- Should there be a CLI client? (Nice-to-have.)

---

## 12. Appendices

### 12.1 References

- Run:ai: https://run-ai-docs.nvidia.com/saas

- Slurm-on-Kubernetes: https://developer.nvidia.com/blog/running-large-scale-gpu-workloads-on-kubernetes-with-slurm/

- Volcano: https://volcano.sh/docs

- PostgreSQL Explicit Locking: https://www.postgresql.org/docs/current/explicit-locking.html

- FastAPI: https://fastapi.tiangolo.com/

- Celery (queue pattern): https://docs.celeryq.dev/en/stable/

### 12.2 Document Revision History

| Version | Date | Author | Changes |

|---------|------|--------|---------|

| 1.0 | 2026-09-24 | Team | Initial release |

---

**End of PRD**

Create a GitHub-ready Markdown file at docs/contracts/CONTRACTS.md that finalizes all external dependencies for Mini-Project 4. Structure it with these sections:

1. Job Submission Payload (agreed with Vansh Sagar):

   - Field table: job_name (string), script_path (string, must start with /mnt/nfs/users/), required_gpus (integer ≥1), memory_mb (integer), estimated_runtime (integer, seconds), priority (integer 1-5), user_id (string)

   - Validation rules: script_path prefix check, priority ∈ [1,5], required_gpus ≥1

2. Job Status Response (agreed with Vansh Sagar):

   - Field table: job_id (integer), user_id (string), status (enum: QUEUED|RUNNING|COMPLETED|FAILED|CANCELLED), submitted_at (ISO 8601 datetime), started_at (nullable ISO 8601), ended_at (nullable ISO 8601), logs_url (string, NFS path), output_url (string, NFS path)

3. Worker Registration Payload (agreed with Suryansh):

   - Field table: hostname (string), gpu_count (integer), gpu_utilization (float 0.0-1.0), memory_utilization (float 0.0-1.0), state (enum: ONLINE|DRAINING|OFFLINE), last_heartbeat (ISO 8601 datetime)

4. NFS Shared-Folder Convention (agreed with Harshvardhan):

   - Statement: Base path: /mnt/nfs/users/{username}/jobs/{job_id}/

   - Validation: script_path must be a subpath under this convention (e.g., /mnt/nfs/users/alice/jobs/123/train.py)

Output Requirements:

   - Use clear markdown tables for all field definitions

   - Include validation rules as bullet points under each section

   - Add a ## Readiness Check section stating:

     ✅ All three leads (Vansh Sagar, Harshvardhan, Suryansh) have approved this document via GitHub issue comments.

   - Save as docs/contracts/CONTRACTS.md in the repository root

This project was built with [Lovable](https://lovable.dev).

## Build with Lovable

Continue developing this project in the [Lovable editor](https://lovable.dev/projects/8482f399-984a-424f-9d49-2b21cc40ea10).

- **Ship faster**: describe what you want to build and Lovable handles the code.
- **Stay in sync**: every change made in Lovable is committed straight to this repository.
- **Full ownership**: this code is yours. Push to `main` on GitHub and your changes sync back into Lovable, ready for your next prompt.

## Development

Prefer working locally? You need Node.js and npm — [install with nvm](https://github.com/nvm-sh/nvm#installing-and-updating).

```sh
git clone <this-repository-url>
cd <repository-name>
npm i
npm run dev
```
