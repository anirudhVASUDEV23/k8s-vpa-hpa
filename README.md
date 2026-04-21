# Kubernetes HPA and VPA Demonstration

This project demonstrates **Horizontal Pod Autoscaling (HPA)** and **Vertical Pod Autoscaling (VPA)** in Kubernetes using Apache as a test workload in the `apache` namespace.

## Overview

This setup showcases two different autoscaling strategies:

- **HPA (Horizontal Pod Autoscaler)**: Scales the number of pod replicas based on CPU/memory metrics
- **VPA (Vertical Pod Autoscaler)**: Scales the CPU/memory requests and limits of individual pods based on usage patterns

## Project Structure

```
.
├── namespace.yml          # Kubernetes namespace definition
├── deployment.yml         # Apache deployment configuration
├── service.yml            # Service to expose Apache pods
├── hpa.yml                # Horizontal Pod Autoscaler configuration
├── vpa.yml                # Vertical Pod Autoscaler configuration
└── README.md              # This file
```

## Prerequisites

- Kubernetes cluster (1.19+)
- `kubectl` configured to access your cluster
- Metrics Server installed (for HPA to work)
- Access to clone the [Kubernetes Autoscaler](https://github.com/kubernetes/autoscaler) repository

## Files to Apply

### Phase 1: Setup Base Infrastructure

1. **namespace.yml** - Creates the `apache` namespace
2. **deployment.yml** - Deploys Apache web server with resource requests
3. **service.yml** - Exposes Apache pods as a Kubernetes service

### Phase 2: HPA Testing

4. **hpa.yml** - Horizontal Pod Autoscaler that scales replicas based on CPU usage

### Phase 3: VPA Testing

5. **vpa.yml** - Vertical Pod Autoscaler for automatic resource requests/limits adjustment

## Step-by-Step Workflow

### Step 1: Create Namespace and Deploy Base Resources

```bash
# Apply namespace
kubectl apply -f namespace.yml

# Apply deployment
kubectl apply -f deployment.yml

# Apply service
kubectl apply -f service.yml

# Verify deployment
kubectl get pods -n apache
kubectl get svc -n apache
```

### Step 2: Test Horizontal Pod Autoscaler (HPA)

```bash
# Apply HPA configuration
kubectl apply -f hpa.yml

# Verify HPA is active
kubectl get hpa -n apache
kubectl describe hpa -n apache
```

### Step 3: Generate Load

In a separate terminal, create a load-generating pod:

```bash
# Start an interactive pod
kubectl run -i --tty load-generator --image=busybox -n apache /bin/bash

# Inside the pod, run the load generation script
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

### Step 4: Monitor HPA Scaling

In another terminal, watch the pods and HPA metrics:

```bash
# Watch pods scaling in real-time
kubectl get pods -n apache --watch

# Monitor HPA status
kubectl get hpa -n apache --watch

# Check detailed HPA metrics
kubectl describe hpa -n apache
```

**Expected Behavior:**

- Initial pod count increases as CPU usage rises above threshold
- Pods scale up to handle the load
- Metrics should show current CPU usage vs. target

### Step 5: Cleanup HPA

Stop the load generator and remove HPA:

```bash
# Delete the load generator pod (Ctrl+C in the load-generator terminal first)
kubectl delete pod load-generator -n apache

# Delete HPA
kubectl delete -f hpa.yml

# Wait for metrics to stabilize, then scale down to 1 replica
kubectl scale deployment/apache --replicas=1 -n apache
```

### Step 6: Setup Vertical Pod Autoscaler (VPA)

VPA requires additional setup. Clone and install VPA components:

```bash
# Clone the autoscaler repository
git clone https://github.com/kubernetes/autoscaler.git

# Navigate to VPA directory
cd autoscaler/vertical-pod-autoscaler

# Run the VPA setup script
./hack/vpa-process-yamls.sh apply
```

This installs:

- VPA CRDs (CustomResourceDefinitions)
- VPA Admission Controller
- VPA Recommender
- VPA Updater

### Step 7: Apply VPA Configuration

```bash
# Apply VPA policy to Apache deployment
kubectl apply -f vpa.yml

# Verify VPA is active
kubectl get vpa -n apache
kubectl describe vpa -n apache
```

### Step 8: Generate Load for VPA Testing

```bash
# Start load generator again
kubectl run -i --tty load-generator --image=busybox -n apache /bin/bash

# Inside the pod:
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

### Step 9: Monitor VPA Recommendations

```bash
# Watch pod resource changes
kubectl get pods -n apache --watch

# Check VPA recommendations
kubectl describe vpa -n apache

# Monitor pod resource requests over time
kubectl get pods -n apache -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources}{"\n"}{end}'
```

**Expected Behavior:**

- VPA analyzes actual resource usage
- Pod recommendations are adjusted based on observed metrics
- Pods may be recreated with updated resource requests/limits
- Over time, resource requests align with actual usage

### Step 10: Cleanup VPA

```bash
# Delete VPA configuration
kubectl delete -f vpa.yml

# Optionally, remove VPA from cluster
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-process-yamls.sh delete
```

## Key Monitoring Commands

```bash
# Check current state
kubectl get pods -n apache
kubectl get svc -n apache

# HPA monitoring
kubectl get hpa -n apache
kubectl describe hpa -n apache

# VPA monitoring
kubectl get vpa -n apache
kubectl describe vpa -n apache
kubectl get vpa -n apache -o yaml

# View resource metrics
kubectl top pods -n apache
kubectl top nodes
```

## File Descriptions

| File             | Purpose                                                |
| ---------------- | ------------------------------------------------------ |
| `namespace.yml`  | Defines the `apache` namespace for isolation           |
| `deployment.yml` | Apache deployment with resource requests (CPU/Memory)  |
| `service.yml`    | ClusterIP service for internal pod communication       |
| `hpa.yml`        | Scales pod replicas based on CPU threshold (e.g., 50%) |
| `vpa.yml`        | Adjusts CPU/memory requests based on actual usage      |

## Troubleshooting

### HPA Shows "unknown" for metrics

- Ensure Metrics Server is installed: `kubectl get deployment metrics-server -n kube-system`
- Install if missing: `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`

### VPA Not Making Changes

- Verify VPA pods are running: `kubectl get pods -n kube-system | grep vpa`
- Check VPA logs: `kubectl logs -n kube-system -l app=vpa-recommender`
- Ensure deployment has resource requests defined in `deployment.yml`

### Load Generator Unresponsive

- Verify service exists: `kubectl get svc -n apache`
- Test connectivity: `kubectl run -it --rm debug --image=busybox -n apache -- wget -O- http://apache-service.apache.svc.cluster.local`

## Cleanup Everything

```bash
# Delete all resources
kubectl delete namespace apache
```

## References

- [HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Kubernetes Autoscaler Project](https://github.com/kubernetes/autoscaler)
