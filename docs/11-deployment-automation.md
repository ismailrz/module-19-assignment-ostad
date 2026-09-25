# Part 11 — Deployment Automation

## Building a new application image
Every push to `main` triggers a CI build: the pipeline checks out the repo, installs dependencies (`npm install`), and builds a new Docker image from the `Dockerfile` at the repo root (`docker build -t roy35/node-express-app:<tag> .`). This repo's `Jenkinsfile` already implements this stage.

## Tagging the new image version
Each build is tagged twice: an immutable, traceable tag (this repo uses the Jenkins `BUILD_NUMBER`, e.g. `roy35/node-express-app:42`) and a floating `latest` tag for convenience. The immutable tag is what should actually be referenced in the Kubernetes Deployment, since `latest` gives no way to know or roll back to a specific version.

## Pushing the image to Docker Hub
After a successful build, the pipeline authenticates to Docker Hub with credentials stored in the CI system's secret store (this repo's `Jenkinsfile` uses a `dockerhub-credentials` Jenkins credential, piped into `docker login` via stdin — never as a plaintext CLI arg), then pushes both the versioned and `latest` tags.

## Updating the Kubernetes Deployment
Once the new image is in Docker Hub, the cluster needs to be told to use it. Two common approaches:
- **Imperative (fast, good for demos):**
  `kubectl set image deployment/node-express-app node-express-app=roy35/node-express-app:<new-tag>`
- **Declarative (better for real pipelines):** update the `image:` field in `k8s/06-app-deployment.yaml` (ideally via a templating/patch step in CI, e.g. `yq` or `kustomize edit set image`), commit it, and apply with `kubectl apply -f k8s/06-app-deployment.yaml`. This keeps the manifest in git as the source of truth (GitOps), rather than the cluster's state drifting from what's in version control.

Because the Deployment's Pod template changed (new image), Kubernetes creates a new ReplicaSet and performs a rolling update automatically — no manual Pod deletion needed.

## Verifying the new version
After updating, confirm the rollout completed and the right version is live:
```bash
kubectl rollout status deployment/node-express-app
kubectl get pods -l app=node-express-app -o jsonpath='{.items[*].spec.containers[0].image}'
curl http://<node-ip>:30080/api   # sanity-check the app actually responds
```
If the rollout fails health checks, `kubectl rollout undo deployment/node-express-app` reverts to the previous ReplicaSet/image.

## Automating the process using CI/CD
Tying the above into a single automated pipeline (this repo already has the skeleton in `Jenkinsfile` and `.github/workflows/`):
1. **Trigger:** on push/merge to `main` (or on a tag/release for stricter environments).
2. **Test stage:** `npm install && npm run check` — fail fast before building an image from broken code.
3. **Build & tag stage:** `docker build`, tag with both a unique identifier (commit SHA or build number) and `latest`.
4. **Push stage:** authenticate to Docker Hub using CI-managed secrets, push both tags.
5. **Deploy stage:** update the Deployment's image (via `kubectl set image` or a GitOps commit) against the target cluster, using a `kubeconfig` stored as a CI secret (or a self-hosted runner with cluster access, as this repo's GitHub Actions workflow does for the `Deploy` job).
6. **Verify stage:** `kubectl rollout status` as the pipeline's success/failure signal — if the rollout doesn't finish within a timeout, fail the pipeline and optionally auto-rollback.

This closes the loop so a merged code change reaches the running cluster with no manual `docker build`/`kubectl` steps, while every deployed version stays traceable to the commit and image tag that produced it.
