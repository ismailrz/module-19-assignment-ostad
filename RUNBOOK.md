# RUNBOOK — Docker & Kubernetes Deployment on AWS EC2

Execute these on your own machine / AWS EC2 instances (I can't provision AWS
resources or take screenshots for you). Every command block below maps to a
Part in `requirements.md`. Take the listed screenshot(s) right after each
✅ step — don't wait until the end.

Replace placeholders as you go: `<DOCKERHUB_USERNAME>` (Jenkinsfile in this
repo already uses `roy35`), `<CONTROL_PLANE_IP>`, `<WORKER_NODE_NAME>`.

---

## Part 1 — Containerize the app

The app is a small Express server (`src/server.js`), listens on port `5000`
(`PORT` env override), routes: `/` (HTML) and `/api` (JSON). `Dockerfile` at
the repo root already builds/runs it correctly — no changes needed.

```bash
# Build
docker build -t <DOCKERHUB_USERNAME>/node-express-app:v1 .

# Run locally and verify
docker run -d -p 5000:5000 --name node-app-test <DOCKERHUB_USERNAME>/node-express-app:v1
curl http://localhost:5000/
curl http://localhost:5000/api
docker ps                       # 📸 screenshot: container running
docker logs node-app-test       # 📸 screenshot: server started log line

# Push
docker login
docker push <DOCKERHUB_USERNAME>/node-express-app:v1
docker tag <DOCKERHUB_USERNAME>/node-express-app:v1 <DOCKERHUB_USERNAME>/node-express-app:latest
docker push <DOCKERHUB_USERNAME>/node-express-app:latest
```
📸 Screenshot: `curl` output from `/` and `/api`, plus the Docker Hub repo page showing the pushed tags.

```bash
docker rm -f node-app-test
```

---

## Part 2 — Kubernetes cluster on EC2

On AWS: launch 2+ EC2 instances (e.g. Ubuntu 22.04, t3.medium+), one as
control-plane and the rest as workers, in the same VPC/subnet. Security
group must allow, between the instances: TCP 6443 (API server), 2379-2380
(etcd, control-plane only), 10250 (kubelet), 10251/10252 (scheduler/
controller-manager, control-plane only), and UDP/TCP for your CNI (e.g.
Flannel needs UDP 8472). Allow SSH (22) from your IP only.

On **every** node:
```bash
sudo swapoff -a   # kubeadm requires swap disabled
# install a container runtime (containerd) and kubeadm/kubelet/kubectl
# per the official Kubernetes docs for your OS/version
```

On the **control-plane** node:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# install a CNI, e.g. Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
Save the `kubeadm join ...` command it prints.

On **each worker** node:
```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

Verify from the control plane:
```bash
kubectl get nodes -o wide       # 📸 screenshot: all nodes Ready
kubectl get pods -A             # 📸 screenshot: kube-system pods Running
```
📸 Screenshot: AWS EC2 console showing the running instances.

---

## Part 3 — Secure etcd

```bash
# 1. Test BEFORE encryption
kubectl apply -f k8s/01-secret-test.yaml
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-secret | hexdump -C
# 📸 screenshot: readable base64 values in the etcd record
```

Follow the setup steps at the top of `k8s/02-etcd-encryption-config.yaml`
(generate key, place config, edit `kube-apiserver.yaml`), then:

```bash
kubectl get --raw='/readyz'                            # confirm API server is back
kubectl get secrets --all-namespaces -o json | kubectl replace -f -   # re-encrypt existing secrets

sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-secret | hexdump -C
# 📸 screenshot: now prefixed k8s:enc:aescbc:v1:key1: followed by ciphertext
```
📸 Screenshot: `cat /etc/kubernetes/manifests/kube-apiserver.yaml` showing the `--encryption-provider-config` flag.

---

## Part 4 — Static NGINX Pod

```bash
sudo cp k8s/00-static-nginx-pod.yaml /etc/kubernetes/manifests/static-nginx.yaml
kubectl get pods -A | grep static-nginx    # 📸 screenshot: Running

curl http://localhost/           # from the node that's running it
# or: kubectl port-forward pod/<static-nginx-pod-name> 8080:80 (from control plane) then curl localhost:8080
```
📸 Screenshot: the default "Welcome to nginx!" page in a browser or via curl.

---

## Part 5 — Deploy the application

```bash
kubectl apply -f k8s/06-app-deployment.yaml   # after PV/PVC in Part 7 exist, or apply Part 7 first
kubectl get deployments                       # 📸 screenshot
kubectl get rs                                # 📸 screenshot
kubectl get pods -l app=node-express-app -o wide  # 📸 screenshot
```

---

## Part 6 — PostgreSQL StatefulSet

```bash
# on the chosen node:
sudo mkdir -p /mnt/data/postgres && sudo chmod 777 /mnt/data/postgres

kubectl apply -f k8s/03-storageclass-manual.yaml
kubectl apply -f k8s/08-postgres-pv.yaml        # fill in <WORKER_NODE_NAME> first
kubectl apply -f k8s/09-postgres-statefulset.yaml

kubectl get statefulset postgres     # 📸 screenshot
kubectl get pods -l app=postgres     # 📸 screenshot: postgres-0 Running
kubectl get pvc                      # 📸 screenshot: Bound
```

---

## Part 7 — PV and PVC for the application

```bash
# on the chosen node:
sudo mkdir -p /mnt/data/app && sudo chmod 777 /mnt/data/app

kubectl apply -f k8s/04-app-pv.yaml    # fill in <WORKER_NODE_NAME> first
kubectl apply -f k8s/05-app-pvc.yaml
kubectl get pv                          # 📸 screenshot: app-pv Bound
kubectl get pvc                         # 📸 screenshot: app-pvc Bound

# app-deployment.yaml (Part 5) already mounts this PVC at /app/data
kubectl exec -it deploy/node-express-app -- df -h /app/data   # 📸 screenshot: mounted volume
```

---

## Part 8 — Expose the application

```bash
kubectl apply -f k8s/07-app-service-nodeport.yaml
kubectl get svc node-express-app-svc    # 📸 screenshot: NodePort 30080
```
Open TCP 30080 in the EC2 security group (from your IP or 0.0.0.0/0 for the demo), then:
```bash
curl http://<ANY_NODE_PUBLIC_IP>:30080/
curl http://<ANY_NODE_PUBLIC_IP>:30080/api
```
📸 Screenshot: the app loading in a browser via `http://<NODE_PUBLIC_IP>:30080/`.

---

## Written submission (Parts 9-12)

Already drafted — see:
- `docs/09-kubernetes-resources.md`
- `docs/10-kubernetes-security.md`
- `docs/11-deployment-automation.md`
- `docs/12-deployment-strategies.md`

Assemble all screenshots + these docs into the final submission document per
`requirements.md`'s "Submission Requirements" section.
