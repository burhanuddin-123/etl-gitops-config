# ETL GitOps: Config Repo

The **desired state** of the ETL app, kept in Git. [Argo CD](https://argo-cd.readthedocs.io/) watches this repo and keeps a Kubernetes cluster in sync with it (automated sync, pruning and self-heal are on).

The application and its CI live in [`etl-gitops-app`](https://github.com/burhanuddin-123/etl-gitops-app). On every merge to `main` its pipeline builds an image and commits the new image tag into `app/etl/dev/job.yaml` here, which triggers a sync.

```
etl-gitops-app (CI)  ──commit image tag──►  etl-gitops-config  ◄──watches──  Argo CD ──► cluster
```

## Layout

```
argocd-application.yaml     Argo CD Application: points at app/etl/dev, auto-sync + prune + selfHeal
app/etl/dev/job.yaml        Kubernetes Job running the ETL image (resource requests/limits, TTL cleanup)
app/etl/dev/pvc.yaml        1Gi PersistentVolumeClaim mounted at /app/output for the job's results
```

## Use it

1. Install Argo CD on your cluster.
2. Replace `myregistry.azurecr.io/etl-app:initial` in `job.yaml` with your registry and image (CI rewrites the tag afterwards).
3. Register the app:
   ```bash
   kubectl apply -n argocd -f argocd-application.yaml
   ```

## Design notes

- **Git is the source of truth.** A deploy is a commit, and a rollback is a revert.
- **Resource governance:** the Job sets CPU and memory requests and limits, and `ttlSecondsAfterFinished` cleans up finished runs.
- **Known limitation:** a Kubernetes Job's pod template is immutable, so when CI changes the image on an existing Job, Argo CD needs the `Replace=true` sync option (or the Job should become a CronJob, or use `generateName`). I would make that change before using this for anything beyond a demo.

## License

MIT, see [LICENSE](LICENSE).
