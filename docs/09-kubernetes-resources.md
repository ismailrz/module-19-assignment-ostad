# Part 9 — Kubernetes Resource Documentation

For each resource: what it is, what problem it solves, when to use it, and a practical example (drawn from this project's own manifests where possible).

---

## Pod
**What it is:** The smallest deployable unit in Kubernetes — one or more containers that share network namespace (same IP/port space) and storage volumes, always scheduled together on the same node.

**Problem it solves:** Containers on their own have no concept of "this group of processes belongs together." A Pod groups tightly-coupled containers (e.g. an app + a log-shipping sidecar) so they share a lifecycle, network, and volumes.

**When to use it:** Rarely created directly in production — normally you let a Deployment, StatefulSet, DaemonSet, or Job create and manage Pods for you, since a bare Pod has no self-healing (if the node dies, the Pod is gone for good).

**Example:** `k8s/00-static-nginx-pod.yaml` in this repo — a single-container Pod running nginx.

---

## Deployment
**What it is:** A controller that manages a set of identical Pods via an underlying ReplicaSet, and drives the Pods from one desired state to another (e.g. rolling out a new image version).

**Problem it solves:** Manually creating/updating/deleting Pods to roll out changes or recover from failures doesn't scale and is error-prone. A Deployment declares "I want N replicas of this Pod spec running" and continuously reconciles reality to match that.

**When to use it:** Any stateless workload — web servers, APIs, background workers — where any replica can serve any request and none needs stable identity or storage.

**Example:** `k8s/06-app-deployment.yaml` — runs 2 replicas of `node-express-app`.

---

## ReplicaSet
**What it is:** A controller that ensures a specified number of identical Pod replicas are running at all times. It's the mechanism a Deployment uses under the hood.

**Problem it solves:** Guarantees self-healing at the Pod-count level — if a Pod is deleted or its node fails, the ReplicaSet notices the count dropped below desired and creates a replacement.

**When to use it:** Almost never created directly — Deployments manage ReplicaSets for you (one ReplicaSet per revision, which is how rolling updates and rollbacks work).

**Example:** `kubectl get rs` after applying `06-app-deployment.yaml` shows the ReplicaSet the Deployment created and owns.

---

## StatefulSet
**What it is:** Like a Deployment, but for workloads that need stable, unique network identities (`<name>-0`, `<name>-1`, ...) and stable per-replica persistent storage that follows the Pod across rescheduling.

**Problem it solves:** Deployments treat all Pods as interchangeable and give each a random name/IP — that breaks stateful systems (databases, message queues) that need each replica to keep its own identity and its own data volume across restarts.

**When to use it:** Databases, clustered systems where members need to find each other by stable name (e.g. via a headless Service), or anything where "replica 0" must always come back as "replica 0" with its own disk.

**Example:** `k8s/09-postgres-statefulset.yaml` — PostgreSQL, one replica, each pod gets its own PVC via `volumeClaimTemplates`.

---

## DaemonSet
**What it is:** A controller that ensures exactly one copy of a Pod runs on every node (or every node matching a selector) in the cluster, automatically adding/removing Pods as nodes join or leave.

**Problem it solves:** Some workloads are inherently "one per node" — log collectors, node monitoring agents, network/CNI plugins — and need to scale with the cluster's node count, not be manually replicated.

**When to use it:** Cluster-wide infrastructure agents: `kube-proxy`, CNI plugins (Calico/Flannel), log shippers (Fluentd/Filebeat), node exporters for monitoring.

**Example:** kubeadm itself deploys `kube-proxy` as a DaemonSet — check with `kubectl get daemonset -n kube-system`.

---

## Static Pod
**What it is:** A Pod defined directly on a node by dropping a manifest file into a directory the kubelet watches (default `/etc/kubernetes/manifests`), managed by that node's kubelet instead of the API server/scheduler.

**Problem it solves:** Bootstrapping problem — control-plane components (`kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `etcd`) need to run as Pods before the API server itself is even up. Static Pods let the kubelet run them without depending on the API server existing yet.

**When to use it:** Control-plane components (this is exactly how kubeadm runs them), or node-specific one-off Pods that must survive even if the API server is unreachable.

**Example:** `k8s/00-static-nginx-pod.yaml`, deployed by copying it into `/etc/kubernetes/manifests/` on a node rather than via `kubectl apply`.

---

## Service
**What it is:** A stable virtual IP and DNS name that load-balances traffic across a dynamic set of Pods matched by a label selector.

**Problem it solves:** Pod IPs are ephemeral — they change every time a Pod is recreated. A Service gives clients one stable address to talk to regardless of which Pods are currently backing it.

**When to use it:** Any time one set of Pods (or an outside client) needs to reliably reach another set of Pods, whether inside the cluster (`ClusterIP`), from outside via each node (`NodePort`), or via a cloud load balancer (`LoadBalancer`).

**Example:** `k8s/09-postgres-statefulset.yaml` includes a headless Service (`clusterIP: None`) so each Postgres Pod gets its own stable DNS record instead of a load-balanced VIP.

---

## NodePort
**What it is:** A Service type that opens a fixed port (default range 30000-32767) on every node in the cluster and forwards traffic from `<any-node-IP>:<nodePort>` to the Service's backing Pods.

**Problem it solves:** Gives external clients a way to reach a Service without a cloud load balancer — useful when running on bare-metal/self-managed clusters (like kubeadm on EC2) where a `LoadBalancer` Service type has nothing to provision it.

**When to use it:** Bare-metal/self-hosted clusters, demos, or dev/test environments needing external access without extra infrastructure. Not typically used directly in production behind a real ingress/load balancer.

**Example:** `k8s/07-app-service-nodeport.yaml` exposes `node-express-app` on port `30080` on every node's IP.

---

## PersistentVolume (PV)
**What it is:** A cluster-scoped storage resource representing an actual piece of storage (a directory, an EBS volume, an NFS export, etc.), provisioned either manually by an admin (static) or automatically by a StorageClass (dynamic).

**Problem it solves:** Decouples "what storage physically exists" from "which Pod is using it" — storage has its own lifecycle independent of any one Pod or even the cluster's Pods being deleted/recreated.

**When to use it:** Any time a workload needs data to survive Pod restarts/rescheduling. On a bare kubeadm cluster (no cloud provisioner), PVs are typically created manually with `hostPath` or NFS.

**Example:** `k8s/04-app-pv.yaml` and `k8s/08-postgres-pv.yaml` — manually-provisioned `hostPath` volumes.

---

## PersistentVolumeClaim (PVC)
**What it is:** A namespaced request for storage made by a user/Pod, specifying size and access mode; Kubernetes binds it to a matching PV (or dynamically provisions one via a StorageClass).

**Problem it solves:** Pods shouldn't need to know low-level storage details (hostPath, EBS volume ID, NFS server). A PVC is the abstraction a Pod spec references; the PV behind it can change without touching the Pod spec.

**When to use it:** Whenever a Pod needs persistent storage — always go through a PVC rather than referencing a PV or hostPath directly in the Pod spec.

**Example:** `k8s/05-app-pvc.yaml` binds to `app-pv` and is mounted by the Deployment in `06-app-deployment.yaml`.

---

## StorageClass
**What it is:** A named "class" of storage that either describes parameters for a dynamic provisioner (e.g. AWS EBS `gp3`, IOPS, encryption) or, with `provisioner: kubernetes.io/no-provisioner`, simply groups PVs meant to be manually/statically bound.

**Problem it solves:** Lets PVCs request "a class of storage" (e.g. "fast SSD") without the requester needing to know provisioning details, and lets admins offer several tiers of storage side by side.

**When to use it:** Any cluster where storage should be provisioned automatically on demand (cloud clusters); on bare kubeadm clusters it's still useful as a matching label between manually-created PVs and PVCs, as done here.

**Example:** `k8s/03-storageclass-manual.yaml`, referenced by both `app-pvc` and the Postgres `volumeClaimTemplates`.

---

## Secret
**What it is:** An API object for storing small amounts of sensitive data (passwords, tokens, keys), base64-encoded in the API and made available to Pods as env vars or mounted files.

**Problem it solves:** Keeps sensitive values out of Pod specs, container images, and ConfigMaps, and lets access be controlled separately (RBAC) from the workloads that consume them.

**When to use it:** Any credential, API key, TLS certificate/key, or other sensitive value a Pod needs at runtime. Note base64 alone is *not* encryption — see Part 10 for etcd encryption at rest.

**Example:** `k8s/01-secret-test.yaml`, and `postgres-credentials` in `09-postgres-statefulset.yaml` consumed via `secretKeyRef`.

---

## ConfigMap
**What it is:** An API object for storing non-sensitive configuration data as key-value pairs, consumable by Pods as env vars, command-line args, or mounted files.

**Problem it solves:** Separates configuration from container images, so the same image can run in different environments (dev/staging/prod) by swapping which ConfigMap it's given, without rebuilding the image.

**When to use it:** Any non-sensitive configuration — feature flags, URLs, log levels, application config files. Use a Secret instead for anything sensitive.

**Example:** Not currently in this repo's manifests, but e.g. `POSTGRES_DB`/`POSTGRES_USER` values in `09-postgres-statefulset.yaml` are good ConfigMap candidates (they're plain-text config, only `POSTGRES_PASSWORD` needs to be a Secret).
