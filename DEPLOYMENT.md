# ProgrammersArena Judge-Box Integration & Deployment Guide

This guide explains the new separated judge-box architecture for secure code execution.

## Architecture Overview

```
Frontend (React)
    ↓
Backend (Laravel) → Redis Queue → Judge-Box (Node.js)
    ↑                                    ↓
    ←──── Judge Results Queue ←─────────
```

**Components:**
- **Backend (Laravel)**: API server; creates `Submission` records and pushes jobs to Redis `judge:jobs` queue
- **Judge-Box (Node.js daemon)**: Pulls jobs from Redis, executes code in sandboxed environments (Docker/isolate), pushes results to Redis `judge:results` queue
- **Judge Consumer**: Laravel command that pulls results from `judge:results` queue and updates submissions in the database

## Prerequisites

### System Requirements
- Docker & Docker Compose (for containerized setup)
- Redis (shared between backend and judge-box)
- PostgreSQL (already in backend)
- For isolate mode: Linux kernel with cgroups v2, isolate CLI installed

### Environment Variables

Create `.env` files based on `.env.example` in both `backend/` and `judge-box/` directories.

**Backend `.env`:**
```env
JUDGE_MODE=queue
JUDGE_REDIS_URL=redis://redis:6379/0
REDIS_HOST=redis
REDIS_PORT=6379
```

**Judge-Box:**
```env
EXEC_MODE=docker
ENABLE_REDIS=1
REDIS_URL=redis://redis:6379
```

## Quick Start: Docker Compose (Development)

### Option 1: Full Stack (Backend + Judge-Box)

From the project root:

```bash
# Backend + database + Redis
cd backend
docker-compose up --build

# In another terminal: Judge-Box (shares Redis)
cd judge-box
docker-compose up --build
```

**Verify:**
- Backend API: http://localhost:8000
- Frontend: http://localhost:3000
- Judge daemon HTTP: http://localhost:3001 (direct HTTP for testing)
- Redis: localhost:6379

### Option 2: Run Services Manually (No Docker)

**Terminal 1: Redis**
```bash
redis-server --port 6379
```

**Terminal 2: Backend**
```bash
cd backend
php artisan serve --port 8000
php artisan queue:work redis
php artisan judge:worker  # Consumer
```

**Terminal 3: Judge-Box**
```bash
cd judge-box
npm install
EXEC_MODE=docker ENABLE_REDIS=1 node src/daemon/index.js
```

## Testing the System

### Test 1: Direct HTTP Job (Judge-Box)

```bash
curl -X POST http://localhost:3001/job \
  -H 'Content-Type: application/json' \
  -d '{
    "language": "gcc",
    "version": "10",
    "files": {
      "submission.cpp": "#include<bits/stdc++.h>\nusing namespace std;\nint main(){int a,b; cin>>a>>b; cout<<a+b; return 0;}"
    },
    "input": "1 2",
    "expected_output": "3",
    "time_limit": 1,
    "memory_limit": 256
  }'
```

Expected response:
```json
{
  "status": "OK",
  "time_used_ms": 5,
  "memory_used_kb": 1024,
  "raw_output": "3",
  "error_message": null
}
```

### Test 2: Via Backend API (Full Flow)

1. Create a contest and problem via the backend API
2. Submit code as a user
3. Watch the submission status change: Queued → Judging → Accepted/Wrong Answer
4. Results appear in real-time

```bash
# Example: Submit via backend
curl -X POST http://localhost:8000/api/contests/{id}/problems/A/submit \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "code": "#include<bits/stdc++.h>\nusing namespace std;\nint main(){cout<<\"Hello World\";}",
    "language": "gcc-10"
  }'
```

## Production Deployment

### Using isolate (Recommended for Security)

**1. Host Setup:**
```bash
# Install isolate on the judge-box host
apt-get update && apt-get install isolate

# Enable cgroups v2
echo "cgroup2 /sys/fs/cgroup cgroup2 rw,nosuid,nodev,noexec,relatime 0 0" >> /etc/fstab
mount -a

# Initialize isolate
isolate --init
```

**2. Run Judge-Box with isolate:**
```bash
EXEC_MODE=isolate ENABLE_REDIS=1 node src/daemon/index.js
```

### Docker Stack (Production-Ready)

Update `judge-box/docker-compose.yml` for production:

```yaml
judge-daemon:
  environment:
    EXEC_MODE: isolate  # Use isolate instead of docker
    ENABLE_REDIS: "1"
  volumes:
    - /var/lib/isolate:/var/lib/isolate  # Mount isolate boxes
```

### Separate Judge-Box Host

For maximum security, run judge-box on a **separate VM or physical machine**:

```bash
# On judge-box host:
docker run -d \
  -e REDIS_URL=redis://backend-redis:6379 \
  -e EXEC_MODE=isolate \
  -e ENABLE_REDIS=1 \
  -p 3001:3001 \
  judge-box:latest
```

**Network:** Ensure backend and judge-box can communicate via Redis. Use a private network or VPN.

## Architecture Details

### Job Flow

1. **Submission Created:**
   - User submits code via frontend
   - Backend creates `Submission` record with status='Queued'

2. **Job Dispatched:**
   - `GradeSubmissionJob` is queued
   - Queue worker sends JSON to Redis `judge:jobs`

   Job format:
   ```json
   {
     "id": "sub-{submission_id}-{uniqid}",
     "language": "gcc",
     "version": "10",
     "files": {"submission.cpp": "...code..."},
     "input": "1 2\n",
     "expected_output": "3\n",
     "time_limit": 1,
     "memory_limit": 256
   }
   ```

3. **Judge-Box Processes:**
   - Pulls job from Redis `judge:jobs` queue (BRPOP)
   - Extracts files to `/tmp/submission-{id}`
   - Compiles (if needed) in Docker/isolate
   - Runs with resource limits
   - Checks output against expected result
   - Pushes result to Redis `judge:results`

   Result format:
   ```json
   {
     "id": "sub-{submission_id}-{uniqid}",
     "result": {
       "status": "AC|WA|CE|TLE|MLE|RE",
       "time_used_ms": 142,
       "memory_used_kb": 4096,
       "raw_output": "3",
       "error_message": null
     }
   }
   ```

4. **Result Consumed:**
   - `JudgeQueueWorker` command pulls results (BRPOP on `judge:results`)
   - Updates `Submission` with status, time, memory, output
   - Updates `Standing` records for contests

### Supported Languages

- **C++**: `gcc` (versions 10, 11, 12)
- **Python**: `python` (versions 3.8, 3.9, 3.10)
- **PHP**: `php` (versions 7.4, 8.0, 8.1)
- **Java**: `java` (versions 8, 11, 17)
- **JavaScript**: `javascript` (versions 12, 14, 16)
- **Rust**: `rust` (versions 1.70, 1.75)

Add more in `judge-box/config/languages.json` and `backend/config/languages.php`.

## Monitoring & Debugging

### Redis Queue Monitoring

```bash
# Watch queue sizes
redis-cli LLEN judge:jobs
redis-cli LLEN judge:results

# Inspect job (peek without removing)
redis-cli LRANGE judge:jobs 0 -1
redis-cli LRANGE judge:results 0 -1

# Clear queues (be careful!)
redis-cli DEL judge:jobs judge:results
```

### Judge-Box Logs

```bash
# Docker
docker logs judge-daemon

# Direct Node.js
node src/daemon/index.js  # Outputs to stdout
```

### Backend Logs

```bash
cd backend
tail -f storage/logs/laravel.log
```

## Troubleshooting

### Issue: Judge-Box Can't Connect to Redis
- **Check:** `REDIS_URL` env var is correct
- **Check:** Redis service is running and accessible
- **Fix:** `redis-cli PING` from judge-box container

### Issue: Submissions Stuck in "Queued"
- **Check:** Judge-box daemon is running: `docker ps | grep judge`
- **Check:** Queue consumer is running: `ps aux | grep judge:worker`
- **Fix:** Restart both services

### Issue: TLE or MLE Not Detected
- **Check:** `timeout` command is available in Docker image
- **Check:** `--memory` limit is enforced (needs cgroups v2 for isolate)
- **Fix:** Update Dockerfile or isolate config

### Issue: Output Checker Failing
- **Check:** Expected output file has correct line endings (LF, not CRLF)
- **Debug:** Check `check.sh` script trims whitespace properly
- **Fix:** Update checker logic in judge-box

## Performance Tuning

### Increase Judge-Box Throughput
- Run multiple judge-daemon instances (load balance via Redis)
- Increase memory limits for concurrent containers
- Use isolate instead of Docker (faster, lower overhead)

### Optimize Compilation
- Cache compiled artifacts per language version
- Pre-compile common libraries
- Use `-O2` optimization for C++

## Security Hardening Checklist

- [ ] Judge-box runs on separate host/VM from backend
- [ ] Network communication encrypted (VPN or SSH tunnel)
- [ ] Redis requires authentication (`requirepass` in redis.conf)
- [ ] isolate is used instead of Docker in production
- [ ] cgroups v2 enabled on judge-box host
- [ ] File upload limits enforced (100 MB default)
- [ ] Output truncation enabled (1 MB default)
- [ ] No compilation of unsafe code (disable preprocessor directives for C++)

## License

MIT
