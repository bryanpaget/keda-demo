# KEDA Demo Examples

This directory contains standalone demos for KEDA (Kubernetes Event-driven Autoscaling).

## Demos

### 1. cpu-scaler.yaml
Demonstrates traditional CPU-based scaling (similar to HPA). Shows resource utilization triggering scaling. Use this to contrast with event-driven approaches - CPU scaling requires actual load and can't scale to zero as easily.

**Usage:**
```bash
kubectl apply -f cpu-scaler.yaml -n default
# Generate load to trigger scaling
# Generate load (pod security compliant - restricted PSP)
kubectl run loadgen --namespace=default --restart=Never \
  --image=docker.io/library/busybox:1.36 \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":1000,"runAsGroup":1000,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"loadgen","image":"docker.io/library/busybox:1.36","securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}},"command":["/bin/sh","-c"],"args":["while true; do wget -q -O- http://keda-test-svc.default.svc.cluster.local > /dev/null 2>&1; sleep 0.01; done"]}]}}' \
  -- /bin/sh -c "while true; do wget -q -O- http://keda-test-svc.default.svc.cluster.local > /dev/null 2>&1; sleep 0.01; done"
# Stop load
kubectl delete pod loadgen -n default
# Watch scaling
watch -n 2 "kubectl get hpa -n default && echo "---" && kubectl get pods -n default"
```

### 2. cron-scaler.yaml
Demonstrates scaling based on time-based cron triggers. Shows how KEDA can scale workloads based on scheduled events rather than CPU/Memory metrics. The threat-analyzer deployment scales to 3 replicas during the active window.

**Usage:**
```bash
kubectl apply -f cron-scaler.yaml -n keda-system
# Watch scaling
kubectl get pods -w -n keda-system
kubectl describe scaledobject -n keda-system
```

### 3. redis-demo.yaml
Demonstrates event-driven scaling based on Redis list length. A load-generator pushes messages to a Redis queue, and KEDA scales the threat-analyzer deployment based on queue depth. Shows scale-to-zero capability when queue is empty. 

**Includes:** Full stack (Redis, load-generator, threat-analyzer), KEDA ScaledObject with activationListLength, and Cilium NetworkPolicies for proper connectivity in this cluster. 

**Implementation notes (thanks to Deepesh):** Uses `redis:alpine` with `redis-cli` (no runtime pip installs), Cilium NetworkPolicies to allow KEDA operator to poll Redis, and `activationListLength: "1"` for scale-from-zero.

**Usage:**
```bash
kubectl apply -f redis-demo.yaml -n keda-system
# Monitor queue and scaling
kubectl exec -n keda-system $(kubectl get pod -n keda-system -l app=redis -o jsonpath='{.items[0].metadata.name}') -- redis-cli LLEN security-alerts
kubectl get pods -w -n keda-system
```

## Visualization & Demo Playbook

To effectively demo KEDA, you'll want multiple terminals/console sessions open to observe different things simultaneously.

### Recommended Setup (2-3 Terminals)

**Terminal 1: Watch pods/scaling in real-time**
```bash
# For keda-system namespace
watch -n 2 'kubectl get pods,hpa,scaledobject -n keda-system'

# For default namespace  
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
# Generate load (pod security compliant - restricted PSP)
kubectl run loadgen --namespace=default --restart=Never \
  --image=docker.io/library/busybox:1.36 \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":1000,"runAsGroup":1000,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"loadgen","image":"docker.io/library/busybox:1.36","securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}},"command":["/bin/sh","-c"],"args":["while true; do wget -q -O- http://keda-test-svc.default.svc.cluster.local > /dev/null 2>&1; sleep 0.01; done"]}]}}' \
  -- /bin/sh -c "while true; do wget -q -O- http://keda-test-svc.default.svc.cluster.local > /dev/null 2>&1; sleep 0.01; done"
# Stop load
kubectl delete pod loadgen -n default

# Watch pods scale up (may take 1-2 minutes depending on metrics-server)
# Stop with Ctrl+C to see scale down after cooldown
```

#### Demo 2: Cron Scaling (Time-based events)
Show scaling based on scheduled/time-based triggers.
```bash
# Apply
kubectl apply -f cron-scaler.yaml -n keda-system

# Watch
watch -n 2 'kubectl get pods -n keda-system | grep threat-analyzer'
```

#### Demo 3: Event-Driven Scaling (Redis Queue) - Most impressive
Show true event-driven scaling with scale-to-zero.
```bash
# Apply the full stack
kubectl apply -f redis-demo.yaml -n keda-system

# Initial state - threat-analyzer should be at 0 replicas (scale to zero!)
kubectl get pods -n keda-system | grep -E 'redis|load-generator|threat-analyzer'

# Watch all three terminals as messages flow

# Check status
kubectl get scaledobject,hpa -n keda-system
```

### Tips for Presentation
- **Start with CPU demo** to establish baseline ("this is how HPA works")
- **Show cron demo** to show time-based event triggers
- **End with Redis demo** - most compelling showing scale-to-zero based on actual events/queue depth
- **Point out the key difference**: With event-driven scaling, you only pay for compute when there's work to do
- **Show the ScaledObject status**: `kubectl describe scaledobject <name> -n <ns>` to show triggers and conditions

## Lessons Learned (Thanks to Deepesh)
- **CNI matters**: This cluster uses Cilium - need `CiliumNetworkPolicy`/`CiliumClusterwideNetworkPolicy`, not just standard NetworkPolicies. KEDA operator needs explicit egress allowed to scrape Redis.
- **KEDA polls too**: The operator polls Redis for metrics, not just app pods. Must allow egress from `keda-operator` to Redis.
- **No runtime dependencies**: Use `redis:alpine` with built-in `redis-cli`. Avoid `pip install` at runtime as clusters may be air-gapped/no internet.
- **Activation thresholds**: Use `activationListLength` to control when scaling activates from zero (vs just scaling based on listLength).
- **Look at working examples**: Check existing working resources in the cluster first before building from scratch.
