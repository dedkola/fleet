# fleet

GitOps repo for my-cluster using Flux v2.

## Structure

```
clusters/my-cluster/
  flux-system/           # Flux bootstrap (includes image automation controllers)
  chat2md/               # app: chat2md (public repo)
    gitrepository.yaml
    flux-kustomization.yaml
    image-automation.yaml  # ImageRepository + ImagePolicy + ImageUpdateAutomation
    kustomization.yaml
  tk-doc/                # app: tk-doc (private repo)
    gitrepository.yaml       # production (main branch)
    gitrepository-dev.yaml   # development (dev branch)
    flux-kustomization.yaml      # production → default namespace
    flux-kustomization-dev.yaml  # development → dev namespace
    image-automation.yaml    # ImageRepository + ImagePolicy + ImageUpdateAutomation
    kustomization.yaml
```

## Apps

| App        | Repo              | Branch | Namespace | Image                     | Registry |
| ---------- | ----------------- | ------ | --------- | ------------------------- | -------- |
| chat2md    | `dedkola/chat2MD` | main   | default   | `ghcr.io/dedkola/chat2md` | public   |
| tk-doc     | `dedkola/tk-doc`  | main   | default   | `ghcr.io/dedkola/tk-doc`  | private  |
| tk-doc-dev | `dedkola/tk-doc`  | dev    | dev       | `ghcr.io/dedkola/tk-doc`  | private  |

## Check status

```bash
# All Flux resources
flux get all

# Git sources
flux get sources git -A

# Kustomizations
flux get kustomizations -A

# Image automation
flux get image repository -A
flux get image policy -A
flux get image update -A
```

## Image Automation

Flux automatically updates container images when new tags are pushed to GHCR. This requires:

1. **Image automation controllers** — installed via `gotk-components.yaml` (components-extra: `image-reflector-controller`, `image-automation-controller`)
2. **Timestamp-based image tags** — CI must tag images as `YYYYMMDDHHmmss-<short-sha>` (not just `:latest`)
3. **Image policy markers** — Deployment manifests in app repos need a marker comment

### How it works

1. CI builds and pushes image with tag `YYYYMMDDHHmmss-<short-sha>` (e.g. `20260327124823-dc0f3eb`) to GHCR
2. `ImageRepository` scans the registry every 1m for new tags
3. `ImagePolicy` selects the newest tag matching `^\d{14}-[a-f0-9]+$` (alphabetical sort = chronological)
4. `ImageUpdateAutomation` commits the updated image tag to the app repo
5. Flux detects the new commit and rolls out the updated deployment

### CI setup (GitHub Actions)

Each app repo needs a workflow that tags images with timestamps. Add this to `.github/workflows/docker-build.yml`:

```yaml
- name: Extract metadata
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: ghcr.io/dedkola/<app-name>
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

### Image policy marker (app repos)

In each app's `k8s/deployment.yaml`, add the marker comment on the image line:

```yaml
# chat2md
image: ghcr.io/dedkola/chat2md:latest # {"$imagepolicy": "flux-system:chat2md"}

# tk-doc
image: ghcr.io/dedkola/tk-doc:latest # {"$imagepolicy": "flux-system:tk-doc"}
```

Flux will replace the tag portion automatically when a new timestamp tag is detected.

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


