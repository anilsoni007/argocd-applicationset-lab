# Stage 2 — Sync policy, drift, pruning, and health

This stage changes one behavior at a time. Commit and push each Git edit before refreshing the Application. Keep the explicitly manual Application available for comparison.

## 2.1 Automated sync

In [bootstrap/applications/hello-dev.yaml](../bootstrap/applications/hello-dev.yaml), add an automated sync policy under `spec.syncPolicy`:

```yaml
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
    syncOptions:
      - CreateNamespace=true
```

Commit and push. Observe status:

```bash
argocd app get hello-dev
```

Change a desired value (for example the dev replica patch), push it, and wait for the configured repo refresh interval or run:

```bash
argocd app get hello-dev --hard-refresh
argocd app wait hello-dev --sync --health --timeout 180
```

Expected: Argo CD syncs the changed desired state without an explicit `argocd app sync`.

## 2.2 Pruning

With automated sync enabled but `prune: false`, remove the Service from the source and push. Expected: Application is `OutOfSync`; the live Service remains. Restore the Service manifest before continuing.

Now set `prune: true`, push, and remove the Service manifest again. Expected: Argo CD deletes the no-longer-desired Service on auto-sync. Restore the file and push to recreate it.

**Important:** prune is destructive. In a real cluster, review the diff and resource ownership first. Do not enable `allowEmpty` in this exercise.

## 2.3 Self-heal

Enable:

```yaml
    automated:
      prune: true
      selfHeal: true
```

Sync the current Git state, then introduce a small live-only drift:

```bash
kubectl scale deploy/hello -n argocd-lab-dev --replicas=4
```

Expected: Argo CD detects `OutOfSync`, then restores the Git replica count. Check:

```bash
kubectl get deploy hello -n argocd-lab-dev
argocd app get hello-dev
```

If it does not revert promptly, verify that self-heal is enabled on the live Application and that your version's reconciliation interval has elapsed. Do not use live edits as a deployment method.

## 2.4 Sync options and automated sync safety

`CreateNamespace=true` is already used by the sample. Inspect sync options in the Application and the corresponding destination namespace. Read the installed-version documentation before using `ServerSideApply`, `Replace`, `PruneLast`, or `FailOnSharedResource`: these options can alter apply semantics or cause resource replacement.

Delete the Application object (not the `argocd` namespace) only after deciding whether its resources should remain:

```bash
kubectl delete -n argocd application hello-dev
```

Without an Application finalizer, deleting the Application object normally leaves managed resources behind. Automated prune concerns resources removed from desired state; it is distinct from deleting the Application object. For a controlled cascade experiment, add `resources-finalizer.argocd.argoproj.io` to Application metadata and delete the Application only in the disposable lab. Observe the effect, then reapply the Application. Do not add finalizers casually to shared workloads.

## 2.5 Health and sync waves

Argo CD derives built-in health for common Kubernetes resources such as Deployments and Services. The workload should become Healthy when the Deployment has available replicas.

Optional exercise: create a `PreSync` Job that validates a prerequisite, and a `PostSync` Job that runs after the workload sync. Add the Job manifest in the same Application source and use hook annotations:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

Use a harmless, short-lived container command and set `backoffLimit: 0`. A hook is an operational action, not a normal long-running workload. Check hook Job status and Application operation history. Remove the exercise Job and push when done.

For ordering regular resources, add `argocd.argoproj.io/sync-wave: "-1"` to a prerequisite and `"0"` to the workload. Argo CD applies lower-numbered waves first and waits for health where applicable. Hooks and waves solve different sequencing cases.

### Checkpoint

- [ ] Automated sync responds to a Git commit.
- [ ] You distinguished `prune` from removing an Application object.
- [ ] You observed self-heal correcting live drift.
- [ ] You can explain when hooks and sync waves are appropriate.
