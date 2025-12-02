# M2-01 Imperative → Declarative Practice

**Goal:** Become fluent in converting imperative `kubectl` commands into declarative YAML quickly.

---

## Why Imperative → Declarative + Manual Fixes?

**kubectl generates the skeleton, you add the application logic.**

### What `kubectl` provides (speed):
- ✅ Correct `apiVersion` and `kind`
- ✅ Valid base structure and required fields
- ✅ Container image, ports, selectors
- ✅ Saves 20-30 seconds of typing boilerplate

### What `kubectl` cannot know (requires manual fixes):
- ❌ **Probes** - Your app needs `/healthz`, but kubectl doesn't know
- ❌ **Resources** - You decide the limits, not the CLI
- ❌ **Labels** - CLI generates minimal; exams require specific labels
- ❌ **Environment variables** - CLI cannot know what your app needs
- ❌ **Volume mounts** - Cannot infer volumes from container image
- ❌ **Advanced configs** - Affinity, init containers, custom strategies

**Summary:** Imperative generation = fast scaffolding. Manual fixes = application-specific requirements. This is the fastest, safest way for CKAD.

---

## 1. Deployment: web-deploy

**Command:**
```bash
kubectl create deployment web-deploy \
  --image=nginx:1.27 \
  --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml
```

**Manual fixes:**
- Removed `creationTimestamp: null` and `status: {}`
- Removed empty `strategy: {}` and `resources: {}`
- Updated labels to `app=web`
- Added `ports` section with `containerPort: 80`
- Added resource requests and limits
- Added `livenessProbe`

---

## 2. Service: web-svc

**Command:**
```bash
kubectl create service clusterip web-svc \
  --tcp=80:80 \
  --dry-run=client -o yaml > service.yaml
```

**Manual fixes:**
- Removed `creationTimestamp: null` and `status: {}`
- Removed auto-generated port name
- Ensured selector matches `app=web`

---

## 3. Pod: debug-pod

**Command:**
```bash
kubectl run debug-pod \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600 \
  --dry-run=client -o yaml > pod.yaml
```

**Manual fixes:**
- Removed `creationTimestamp: null` and `status: {}`
- Removed empty `resources: {}`
- Ensured command array format is correct

---

## 4. CronJob: hello-cron

**Command:**
```bash
kubectl create cronjob hello-cron \
  --image=busybox:1.36 \
  --schedule="*/5 * * * *" \
  --dry-run=client -o yaml \
  -- /bin/sh -c 'date; echo Hello from CronJob' \
  > cronjob.yaml
```

**Manual fixes:**
- Removed `creationTimestamp: null` and `status: {}`
- Removed unnecessary `jobTemplate` metadata
- Removed empty objects
- Quoted schedule properly

---

## 5. ConfigMap: app-config

**Command:**
```bash
kubectl create configmap app-config \
  --from-literal=MODE=dev \
  --from-literal=LOG_LEVEL=info \
  --dry-run=client -o yaml > configmap.yaml
```

**Manual fixes:**
- Removed `creationTimestamp: null`
- Reordered `apiVersion`, `kind`, `metadata`, `data`

---

## 6. ConfigMap: app-config-file

**Command:**
```bash
echo "MAX_RETRY=5" > app.env
kubectl create configmap app-config-file \
  --from-env-file=app.env \
  --dry-run=client -o yaml > configmap-file.yaml
```

**Manual fixes:**
- Removed `creationTimestamp: null`
- Reordered fields

---

## 7. Secret: app-secret

**Command:**
```bash
kubectl create secret generic app-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=supersecret \
  --dry-run=client -o yaml > secret.yaml
```

**Manual fixes:**
- Removed `creationTimestamp: null`
- Added `type: Opaque`
- Reordered `apiVersion`, `kind`, `metadata`, `type`, `data`

---

## Common Cleanup Patterns

**Remove:**
- `creationTimestamp: null`
- `status: {}`
- Empty objects (`resources: {}`, `strategy: {}`)
- Auto-generated nested metadata

**Fixes:**
- Ensure selectors match labels
- Add `containerPort` for deployments
- Add probes
- Add resources
- Quote cron schedules
- Add explicit Secret type

---

## Quick Reference

```bash
# Deployment
kubectl create deployment <name> --image=<image> --replicas=<n> --dry-run=client -o yaml

# Service
kubectl create service clusterip <name> --tcp=<port>:<targetPort> --dry-run=client -o yaml

# Pod
kubectl run <name> --image=<image> --restart=Never --dry-run=client -o yaml

# CronJob
kubectl create cronjob <name> --image=<image> --schedule="<cron>" --dry-run=client -o yaml

# ConfigMap (literal)
kubectl create configmap <name> --from-literal=<key>=<value> --dry-run=client -o yaml

# ConfigMap (file)
kubectl create configmap <name> --from-env-file=<file> --dry-run=client -o yaml

# Secret
kubectl create secret generic <name> --from-literal=<key>=<value> --dry-run=client -o yaml
```
