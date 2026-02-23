# fleet

GitOps repo for my-cluster using Flux v2.

## Structure

```
clusters/my-cluster/
  flux-system/   # Flux bootstrap
  chat2md/       # app: chat2md
  tk-doc/        # app: tk-doc (private repo)
```

## Check status

```bash
flux get all
```

## Add a new app

1. Create `clusters/my-cluster/<app>/gitrepository.yaml`
2. Create `clusters/my-cluster/<app>/flux-kustomization.yaml`
3. Create `clusters/my-cluster/<app>/kustomization.yaml` (lists the above two)
4. If private repo: `flux create secret git <app> --url=... --username=git --password=<token>`

## Notes

- `tk-doc` uses a dedicated secret (`tk-doc`) for private repo auth
- Flux reconciles every 1m (sources) / 10m (kustomizations)
