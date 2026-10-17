# Stage 1 — Your first Argo CD Application

An `Application` connects a source (repository, revision, and path) to a destination (cluster and namespace), then compares rendered desired state with live state. Start with manual sync so the reconciliation loop is visible and under your control.

## 1.1 Create the lab AppProject

Set the repository URL in [bootstrap/project-lab.yaml](../bootstrap/project-lab.yaml), commit, and push it. Then apply the project directly to the Argo CD control cluster:

```bash
kubectl apply -n argocd -f bootstrap/project-lab.yaml
kubectl -n argocd get appproject lab
```

Expected:

```text
NAME   AGE
lab    ...
```

The project permits only this source repository and the `argocd-lab-dev` and `argocd-lab-prod` namespaces on the in-cluster destination. If you change the repo URL in an Application, change the AppProject allow-list too.

## 1.2 Create a manually synced Application

Set the source repo URL in [bootstrap/applications/hello-dev.yaml](../bootstrap/applications/hello-dev.yaml), commit, and push. Apply it:

```bash
kubectl apply -n argocd -f bootstrap/applications/hello-dev.yaml
argocd app get hello-dev
```

Expected before the first sync: the Application exists and reports `OutOfSync`; the workload has not yet been created. Depending on Argo CD version, the health display can initially be `Missing`.

Inspect the desired/live difference:

```bash
argocd app diff hello-dev
```

Sync it:

```bash
argocd app sync hello-dev
argocd app wait hello-dev --sync --health --timeout 180
kubectl get deploy,svc -n argocd-lab-dev
```

Expected:

```text
NAME                         READY   UP-TO-DATE   AVAILABLE
deployment.apps/hello        1/1     1            1

NAME            TYPE        CLUSTER-IP      PORT(S)
service/hello   ClusterIP   <cluster-ip>    80/TCP
```

The deployment image is a small nginx container. Test it without installing an Ingress controller:

```bash
kubectl port-forward -n argocd-lab-dev svc/hello 8080:80
```

In another terminal:

```bash
curl -I http://127.0.0.1:8080/
```

Expected: `HTTP/1.1 200 OK`. Stop the port-forward with Ctrl+C.

## 1.3 Read the Application fields

Open [bootstrap/applications/hello-dev.yaml](../bootstrap/applications/hello-dev.yaml) and identify:

- `spec.project`: authorization boundary for source and destination.
- `spec.source.repoURL`, `targetRevision`, `path`: where and which revision to render.
- `spec.destination.server`, `namespace`: where to apply rendered objects.
- `spec.syncPolicy`: absent automation means sync is manual.
- `CreateNamespace=true`: create the destination namespace if needed.

Useful inspection commands:

```bash
kubectl -n argocd get application hello-dev -o yaml
argocd app resources hello-dev
argocd app history hello-dev
argocd app manifests hello-dev
```

`argocd app manifests` shows rendered desired manifests, not necessarily the exact response from the API server.

## 1.4 Git change versus live change

Change the dev replica count in `gitops/envs/dev/hello/replica-patch.yaml` from `1` to `2`, commit, and push. Then refresh and inspect:

```bash
argocd app get hello-dev --hard-refresh
argocd app diff hello-dev
```

Expected: the diff proposes two replicas while the live Deployment is still at one. Sync and verify:

```bash
argocd app sync hello-dev
kubectl get deploy hello -n argocd-lab-dev
```

Restore the replica count to `1` and push before continuing. This is a Git-driven change; editing the cluster directly is not the desired workflow.

### Checkpoint

- [ ] Application is `Synced` and `Healthy`.
- [ ] You can explain source, destination, project, and sync mode.
- [ ] You observed an `OutOfSync` diff and reconciled it.
- [ ] Service responds through port-forward.
