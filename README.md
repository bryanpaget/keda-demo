# KEDA Demo Examples

This directory contains standalone demos for KEDA (Kubernetes Event-driven Autoscaling).

## Demos

### 1. cron-scaler.yaml
Demonstrates scaling based on time-based cron triggers. Shows how KEDA can scale workloads based on scheduled events rather than CPU/Memory metrics. The threat-analyzer deployment scales to 3 replicas during the active window (every minute for demo purposes).

**Usage:**
```bash
kubectl apply -f cron-scaler.yaml -n <namespace>
# Watch scaling
kubectl get pods -w
kubectl describe scaledobject
```

### 2. redis-demo.yaml
Demonstrates event-driven scaling based on Redis list length. A load-generator pushes messages to a Redis queue, and KEDA scales the threat-analyzer deployment based on queue depth. Shows scale-to-zero capability when queue is empty.

**Usage:**
```bash
kubectl apply -f redis-demo.yaml -n <namespace>
# Monitor queue and scaling
kubectl exec -it redis-<pod> -- redis-cli LLEN security-alerts
kubectl get pods -w
```

## Notes
- These are standalone demos - apply each file independently
- Both demos use the same deployment name (threat-analyzer) - apply one at a time
- Cron scaler works out of the box; Redis demo requires network connectivity between KEDA operator and Redis
# keda-demo
---

### 3. cpu-scaler.yaml
Demonstrates traditional CPU-based scaling (similar to HPA). Shows resource utilization triggering scaling. Use this to contrast with event-driven approaches - CPU scaling requires actual load and can't scale to zero as easily.

**Usage:**
```bash
kubectl apply -f cpu-scaler.yaml -n default
# Generate load to trigger scaling
kubectl run loadgen --rm -it --restart=Never --image=docker.io/library/busybox:1.36 -- \
  /bin/sh -c "while true; do wget -q -O- http://keda-test-svc.default.svc.cluster.local; done"
# Watch scaling
kubectl get hpa,pods -w
```
