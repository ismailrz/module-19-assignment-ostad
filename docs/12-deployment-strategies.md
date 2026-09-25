# Part 12 — Kubernetes Deployment Strategies

## Rolling Update
**How it works:** The default strategy for a Kubernetes `Deployment`. New Pods (new ReplicaSet) are created incrementally while old Pods (old ReplicaSet) are terminated incrementally, controlled by `maxSurge` (how many extra Pods above `replicas` can exist during the rollout) and `maxUnavailable` (how many can be down at once). At every point during the rollout, a mix of old and new Pods serves traffic.

**Advantages:** Zero downtime (with `maxUnavailable > 0` avoided or traffic-aware readiness probes); no extra infrastructure needed beyond what the Deployment already manages; built in natively — no extra tooling; easy automatic rollback via `kubectl rollout undo`.

**Limitations:** Both old and new versions run simultaneously for the duration of the rollout, which requires the two versions to be backward/forward compatible (e.g. shared database schema must work with both); a bad new version still gets *some* live traffic before health checks catch it and halt the rollout; slower than an instant cutover.

**When to use it:** The default choice for most stateless services — this repo's `node-express-app` Deployment (`k8s/06-app-deployment.yaml`) uses it implicitly, since no `strategy` override is set.

---

## Recreate
**How it works:** All existing Pods are terminated first, then new Pods are created only after the old ones are fully gone. Set via `spec.strategy.type: Recreate` on a Deployment.

**Advantages:** Simple and guarantees only one version's Pods ever run at once — no risk of old/new version incompatibility during the transition; useful when the app itself can't tolerate two versions running concurrently (e.g. it holds an exclusive lock, or a schema migration is breaking/non-backward-compatible).

**Limitations:** Guaranteed downtime between the old Pods stopping and new Pods becoming ready — unacceptable for most user-facing services.

**When to use it:** Non-critical or internal services where brief downtime is acceptable, or any deployment where running two versions simultaneously would actively cause incorrect behavior (e.g. a singleton batch processor, or a breaking DB migration that isn't backward-compatible).

---

## Canary Deployment
**How it works:** A small subset of new-version Pods ("the canary") is deployed alongside the full set of old-version Pods, and a small percentage of live traffic is routed to the canary (via a Service that matches both versions' labels with weighted routing, or more precisely via a service-mesh/ingress controller like Istio, Linkerd, or NGINX Ingress with canary annotations). If the canary looks healthy (error rates, latency, business metrics), traffic is gradually shifted until the canary fully replaces the old version; if not, it's rolled back having only affected a small slice of users.

**Advantages:** Limits the blast radius of a bad release to a small fraction of real traffic instead of all of it; allows validating a new version against real production traffic/data before full rollout; can be tied to automated metrics-based promotion/rollback.

**Limitations:** Not natively supported by a plain Kubernetes Deployment/Service — needs an ingress controller, service mesh, or a tool like Argo Rollouts/Flagger to do weighted/percentage-based traffic splitting; more operational complexity to set up and monitor; like Rolling Update, requires the two versions to be compatible with shared state/data during the overlap.

**When to use it:** High-traffic or high-risk production services where you want real-user validation before a full rollout, and where the infrastructure to support traffic splitting (mesh/ingress) already exists or is worth the investment.

---

## Comparison & Use Cases

| Strategy | Downtime | Version overlap | Rollback risk exposure | Extra tooling needed |
|---|---|---|---|---|
| Rolling Update | None (tunable) | Yes, brief | Some traffic hits bad version before detection | None (built into Deployment) |
| Recreate | Yes | None | None hit bad version until it's fully up | None (built into Deployment) |
| Canary | None | Yes, prolonged/controlled | Only the canary slice hits bad version | Ingress/mesh or rollout controller |

- **Rolling Update** — default for `node-express-app`-style stateless web APIs where brief version overlap is safe.
- **Recreate** — a job/worker that can't have two instances racing on the same external resource, or a release paired with a breaking DB migration.
- **Canary** — a customer-facing checkout service where a bad release must be caught against a small slice of real traffic before it reaches everyone.
