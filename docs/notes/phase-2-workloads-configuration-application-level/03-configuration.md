# Configuration: ConfigMaps & Secrets

## Quick Reference

### ConfigMap vs Secret

| Criteria | ConfigMap | Secret |
|----------|-----------|--------|
| **Data type** | Non-sensitive | Sensitive |
| **Encoding** | Plain text | Base64 encoded |
| **Examples** | App config, env vars | Passwords, tokens, keys |
| **Git commit** | OK | Never |

### Injection Methods

| Method | Use Case | Example |
|--------|----------|---------|
| **env (individual)** | Specific keys, rename | `valueFrom.configMapKeyRef` |
| **envFrom (all keys)** | All keys needed | `envFrom.configMapRef` |
| **Volume mounts** | Files, hot reload | Mount at `/etc/config` |

## Common Use Cases

### When to Use ConfigMap vs Secret

**Use ConfigMap for:**
- Non-sensitive config (APP_MODE, LOG_LEVEL, API_ENDPOINT)
- Configuration files
- Environment variables

**Use Secret for:**
- Passwords, tokens, API keys
- Database credentials
- TLS certificates

**Note:** Secrets are base64 encoded, NOT encrypted. Use external secret management for production.

## Essential Commands

### ConfigMap & Secret Operations

```bash
# Create ConfigMap
kubectl create configmap <name> --from-literal=KEY=value
kubectl create configmap <name> --from-file=config.properties
kubectl apply -f configmap.yaml

# Create Secret
kubectl create secret generic <name> --from-literal=KEY=value
kubectl create secret generic <name> --from-file=username.txt
echo -n "value" | base64  # Encode for YAML

# View ConfigMap/Secret
kubectl get configmap <name> -o yaml
kubectl get secret <name> -o yaml
kubectl describe configmap <name>
kubectl describe secret <name>

# Decode Secret value
kubectl get secret <name> -o jsonpath='{.data.KEY}' | base64 -d
```

## Common Issues & Solutions

### Configuration Not Working

**Symptoms:** Missing env vars, wrong values, files not mounted

**Commands:**
```bash
# Check env vars
kubectl exec <pod-name> -- env | grep KEY

# Check ConfigMap/Secret
kubectl get configmap <name> -o yaml
kubectl get secret <name> -o yaml

# Check mounted volumes
kubectl exec <pod-name> -- ls -la /etc/config
kubectl exec <pod-name> -- cat /etc/config/<file>

# Check pod spec
kubectl get pod <pod-name> -o yaml | grep -A 20 "env\|volumeMounts"
```

**Common causes:**
- ConfigMap/Secret not created → `kubectl get configmap <name>`
- Wrong key names → Verify keys match in ConfigMap/Secret and pod spec
- Volume mount path incorrect → Check `mountPath` in pod spec
- Env var not set → Verify `env` or `envFrom` in container spec

**Fix:**
```yaml
# Verify ConfigMap exists
kubectl get configmap app-config

# Verify keys match
kubectl get configmap app-config -o jsonpath='{.data}'

# In pod spec, ensure key names match
env:
- name: APP_MODE
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_MODE  # Must match key in ConfigMap
```
