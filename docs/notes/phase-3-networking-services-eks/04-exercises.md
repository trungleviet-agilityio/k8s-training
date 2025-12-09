# Related Exercises

## Phase 3 Exercises

The following exercises are available in the `exercises/` directory to practice the concepts covered in this phase.

| Topic | Directory | Description |
|-------|-----------|-------------|
| **Services** | `exercises/services/` | ClusterIP and NodePort services for internal and external access. |
| **Ingress** | `exercises/ingress/` | Ingress resources for HTTP/HTTPS routing from outside the cluster. |
| **NetworkPolicies** | `exercises/networkpolicies/` | Network policies for traffic control and pod-to-pod communication restrictions. |
| **Deployments** | `exercises/deployments/` | Application deployments that work with Services and Ingress. |

## Recommended Practice Path

1. **Start with Services:** Deploy an application and expose it with a ClusterIP Service to understand internal service discovery.
2. **Add External Access:** Try NodePort for simple external access, then move to Ingress for production-like HTTP routing.
3. **Understand DNS:** Practice accessing services using service names, ClusterIPs, and FQDNs.
4. **Secure with NetworkPolicies:** Start with default deny-all, then add specific allow rules for required traffic.
5. **Test Traffic Control:** Verify NetworkPolicy enforcement by testing allowed and denied connections.
6. **Put it Together:** Deploy a complete stack (Deployment → Service → Ingress → NetworkPolicy) and test end-to-end.

## Lab Guides

### Lab 1: Services (Internal & External Access)

**Objective:** Expose an Nginx deployment internally using ClusterIP and externally using NodePort.

1. **Deploy Application:**
   ```bash
   kubectl apply -f exercises/deployments/nginx-deployment.yaml
   ```

2. **Create ClusterIP Service:**
   ```bash
   kubectl apply -f exercises/services/nginx-clusterip.yaml
   ```
   *   **Verify:** `kubectl get svc nginx-clusterip`
   *   **Test Internal Access:**
       ```bash
       # Create a temporary pod to test connectivity
       kubectl run curl-pod --image=curlimages/curl --restart=Never -- sleep infinity

       # Curl the service name (DNS resolution)
       kubectl exec -it curl-pod -- curl http://nginx-clusterip
       ```

3. **Create NodePort Service:**
   ```bash
   kubectl apply -f exercises/services/nginx-nodeport.yaml
   ```
   *   **Verify:** `kubectl get svc nginx-nodeport`
   *   **Test External Access:**
       *   **Minikube:** `minikube service nginx-nodeport --url` (or `curl $(minikube ip):30080`)
       *   **Kind/Local:** If using Kind, you might need `kubectl port-forward svc/nginx-nodeport 30080:80` then access `localhost:30080`.

### Lab 2: Ingress (HTTP Routing)

**Objective:** Expose the Nginx service via an Ingress Controller with path-based routing.

1. **Enable Ingress Controller (Minikube):**
   ```bash
   minikube addons enable ingress
   kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=90s
   ```

2. **Deploy Service & Ingress:**
   ```bash
   kubectl apply -f exercises/ingress/nginx-service.yaml
   kubectl apply -f exercises/ingress/nginx-ingress.yaml
   ```

3. **Verify Ingress:**
   ```bash
   kubectl get ingress nginx-ingress
   kubectl get endpoints nginx-service
   ```

4. **Test Access:**
   ```bash
   MINIKUBE_IP=$(minikube ip)
   NODEPORT=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.ports[0].nodePort}')
   curl http://$MINIKUBE_IP:$NODEPORT/
   ```
   **Expected:** Nginx welcome page HTML

### Lab 3: NetworkPolicies (Traffic Control)

**Objective:** Implement a "Default Deny" policy and selectively allow traffic.

**Prerequisites:** Ensure your cluster supports NetworkPolicies (Minikube with CNI plugin like Calico or Cilium).
*   *If using standard Minikube, NetworkPolicies might not be enforced.*
*   Start minikube with CNI: `minikube start --cni=calico`

1. **Deploy Test Pods:**
   ```bash
   kubectl apply -f exercises/networkpolicies/backend-pod.yaml
   kubectl apply -f exercises/networkpolicies/frontend-pod.yaml
   kubectl apply -f exercises/networkpolicies/other-pod.yaml
   ```

2. **Test Default Connectivity (Should be Allowed):**
   ```bash
   # Get backend pod IP
   BACKEND_IP=$(kubectl get pod backend-pod -o jsonpath='{.status.podIP}')

   # Test from frontend
   kubectl exec -it frontend-pod -- wget -qO- --timeout=2 http://$BACKEND_IP

   # Test from other
   kubectl exec -it other-pod -- wget -qO- --timeout=2 http://$BACKEND_IP
   ```

3. **Apply Default Deny:**
   ```bash
   kubectl apply -f exercises/networkpolicies/default-deny-all.yaml
   ```
   *   **Verify:** Repeat step 2. Both commands should TIMEOUT or fail.

4. **Apply Allow Rule:**
   ```bash
   kubectl apply -f exercises/networkpolicies/frontend-to-backend.yaml
   ```
   *   **Verify:**
       *   **Frontend:** `kubectl exec -it frontend-pod -- wget -qO- --timeout=2 http://$BACKEND_IP` -> **SUCCESS**
       *   **Other:** `kubectl exec -it other-pod -- wget -qO- --timeout=2 http://$BACKEND_IP` -> **FAIL/TIMEOUT**

## Troubleshooting

### Lab 2: Ingress Issues

**504 Gateway Timeout:**
```bash
# Check 1: Service has endpoints
kubectl get endpoints nginx-service

# Check 2: NetworkPolicy blocking Ingress Controller (MOST COMMON)
kubectl get networkpolicies
# If default-deny-all-ingress exists:
kubectl label namespace ingress-nginx name=ingress-nginx --overwrite
kubectl apply -f exercises/networkpolicies/allow-ingress-controller.yaml

# Check 3: Ingress Controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=20 | grep -i "timeout\|504"
```

**404 Not Found:**
```bash
kubectl describe ingress nginx-ingress
kubectl get ingress nginx-ingress -o yaml
```

**Connection Refused:**
```bash
# Verify NodePort and minikube IP
kubectl get svc -n ingress-nginx ingress-nginx-controller
minikube ip
```

## Cleanup
```bash
kubectl delete -f exercises/deployments/nginx-deployment.yaml
kubectl delete -f exercises/services/
kubectl delete -f exercises/ingress/
kubectl delete -f exercises/networkpolicies/
kubectl delete pod curl-pod
```
