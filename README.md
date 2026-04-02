# fleet

GitOps repo for my-cluster using Flux v2.

## Structure

```
clusters/my-cluster/
  flux-system/           # Flux bootstrap (DO NOT EDIT — managed by flux)
  <app-name>/            # one directory per app
    gitrepository.yaml
    flux-kustomization.yaml
    image-automation.yaml  # ImageRepository + ImagePolicy + ImageUpdateAutomation
    kustomization.yaml     # Kustomize overlay that ties the above together
```

## Apps

| App        | Repo              | Branch | Namespace | Image                     | Registry |
| ---------- | ----------------- | ------ | --------- | ------------------------- | -------- |
| chat2md    | `dedkola/chat2MD` | main   | default   | `ghcr.io/dedkola/chat2md` | public   |
| tk-doc     | `dedkola/tk-doc`  | main   | default   | `ghcr.io/dedkola/tk-doc`  | private  |
| tk-doc-dev | `dedkola/tk-doc`  | dev    | dev       | `ghcr.io/dedkola/tk-doc`  | private  |

---

## Part 1 — Bootstrap Flux on a fresh k3s cluster

### Prerequisites

- A running k3s node with `kubectl` access (`KUBECONFIG` set or `~/.kube/config` present)
- [Flux CLI](https://fluxcd.io/flux/installation/) installed
- A GitHub PAT with `repo` scope

### Step 1 — Verify the cluster

```bash
kubectl get nodes
# should show your node(s) in Ready state
```

### Step 2 — Export your GitHub token

```bash
export GITHUB_TOKEN=<your-pat>
```

### Step 3 — Run the bootstrap

```bash
flux bootstrap github \
  --owner=dedkola \
  --repository=fleet \
  --branch=main \
  --path=clusters/my-cluster \
  --components-extra=image-reflector-controller,image-automation-controller
```

This will:

1. Install Flux controllers (including image automation) into the `flux-system` namespace
2. Create a `GitRepository` + `Kustomization` pointing at `clusters/my-cluster`
3. Push the generated manifests to the repo

### Step 4 — Fix the SSH secret

The bootstrap creates an HTTPS-based secret, but `gotk-sync.yaml` uses an SSH URL.
You need to replace the secret with SSH credentials.

```bash
# Delete the HTTPS-only secret
kubectl delete secret flux-system -n flux-system

# Recreate with the SSH deploy key and known_hosts
kubectl create secret generic flux-system \
  --namespace=flux-system \
  --from-file=identity=./identity \
  --from-file=identity.pub=./identity.pub \
  --from-file=known_hosts=<(ssh-keyscan github.com 2>/dev/null)
```

> **Note:** The `identity` / `identity.pub` files are the SSH deploy key for this repo.
> They must be registered as a deploy key on GitHub → Repo → Settings → Deploy keys.

### Step 5 — Trigger reconciliation

```bash
flux reconcile source git flux-system
flux reconcile kustomization flux-system
```

### Step 6 — Verify everything is healthy

```bash
flux check
flux get all -A
```

All resources should show `Ready: True`.

---

## Part 2 — Regenerate / rotate the SSH deploy key

Use this when the deploy key is lost, compromised, or you're setting up a new cluster from scratch.

### Step 1 — Generate a new ed25519 key pair

```bash
ssh-keygen -t ed25519 -C "flux" -f ./identity -N ""
```

This creates `identity` (private) and `identity.pub` (public) in the repo root.
Both are listed in `.gitignore` — **never commit private keys**.

### Step 2 — Register the public key on GitHub

1. Go to **github.com/dedkola/fleet** → **Settings** → **Deploy keys**
2. Remove the old deploy key (if any)
3. Click **Add deploy key**
4. Paste the contents of `identity.pub`
5. Check **Allow write access** (required for image automation to push commits)
6. Save

### Step 3 — Update the Kubernetes secret

# Git auth (HTTPS) — one PAT for all repos

```
kubectl create secret generic fleet-secret \
  --namespace=flux-system \
  --from-literal=username=dedkola \
  --from-literal=password=pat

  kubectl create secret generic flux-system \
  --namespace=flux-system \
  --from-literal=username=dedkola  \
  --from-literal=password=pat
```

## Part 3 — Add a new app to the fleet

### Step 1 — Create the app directory

```bash
APP=my-app
mkdir -p clusters/my-cluster/$APP
```

### Step 2 — Create `gitrepository.yaml`

```yaml
# clusters/my-cluster/<app>/gitrepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: <app>
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system # use a different secret for private app repos
  url: https://github.com/dedkola/<app>.git
```

> For private repos, create a dedicated secret with a PAT or SSH key and reference it in `secretRef`.

### Step 3 — Create `flux-kustomization.yaml`

```yaml
# clusters/my-cluster/<app>/flux-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: <app>
  namespace: flux-system
spec:
  interval: 10m
  path: ./k8s
  prune: true
  targetNamespace: default
  sourceRef:
    kind: GitRepository
    name: <app>
```

### Step 4 — Create `image-automation.yaml`

```yaml
# clusters/my-cluster/<app>/image-automation.yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: <app>
  namespace: flux-system
spec:
  image: ghcr.io/dedkola/<app>
  interval: 1m0s
  # secretRef:             # uncomment for private registries
  #   name: ghcr-<app>
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: <app>
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: <app>
  filterTags:
    pattern: "^\\d{14}-[a-f0-9]+$"
  policy:
    alphabetical:
      order: asc
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: <app>
  namespace: flux-system
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: <app>
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: "chore: update <app> image to {{range .Changed.Changes}}{{print .NewValue}}{{end}}"
    push:
      branch: main
  update:
    path: ./k8s
    strategy: Setters
```

### Step 5 — Create `kustomization.yaml`

```yaml
# clusters/my-cluster/<app>/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gitrepository.yaml
  - flux-kustomization.yaml
  - image-automation.yaml
```

### Step 6 — Set up the app repo

In the app repository, ensure:

1. **Kubernetes manifests** live under `k8s/` (matching the `path: ./k8s` in the Kustomization)

2. **The deployment image line** has the Flux image policy marker:

   ```yaml
   image: ghcr.io/dedkola/<app>:latest # {"$imagepolicy": "flux-system:<app>"}
   ```

3. **CI tags images with timestamps** (`.github/workflows/docker-build.yml`):

   ```yaml
   - name: Extract metadata
     id: meta
     uses: docker/metadata-action@v5
     with:
       images: ghcr.io/dedkola/<app>
       tags: |
         type=raw,value={{date 'YYYYMMDDHHmmss'}}-{{sha}}
         type=raw,value=latest

   - name: Build and push
     uses: docker/build-push-action@v5
     with:
       context: .
       push: true
       tags: ${{ steps.meta.outputs.tags }}
   ```

### Step 7 — Commit, push, and verify

```bash
git add clusters/my-cluster/$APP
git commit -m "feat: add $APP to fleet"
git push

# Wait for Flux to pick it up, or force reconciliation
flux reconcile kustomization flux-system

# Verify
flux get sources git -A
flux get kustomizations -A
flux get image repository -A
```

---

## Quick reference

```bash
# Health check
flux check

# All resources
flux get all -A

# Force reconcile everything
flux reconcile source git flux-system && flux reconcile kustomization flux-system

# Suspend/resume an app
flux suspend kustomization <app>
flux resume kustomization <app>

# View Flux logs
flux logs --all-namespaces
```

## Secrets

### Git secrets (for Flux to pull/push repos)

| Secret        | Namespace   | Used by              | Purpose                               |
| ------------- | ----------- | -------------------- | ------------------------------------- |
| `flux-system` | flux-system | flux-system, chat2md | Git access to fleet and chat2MD repos |
| `tk-doc`      | flux-system | tk-doc, tk-doc-dev   | Git access to tk-doc repo (private)   |

### Registry secrets (for scanning and pulling private images)

| Secret        | Namespace   | Used by                | Purpose                                 |
| ------------- | ----------- | ---------------------- | --------------------------------------- |
| `ghcr-secret` | default     | tk-doc pods            | Pull private images from GHCR           |
| `ghcr-secret` | dev         | tk-doc-dev pods        | Pull private images from GHCR           |
| `ghcr-tk-doc` | flux-system | ImageRepository/tk-doc | Scan private GHCR registry for new tags |

Create the registry scanning secret:

```bash
kubectl create secret docker-registry ghcr-tk-doc \
  --namespace=flux-system \
  --docker-server=ghcr.io \
  --docker-username=dedkola \
  --docker-password=<GITHUB_PAT_WITH_READ_PACKAGES>
```

Copy image pull secret to dev namespace:

```bash
kubectl get secret ghcr-secret -n default -o json \
  | jq 'del(.metadata.namespace, .metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp)' \
  | kubectl apply -n dev -f -
```

### Git push access (for ImageUpdateAutomation)

The `flux-system` and `tk-doc` secrets need **write access** to their respective app repos so ImageUpdateAutomation can push commits. If using HTTPS PAT, ensure `repo` scope. If using deploy keys, enable write access.

## Add a new app

1. Create `clusters/my-cluster/<app>/gitrepository.yaml`
2. Create `clusters/my-cluster/<app>/flux-kustomization.yaml`
3. Create `clusters/my-cluster/<app>/image-automation.yaml` (ImageRepository + ImagePolicy + ImageUpdateAutomation)
4. Create `clusters/my-cluster/<app>/kustomization.yaml` (lists the above files)
5. If private repo: create Git secret — `flux create secret git <app> --url=... --username=git --password=<token>`
6. If private registry: create registry secret for image scanning (see Secrets section)
7. Update app CI to push timestamp-based tags (`type=raw,value={{date 'YYYYMMDDHHmmss'}}-{{sha}}`)
8. Add `# {"$imagepolicy": "flux-system:<app>"}` marker to the app's deployment manifest

## Re-bootstrap Flux (with image automation)

If image automation controllers are missing, re-bootstrap:

```bash
flux bootstrap github \
  --owner=dedkola \
  --repository=fleet \
  --branch=main \
  --path=clusters/my-cluster \
  --components-extra=image-reflector-controller,image-automation-controller
```

Or generate the components manifest manually:

```bash
flux install --export \
  --components-extra=image-reflector-controller,image-automation-controller \
  > clusters/my-cluster/flux-system/gotk-components.yaml
```

Futher Reading and Resources about Flux:

- [Flux documentation](https://fluxcd.io/docs/)
- [Flux onboarding guide](https://doc.tkweb.site/docs/Kubernetes/flux-onboarding)
- [Flux to Node.js app guide](https://doc.tkweb.site/docs/Kubernetes/flux-to-node-app)

# bootstrap fix

# bootstrap fix
