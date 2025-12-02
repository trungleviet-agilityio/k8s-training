# Day 06 — ConfigMaps and Secrets

**Goal:** Practice using ConfigMaps and Secrets via env vars and volumes.

---

## What are ConfigMaps and Secrets?

- **ConfigMap:** Stores non-sensitive configuration data (key-value pairs)
- **Secret:** Stores sensitive data (passwords, tokens, keys) - base64 encoded

---

## ConfigMap Manifest

**File:** `exercises/configmaps-secrets/app-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-configmap
data:
  APP_MODE: dev
  LOG_LEVEL: debug
```

**Apply:**
```bash
kubectl apply -f exercises/configmaps-secrets/app-configmap.yaml
```

---

## Secret Manifest

**File:** `exercises/configmaps-secrets/db-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=                  # base64 encoded "admin"
  password: c2VjcmV0cGFzc3dvcmQxMjM=  # base64 encoded "secretpassword123"
```

**Apply:**
```bash
kubectl apply -f exercises/configmaps-secrets/db-secret.yaml
```

---

## Base64 Encoding for Secrets

Secrets must be base64-encoded in YAML manifests.

**Encode:**
```bash
echo -n "admin" | base64
# Output: YWRtaW4=

echo -n "secretpassword123" | base64
# Output: c2VjcmV0cGFzc3dvcmQxMjM=
```

**Decode (for verification):**
```bash
echo "YWRtaW4=" | base64 -d
# Output: admin
```

**Note:** Use `-n` flag with echo to avoid newline characters.

---

## Using ConfigMap and Secret in Pod

**File:** `exercises/configmaps-secrets/demo-pod.yaml`

### Method 1: envFrom (ConfigMap)

Loads **all keys** from ConfigMap as environment variables:

```yaml
envFrom:
- configMapRef:
    name: app-configmap
```

**Result:** `APP_MODE=dev`, `LOG_LEVEL=debug` available as env vars.

### Method 2: env with secretKeyRef (Secret)

Loads **specific keys** from Secret as environment variables:

```yaml
env:
- name: DB_USERNAME
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: username
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

**Result:** `DB_USERNAME=admin`, `DB_PASSWORD=secretpassword123` available as env vars.

### Method 3: Volume Mounts

Mount ConfigMap and Secret as files:

```yaml
volumeMounts:
- name: config-volume
  mountPath: /etc/config
  readOnly: true
- name: secret-volume
  mountPath: /etc/secret
  readOnly: true
volumes:
- name: config-volume
  configMap:
    name: app-configmap
- name: secret-volume
  secret:
    secretName: db-secret
```

**Result:** Files available at `/etc/config/APP_MODE`, `/etc/config/LOG_LEVEL`, `/etc/secret/username`, `/etc/secret/password`.

---

## Difference: env vs envFrom

### env (Individual keys)

```yaml
env:
- name: APP_MODE
  valueFrom:
    configMapKeyRef:
      name: app-configmap
      key: APP_MODE
```

**Use when:**
- Need specific keys only
- Want to rename env var names
- Mix ConfigMap and Secret keys

### envFrom (All keys)

```yaml
envFrom:
- configMapRef:
    name: app-configmap
```

**Use when:**
- Need all keys from ConfigMap/Secret
- Key names are already correct
- Simpler configuration

**Key difference:**
- `env`: Select specific keys, can rename
- `envFrom`: Load all keys, original names preserved

---

## Verify Configuration

### Check environment variables:

```bash
kubectl exec demo-pod -- env | grep -E "(APP_MODE|LOG_LEVEL|DB_USERNAME|DB_PASSWORD)"
```

**Output:**
```
APP_MODE=dev
LOG_LEVEL=debug
DB_USERNAME=admin
DB_PASSWORD=secretpassword123
```

### Check volume mounts:

```bash
# ConfigMap files
kubectl exec demo-pod -- ls -la /etc/config
kubectl exec demo-pod -- cat /etc/config/APP_MODE

# Secret files
kubectl exec demo-pod -- ls -la /etc/secret
kubectl exec demo-pod -- cat /etc/secret/username
```

---

## Common Commands

### ConfigMap

```bash
# Create from file
kubectl create configmap app-config --from-file=config.properties

# Create from literals
kubectl create configmap app-config --from-literal=APP_MODE=dev --from-literal=LOG_LEVEL=debug

# List ConfigMaps
kubectl get configmap

# Describe ConfigMap
kubectl describe configmap app-configmap

# View ConfigMap data
kubectl get configmap app-configmap -o yaml

# Edit ConfigMap
kubectl edit configmap app-configmap

# Delete ConfigMap
kubectl delete configmap app-configmap
```

### Secret

```bash
# Create from file
kubectl create secret generic db-secret --from-file=username.txt --from-file=password.txt

# Create from literals (auto base64-encoded)
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=secret123

# List Secrets
kubectl get secret

# Describe Secret (data hidden)
kubectl describe secret db-secret

# View Secret (base64 encoded)
kubectl get secret db-secret -o yaml

# Decode Secret value
kubectl get secret db-secret -o jsonpath='{.data.username}' | base64 -d

# Edit Secret
kubectl edit secret db-secret

# Delete Secret
kubectl delete secret db-secret
```

### Pod/Deployment

```bash
# Apply Pod with ConfigMap/Secret
kubectl apply -f demo-pod.yaml

# Check Pod status
kubectl get pod demo-pod

# View environment variables
kubectl exec demo-pod -- env

# View mounted files
kubectl exec demo-pod -- ls -la /etc/config
kubectl exec demo-pod -- cat /etc/config/APP_MODE

# View logs
kubectl logs demo-pod
```

---

## Key Learnings

1. **ConfigMap vs Secret:**
   - ConfigMap: Non-sensitive data (plain text)
   - Secret: Sensitive data (base64 encoded, but not encrypted)

2. **env vs envFrom:**
   - `env`: Select specific keys, can rename
   - `envFrom`: Load all keys, original names

3. **Base64 Encoding:**
   - Required for Secrets in YAML
   - Use `echo -n "value" | base64` to encode
   - Kubernetes decodes automatically when used

4. **Volume Mounts:**
   - ConfigMap/Secret mounted as files
   - Files update automatically when ConfigMap/Secret changes
   - Read-only by default

---

## Deliverables

**app-configmap.yaml** - ConfigMap with APP_MODE and LOG_LEVEL  
**db-secret.yaml** - Secret with username/password (base64)  
**demo-pod.yaml** - Pod using both via envFrom, secretKeyRef, and volumes  
**Verified** - Environment variables and volume mounts tested

---

## Summary

- ConfigMaps store non-sensitive configuration
- Secrets store sensitive data (base64 encoded)
- Use `envFrom` to load all keys, `env` for specific keys
- Mount as volumes for file-based access
- Base64 encoding required for Secrets in YAML manifests
