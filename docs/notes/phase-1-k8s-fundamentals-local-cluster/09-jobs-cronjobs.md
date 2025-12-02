# Day 09 — Jobs and CronJobs

**Goal:** Learn how to run one-off and scheduled workloads.

---

## What are Jobs and CronJobs?

- **Job:** Runs a task to completion (one-time or parallel)
- **CronJob:** Creates Jobs on a schedule (recurring tasks)

---

## Job Manifest

**File:** `exercises/jobs-cronjobs/simple-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: simple-job
spec:
  completions: 1
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: job-container
        image: busybox:1.36
        command: ["sh", "-c"]
        args:
        - echo "Job completed successfully at $(date)"
```

**Apply:**
```bash
kubectl apply -f exercises/jobs-cronjobs/simple-job.yaml
```

**Observe:**
```bash
kubectl get jobs
kubectl get pods -l app=job-demo
kubectl logs -l app=job-demo
```

**Expected output:**
```
NAME         COMPLETIONS   DURATION   AGE
simple-job   1/1           3s         12s

NAME               READY   STATUS      RESTARTS   AGE
simple-job-kf2nc   0/1     Completed   0          16s
```

---

## CronJob Manifest

**File:** `exercises/jobs-cronjobs/simple-cronjob.yaml`

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: simple-cronjob
spec:
  schedule: "*/1 * * * *"  # Every minute
  jobTemplate:
    spec:
      completions: 1
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cronjob-container
            image: busybox:1.36
            command: ["sh", "-c"]
            args:
            - echo "CronJob executed at $(date)"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

**Apply:**
```bash
kubectl apply -f exercises/jobs-cronjobs/simple-cronjob.yaml
```

**Observe:**
```bash
kubectl get cronjobs
kubectl get jobs -l app=cronjob-demo
kubectl get pods -l app=cronjob-demo
```

**Expected output:**
```
NAME             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
simple-cronjob   */1 * * * *   False     0        14s             117s

NAME                      COMPLETIONS   DURATION   AGE
simple-cronjob-29399628   1/1           3s         78s
simple-cronjob-29399629   1/1           3s         18s
```

---

## Key Fields Explained

### .spec.completions

**Purpose:** Number of successful completions required for Job to be considered complete.

**Values:**
- `1` (default): Job completes when 1 pod succeeds
- `N`: Job completes when N pods succeed (parallel jobs)
- `null`: Job completes when any pod succeeds

**Example:**
```yaml
spec:
  completions: 3  # Job needs 3 successful pods
```

**Use case:** Parallel processing, batch jobs that need multiple workers.

### .spec.backoffLimit

**Purpose:** Number of retries before Job is marked as failed.

**Values:**
- `3` (default): Retry up to 3 times
- `0`: No retries
- `N`: Retry N times

**Behavior:**
- If pod fails, Job creates new pod
- After `backoffLimit` failures, Job marked as Failed
- Exponential backoff between retries

**Example:**
```yaml
spec:
  backoffLimit: 5  # Retry up to 5 times
```

**Use case:** Handle transient failures, network issues.

### .spec.schedule (CronJob only)

**Purpose:** Cron expression defining when Jobs should be created.

**Format:** `"minute hour day month weekday"`

**Examples:**
```yaml
schedule: "*/1 * * * *"    # Every minute
schedule: "0 * * * *"      # Every hour
schedule: "0 0 * * *"      # Every day at midnight
schedule: "0 0 * * 0"      # Every Sunday at midnight
schedule: "*/5 * * * *"    # Every 5 minutes
schedule: "0 9 * * 1-5"    # Every weekday at 9 AM
```

**Cron syntax:**
- `*` = any value
- `*/N` = every N units
- `N-M` = range
- `N,M` = specific values

**Use case:** Scheduled backups, reports, maintenance tasks.

---

## Observing Status

### Jobs

```bash
# List all jobs
kubectl get jobs

# Describe job
kubectl describe job simple-job

# View job pods
kubectl get pods -l app=job-demo

# View job logs
kubectl logs -l app=job-demo
```

**Status indicators:**
- `COMPLETIONS: 1/1` - Job completed successfully
- `COMPLETIONS: 0/1` - Job still running
- Pod `STATUS: Completed` - Pod finished successfully
- Pod `STATUS: Error` - Pod failed

### CronJobs

```bash
# List cronjobs
kubectl get cronjobs

# Describe cronjob
kubectl describe cronjob simple-cronjob

# View jobs created by cronjob
kubectl get jobs -l app=cronjob-demo

# View cronjob logs
kubectl logs -l app=cronjob-demo
```

**Status indicators:**
- `LAST SCHEDULE` - When last job was created
- `ACTIVE` - Number of currently running jobs
- `SUSPEND: False` - CronJob is active

---

## Clean Up Old Jobs/Pods

### Delete Specific Job

```bash
kubectl delete job simple-job
```

**Result:** Job and its pods are deleted.

### Delete Jobs Created by CronJob

```bash
# Delete all jobs created by cronjob
kubectl delete job -l app=cronjob-demo

# Delete only completed jobs
kubectl delete job -l app=cronjob-demo --field-selector status.successful=1

# Delete only failed jobs
kubectl delete job -l app=cronjob-demo --field-selector status.failed=1
```

### Automatic Cleanup (CronJob)

CronJob automatically cleans up old jobs based on:

```yaml
spec:
  successfulJobsHistoryLimit: 3  # Keep last 3 successful jobs
  failedJobsHistoryLimit: 1       # Keep last 1 failed job
```

**Behavior:**
- Old successful jobs deleted automatically
- Old failed jobs deleted automatically
- Keeps only the specified number

### Delete CronJob

```bash
kubectl delete cronjob simple-cronjob
```

**Note:** Deleting CronJob does NOT delete its created Jobs. Delete them separately if needed.

---

## Common Commands

```bash
# Create job
kubectl apply -f simple-job.yaml

# Create cronjob
kubectl apply -f simple-cronjob.yaml

# List jobs
kubectl get jobs

# List cronjobs
kubectl get cronjobs

# Describe job
kubectl describe job <job-name>

# Describe cronjob
kubectl describe cronjob <cronjob-name>

# View job pods
kubectl get pods -l app=<label>

# View logs
kubectl logs -l app=<label>

# Delete job
kubectl delete job <job-name>

# Delete cronjob
kubectl delete cronjob <cronjob-name>

# Suspend cronjob
kubectl patch cronjob simple-cronjob -p '{"spec":{"suspend":true}}'

# Resume cronjob
kubectl patch cronjob simple-cronjob -p '{"spec":{"suspend":false}}'
```

---

## Key Differences: Job vs CronJob

| Feature | Job | CronJob |
|---------|-----|---------|
| **Purpose** | One-time task | Recurring task |
| **Schedule** | Runs immediately | Runs on schedule |
| **Creates** | Pods directly | Creates Jobs |
| **Completion** | Runs to completion | Creates new Job each run |
| **Use case** | Batch processing, one-off tasks | Scheduled backups, reports |

---

## Restart Policy

**Important:** Jobs require `restartPolicy: OnFailure` or `Never` (not `Always`).

```yaml
spec:
  template:
    spec:
      restartPolicy: OnFailure  # Restart on failure
      # OR
      restartPolicy: Never      # Never restart
```

**Why:** Jobs are meant to complete, not run indefinitely.

---

## Key Learnings

1. **completions:**
   - Number of successful pods needed
   - `1` = single task, `N` = parallel tasks

2. **backoffLimit:**
   - Retry attempts before marking as failed
   - Exponential backoff between retries

3. **schedule:**
   - Cron expression for recurring tasks
   - Format: `"minute hour day month weekday"`

4. **Job vs CronJob:**
   - Job: One-time execution
   - CronJob: Creates Jobs on schedule

5. **Cleanup:**
   - Jobs persist after completion
   - CronJob can auto-cleanup old jobs
   - Manual cleanup via `kubectl delete`

---

## Deliverables

**simple-job.yaml** - One-time Job that prints message and exits  
**simple-cronjob.yaml** - CronJob that runs every minute and logs timestamp  
**Verified** - Jobs and CronJobs created and executed successfully  
**Documentation** - Key fields (completions, backoffLimit, schedule) explained

---

## Summary

- **Job:** Runs task to completion, use for one-time or parallel tasks
- **CronJob:** Creates Jobs on schedule, use for recurring tasks
- **completions:** Number of successful pods needed
- **backoffLimit:** Retry attempts before failure
- **schedule:** Cron expression for timing

**Jobs and CronJobs enable batch processing and scheduled tasks in Kubernetes.**
