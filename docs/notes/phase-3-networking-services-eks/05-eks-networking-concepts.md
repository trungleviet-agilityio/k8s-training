# EKS Networking Concepts

This document provides a deep dive into Amazon EKS networking, explaining how the VPC CNI plugin works, how pods get IP addresses, and how the AWS Load Balancer Controller maps Ingress to ALBs.

## 1. AWS VPC CNI & Pod Networking

### Concept: Flat Networking
Amazon EKS uses the **Amazon VPC CNI plugin** for Kubernetes networking. Unlike standard Kubernetes overlay networks (like Flannel or Calico's default mode), the VPC CNI connects pods **directly** to the AWS VPC network.

*   **Pods receive first-class VPC IP addresses.**
*   Pods can communicate directly with other VPC resources (RDS, ElastiCache, EC2) without NAT.
*   The IP visibility makes debugging and security group configuration simpler.

### How IP Allocation Works (ipamd)
The CNI plugin runs a daemon called `ipamd` (part of the `aws-node` DaemonSet) on every worker node.

1.  **Warm Pool:** `ipamd` maintains a "warm pool" of available IP addresses on the node to ensure fast pod startup.
2.  **ENI Attachment:** When the pool is exhausted, `ipamd` calls the EC2 API to attach a new **Elastic Network Interface (ENI)** to the node.
3.  **IP Assignment:** When a pod is scheduled, the CNI assigns it a free IP from one of the attached ENIs.

### Architecture Diagram: Pod IP Allocation

```ascii
+---------------------------------------------------------------+
|                        Worker Node (EC2)                      |
|                                                               |
|   +-------------------+       +---------------------------+   |
|   |   Primary ENI     |       |      Secondary ENI        |   |
|   |   (Node IP)       |       |    (Pod IP Pool)          |   |
|   |   IP: 10.0.1.10   |       |    IPs: 10.0.1.11-20      |   |
|   +---------+---------+       +-------------+-------------+   |
|             |                               |                 |
|             | route                         | assigns         |
|             v                               v                 |
|   +-------------------+       +---------------------------+   |
|   |   Host Network    |       |         Pod 1             |   |
|   |   Namespace       |       | IP: 10.0.1.11 (VPC IP)    |   |
|   +-------------------+       +---------------------------+   |
|                                                               |
+---------------------------------------------------------------+
              |                               |
              |                               |
+-------------v-------------------------------v------------------+
|                  VPC Subnet (10.0.1.0/24)                     |
+---------------------------------------------------------------+
```

### Key Takeaways
*   **Pod Density:** The maximum number of pods per node is limited by the number of ENIs and IPs per ENI supported by the EC2 instance type.
    *   *Formula:* `(Max ENIs * (IPs per ENI - 1)) + 2`
*   **Subnet Sizing:** Subnets must be large enough to accommodate all pods, not just nodes.

---

## 2. Subnet Architecture (Public vs. Private)

EKS clusters typically use a split-subnet architecture for security and functionality.

### Network Layout Example

**VPC Structure:**
- **VPC CIDR:** `10.0.0.0/16`
- **Public Subnets:**
  - `10.0.1.0/24` (ap-southeast-1a)
  - `10.0.4.0/24` (ap-southeast-1b)
  - *Contains:* NAT Gateway, Internet Gateway, Public Node Group
- **Private Subnets:**
  - `10.0.2.0/24` (ap-southeast-1a)
  - `10.0.3.0/24` (ap-southeast-1b)
  - *Contains:* Private Node Group, Pods

### Public Subnets
*   **Connectivity:** Have a route to an **Internet Gateway (IGW)**.
*   **Resources:**
    *   **Public Load Balancers:** Internet-facing ALBs or NLBs.
    *   **NAT Gateways:** Allow outbound internet access for private resources.
    *   **Bastion Hosts:** For SSH access (optional).

### Private Subnets
*   **Connectivity:** No direct route to IGW. Outbound traffic goes through a **NAT Gateway** in a public subnet.
*   **Resources:**
    *   **Worker Nodes:** Security best practice is to place all worker nodes here.
    *   **Pods:** Pods running on these nodes get private IPs.
    *   **Internal Load Balancers:** For internal service-to-service communication.

### Traffic Flow Diagram

```ascii
       Internet
          ^
          |
    +---------v----------+
    |   Internet Gateway |
    +---------+----------+
              |
              | (Public Subnet)
    +---------v---------------------------------+
    |  +-------------+      +----------------+  |
    |  | Public ALB  |      |  NAT Gateway   |  |
    |  +------+------+      +-------+--------+  |
    |         |                     ^           |
    +---------|---------------------|-----------+
              | Ingress             | Outbound
              v                     |
    +-------------------------------|-----------+
    |  (Private Subnet)             |           |
    |                               |           |
    |      +------------------------+           |
    |      |                                    |
    |  +---+-----------+                        |
    |  |  Worker Node  |                        |
    |  |  +---------+  |                        |
    |  |  |   Pod   |  |                        |
    |  |  +---------+  |                        |
    |  +---------------+                        |
    +-------------------------------------------+
```

---

## 3. AWS Load Balancer Controller (Ingress)

The **AWS Load Balancer Controller** is a controller that watches Kubernetes API resources and provisions AWS Elastic Load Balancers.

### Mapping Concepts

| Kubernetes Resource | AWS Resource | OSI Layer | Usage |
|---------------------|--------------|-----------|-------|
| **Ingress** | **Application Load Balancer (ALB)** | Layer 7 | HTTP/HTTPS routing, SSL termination, path-based routing. |
| **Service** (LoadBalancer) | **Network Load Balancer (NLB)** | Layer 4 | TCP/UDP traffic, high performance, static IPs. |

### Target Types: Instance vs. IP

When configuring Ingress (ALB), you can choose how traffic reaches your pods.

#### 1. Instance Mode (Legacy/Default)
*   ALB sends traffic to the **NodePort** on the worker node.
*   `kube-proxy` (iptables) forwards traffic to the Pod.
*   *Downside:* Extra network hop, potentially uneven load balancing.

#### 2. IP Mode (Recommended for EKS)
*   **Requires:** VPC CNI (which provides real VPC IPs to pods).
*   ALB sends traffic **directly to the Pod IP**.
*   *Benefit:* Lower latency, removes `kube-proxy` hop, better visibility.
*   *Annotation:* `alb.ingress.kubernetes.io/target-type: ip`

### Architecture Diagram: ALB IP Mode

```ascii
      +---------------------------+
      |   Application Load        |
      |   Balancer (ALB)          |
      +-------------+-------------+
                    |
                    | Traffic (Direct to Pod IP)
                    | Target Group: IP Mode
                    v
      +-----------------------------------------+
      |             VPC                         |
      |                                         |
      |  +-----------------------------------+  |
      |  |        Worker Node                |  |
      |  |                                   |  |
      |  |  +-----------+   +-----------+    |  |
      |  |  |   Pod A   |   |   Pod B   |    |  |
      |  |  | 10.0.2.5  |   | 10.0.2.6  |    |  |
      |  |  +-----------+   +-----------+    |  |
      |  |                                   |  |
      |  +-----------------------------------+  |
      +-----------------------------------------+
```
