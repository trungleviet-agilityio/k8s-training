# Probes and Lifecycle

## Quick Reference

### Probe Types

| Probe Type | Purpose | Failure Action | When to Use |
|------------|---------|----------------|-------------|
| **Readiness** | Is container ready? | Remove from Service | Container needs initialization |
| **Liveness** | Is container alive? | Restart container | App can get stuck |
| **Startup** | Is container started? | Restart container | Slow-starting apps |

## Common Use Cases

### When to Use Each Probe Type

**Readiness Probe:**
- Container needs initialization time
- Database connections must be established
- Configuration must be loaded
- **Failure:** Pod removed from Service, container keeps running

**Liveness Probe:**
- Application can get stuck (deadlocks, infinite loops)
- Need automatic recovery
- **Failure:** Container restarted

**Startup Probe:**
- Slow-starting applications (Java apps, databases)
- Prevents premature restarts
- **Failure:** Container restarted, other probes disabled until startup succeeds

## Configuration Examples

### Probe Configuration (API Deployment Example)

**Readiness Probe:**
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5   # Short delay, route traffic quickly
  periodSeconds: 10
  failureThreshold: 3
```

**Liveness Probe:**
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 30  # Longer delay, avoid premature restarts
  periodSeconds: 10
  failureThreshold: 3
```

**Key Differences:**
- Readiness: Shorter delay (5s) to route traffic quickly
- Liveness: Longer delay (30s) to avoid unnecessary restarts
- Both use same endpoint for simplicity

**Common Pitfalls:**
- Same `initialDelaySeconds` for both → Causes premature restarts
- Too low `initialDelaySeconds` → Causes restart loops
- Too low `failureThreshold` → Causes unnecessary restarts
- Heavy probe endpoints → Impacts application performance

## Common Issues & Solutions

### Probe Failures

**Symptoms:** `READY: 0/1` (readiness), `RESTARTS` increasing (liveness)

**Commands:**
```bash
kubectl describe pod <pod-name> | grep -A 10 "Readiness\|Liveness"
kubectl exec <pod-name> -- curl http://localhost:8080/health  # Test endpoint
```

**Common causes:**
- Wrong probe path → Verify endpoint exists
- Wrong probe port → Check container port
- Application not responding → Check app logs
- Probe timing too aggressive → Increase `initialDelaySeconds` or `failureThreshold`

**Fix:**
```yaml
readinessProbe:
  httpGet:
    path: /ready  # Verify path exists
    port: 8080    # Verify port matches
  initialDelaySeconds: 10  # Increase if app starts slowly
  failureThreshold: 3       # Allow transient failures
```
