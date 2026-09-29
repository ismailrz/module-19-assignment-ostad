# Docker & Kubernetes Deployment on AWS EC2 — Submission

**Name:** _\<your name\>_
**Date:** _\<date\>_
**App repo:** https://github.com/roy35-909/Module-3-deployment
**Docker Hub repo:** _\<https://hub.docker.com/r/DOCKERHUB_USERNAME/node-express-app\>_
**EC2 control-plane public IP:** _\<CONTROL_PLANE_IP\>_

> How to use this file: do the step in `RUNBOOK.md`, save the screenshot into
> `screenshots/` using the **exact filename** shown under each item below,
> and it will render automatically here. Nothing else needs editing except
> the placeholders above and any `_<...>_` notes.

---

## Part 1 — Application Containerization

**Docker image running locally** (`docker ps` + `docker logs`):

![docker ps output](screenshots/part1-docker-ps.png)

![docker logs output](screenshots/part1-docker-logs.png)

**`curl` output from `/` and `/api`:**

![curl localhost:5000](screenshots/part1-curl.png)

**Image pushed and visible on Docker Hub:**

![Docker Hub repo page](screenshots/part1-dockerhub.png)

---

## Part 2 — Kubernetes Cluster on AWS EC2

**AWS EC2 console — running instances:**

![EC2 instances](screenshots/part2-ec2-instances.png)

**`kubectl get nodes -o wide` — all nodes Ready:**

![kubectl get nodes](screenshots/part2-nodes-ready.png)

**`kubectl get pods -A` — kube-system pods Running:**

![kube-system pods](screenshots/part2-kube-system-pods.png)

---

## Part 3 — Secure etcd

**Before encryption — Secret readable in etcd (base64, not encrypted):**

![etcd secret before encryption](screenshots/part3-etcd-before.png)

**`kube-apiserver.yaml` showing `--encryption-provider-config` flag:**

![encryption provider config flag](screenshots/part3-apiserver-flag.png)

**After encryption — Secret stored as ciphertext (`k8s:enc:aescbc:v1:key1:...`):**

![etcd secret after encryption](screenshots/part3-etcd-after.png)

---

## Part 4 — Static NGINX Pod

**Static Pod Running:**

![static nginx pod running](screenshots/part4-static-pod-running.png)

**Default NGINX welcome page:**

![nginx welcome page](screenshots/part4-nginx-page.png)

---

## Part 5 — Deploy the Application

**Deployment:**

![kubectl get deployments](screenshots/part5-deployments.png)

**ReplicaSet:**

![kubectl get rs](screenshots/part5-replicaset.png)

**Pods:**

![kubectl get pods -l app=node-express-app](screenshots/part5-pods.png)

---

## Part 6 — PostgreSQL StatefulSet

**StatefulSet:**

![kubectl get statefulset postgres](screenshots/part6-statefulset.png)

**PostgreSQL Pod Running:**

![postgres-0 running](screenshots/part6-postgres-pod.png)

**PVC Bound:**

![postgres pvc bound](screenshots/part6-pvc-bound.png)

---

## Part 7 — Persistent Volume and PVC (Application)

**PersistentVolume `app-pv` Bound:**

![app-pv bound](screenshots/part7-pv-bound.png)

**PersistentVolumeClaim `app-pvc` Bound:**

![app-pvc bound](screenshots/part7-pvc-bound.png)

**Volume mounted inside the application Pod:**

![df -h /app/data inside pod](screenshots/part7-mount-verify.png)

---

## Part 8 — Expose the Application

**NodePort Service:**

![node-express-app-svc nodeport 30080](screenshots/part8-service.png)

**Application accessible from outside the cluster via NodePort:**

![app in browser via NodePort](screenshots/part8-app-external.png)

---

## Part 9 — Kubernetes Resource Documentation

For each resource: what it is, what problem it solves, when to use it, and a practical example (drawn from this project's own manifests).

### Pod
**What it is:** The smallest deployable unit in Kubernetes — one or more containers that share network namespace (same IP/port space) and storage volumes, always scheduled together on the same node.
**Problem it solves:** Containers on their own have no concept of "this group of processes belongs together." A Pod groups tightly-coupled containers (e.g. an app + a log-shipping sidecar) so they share a lifecycle, network, and volumes.
**When to use it:** Rarely created directly in production — normally a Deployment, StatefulSet, DaemonSet, or Job creates and manages Pods, since a bare Pod has no self-healing (if the node dies, the Pod is gone for good).
**Example:** `k8s/00-static-nginx-pod.yaml` — a single-container Pod running nginx.

### Deployment
**What it is:** A controller that manages a set of identical Pods via an underlying ReplicaSet, and drives the Pods from one desired state to another (e.g. rolling out a new image version).
**Problem it solves:** Manually creating/updating/deleting Pods to roll out changes or recover from failures doesn't scale and is error-prone. A Deployment declares "I want N replicas of this Pod spec running" and continuously reconciles reality to match that.
**When to use it:** Any stateless workload — web servers, APIs, background workers — where any replica can serve any request and none needs stable identity or storage.
**Example:** `k8s/06-app-deployment.yaml` — runs 2 replicas of `node-express-app`.

### ReplicaSet
**What it is:** A controller that ensures a specified number of identical Pod replicas are running at all times. It's the mechanism a Deployment uses under the hood.
**Problem it solves:** Guarantees self-healing at the Pod-count level — if a Pod is deleted or its node fails, the ReplicaSet notices the count dropped below desired and creates a replacement.
**When to use it:** Almost never created directly — Deployments manage ReplicaSets for you (one ReplicaSet per revision, which is how rolling updates and rollbacks work).
**Example:** `kubectl get rs` after applying `06-app-deployment.yaml` shows the ReplicaSet the Deployment created and owns.

### StatefulSet
**What it is:** Like a Deployment, but for workloads that need stable, unique network identities (`<name>-0`, `<name>-1`, ...) and stable per-replica persistent storage that follows the Pod across rescheduling.
**Problem it solves:** Deployments treat all Pods as interchangeable and give each a random name/IP — that breaks stateful systems (databases, message queues) that need each replica to keep its own identity and its own data volume across restarts.
**When to use it:** Databases, clustered systems where members need to find each other by stable name (e.g. via a headless Service), or anything where "replica 0" must always come back as "replica 0" with its own disk.
**Example:** `k8s/09-postgres-statefulset.yaml` — PostgreSQL, one replica, each pod gets its own PVC via `volumeClaimTemplates`.

### DaemonSet
**What it is:** A controller that ensures exactly one copy of a Pod runs on every node (or every node matching a selector) in the cluster, automatically adding/removing Pods as nodes join or leave.
**Problem it solves:** Some workloads are inherently "one per node" — log collectors, node monitoring agents, network/CNI plugins — and need to scale with the cluster's node count, not be manually replicated.
**When to use it:** Cluster-wide infrastructure agents: `kube-proxy`, CNI plugins (Calico/Flannel), log shippers (Fluentd/Filebeat), node exporters for monitoring.
**Example:** kubeadm itself deploys `kube-proxy` as a DaemonSet — check with `kubectl get daemonset -n kube-system`.

### Static Pod
**What it is:** A Pod defined directly on a node by dropping a manifest file into a directory the kubelet watches (default `/etc/kubernetes/manifests`), managed by that node's kubelet instead of the API server/scheduler.
**Problem it solves:** Bootstrapping problem — control-plane components (`kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `etcd`) need to run as Pods before the API server itself is even up. Static Pods let the kubelet run them without depending on the API server existing yet.
**When to use it:** Control-plane components (this is exactly how kubeadm runs them), or node-specific one-off Pods that must survive even if the API server is unreachable.
**Example:** `k8s/00-static-nginx-pod.yaml`, deployed by copying it into `/etc/kubernetes/manifests/` on a node rather than via `kubectl apply`.

### Service
**What it is:** A stable virtual IP and DNS name that load-balances traffic across a dynamic set of Pods matched by a label selector.
**Problem it solves:** Pod IPs are ephemeral — they change every time a Pod is recreated. A Service gives clients one stable address to talk to regardless of which Pods are currently backing it.
**When to use it:** Any time one set of Pods (or an outside client) needs to reliably reach another set of Pods, whether inside the cluster (`ClusterIP`), from outside via each node (`NodePort`), or via a cloud load balancer (`LoadBalancer`).
**Example:** `k8s/09-postgres-statefulset.yaml` includes a headless Service (`clusterIP: None`) so each Postgres Pod gets its own stable DNS record instead of a load-balanced VIP.

### NodePort
**What it is:** A Service type that opens a fixed port (default range 30000-32767) on every node in the cluster and forwards traffic from `<any-node-IP>:<nodePort>` to the Service's backing Pods.
**Problem it solves:** Gives external clients a way to reach a Service without a cloud load balancer — useful when running on bare-metal/self-managed clusters (like kubeadm on EC2) where a `LoadBalancer` Service type has nothing to provision it.
**When to use it:** Bare-metal/self-hosted clusters, demos, or dev/test environments needing external access without extra infrastructure. Not typically used directly in production behind a real ingress/load balancer.
**Example:** `k8s/07-app-service-nodeport.yaml` exposes `node-express-app` on port `30080` on every node's IP.

### PersistentVolume (PV)
**What it is:** A cluster-scoped storage resource representing an actual piece of storage (a directory, an EBS volume, an NFS export, etc.), provisioned either manually by an admin (static) or automatically by a StorageClass (dynamic).
**Problem it solves:** Decouples "what storage physically exists" from "which Pod is using it" — storage has its own lifecycle independent of any one Pod or even the cluster's Pods being deleted/recreated.
**When to use it:** Any time a workload needs data to survive Pod restarts/rescheduling. On a bare kubeadm cluster (no cloud provisioner), PVs are typically created manually with `hostPath` or NFS.
**Example:** `k8s/04-app-pv.yaml` and `k8s/08-postgres-pv.yaml` — manually-provisioned `hostPath` volumes.

### PersistentVolumeClaim (PVC)
**What it is:** A namespaced request for storage made by a user/Pod, specifying size and access mode; Kubernetes binds it to a matching PV (or dynamically provisions one via a StorageClass).
**Problem it solves:** Pods shouldn't need to know low-level storage details (hostPath, EBS volume ID, NFS server). A PVC is the abstraction a Pod spec references; the PV behind it can change without touching the Pod spec.
**When to use it:** Whenever a Pod needs persistent storage — always go through a PVC rather than referencing a PV or hostPath directly in the Pod spec.
**Example:** `k8s/05-app-pvc.yaml` binds to `app-pv` and is mounted by the Deployment in `06-app-deployment.yaml`.

### StorageClass
**What it is:** A named "class" of storage that either describes parameters for a dynamic provisioner (e.g. AWS EBS `gp3`, IOPS, encryption) or, with `provisioner: kubernetes.io/no-provisioner`, simply groups PVs meant to be manually/statically bound.
**Problem it solves:** Lets PVCs request "a class of storage" (e.g. "fast SSD") without the requester needing to know provisioning details, and lets admins offer several tiers of storage side by side.
**When to use it:** Any cluster where storage should be provisioned automatically on demand (cloud clusters); on bare kubeadm clusters it's still useful as a matching label between manually-created PVs and PVCs, as done here.
**Example:** `k8s/03-storageclass-manual.yaml`, referenced by both `app-pvc` and the Postgres `volumeClaimTemplates`.

### Secret
**What it is:** An API object for storing small amounts of sensitive data (passwords, tokens, keys), base64-encoded in the API and made available to Pods as env vars or mounted files.
**Problem it solves:** Keeps sensitive values out of Pod specs, container images, and ConfigMaps, and lets access be controlled separately (RBAC) from the workloads that consume them.
**When to use it:** Any credential, API key, TLS certificate/key, or other sensitive value a Pod needs at runtime. Note base64 alone is *not* encryption — see Part 10 for etcd encryption at rest.
**Example:** `k8s/01-secret-test.yaml`, and `postgres-credentials` in `09-postgres-statefulset.yaml` consumed via `secretKeyRef`.

### ConfigMap
**What it is:** An API object for storing non-sensitive configuration data as key-value pairs, consumable by Pods as env vars, command-line args, or mounted files.
**Problem it solves:** Separates configuration from container images, so the same image can run in different environments (dev/staging/prod) by swapping which ConfigMap it's given, without rebuilding the image.
**When to use it:** Any non-sensitive configuration — feature flags, URLs, log levels, application config files. Use a Secret instead for anything sensitive.
**Example:** Not currently in this repo's manifests, but e.g. `POSTGRES_DB`/`POSTGRES_USER` values in `09-postgres-statefulset.yaml` are good ConfigMap candidates (they're plain-text config, only `POSTGRES_PASSWORD` needs to be a Secret).

---

## Part 10 — Kubernetes Security

**Kubernetes Secrets** — Secrets take sensitive values out of Pod specs/images and let access be controlled independently via RBAC. On their own they only provide *separation*, not protection: by default they're stored in etcd as base64 (trivially reversible) and any Pod/user with API access to read a Secret gets the plaintext value. Treat Secrets as "controlled-access," not "encrypted," unless the next two controls are also in place.

**Encryption at rest** — By default, everything the API server writes to etcd — including Secrets — is stored effectively in plaintext (base64). Encryption at rest configures the API server (`--encryption-provider-config`) to encrypt specific resource types (typically `secrets`) before they're written to etcd, using a provider such as `aescbc` or `kms`. See `k8s/02-etcd-encryption-config.yaml` for a working example. Encryption only applies to *new writes*, so existing Secrets must be rewritten (`kubectl get secrets -A -o json | kubectl replace -f -`) after enabling it; the encryption key itself must be protected (ideally via a KMS provider rather than a local key file).

**etcd security** — etcd holds the entire cluster state, including every Secret — read access to etcd is equivalent to cluster-admin-plus. It should run only on control-plane nodes, never be exposed beyond `localhost`/the control-plane private network, require mutual TLS for all client and peer connections (kubeadm sets this up by default — client certs under `/etc/kubernetes/pki/etcd/`), and have its data directory encrypted/protected at the filesystem level and included in access-restricted, encrypted backups.

**RBAC** — Role-Based Access Control restricts *who* can do *what* to *which resources*, via `Role`/`ClusterRole` (permissions) bound to users/groups/ServiceAccounts via `RoleBinding`/`ClusterRoleBinding`. Without RBAC properly scoped, any authenticated identity can act with full cluster-admin power. Default to least privilege, avoid binding to `cluster-admin` except for actual cluster operators, and scope Roles to a namespace whenever the workload doesn't need cluster-wide access.

**Service Accounts** — Every Pod runs as a ServiceAccount (default: the `default` SA in its namespace), which is how it authenticates to the API server. The `default` SA is often over-permissioned or auto-mounted into every Pod even when the Pod never talks to the API. Give workloads dedicated ServiceAccounts scoped via RBAC to only the permissions they need, and set `automountServiceAccountToken: false` on Pods/SAs that don't need API access at all.

**Network Policies** — By default, Kubernetes networking is flat — any Pod can reach any other Pod in the cluster. A `NetworkPolicy` (enforced by the CNI plugin — Calico, Cilium, etc.; kubeadm's default `flannel` does *not* enforce them) restricts which Pods can talk to which, by label/namespace/port. Default-deny ingress per namespace, then explicitly allow only the traffic paths a workload actually needs (e.g. only the app Deployment can reach the Postgres StatefulSet on port 5432).

**TLS** — All control-plane communication (API server ↔ kubelet, API server ↔ etcd, client ↔ API server) should be encrypted with TLS, using certificates kubeadm generates automatically under `/etc/kubernetes/pki/`. Beyond the control plane, application ingress traffic should also terminate TLS (e.g. via an Ingress controller with cert-manager/Let's Encrypt) rather than serving plaintext HTTP externally.

**API Server security** — The API server is the single control point for the whole cluster and the highest-value target. Harden it by disabling anonymous/insecure access (no `--insecure-port`), enabling and correctly ordering authorization modes (`Node,RBAC`), enabling admission controllers (`PodSecurity`, `NodeRestriction`), restricting which networks can reach port 6443 (security group/firewall), and enabling audit logging (`--audit-log-path`) so privileged actions are traceable.

**Node security** — Worker/control-plane nodes should run a minimal, regularly-patched OS, expose only the ports Kubernetes needs (kubelet 10250, etcd 2379-2380 on control planes only, API server 6443), have SSH access locked down (key-only, restricted security groups/bastion), and use `NodeRestriction` admission plus kubelet authn/authz so a compromised kubelet can't read/modify objects outside its own node.

**Container image security** — Only pull images from trusted registries, scan images for known CVEs (Trivy, Grype, or Docker Hub/ECR scanning) before deploying, pin to specific digests/tags rather than floating `latest` in production, and avoid running containers as root inside the image (`USER` directive / `runAsNonRoot`) to reduce the blast radius of a container escape.

**Pod security** — Enforce least-privilege at the Pod level via `securityContext`: `runAsNonRoot: true`, drop all Linux capabilities and add back only what's needed, `readOnlyRootFilesystem: true` where possible, and disallow privilege escalation (`allowPrivilegeEscalation: false`). Cluster-wide, use the built-in Pod Security Admission (`restricted`/`baseline` levels per namespace) to reject Pod specs that violate these rules.

**Kubernetes version and patch management** — Kubernetes has a defined support window (typically the latest 3 minor versions); running an unsupported version means no security patches. Track the upstream release/CVE feed, patch control-plane and node components on a regular cadence, and upgrade one minor version at a time (`kubeadm upgrade`) rather than large multi-version jumps.

**Limiting public exposure of cluster components** — Only the application's own ingress/NodePort traffic should be reachable from the public internet — the API server (6443), etcd (2379-2380), and kubelet (10250) should be restricted to the private VPC/security group and never exposed publicly. On AWS EC2, this means locking down security groups to specific CIDR ranges (or a bastion/VPN) for control-plane ports, while only the NodePort/LoadBalancer port used for the app itself is opened broadly.

---

## Part 11 — Deployment Automation

**Building a new application image** — Every push to `main` triggers a CI build: the pipeline checks out the repo, installs dependencies (`npm install`), and builds a new Docker image from the `Dockerfile` at the repo root (`docker build -t roy35/node-express-app:<tag> .`). This repo's `Jenkinsfile` already implements this stage.

**Tagging the new image version** — Each build is tagged twice: an immutable, traceable tag (this repo uses the Jenkins `BUILD_NUMBER`, e.g. `roy35/node-express-app:42`) and a floating `latest` tag for convenience. The immutable tag is what should actually be referenced in the Kubernetes Deployment, since `latest` gives no way to know or roll back to a specific version.

**Pushing the image to Docker Hub** — After a successful build, the pipeline authenticates to Docker Hub with credentials stored in the CI system's secret store (this repo's `Jenkinsfile` uses a `dockerhub-credentials` Jenkins credential, piped into `docker login` via stdin — never as a plaintext CLI arg), then pushes both the versioned and `latest` tags.

**Updating the Kubernetes Deployment** — Once the new image is in Docker Hub, the cluster needs to be told to use it. Two common approaches:
- *Imperative (fast, good for demos):* `kubectl set image deployment/node-express-app node-express-app=roy35/node-express-app:<new-tag>`
- *Declarative (better for real pipelines):* update the `image:` field in `k8s/06-app-deployment.yaml` (ideally via a templating/patch step in CI, e.g. `yq` or `kustomize edit set image`), commit it, and apply with `kubectl apply -f k8s/06-app-deployment.yaml`. This keeps the manifest in git as the source of truth (GitOps), rather than the cluster's state drifting from what's in version control.

Because the Deployment's Pod template changed (new image), Kubernetes creates a new ReplicaSet and performs a rolling update automatically — no manual Pod deletion needed.

**Verifying the new version:**
```bash
kubectl rollout status deployment/node-express-app
kubectl get pods -l app=node-express-app -o jsonpath='{.items[*].spec.containers[0].image}'
curl http://<node-ip>:30080/api   # sanity-check the app actually responds
```
If the rollout fails health checks, `kubectl rollout undo deployment/node-express-app` reverts to the previous ReplicaSet/image.

**Automating the process using CI/CD** — Tying the above into a single automated pipeline (this repo already has the skeleton in `Jenkinsfile` and `.github/workflows/`):
1. **Trigger:** on push/merge to `main` (or on a tag/release for stricter environments).
2. **Test stage:** `npm install && npm run check` — fail fast before building an image from broken code.
3. **Build & tag stage:** `docker build`, tag with both a unique identifier (commit SHA or build number) and `latest`.
4. **Push stage:** authenticate to Docker Hub using CI-managed secrets, push both tags.
5. **Deploy stage:** update the Deployment's image (via `kubectl set image` or a GitOps commit) against the target cluster, using a `kubeconfig` stored as a CI secret (or a self-hosted runner with cluster access, as this repo's GitHub Actions workflow does for the `Deploy` job).
6. **Verify stage:** `kubectl rollout status` as the pipeline's success/failure signal — if the rollout doesn't finish within a timeout, fail the pipeline and optionally auto-rollback.

This closes the loop so a merged code change reaches the running cluster with no manual `docker build`/`kubectl` steps, while every deployed version stays traceable to the commit and image tag that produced it.

---

## Part 12 — Kubernetes Deployment Strategies

### Rolling Update
**How it works:** The default strategy for a Kubernetes `Deployment`. New Pods (new ReplicaSet) are created incrementally while old Pods (old ReplicaSet) are terminated incrementally, controlled by `maxSurge` (how many extra Pods above `replicas` can exist during the rollout) and `maxUnavailable` (how many can be down at once). At every point during the rollout, a mix of old and new Pods serves traffic.
**Advantages:** Zero downtime (with `maxUnavailable > 0` avoided or traffic-aware readiness probes); no extra infrastructure needed beyond what the Deployment already manages; built in natively; easy automatic rollback via `kubectl rollout undo`.
**Limitations:** Both old and new versions run simultaneously for the duration of the rollout, which requires the two versions to be backward/forward compatible (e.g. shared database schema must work with both); a bad new version still gets *some* live traffic before health checks catch it and halt the rollout; slower than an instant cutover.
**When to use it:** The default choice for most stateless services — this repo's `node-express-app` Deployment (`k8s/06-app-deployment.yaml`) uses it implicitly, since no `strategy` override is set.

### Recreate
**How it works:** All existing Pods are terminated first, then new Pods are created only after the old ones are fully gone. Set via `spec.strategy.type: Recreate` on a Deployment.
**Advantages:** Simple and guarantees only one version's Pods ever run at once — no risk of old/new version incompatibility during the transition; useful when the app can't tolerate two versions running concurrently (e.g. it holds an exclusive lock, or a schema migration is breaking/non-backward-compatible).
**Limitations:** Guaranteed downtime between the old Pods stopping and new Pods becoming ready — unacceptable for most user-facing services.
**When to use it:** Non-critical or internal services where brief downtime is acceptable, or any deployment where running two versions simultaneously would actively cause incorrect behavior (e.g. a singleton batch processor, or a breaking DB migration that isn't backward-compatible).

### Canary Deployment
**How it works:** A small subset of new-version Pods ("the canary") is deployed alongside the full set of old-version Pods, and a small percentage of live traffic is routed to the canary (via a Service that matches both versions' labels with weighted routing, or more precisely via a service-mesh/ingress controller like Istio, Linkerd, or NGINX Ingress with canary annotations). If the canary looks healthy (error rates, latency, business metrics), traffic is gradually shifted until the canary fully replaces the old version; if not, it's rolled back having only affected a small slice of users.
**Advantages:** Limits the blast radius of a bad release to a small fraction of real traffic instead of all of it; allows validating a new version against real production traffic/data before full rollout; can be tied to automated metrics-based promotion/rollback.
**Limitations:** Not natively supported by a plain Kubernetes Deployment/Service — needs an ingress controller, service mesh, or a tool like Argo Rollouts/Flagger to do weighted/percentage-based traffic splitting; more operational complexity to set up and monitor; like Rolling Update, requires the two versions to be compatible with shared state/data during the overlap.
**When to use it:** High-traffic or high-risk production services where you want real-user validation before a full rollout, and where the infrastructure to support traffic splitting (mesh/ingress) already exists or is worth the investment.

### Comparison & Use Cases

| Strategy | Downtime | Version overlap | Rollback risk exposure | Extra tooling needed |
|---|---|---|---|---|
| Rolling Update | None (tunable) | Yes, brief | Some traffic hits bad version before detection | None (built into Deployment) |
| Recreate | Yes | None | None hit bad version until it's fully up | None (built into Deployment) |
| Canary | None | Yes, prolonged/controlled | Only the canary slice hits bad version | Ingress/mesh or rollout controller |

- **Rolling Update** — default for `node-express-app`-style stateless web APIs where brief version overlap is safe.
- **Recreate** — a job/worker that can't have two instances racing on the same external resource, or a release paired with a breaking DB migration.
- **Canary** — a customer-facing checkout service where a bad release must be caught against a small slice of real traffic before it reaches everyone.

---

## Checklist before submitting

- [ ] All `screenshots/part*.png` files added and rendering above (no broken image links)
- [ ] Placeholders at the top of this file filled in (name, Docker Hub URL, EC2 IP)
- [ ] Docker Hub repo confirmed public/accessible and shows the pushed tags
- [ ] `kubectl get nodes`, `get pods -A`, `get deploy/rs/pods/svc/pv/pvc/statefulset` screenshots all show healthy/Running/Bound state, not errors
- [ ] Application reachable externally via `http://<NODE_PUBLIC_IP>:30080/`
- [ ] Exported this file to PDF/Word (or shared as a rendered Markdown/GitHub link) for final submission
