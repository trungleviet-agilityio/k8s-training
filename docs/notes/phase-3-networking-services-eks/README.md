# Phase 3: Networking, Services & AWS EKS Basics

This directory contains detailed documentation for Phase 3, focused on Kubernetes networking, Services, Ingress, and NetworkPolicies.

## Documentation Index

1. [**Services**](./01-services.md)
   - Service Types (ClusterIP, NodePort, LoadBalancer)
   - DNS-based Service Discovery
   - Port Configuration
   - Common Issues

2. [**Ingress**](./02-ingress.md)
   - Services vs Ingress Evolution
   - Ingress Controller vs Ingress Resources
   - Path-based and Host-based Routing
   - Multiple Rules and Paths
   - API Version Changes
   - Troubleshooting

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
   - Troubleshooting guide

## Goal
The goal of this phase is to master Kubernetes networking concepts, enabling you to expose applications, control traffic, and understand service discovery.
