# NetworkPolicies: Basic Traffic Control

**Goal:** Get a first feel for securing Pod-to-Pod communication.

---

## NetworkPolicy Requirements

**Important:** NetworkPolicies require a CNI plugin that supports them.

### For Minikube

Minikube's default CNI (bridge) **does NOT support NetworkPolicies**.

**To Enable NetworkPolicy Enforcement:**

**Start Minikube with Calico CNI:**
```bash
minikube stop
minikube start --cni=calico
```

**Verify Calico is running:**
```bash
kubectl get pods -n kube-system | grep calico
```

**Expected output:**
```
calico-kube-controllers-xxx   1/1     Running
calico-node-xxx                1/1     Running
```

**Once Calico is installed, NetworkPolicies will be enforced!**

---

## Default Deny-All Ingress NetworkPolicy

**File:** `exercises/networkpolicies/default-deny-all.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress traffic
```

**Key components:**
- `podSelector: {}` - Matches all pods (empty selector)
- `policyTypes: [Ingress]` - Only controls ingress traffic
- No `ingress` rules - Denies all incoming traffic

**Apply:**
```bash
kubectl apply -f exercises/networkpolicies/default-deny-all.yaml
```

**Verify:**
```bash
kubectl get networkpolicies
```

**Expected output:**
```
NAME                       POD-SELECTOR   AGE
default-deny-all-ingress   <none>         4s
```

---

## Allow Frontend to Backend NetworkPolicy

**File:** `exercises/networkpolicies/frontend-to-backend.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

**Key components:**
- `podSelector.matchLabels.app: backend` - Applies to backend pods
- `ingress.from.podSelector.matchLabels.app: frontend` - Allows traffic from frontend pods
- `ports` - Allows TCP port 80

**Apply:**
```bash
kubectl apply -f exercises/networkpolicies/frontend-to-backend.yaml
```

---

## Test Pods

### Backend Pod

**File:** `exercises/networkpolicies/backend-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  labels:
    app: backend
spec:
  containers:
  - name: backend
    image: nginx:1.25
    ports:
    - containerPort: 80
```

### Frontend Pod

**File:** `exercises/networkpolicies/frontend-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend
spec:
  containers:
  - name: frontend
    image: busybox:1.36
    command: ["sleep", "3600"]
```

### Other Pod (for testing denial)

**File:** `exercises/networkpolicies/other-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: other-pod
  labels:
    app: other
spec:
  containers:
  - name: other
    image: busybox:1.36
    command: ["sleep", "3600"]
```

---

## How podSelector Works

### podSelector in spec

```yaml
spec:
  podSelector:
    matchLabels:
      app: backend
```

**Meaning:** This NetworkPolicy applies to pods matching the selector.

**Examples:**
- `podSelector: {}` - Matches ALL pods (empty selector)
- `podSelector.matchLabels.app: backend` - Matches pods with `app=backend` label
- `podSelector.matchLabels.app: backend, tier: api` - Matches pods with both labels

**Key point:** NetworkPolicy applies to pods selected by `podSelector`.

---

## How from Field Works

### from in ingress rules

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
```

**Meaning:** Allows traffic FROM pods matching the selector.

**Examples:**
- `from: []` - No sources allowed (deny all)
- `from.podSelector.matchLabels.app: frontend` - Allow from pods with `app=frontend`
- Multiple `from` entries = OR logic (allow from any match)

**Key point:** `from` defines which pods can send traffic TO the selected pods.

---

## Testing Connectivity

### Before Applying NetworkPolicies

**Setup:**
```bash
# Create test pods
kubectl apply -f exercises/networkpolicies/backend-pod.yaml
kubectl apply -f exercises/networkpolicies/frontend-pod.yaml
kubectl apply -f exercises/networkpolicies/other-pod.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod backend-pod --timeout=30s
kubectl wait --for=condition=ready pod frontend-pod --timeout=30s
kubectl wait --for=condition=ready pod other-pod --timeout=30s
```

**Test connectivity:**
```bash
# Get backend IP
BACKEND_IP=$(kubectl get pod backend-pod -o jsonpath='{.status.podIP}')

# Test from frontend pod
kubectl exec frontend-pod -- wget -q -O- --timeout=3 http://$BACKEND_IP

# Test from other pod
kubectl exec other-pod -- wget -q -O- --timeout=3 http://$BACKEND_IP
```

**Expected result:** Both connections succeed (no policies yet).

### After Applying Default Deny-All

**Apply policy:**
```bash
kubectl apply -f exercises/networkpolicies/default-deny-all.yaml
```

**Test connectivity:**
```bash
# Test from frontend pod
kubectl exec frontend-pod -- wget -q -O- --timeout=3 http://$BACKEND_IP

# Test from other pod
kubectl exec other-pod -- wget -q -O- --timeout=3 http://$BACKEND_IP
```

**Expected result:** Both connections fail (all ingress denied).

**Actual result (with Calico CNI):** Both connections are BLOCKED ✅ (policy enforced!)

### After Applying Frontend-to-Backend Policy

**Apply policy:**
```bash
kubectl apply -f exercises/networkpolicies/frontend-to-backend.yaml
```

**Test connectivity:**
```bash
# Test from frontend pod (should succeed)
kubectl exec frontend-pod -- wget -q -O- --timeout=3 http://$BACKEND_IP

# Test from other pod (should fail)
kubectl exec other-pod -- wget -q -O- --timeout=3 http://$BACKEND_IP
```

**Expected result:**
- Frontend → Backend: **Succeeds** ✅ (allowed by policy)
- Other → Backend: **Fails** ❌ (not allowed by policy)

**Actual result (with Calico CNI):**
- Frontend → Backend: **Succeeds** ✅ (policy allows it)
- Other → Backend: **BLOCKED** ❌ (correctly denied by policy)

**NetworkPolicies are now being enforced!**

---

## What Happened: Before and After

### Before NetworkPolicies

**Behavior:**
- All pods can communicate freely
- No restrictions on Pod-to-Pod traffic
- Default: **Allow all**

**Test results:**
- Frontend pod → Backend pod: ✅ **Connected**
- Other pod → Backend pod: ✅ **Connected**

### After Default Deny-All Policy

**Behavior:**
- All ingress traffic denied
- Pods cannot receive traffic from any source
- Default: **Deny all ingress**

**Test results (with Calico CNI):**
- Frontend pod → Backend pod: ❌ **BLOCKED** (policy enforced)
- Other pod → Backend pod: ❌ **BLOCKED** (policy enforced)

### After Frontend-to-Backend Policy

**Behavior:**
- Backend pods can receive traffic ONLY from frontend pods
- Other pods still blocked
- Specific allow rule overrides default deny

**Test results (with Calico CNI):**
- Frontend pod → Backend pod: ✅ **ALLOWED** (policy allows it)
- Other pod → Backend pod: ❌ **BLOCKED** (correctly denied by policy)

**NetworkPolicies are working correctly!**

---

## NetworkPolicy Structure

### Complete Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
spec:
  # Which pods this policy applies to
  podSelector:
    matchLabels:
      app: backend
  # What types of traffic to control
  policyTypes:
  - Ingress  # Incoming traffic
  - Egress   # Outgoing traffic (optional)
  # Ingress rules
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
  # Egress rules (if policyTypes includes Egress)
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

---

## Key Fields Explained

### podSelector

**Location:** `spec.podSelector`

**Purpose:** Selects which pods this NetworkPolicy applies to.

**Values:**
- `{}` - Empty selector matches ALL pods
- `matchLabels` - Match pods with specific labels
- `matchExpressions` - Match pods using expressions

**Example:**
```yaml
podSelector:
  matchLabels:
    app: backend
    tier: api
```

### from (in ingress)

**Location:** `spec.ingress[].from[]`

**Purpose:** Defines sources allowed to send traffic.

**Options:**
- `podSelector` - Allow from pods matching selector
- `namespaceSelector` - Allow from namespaces matching selector
- `ipBlock` - Allow from specific IP ranges

**Example:**
```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
  - namespaceSelector:
      matchLabels:
        name: production
```

**Logic:** Multiple entries = OR (allow if matches any)

### to (in egress)

**Location:** `spec.egress[].to[]`

**Purpose:** Defines destinations allowed to receive traffic.

**Options:** Same as `from` (podSelector, namespaceSelector, ipBlock)

---

## Common Commands

```bash
# Create NetworkPolicy
kubectl apply -f default-deny-all.yaml

# List NetworkPolicies
kubectl get networkpolicies
kubectl get netpol  # Short form

# Describe NetworkPolicy
kubectl describe networkpolicy default-deny-all-ingress

# View NetworkPolicy YAML
kubectl get networkpolicy default-deny-all-ingress -o yaml

# Delete NetworkPolicy
kubectl delete networkpolicy default-deny-all-ingress

# Test connectivity
kubectl exec frontend-pod -- wget -q -O- http://<backend-ip>
kubectl exec frontend-pod -- curl http://<backend-ip>
```

---

## NetworkPolicy Best Practices

1. **Start with deny-all:** Apply default deny-all, then add specific allows
2. **Use labels consistently:** NetworkPolicies rely on pod labels
3. **Test thoroughly:** Verify policies work as expected
4. **Document policies:** Keep track of what each policy does
5. **Use namespaces:** Isolate policies by namespace when possible

---

## Key Learnings

1. **podSelector:**
   - Selects which pods the policy applies to
   - Empty `{}` = all pods
   - Uses label matching

2. **from field:**
   - Defines allowed traffic sources
   - Can use podSelector, namespaceSelector, or ipBlock
   - Multiple entries = OR logic

3. **Default behavior:**
   - No NetworkPolicy = Allow all
   - NetworkPolicy with no rules = Deny all
   - Specific rules override defaults

4. **Policy evaluation:**
   - Policies are additive (if any policy allows, traffic allowed)
   - More specific policies take precedence
   - Deny-all + allow rules = only allowed traffic passes

---

## Deliverables

**default-deny-all.yaml** - NetworkPolicy denying all ingress traffic  
**frontend-to-backend.yaml** - NetworkPolicy allowing frontend to backend  
**Test pods** - backend-pod, frontend-pod, other-pod for testing  
**Documentation** - podSelector and from fields explained, before/after behavior documented

---

## Summary

- **podSelector:** Selects pods the policy applies to
- **from:** Defines allowed traffic sources
- **Default:** No policy = allow all, policy with no rules = deny all
- **Testing:** Verify connectivity before and after applying policies
- **CNI Requirement:** NetworkPolicies require a CNI plugin that supports them (Calico, Cilium, etc.)
- **Minikube Setup:** Use `minikube start --cni=calico` to enable NetworkPolicy enforcement

**NetworkPolicies enable fine-grained control over Pod-to-Pod communication when using a supporting CNI plugin like Calico.**
