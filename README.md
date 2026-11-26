# ☸️ Kubernetes Production Patterns

A small, opinionated set of manifests I use as the starting point for a new service on Kubernetes. Everything here is generic — swap the image, ports and names and apply.

| File | What it gives you |
|---|---|
| [`manifests/deployment.yaml`](manifests/deployment.yaml) | Hardened Deployment: zero-downtime rollout, startup/readiness/liveness probes, resource requests, non-root + read-only filesystem, zone spreading |
| [`manifests/service.yaml`](manifests/service.yaml) | ClusterIP Service in front of the pods |
| [`manifests/hpa.yaml`](manifests/hpa.yaml) | HorizontalPodAutoscaler (autoscaling/v2) with scale-up/scale-down behaviour |
| [`manifests/pdb.yaml`](manifests/pdb.yaml) | PodDisruptionBudget so node drains and upgrades never take every replica |
| [`manifests/networkpolicy.yaml`](manifests/networkpolicy.yaml) | Default-deny ingress plus an explicit allow from the ingress controller |

## Try it

```bash
kubectl create namespace demo
kubectl -n demo create configmap web-config --from-literal=LOG_LEVEL=info
kubectl -n demo create secret generic web-secrets --from-literal=API_KEY=changeme

# validate without changing anything
kubectl -n demo apply --dry-run=server -f manifests/

kubectl -n demo apply -f manifests/
kubectl -n demo rollout status deployment/web
```

The Deployment references `web-config` and `web-secrets`, so create them first (or remove the `envFrom` block). The image `registry.example.com/team/web:1.0.0` is a placeholder.

## Design decisions

- **`maxUnavailable: 0`** — a rollout adds a new pod before removing an old one, so capacity never drops.
- **Three probes, three jobs** — *startup* protects slow boots, *readiness* gates traffic, *liveness* restarts only truly stuck processes. Liveness deliberately does not call dependencies, so a database blip cannot restart every pod at once.
- **Memory limit = request, no CPU limit** — memory is incompressible (exceeding it means an OOM kill), so it is sized exactly; CPU limits cause throttling even when the node has spare capacity. Measure your own workload before copying this.
- **Non-root, read-only root filesystem, all capabilities dropped** — a compromised container has far less to work with. Writable paths are mounted explicitly (`/tmp` here).
- **HPA scales up fast, down slowly** — a 5-minute stabilisation window stops flapping. HPA utilisation is measured against the CPU *request*, so the request must be set realistically.
- **PodDisruptionBudget** — voluntary disruptions (drains, upgrades, autoscaler scale-down) evict at most one pod at a time.
- **Default-deny NetworkPolicy** — start closed, then open only what is needed. Requires a CNI that enforces NetworkPolicy.

## Useful checks

```bash
kubectl -n demo get pods -o wide
kubectl -n demo describe hpa web
kubectl -n demo get pdb web
kubectl -n demo rollout history deployment/web
kubectl -n demo rollout undo deployment/web
```

## License

MIT — use it, adapt it, ship it.

---
*Maintained by [Rohit Bhosale](https://github.com/rohitbhosale439) — DevOps Engineer*
