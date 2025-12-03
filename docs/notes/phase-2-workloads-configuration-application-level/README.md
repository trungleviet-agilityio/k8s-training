# Phase 2: Workloads & Configuration (Application-level K8s)

This directory contains detailed documentation for Phase 2, focused on designing and configuring applications in Kubernetes.

## Documentation Index

1. [**Pod and Deployment Design**](./01-pod-deployment-design.md)
   - Pod vs Deployment vs ReplicaSet
   - Resource Requests and Limits
   - Common Issues

2. [**Multi-container Pods**](./02-multi-container-pods.md)
   - Design Patterns (Sidecar, Init Containers)
   - Sidecar Logging Pattern
   - Troubleshooting

3. [**Configuration**](./03-configuration.md)
   - ConfigMaps and Secrets
   - Injection Methods (Env, Volume)
   - Best Practices

4. [**Probes and Lifecycle**](./04-probes-lifecycle.md)
   - Liveness, Readiness, and Startup Probes
   - Configuration Examples
   - Troubleshooting Failures

5. [**Jobs and CronJobs**](./05-jobs-cronjobs.md)
   - Workload Types (Batch vs Long-running)
   - Job and CronJob Configuration
   - Failure Handling and Parallelism

6. [**Observability and Debugging**](./06-observability-debugging.md)
   - Debugging Workflow
   - Logging Commands
   - Monitoring Applications

7. [**Exercises**](./07-exercises.md)
   - List of related practical exercises

## Goal
The goal of this phase is to master application-level Kubernetes concepts, enabling you to design robust, scalable, and configurable workloads.
