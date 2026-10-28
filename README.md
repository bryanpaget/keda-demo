# KEDA Demo Examples

Standalone, self-contained demos for KEDA (Kubernetes Event-driven Autoscaling).
Each file can be applied on its own with a single `kubectl apply` and contains
everything it needs (workload, trigger source where applicable, load generator where
applicable, and the networking policies this cluster requires).

## Files

| File | Trigger | Scales to zero | Namespace |
|------|---------|----------------|-----------|
| `cpu-scaler.yaml` | CPU utilization (HPA-style) | No | `default` |
| `cron-scaler.yaml` | Cron schedule (time) | Yes | `keda-system` |
| `redis-demo.yaml` | Redis list length (events) | Yes | `keda-system` |

The demos use distinct resource names and can run side by side. Apply each file
independently; each one is complete on its own.

---

### 1. cpu-scaler.yaml

Traditional resource-based scaling for a threat analyzer, for contrast with the
Redis event-driven threat analyzer. KEDA creates a standard HPA that scales the
CPU-based threat analyzer deployment on CPU utilization.

The file is fully self-contained:

- CPU-based threat analyzer deployment + Service
- A busybox load generator that produces a sustained, wave-shaped HTTP pattern
- `ScaledObject` with a `cpu` trigger at 5% utilization
- Cilium policies allowing the load generator to reach the threat analyzer (required in this cluster)

```bash
kubectl apply -f cpu-scaler.yaml
```

Watch the HPA and pods in one terminal:

```bash
watch -n 2 'kubectl get hpa -n default; echo; kubectl get pods -n default'
```

Watch the wave and pause pattern being generated:

```bash
kubectl logs -f -n default \
  "$(kubectl get pod -n default -l app=load-generator --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"
```

Expected result: the threat analyzer's CPU crosses the 5% target during the load peak and replicas climb above the single-pod floor.

```
NAME                                REFERENCE                              TARGETS       MINPODS   MAXPODS   REPLICAS
keda-hpa-threat-analyzer-cpu-scaler Deployment/threat-analyzer-cpu        cpu: 20%/5%   1         5         2
```

To watch it scale back down, stop the load and wait for the cooldown:

```bash
kubectl scale deployment load-generator -n default --replicas=0
```

Cleanup:

```bash
kubectl delete -f cpu-scaler.yaml
```

Note: `PollingInterval` / `CooldownPeriod` warnings are expected here because
`minReplicaCount > 0` - there is no scale-to-zero in this demo, which is the point.

---

### 2. cron-scaler.yaml

Scaling driven by a schedule instead of a resource metric. The `cron-worker`
deployment sits at zero and is scaled to 3 replicas during the active cron window.

```bash
kubectl apply -f cron-scaler.yaml
watch -n 2 'kubectl get pods -n keda-system | grep cron-worker'
kubectl describe scaledobject cron-worker-scaler -n keda-system
```

The schedule in the file is intentionally aggressive (`*/1` to `*/2` each hour) so
the scale-up is visible during a demo. Adjust `start` / `end` for a realistic window.

Cleanup:

```bash
kubectl delete -f cron-scaler.yaml
```

---

### 3. redis-demo.yaml

The headline demo: event-driven scaling with scale-to-zero.

- A load generator pushes messages into a Redis list in waves (10-50 messages every
  two seconds), then pauses for 30 seconds every ten waves
- `threat-analyzer` workers consume messages
- KEDA scales `threat-analyzer` based on Redis list length
  (`listLength: "5"` per replica, `activationListLength: "1"` to wake from zero)
- Includes Cilium policies so the KEDA operator, generator, and workers can talk to Redis

The pause is deliberate: it lets workers drain the backlog and lets KEDA scale back to
zero, so the whole cycle (0 -> N -> 0) repeats roughly every 50 seconds. Worker
throughput is intentionally higher than generator throughput; otherwise the queue
never drains and the demo never shows scale-down.

```bash
kubectl apply -f redis-demo.yaml
```

Watch three things at once:

```bash
# 1. pods
watch -n 2 'kubectl get pods -n keda-system'

# 2. queue depth
watch -n 1 "kubectl exec -n keda-system \$(kubectl get pod -n keda-system -l app=redis -o jsonpath='{.items[0].metadata.name}') -- redis-cli LLEN security-alerts"

# 3. worker logs
kubectl logs -f -n keda-system -l app=threat-analyzer
```

The queue fills, `threat-analyzer` scales up, workers drain the queue, then it
scales back to zero after the 15s cooldown. Sample cycle:

```
15:01:52  queue=37  replicas=1
15:01:59  queue=0   replicas=4
15:02:15  queue=0   replicas=0   <- drained and scaled to zero
```

Cleanup:

```bash
kubectl delete -f redis-demo.yaml
```

---

## Visualization & Demo Playbook

Use two or three terminals so the audience can see cause and effect at the same time.

**Terminal 1 - scaling state**

```bash
# CPU demo
watch -n 2 'kubectl get hpa -n default; echo; kubectl get pods -n default'

# Event demos
watch -n 2 'kubectl get pods,hpa,scaledobject -n keda-system'
```

Note: `kubectl get ... -w` only accepts a single resource type. When watching
multiple types, use `watch` around a plain `kubectl get` instead of `-w`.

**Terminal 2 - what the workload is doing**

```bash
# CPU demo: HTTP request waves and pauses
kubectl logs -f -n default \
  "$(kubectl get pod -n default -l app=load-generator --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"

# Redis demo: workers processing events
kubectl logs -f -n keda-system -l app=threat-analyzer
```

**Terminal 3 - trigger input**

```bash
# CPU demo: actual CPU usage
watch -n 2 'kubectl top pods -n default'

# Redis demo: queue depth
watch -n 1 "kubectl exec -n keda-system \$(kubectl get pod -n keda-system -l app=redis -o jsonpath='{.items[0].metadata.name}') -- redis-cli LLEN security-alerts"
```

### Suggested flow

1. Apply `cpu-scaler.yaml`. Show that scaling only happens while CPU load exists,
   and that the floor is 1 replica.
2. Apply `cron-scaler.yaml`. Show a trigger that is not a resource metric at all.
3. Apply `redis-demo.yaml`. Show `threat-analyzer` at zero, then watch events in the
   queue wake it up and scale it out, then scale back to zero during the pause.
4. Close on the difference: event-driven workloads can be truly idle (zero cost)
   and respond to backlog rather than to already-burning CPU.

Useful status commands during the talk:

```bash
kubectl describe scaledobject <name> -n <namespace>
kubectl get hpa -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp | tail -20
```

---

## Lessons Learned

### From Deepesh

- **Cilium policies are mandatory here.** This cluster default-denies pod ingress and
  egress for everything outside `kube-system` via clusterwide policies. Any demo that
  needs pods to talk to each other must ship explicit `CiliumNetworkPolicy` rules.
- **KEDA's operator is a client too.** For the Redis scaler, the `keda-operator` pod
  polls Redis for metrics. Policies must allow egress from the operator to Redis, not
  just traffic between the app pods.
- **No runtime dependencies.** Use images that already contain the client
  (`redis:alpine` with `redis-cli`, `busybox:1.36` with a shell and `wget` for load).
  Runtime package installs can fail in restricted/air-gapped clusters.
- **Activation thresholds.** `activationListLength` controls when scaling wakes up
  from zero; `listLength` controls how many items map to each replica.
- **Start from a known-working example.** Inspect what already works in the cluster
  before building from scratch.

### Additional findings from this build-out

- **Metrics can be fine while load is not.** The CPU demo showed `<unknown>` and then
  `0%/5%`; metrics-server was healthy. The real failure was that no traffic reached
  the threat analyzer, so there was no CPU to measure. Always verify the load path end to end.
- **Verify consumption, not just scaling.** Deepesh's original Redis demo scaled to
  10 replicas but its workers could never reach Redis, so the queue grew to ~800k and
  never drained. The cause was an ingress rule with two keys in one `matchLabels`
  (`app: threat-analyzer` and `app: keda-operator`), which matches no pod. Scaling
  looked healthy while the actual work was broken - always check that the workers are
  processing, not just that the HPA moved.
- **Match generator throughput to worker throughput.** If the producer is faster than
  all replicas can consume, the queue never reaches zero and the demo cannot show
  scale-down. Make the load wave-shaped with pauses.
- **Unique policy names in shared namespaces.** Reusing another demo's policy names
  overwrote its selectors. Prefix demo-owned policies (`demo-`) so applying or
  deleting one demo cannot break another.
- **Default-deny is invisible in `kubectl get netpol`.** The blocking rules were
  `CiliumClusterwideNetworkPolicy` objects whose selectors match nearly every
  namespace, plus the absence of an allow policy for the demo's port. Check
  `kubectl get ciliumclusterwidenetworkpolicies` when traffic fails unexpectedly.
- **Generators should be observable.** Both the CPU and Redis generators log the
  current load and pauses, making it easy to correlate workload changes with scaling.
- **`kubectl get a,b -w` is rejected.** `--watch` supports only one resource type;
  wrap a plain multi-type `kubectl get` in `watch` instead.
