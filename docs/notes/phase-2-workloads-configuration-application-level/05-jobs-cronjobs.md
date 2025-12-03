# Jobs and CronJobs

## Workload Types

### Workload Categories

**Long-running Workloads:**
- Web servers, applications, databases
- Run continuously until manually stopped
- Use: Deployments, ReplicaSets

**Batch Processing Workloads:**
- Computation, analytics, reporting, image processing
- Perform specific task and finish
- Short-lived, run to completion
- Use: Jobs, CronJobs

## Comparison

### Jobs vs ReplicaSets

| Feature | ReplicaSet | Job |
|---------|------------|-----|
| **Purpose** | Keep pods running | Run pods to completion |
| **Behavior** | Ensures specified number of pods running at all times | Creates pods to perform task, then stops |
| **Use case** | Web servers, databases | Batch processing, analytics, reporting |
| **Restart policy** | Always (default) | Never or OnFailure |

**Key difference:** ReplicaSet maintains running pods, Job runs pods until task completion.

## Concepts

### What are CronJobs

**CronJobs are scheduled jobs** (like Crontab in Linux):
- Run Jobs periodically based on a schedule
- Create Jobs automatically on schedule
- Use Cron-like format for scheduling

**Use Cases:**
- Generate report and send email periodically
- Scheduled backups, maintenance tasks
- Periodic data processing, analytics

**How it works:**
- CronJob creates a Job on schedule
- Job runs pods to completion
- CronJob can create multiple Jobs over time

## Decision Guide

### When to Use Job vs CronJob

**Use Job when:**
- One-time tasks (batch processing, data migration, cleanup)
- Tasks that need to run to completion
- Parallel processing (multiple pods working on same task)
- Manual execution needed

**Use CronJob when:**
- Scheduled/recurring tasks (backups, reports, maintenance)
- Tasks that run on a schedule (hourly, daily, weekly)
- Automated periodic work
- Need to create Jobs automatically

**Decision:**
```
Need to run on schedule?
├─ No → Use Job
└─ Yes → Use CronJob
```

## Configuration

### Completions and Parallelism

**completions:**
- Number of successful completions needed (default: 1)
- Example: `completions: 3` means job needs 3 successful pods
- Use for: Large datasets requiring multiple pods to process data

**parallelism:**
- Number of pods to run simultaneously (default: 1, sequential)
- Example: `parallelism: 3` creates 3 pods at once
- Use for: Parallel processing to speed up batch jobs

**Sequential Execution (default):**
- Pods created one after another
- Second pod starts only after first completes
- Use when: Tasks must run in order or have dependencies

**Parallel Execution:**
- Multiple pods created at once
- All pods run simultaneously
- Use when: Tasks are independent and can run in parallel

**Failure Handling:**
- Job creates new pods until it reaches desired `completions`
- If pod fails, job creates another pod automatically
- Continues until required number of successful completions
- Example: If `completions: 3` and one pod fails, job creates new pod until 3 succeed

**Example:**
```yaml
apiVersion: batch/v1
kind: Job
spec:
  completions: 3      # Need 3 successful pods
  parallelism: 2      # Run 2 pods at a time
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: job-container
        image: busybox:1.36
        command: ["sh", "-c", "echo 'Processing data'"]
```

### Job Configuration Example

```yaml
apiVersion: batch/v1
kind: Job
spec:
  completions: 1        # Number of successful completions needed
  backoffLimit: 3       # Retry attempts before marking as failed
  template:
    spec:
      restartPolicy: OnFailure  # OnFailure or Never (not Always)
      containers:
      - name: job-container
        image: busybox:1.36
        command: ["sh", "-c", "echo 'Job completed'"]
```

### CronJob Configuration Example

**Structure:** CronJob has three nested spec sections (be careful with nesting):
1. CronJob spec (schedule, jobTemplate)
2. Job spec (under jobTemplate)
3. Pod spec (under job template)

```yaml
apiVersion: batch/v1  # Current version (was batch/v1beta1 in older K8s)
kind: CronJob
spec:
  # Schedule uses Cron-like format: "minute hour day month weekday"
  schedule: "0 * * * *"  # Every hour at minute 0
  # Examples:
  # "*/5 * * * *" - Every 5 minutes
  # "0 0 * * *" - Every day at midnight
  # "0 9 * * 1-5" - Every weekday at 9 AM
  successfulJobsHistoryLimit: 3  # Keep last 3 successful jobs
  failedJobsHistoryLimit: 1     # Keep last 1 failed job
  concurrencyPolicy: Forbid      # Don't run concurrent jobs
  # CronJob spec -> Job template (second spec level)
  jobTemplate:
    spec:
      backoffLimit: 2
      # Job spec -> Pod template (third spec level)
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cronjob-container
            image: busybox:1.36
            command: ["sh", "-c", "echo 'CronJob executed at $(date)'"]
```

**Schedule Format:** `"minute hour day month weekday"`
- `*` = any value
- `*/N` = every N units
- `N-M` = range
- `N,M` = specific values

**Examples:** `exercises/jobs-cronjobs/simple-job.yaml`, `exercises/jobs-cronjobs/simple-cronjob.yaml`

## Operations & Troubleshooting

### Inspection Commands

```bash
# View Jobs
kubectl get jobs
kubectl get jobs -l app=job-demo
kubectl describe job <job-name>

# View CronJobs
kubectl get cronjobs
kubectl get cronjobs -l app=cronjob-demo
kubectl describe cronjob <cronjob-name>

# View Jobs created by CronJob
kubectl get jobs -l app=cronjob-demo
kubectl get jobs --field-selector metadata.ownerReferences.kind=CronJob

# View Pods created by Jobs
kubectl get pods -l app=job-demo
kubectl get pods --field-selector status.phase=Succeeded
kubectl get pods --field-selector status.phase=Failed

# View Job logs
kubectl logs -l app=job-demo
kubectl logs <pod-name>  # Pod created by Job

# View CronJob history
kubectl get jobs -l app=cronjob-demo --sort-by=.metadata.creationTimestamp

# Cleanup old Jobs
kubectl delete job -l app=job-demo --field-selector status.successful=1
kubectl delete job -l app=job-demo --field-selector status.failed=1

# Suspend/Resume CronJob
kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'
kubectl patch cronjob <name> -p '{"spec":{"suspend":false}}'
```

### Key Pitfalls

**1. Too Many Failed Jobs**
- Problem: Job keeps failing, creating many failed pods
- Cause: Low `backoffLimit`, wrong `restartPolicy`, application errors
- Fix: Increase `backoffLimit`, check logs, fix application
- Check: `kubectl get jobs` shows many failed jobs

**2. History Limits**
- Problem: Old jobs accumulate, consuming resources
- Cause: `successfulJobsHistoryLimit` or `failedJobsHistoryLimit` too high
- Fix: Set appropriate limits (3-5 for successful, 1-2 for failed)
- Cleanup: `kubectl delete job -l app=my-app --field-selector status.successful=1`

**3. Restart Policy Requirements**
- Problem: Job fails with "restartPolicy must be OnFailure or Never"
- Cause: Using `restartPolicy: Always` (not allowed for Jobs)
- Why: Default pod behavior is `restartPolicy: Always` (keeps containers running forever). Jobs are meant to complete tasks and finish, not run indefinitely.
- Fix: Use `restartPolicy: OnFailure` or `Never` in job template
- Note: Jobs are meant to complete, not run indefinitely

**4. Schedule Syntax Errors**
- Problem: CronJob doesn't run, schedule invalid
- Cause: Wrong cron syntax format
- Fix: Use format `"minute hour day month weekday"`
- Examples:
  - `"*/5 * * * *"` - Every 5 minutes
  - `"0 * * * *"` - Every hour
  - `"0 0 * * *"` - Every day at midnight
  - `"0 9 * * 1-5"` - Every weekday at 9 AM

**5. Concurrent Jobs**
- Problem: Multiple CronJob instances running simultaneously
- Cause: Job takes longer than schedule interval
- Fix: Set `concurrencyPolicy: Forbid` in CronJob spec
- Check: `kubectl get jobs -l app=my-app` shows multiple active jobs
