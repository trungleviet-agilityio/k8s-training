# Service Architecture in Kubernetes

```
                          +-----------------------------------+
                          |        Control Plane               |
                          |                                    |
                          |  • API Server                      |
                          |  • Scheduler                       |
                          |  • Controller Manager              |
                          |  • etcd                            |
                          +----------------+-------------------+
                                           |
                                           | Service YAML
                                           | (stored in etcd)
                                           |
                          +----------------v-------------------+
                          |            Service                 |
                          |                                    |
                          |  Type: ClusterIP/NodePort/LB       |
                          |  Selector: app=nginx               |
                          |  Port: 80 → targetPort: 80         |
                          +----------------+-------------------+
                                           |
                                           | Creates
                                           | Endpoints
                                           |
                                           v
        ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
        │   Worker Node #1    │  │   Worker Node #2    │  │   Worker Node #3    │
        │                     │  │                     │  │                     │
        │  • kubelet          │  │  • kubelet          │  │  • kubelet          │
        │  • kube-proxy ◄─────┼──┼──► kube-proxy ◄─────┼──┼──► kube-proxy       │
        │                     │  │                     │  │                     │
        │  Pod IPs:           │  │  Pod IPs:           │  │  Pod IPs:           │
        │  10.244.1.10        │  │  10.244.2.10        │  │  10.244.3.10        │
        │  10.244.1.11        │  │  10.244.2.11        │  │                     │
        └─────────────────────┘  └─────────────────────┘  └─────────────────────┘
                 │                        │                        │
                 │                        │                        │
                 └────────────────────────┼────────────────────────┘
                                          │
                                          │ Service IP
                                          │ (e.g., 10.96.0.1)
                                          │
                          +---------------v---------------+
                          |      Service Endpoints         |
                          |                                |
                          |  10.244.1.10:80                |
                          |  10.244.1.11:80                |
                          |  10.244.2.10:80                |
                          |  10.244.2.11:80                |
                          |  10.244.3.10:80                |
                          +--------------------------------+
```

## How It Works

1. **Control Plane**: Stores Service definition in etcd via API Server
2. **Service**: Defines selector and port mapping
3. **Endpoints Controller**: Automatically creates Endpoints object with matching Pod IPs
4. **kube-proxy**: Running on each node, watches Service and Endpoints, updates iptables/ipvs rules
5. **Traffic Flow**: Requests to Service IP are load-balanced to Pod IPs via kube-proxy rules
