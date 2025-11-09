
# queuectl (Node.js)

A minimal, production-grade **CLI job queue** with **workers**, **retries with exponential backoff**, and a **Dead Letter Queue (DLQ)**. Persistent storage via **SQLite**.

> Tech: Node 18+, `better-sqlite3`, `commander`, `execa`

# demo - https://drive.google.com/file/d/15Wk3msi1Fe2Kne2y_nmktIquKhqRWFeQ/view?usp=sharing 

## ✨ Features

- Enqueue shell commands as jobs (e.g., `sleep 2`, `echo hi`)
- Multiple workers with parallel processing
- Atomic job claiming (no duplicate processing)
- Retries with exponential backoff (`delay = base^attempts`)
- Moves to **DLQ** after `max_retries`
- Persistent storage in `~/.queuectl/queue.db`
- Graceful shutdown (SIGTERM): finish current job, then exit
- Configurable via `queuectl config` (retries, backoff, poll interval, timeout)
- Per-job output logs in `~/.queuectl/logs/job-<id>.log`

## 📦 Install & Run

### Linux / macOS

```bash
# 1) Install dependencies
npm install

# 2) Link the CLI (or run via node bin/queuectl.js)
npm link
# now `queuectl` is available on PATH
```

### 🪟 Windows Setup

**Note**: `better-sqlite3` requires native compilation on Windows.

#### Prerequisites:
- Node.js 18+
- Visual Studio Build Tools

#### Install Build Tools:

**Option 1 - Automated**:
```bash
npm install --global windows-build-tools
```

**Option 2 - Manual**:
1. Download Visual Studio: https://visualstudio.microsoft.com/downloads/
2. Install with "Desktop development with C++" workload

Then:
```bash
npm install
npm link
```

#### Alternative: Use Docker
```dockerfile
# See Dockerfile in repo
docker build -t queuectl .
docker run -it queuectl
```

> Uses `~/.queuectl/` for DB, logs, and PID file.

## 🧪 Quick Demo

```bash
# Start 2 workers
queuectl worker start --count 2

# Enqueue a few jobs
queuectl enqueue '{"command":"echo \"Hello World\""}'
queuectl enqueue '{"command":"bash -lc \"exit 1\""}'           # will retry then DLQ
queuectl enqueue '{"command":"sleep 1 && echo done"}'

# Status
queuectl status

# Stop workers (graceful)
queuectl worker stop

# DLQ
queuectl dlq list
queuectl dlq retry <job-id>
```



### Config Keys
- `max_retries` (default `3`)
- `backoff_base` (default `2`)
- `poll_interval_ms` (default `500`)
- `job_timeout_ms` (default `0` = disabled)

## 🏗️ Architecture Overview

- **Persistence**: SQLite (`better-sqlite3`) with WAL mode for safe concurrency.
- **Atomic claim**: Worker uses a transaction to select one `pending` job whose `scheduled_at <= now()`, then `UPDATE ... WHERE state='pending'` to switch it to `processing`. Only one worker can succeed.
- **Execution**: Shell command via `execa` with optional timeout. Stdout/stderr are captured to per-job log file.
- **Retry/Backoff**: On failure, increment `attempts`, compute `delay = base^attempts`, set `scheduled_at = now + delay` and return job to `pending`. If `attempts > max_retries`, move job to `dlq`.
- **Graceful stop**: `queuectl worker stop` sends SIGTERM to PIDs saved in `~/.queuectl/pids.json`. Worker loop exits after current job.
- **DLQ**: Stores original job JSON + reason. `dlq retry <id>` re-enqueues and removes from DLQ.

## 🧷 Schema

Tables: `jobs`, `dlq`, `config`, `workers`

Key job columns:
- `state`: `pending|processing|completed|failed (not stored)|dead (in dlq)`
- `attempts`, `max_retries`
- `scheduled_at` (respects backoff)
- `output_log` (path to log)
- `priority` (higher first; default `0`)

## 🧪 Testing

Basic smoke test:

```bash
npm test
```

Demo run:

```bash
npm run demo
```


