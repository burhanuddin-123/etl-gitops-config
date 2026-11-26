# ETL GitOps: Config Repo

The **desired state** of the ETL app, kept in Git. [Argo CD](https://argo-cd.readthedocs.io/) watches this repo and keeps a Kubernetes cluster in sync with it (automated sync, pruning and self-heal are on).

The application and its CI live in [`etl-gitops-app`](https://github.com/burhanuddin-123/etl-gitops-app). On every merge to `main` its pipeline builds an image and commits the new image tag into `app/etl/dev/cronjob.yaml` here, which triggers a sync.

```
etl-gitops-app (CI)  ──commit image tag──►  etl-gitops-config  ◄──watches──  Argo CD ──► cluster
```

## Layout

```
argocd-application.yaml     Argo CD Application: points at app/etl/dev, auto-sync + prune + selfHeal
app/etl/dev/cronjob.yaml    Kubernetes CronJob running the ETL image hourly (no overlap, resource limits, TTL cleanup)
app/etl/dev/pvc.yaml        1Gi PersistentVolumeClaim mounted at /app/output for the job results
```

## Use it

1. Install Argo CD on your cluster.
2. Replace `myregistry.azurecr.io/etl-app:initial` in `cronjob.yaml` with your registry and image (CI rewrites the tag afterwards).
3. Register the app:
   ```bash
   kubectl apply -n argocd -f argocd-application.yaml
   ```

## Design notes

- **Git is the source of truth.** A deploy is a commit, and a rollback is a revert.
- **Resource governance:** CPU and memory requests and limits are set, and `ttlSecondsAfterFinished` cleans up finished runs.
- **Why a CronJob and not a Job:** a Kubernetes Job's pod template is immutable, so changing the image on an existing Job fails to sync. A CronJob creates a fresh Job each run, so a new image tag from CI applies cleanly. `concurrencyPolicy: Forbid` stops two runs from fighting over the single ReadWriteOnce volume.

## License

MIT, see [LICENSE](LICENSE).
