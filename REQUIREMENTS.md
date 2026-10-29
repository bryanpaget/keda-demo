# Requirements

## Local tools

- `kubectl`, configured with access to the demonstration cluster
- `watch`, for live terminal views
- `stern`, for following logs across pods created or removed by autoscaling

Install `stern` with Homebrew:

```bash
brew install stern
```

## Cluster prerequisites

- Kubernetes access in the namespace containing the selected demo
- KEDA installed with its operator and metrics API server running
- Metrics server installed for the CPU demo
- Redis image access for the Redis demo
- Cilium for the included demo networking policies

All manifests are intended for demonstration environments. Review the manifests
before applying them to a production or multi-tenant cluster.

The Cilium resources in the Redis demo are cluster-specific and assume workloads in
`keda-system` plus KEDA operator components with their standard labels. The CPU demo
uses `default`; the cron and Redis demos use `keda-system`.
