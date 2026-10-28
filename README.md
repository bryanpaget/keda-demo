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
---

### 4. deepesh-redis-demo.yaml
Comprehensive Redis-based event-driven scaling demo with Cilium NetworkPolicies. Uses activationListLength for scaling from zero, different naming to avoid conflicts, and direct IP mapping. Great for environments with strict network policies.

**Usage:**
```bash
kubectl apply -f deepesh-redis-demo.yaml -n keda-system
# Monitor
kubectl get pods -n keda-system | grep deepesh
kubectl logs -n keda-system -l app=deepesh-load-generator --tail=5
kubectl logs -n keda-system -l app=deepesh-threat-analyzer --tail=5
```

## Lessons Learned (Thanks to Deepesh)

- **CNI matters**: This cluster uses Cilium - need `CiliumNetworkPolicy`/`CiliumClusterwideNetworkPolicy`, not just standard NetworkPolicies. KEDA operator needs explicit egress allowed to scrape Redis.
- **KEDA polls too**: The operator polls Redis for metrics, not just app pods. Must allow egress from `keda-operator` to Redis.
- **No runtime dependencies**: Use `redis:alpine` with built-in `redis-cli`. Avoid `pip install` at runtime as clusters may be air-gapped/no internet.
- **Activation thresholds**: Use `activationListLength` to control when scaling activates from zero (vs just scaling based on listLength).
- **Look at working examples**: Check existing working resources in the cluster first before building from scratch.

## Visualization & Demo Playbook

To effectively demo KEDA, you'll want multiple terminals/console sessions open to observe different things simultaneously.

### Recommended Setup (2-3 Terminals)

**Terminal 1: Watch pods/scaling in real-time**
```bash
watch -n 2 'kubectl get pods,hpa,scaledobject -n keda-system'
# or for default namespace
watch -n 2 'kubectl get pods,hpa,scaledobject -n default'
```

**Terminal 2: Watch logs from the workload**
```bash
# For Redis demo
kubectl logs -f -n keda-system -l app=threat-analyzer

# For CPU demo  
kubectl logs -f -n default -l app=keda-test
```

**Terminal 3: Monitor queue/metrics**
```bash
# Redis demo - watch queue depth
watch -n 1 "kubectl exec -n keda-system \$(kubectl get pod -n keda-system -l app=redis -o jsonpath='{.items[0].metadata.name}') -- redis-cli LLEN security-alerts"

# Or check KEDA operator logs
kubectl logs -f -n keda-system -l app=keda-operator --tail=20
```

### Demo Flow

#### Demo 1: CPU Scaling (Traditional)
Show the "old way" - scaling based on resource utilization.
```bash
# Apply
kubectl apply -f cpu-scaler.yaml -n default

# Observe initial state - should have 1 replica
kubectl get pods,hpa -n default

# Generate load
kubectl run loadgen --rm -it --restart=Never --image=docker.io/library/busybox:1.36 -n default -- \
  /bin/sh -c "while true; do wget -q -O- http://keda-test-svc.default.svc.cluster.local > /dev/null; done"

# Watch pods scale up (may take 1-2 minutes depending on metrics-server)
# Stop with Ctrl+C to see scale down after cooldown
```

#### Demo 2: Cron Scaling (Time-based events)
Show scaling based on scheduled/time-based triggers.
```bash
# Apply (scales to 3 replicas during active window, every minute in our config)
kubectl apply -f cron-scaler.yaml -n keda-system

# Watch - will scale up during the trigger window
watch -n 2 'kubectl get pods -n keda-system | grep threat-analyzer'

# Note: Cron triggers based on time, not load - great for predictable patterns
# (batch jobs, maintenance windows, etc.)
```

#### Demo 3: Event-Driven Scaling (Redis Queue) - Most impressive
Show true event-driven scaling with scale-to-zero.
```bash
# Apply the full stack
kubectl apply -f redis-demo.yaml -n keda-system

# Initial state - threat-analyzer should be at 0 replicas (scale to zero!)
kubectl get pods -n keda-system | grep -E 'redis|load-generator|threat-analyzer'

# Watch as load-generator adds messages to queue
# KEDA will detect queue depth and scale up threat-analyzer
# Watch all three terminals

# Watch the magic happen:
# 1. Queue fills up -> KEDA scales threat-analyzer from 0 to N replicas
# 2. Workers consume messages -> queue drains
# 3. After cooldown, scales back to 0

# Check current state
kubectl get scaledobject,hpa -n keda-system
```

### Tips for Presentation
- **Start with CPU demo** to establish baseline ("this is how HPA works")
- **Show cron demo** to show time-based event triggers
- **End with Redis demo** - this is the most compelling showing scale-to-zero based on actual events/queue depth
- **Point out the key difference**: With event-driven scaling, you only pay for compute when there's work to do
- **Show the ScaledObject status**: `kubectl describe scaledobject <name> -n <ns>` to show triggers and conditions
