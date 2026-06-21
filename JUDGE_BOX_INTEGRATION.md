# Judge-Box Integration Summary

## Overview
The judge system has been successfully separated into an isolated **judge-box** service that communicates with the Laravel backend via Redis queues. This provides:
- ✅ Secure sandboxed code execution (Docker/isolate)
- ✅ Scalable architecture (run multiple judge instances)
- ✅ Asynchronous job processing (non-blocking submissions)
- ✅ Support for multiple languages (C++, Python, PHP, Java, JavaScript, Rust)

---

## Files Created

### Judge-Box Directory Structure
```
judge-box/
├── config/
│   ├── languages.json          # Language configs (compile/execute commands)
│   └── judge-box.json          # System config (isolate, cgroups, Redis)
├── src/
│   ├── daemon/
│   │   ├── index.js            # Node.js HTTP/Redis worker
│   │   └── package.json        # Dependencies (express, ioredis)
│   ├── sandbox/
│   │   └── isolate_runner.sh   # Production isolate wrapper
│   └── checkers/
│       └── check.sh            # Output comparison script
├── Dockerfile                   # Multi-compiler image with isolate
├── docker-compose.yml          # Local dev setup with Redis
└── README.md                    # Quick start & architecture docs
```

### Backend Integration Files
```
backend/
├── app/
│   ├── Jobs/
│   │   └── GradeSubmissionJob.php        # Pushes to Redis judge:jobs
│   ├── Console/Commands/
│   │   └── JudgeQueueWorker.php          # Pulls from Redis judge:results
│   ├── Http/Controllers/
│   │   └── SubmissionController.php      # Updated to use new queue
│   └── Models/
│       └── Submission.php                # Added status, time, memory fields
├── config/
│   └── judge.php                         # Updated config with Redis settings
├── database/migrations/
│   └── 2024_06_21_000000_add_judge_queue_columns_to_submissions.php
├── docker-compose.yml                    # Added Redis & judge-consumer services
└── .env.example                          # Added JUDGE_MODE & JUDGE_REDIS_URL
```

### Root Documentation
```
DEPLOYMENT.md                    # Full deployment guide & architecture
```

---

## Key Changes to Backend

### 1. SubmissionController.php
**Before:**
```php
TestCodeJob::dispatch($host, $submission, $language, $version);
```

**After:**
```php
GradeSubmissionJob::dispatch(
    $submission->id,
    $languageKey,
    $version,
    $problem->time_limit ?? 1,
    $problem->memory_limit ?? 256
);
```

### 2. New Queue Jobs
- **GradeSubmissionJob**: Pushes to `judge:jobs` queue
- **JudgeQueueWorker**: Command that listens to `judge:results` queue

### 3. Database Schema
New columns added to `submissions` table:
- `status` (Queued, Judging, Accepted, Wrong Answer, etc.)
- `output` (raw program output)
- `time` (execution time in ms)
- `memory` (memory used in KB)
- `error_message` (compilation or runtime errors)
- `judged_at` (timestamp when judging completed)

---

## Execution Modes

### Development (Docker-based)
```bash
EXEC_MODE=docker node src/daemon/index.js
```
- Each submission runs in a Docker container
- Simple, works everywhere
- Slightly higher overhead

### Production (isolate-based)
```bash
EXEC_MODE=isolate node src/daemon/index.js
```
- Uses lightweight isolate sandbox
- Fine-grained cgroups v2 limits
- Better performance & security
- Requires isolate installed on host

---

## Queue Protocol

### Job Format (→ judge:jobs)
```json
{
  "id": "sub-{submission_id}-{uniqid}",
  "language": "gcc",
  "version": "10",
  "files": {
    "submission.cpp": "...",
    "grader.cpp": "..."
  },
  "input": "1 2\n",
  "expected_output": "3\n",
  "time_limit": 1,
  "memory_limit": 256
}
```

### Result Format (← judge:results)
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

---

## Deployment Steps

### Local Development
```bash
# Terminal 1: Backend + Redis
cd backend && docker-compose up --build

# Terminal 2: Judge-Box
cd judge-box && docker-compose up --build

# Verify
curl http://localhost:3001/job  # Judge daemon HTTP endpoint
redis-cli LLEN judge:jobs       # Check queue sizes
```

### Production (isolate mode)
```bash
# On judge-box host
apt-get install isolate
isolate --init

# Run judge-box with isolate
EXEC_MODE=isolate ENABLE_REDIS=1 REDIS_URL=redis://backend:6379 \
  node src/daemon/index.js
```

### Update Backend
```bash
# Backend .env
JUDGE_MODE=queue
REDIS_HOST=redis  (or IP/hostname of judge-box host)
REDIS_PORT=6379

# Run migrations
php artisan migrate

# Start consumers
php artisan queue:work redis
php artisan judge:worker  # Listens to judge:results
```

---

## Testing

### Direct HTTP (for manual testing)
```bash
curl -X POST http://localhost:3001/job \
  -H 'Content-Type: application/json' \
  -d '{
    "language":"gcc","version":"10",
    "files":{"submission.cpp":"#include<bits/stdc++.h>\nusing namespace std;\nint main(){int a,b; cin>>a>>b; cout<<a+b;}"},
    "input":"1 2","expected_output":"3"
  }'
```

### Via Backend API
1. Submit code as user
2. Check `submissions` table → status changes from Queued → Judging → Accepted/Wrong Answer
3. Results appear in real-time

---

## Supported Languages

| Language   | Versions           | Extension | Compilation |
|------------|-------------------|-----------|------------|
| C++        | 10, 11, 12        | cpp       | Yes        |
| Python     | 3.8, 3.9, 3.10    | py        | No         |
| PHP        | 7.4, 8.0, 8.1     | php       | No         |
| Java       | 8, 11, 17         | java      | Yes        |
| JavaScript | 12, 14, 16        | js        | No         |
| Rust       | 1.70, 1.75, stable| rs        | Yes        |

---

## Next Steps (Optional)

1. **Add more languages**: Update `judge-box/config/languages.json` and `backend/config/languages.php`
2. **Implement interactive problems**: Add support for interactive judge in `check.sh`
3. **Use Testlib**: Replace simple checker with full Testlib integration
4. **Load balancing**: Run multiple judge-box instances, load balance via Redis
5. **Metrics & Monitoring**: Add Prometheus/Grafana for judge performance tracking
6. **Caching**: Cache compiled artifacts per language version

---

## Security Notes

✅ **Completed:**
- Submissions isolated in Docker/isolate sandbox
- Network disabled during execution
- Memory & CPU limits enforced
- Output truncated (prevents DOS)
- Process fork limits (`pids.max`)

⚠️ **Recommended for Production:**
- Run judge-box on separate host/VM
- Use isolate instead of Docker
- Enable cgroups v2 on judge host
- Require Redis authentication
- VPN/encrypted tunnel between backend & judge-box
- Regular security audits of sandbox configuration

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Submissions stuck in "Queued" | Check judge-daemon is running, Redis connected |
| TLE not detected | Ensure `timeout` command available, cgroups v2 enabled |
| Memory limits not working | Use isolate mode, enable cgroups v2 |
| Can't connect to Redis | Check Redis is running, REDIS_URL is correct |
| Compilation errors | Check Docker image has compilers installed |

---

## References

- **Isolate Documentation**: https://github.com/ioi/isolate
- **Competitive Programming Judge Repo**: https://github.com/ioi/cms
- **Redis Queues**: https://redis.io/docs/interact/stream-info/
- **cgroups v2**: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html
