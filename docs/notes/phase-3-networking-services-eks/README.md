# Phase 3: Networking, Services & AWS EKS Basics

This directory contains detailed documentation for Phase 3, focused on Kubernetes networking, Services, Ingress, NetworkPolicies, and AWS EKS fundamentals.

## Documentation Index

### Kubernetes Networking (Local Cluster)

1. [**Services**](./01-services.md)
   - Service Types (ClusterIP, NodePort, LoadBalancer)
   - DNS-based Service Discovery
   - Port Configuration (port, targetPort, nodePort)
   - Common Issues & Solutions
   - Essential Commands Reference

2. [**Ingress**](./02-ingress.md)
   - Services vs Ingress Evolution
   - Ingress Controller vs Ingress Resources
   - Path-based and Host-based Routing
   - Multiple Rules and Paths
   - API Version Changes (networking.k8s.io/v1)
   - Troubleshooting Common Issues

3. [**NetworkPolicies**](./03-networkpolicies.md)
   - Traffic Direction Fundamentals (Ingress vs Egress)
   - Default Allow vs Default Deny
   - Pod, Namespace, and IP Block Selectors
   - Rule Logic (OR vs AND)
   - Real-world Examples
   - CNI Support Requirements

4. [**Exercises**](./04-exercises.md)
   - List of related practical exercises
   - Lab guides for Services, Ingress, and NetworkPolicies
   - Step-by-step testing instructions
   - Troubleshooting guide

### AWS EKS

5. [**EKS Networking Concepts**](./05-eks-networking-concepts.md)
   - VPC CNI & Pod Networking (Flat Network)
   - IP Allocation & ENIs (ipamd)
   - Subnet Architecture (Public vs Private)
   - AWS Load Balancer Controller (Ingress → ALB mapping)
   - Architecture Diagrams

6. [**EKS Setup Guide**](./06-eks-setup-guide.md)
   - Prerequisites and Installation
   - Step-by-step cluster creation using `eksctl`
   - Cluster monitoring and verification
   - Comprehensive Troubleshooting
   - Common `eksctl` Commands
   - Cleanup instructions

7. [**AWS Load Balancer Controller**](./07-alb-controller.md)
   - Controller installation with Helm
   - IAM Roles for Service Accounts (IRSA) setup
   - Automatic ALB provisioning from Ingress resources
   - IP Target Mode vs Instance Mode
   - Common annotations and configuration
   - Testing and troubleshooting

8. [**Helm - Kubernetes Package Manager**](./08-helm.md)
   - What is Helm and why use it
   - Core concepts (Charts, Releases, Repositories, Values)
   - Installation guide (Helm 3 and Helm 4)
   - Essential commands reference
   - Practical examples and use cases
   - Troubleshooting common issues

## Quick Start

**For Local Kubernetes Networking:**
1. Start with [Services](./01-services.md) to understand service discovery
2. Move to [Ingress](./02-ingress.md) for HTTP routing
3. Practice with [Exercises](./04-exercises.md)

**For AWS EKS:**
1. Review [EKS Networking Concepts](./05-eks-networking-concepts.md) for architecture understanding
2. Follow [EKS Setup Guide](./06-eks-setup-guide.md) to create your cluster
3. Learn [Helm basics](./08-helm.md) to understand package management
4. Install [AWS Load Balancer Controller](./07-alb-controller.md) to enable Ingress → ALB provisioning
5. Deploy a test application with Ingress to verify ALB creation

## Goal

The goal of this phase is to master Kubernetes networking concepts, enabling you to:
- Expose applications using Services and Ingress
- Control traffic with NetworkPolicies
- Understand service discovery and DNS
- Use Helm to manage Kubernetes applications and dependencies
- Deploy and manage applications on AWS EKS
- Understand EKS networking architecture (VPC CNI, ENIs, subnets)
- Provision and manage AWS Application Load Balancers from Kubernetes Ingress resources
