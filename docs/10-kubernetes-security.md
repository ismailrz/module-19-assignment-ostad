# Part 10 — Securing a Kubernetes Cluster

## Kubernetes Secrets
Secrets take sensitive values out of Pod specs/images and let access be controlled independently via RBAC. On their own they only provide *separation*, not protection: by default they're stored in etcd as base64 (trivially reversible) and any Pod/user with API access to read a Secret gets the plaintext value. Treat Secrets as "controlled-access," not "encrypted," unless the next two controls are also in place.

## Encryption at rest
By default, everything the API server writes to etcd — including Secrets — is stored effectively in plaintext (base64). Encryption at rest configures the API server (`--encryption-provider-config`) to encrypt specific resource types (typically `secrets`) before they're written to etcd, using a provider such as `aescbc` or `kms`. See `k8s/02-etcd-encryption-config.yaml` in this repo for a working example. Key points: encryption only applies to *new writes*, so existing Secrets must be rewritten (`kubectl get secrets -A -o json | kubectl replace -f -`) after enabling it; and the encryption key itself must be protected (ideally via a KMS provider rather than a local key file) since anyone who gets the key can decrypt everything.

## etcd security
etcd holds the entire cluster state, including every Secret — read access to etcd is equivalent to cluster-admin-plus. It should: run only on control-plane nodes, never be exposed beyond `localhost`/the control-plane private network, require mutual TLS for all client and peer connections (kubeadm sets this up by default — client certs under `/etc/kubernetes/pki/etcd/`), and have its data directory encrypted/protected at the filesystem level and included in access-restricted, encrypted backups.

## RBAC
Role-Based Access Control restricts *who* can do *what* to *which resources*, via `Role`/`ClusterRole` (permissions) bound to users/groups/ServiceAccounts via `RoleBinding`/`ClusterRoleBinding`. Without RBAC properly scoped, any authenticated identity can act with full cluster-admin power. Practice: default to least privilege, avoid binding to `cluster-admin` except for actual cluster operators, and scope Roles to a namespace whenever the workload doesn't need cluster-wide access.

## Service Accounts
Every Pod runs as a ServiceAccount (default: the `default` SA in its namespace) which is how it authenticates to the API server. Problem: the `default` SA is often over-permissioned or auto-mounted into every Pod even when the Pod never talks to the API. Practice: give workloads dedicated ServiceAccounts scoped via RBAC to only the permissions they need, and set `automountServiceAccountToken: false` on Pods/SAs that don't need API access at all.

## Network Policies
By default, Kubernetes networking is flat — any Pod can reach any other Pod in the cluster. A `NetworkPolicy` (enforced by the CNI plugin — Calico, Cilium, etc.; kubeadm's default `flannel` does *not* enforce them) restricts which Pods can talk to which, by label/namespace/port. Practice: default-deny ingress per namespace, then explicitly allow only the traffic paths a workload actually needs (e.g. only the app Deployment can reach the Postgres StatefulSet on port 5432).

## TLS
All control-plane communication (API server ↔ kubelet, API server ↔ etcd, client ↔ API server) should be encrypted with TLS, using certificates kubeadm generates automatically under `/etc/kubernetes/pki/`. Beyond the control plane, application ingress traffic should also terminate TLS (e.g. via an Ingress controller with cert-manager/Let's Encrypt) rather than serving plaintext HTTP externally.

## API Server security
The API server is the single control point for the whole cluster and the highest-value target. Harden it by: disabling anonymous/insecure access (no `--insecure-port`), enabling and correctly ordering authorization modes (`Node,RBAC`), enabling admission controllers (`PodSecurity`, `NodeRestriction`), restricting which networks can reach port 6443 (security group / firewall), and enabling audit logging (`--audit-log-path`) so privileged actions are traceable.

## Node security
Worker/control-plane nodes should run a minimal, regularly-patched OS, expose only the ports Kubernetes needs (kubelet 10250, etcd 2379-2380 on control planes only, API server 6443), have SSH access locked down (key-only, restricted security groups/bastion), and use `NodeRestriction` admission plus kubelet authn/authz so a compromised kubelet can't read/modify objects outside its own node.

## Container image security
Only pull images from trusted registries, scan images for known CVEs (Trivy, Grype, or Docker Hub/ECR scanning) before deploying, pin to specific digests/tags rather than floating `latest` in production, and avoid running containers as root inside the image (`USER` directive / `runAsNonRoot`) to reduce blast radius of a container-escape.

## Pod security
Enforce least-privilege at the Pod level via `securityContext`: `runAsNonRoot: true`, drop all Linux capabilities and add back only what's needed, `readOnlyRootFilesystem: true` where possible, and disallow privilege escalation (`allowPrivilegeEscalation: false`). Cluster-wide, use the built-in Pod Security Admission (`restricted`/`baseline` levels per namespace) to reject Pod specs that violate these rules, rather than relying on every manifest author doing it correctly.

## Kubernetes version and patch management
Kubernetes has a defined support window (typically the latest 3 minor versions); running an unsupported version means no security patches. Practice: track the upstream release/CVE feed, patch control-plane and node components on a regular cadence, and upgrade one minor version at a time (kubeadm supports this directly via `kubeadm upgrade`) rather than large multi-version jumps.

## Limiting public exposure of cluster components
Only the application's own ingress/NodePort traffic should be reachable from the public internet — the API server (6443), etcd (2379-2380), and kubelet (10250) should be restricted to the private VPC/security group and never exposed publicly. On AWS EC2, this means locking down security groups to specific CIDR ranges (or a bastion/VPN) for control-plane ports, while only the NodePort/LoadBalancer port used for the app itself is opened broadly.
