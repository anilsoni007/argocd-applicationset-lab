# Stage 3 — ApplicationSets

An ApplicationSet controller renders one or more `Application` objects from generator parameters. Treat the generated Applications as controller-owned: edit the ApplicationSet template or its generator inputs, not the generated Application by hand.

## 3.1 Git directory generator: one Application per dev workload

Set the repo URL in [bootstrap/applicationsets/dev-directory.yaml](../bootstrap/applicationsets/dev-directory.yaml), commit, and push. Apply it:

```bash
kubectl apply -n argocd -f bootstrap/applicationsets/dev-directory.yaml
kubectl -n argocd get applicationset hello-dev-set
kubectl -n argocd get applications
```

Expected: an Application named `hello-dev` is generated for the `gitops/envs/dev/hello` directory. It deploys the same dev overlay as the manual Application from Stage 1; do not leave both active against the same destination at once. Before applying this set, delete the manual `hello-dev` Application object from Stage 1, then verify whether resources remain and are adopted by the generated Application.

The generator watches folders matching `gitops/envs/dev/*`; adding a new workload folder with a Kustomize root creates another Application. Try adding a second small workload and verify the generated name and path.

## 3.2 Avoid overlapping ownership

An ApplicationSet-generated Application and a manually managed Application must not independently manage the same objects. For a clean ownership transition:

1. Pause automation or delete the old Application definition.
2. Inspect whether its resources are left behind.
3. Create the generated Application and inspect its diff.
4. Sync only after verifying it targets the intended resources.

If Argo CD reports shared resources, stop and inspect both Application resource trees before changing ownership.

## 3.3 Git directory generator: dev and prod

The sample ApplicationSets in [bootstrap/applicationsets/dev-directory.yaml](../bootstrap/applicationsets/dev-directory.yaml) and [bootstrap/applicationsets/prod-directory.yaml](../bootstrap/applicationsets/prod-directory.yaml) generate one Application per environment's workload directory. Apply the prod set when ready to create the prod namespace in the current lab cluster:

```bash
kubectl apply -n argocd -f bootstrap/applicationsets/prod-directory.yaml
kubectl -n argocd get applications
```

The repo contains a `hello` folder per environment. Compare the overlays: dev starts at one replica, prod at three. Both reuse the base and deploy to separate `argocd-lab-dev` and `argocd-lab-prod` namespaces, so they can coexist on one cluster.

## 3.4 Cluster generator

The cluster generator reads cluster Secrets registered in Argo CD. It does not discover arbitrary Kubernetes clusters on the network. The remote registration steps are in Stage 4.

Once you have registered a lab cluster and labeled its Argo CD cluster Secret with `environment=dev`, inspect what the generator selects:

```bash
kubectl -n argocd get secrets -l argocd.argoproj.io/secret-type=cluster \
  -L environment,name
```

Apply [bootstrap/applicationsets/cluster-matrix.yaml](../bootstrap/applicationsets/cluster-matrix.yaml) only after reviewing its selector and namespace template. Initially, it selects registered clusters labeled `environment=dev` and creates the dev workload Application on those clusters.

## 3.5 Matrix generator: environment × cluster

The matrix example combines:

1. A cluster generator selecting cluster Secrets labeled `lab=true`.
2. A Git directory generator whose path uses the cluster's `environment` label.

The expected label set is `lab=true,environment=dev` or `lab=true,environment=prod`. Each selected cluster receives the matching environment overlay. The resulting Application name includes both workload and cluster name to keep names distinct.

The matrix generator can multiply resources quickly. Begin with one registered cluster and one workload. Before expanding the selector, estimate the number of generated Applications:

```text
selected clusters × matching workload directories per environment
```

Do not apply the matrix manifest until Stage 4's connectivity check passes.

### Checkpoint

- [ ] You can identify the generator inputs and template output.
- [ ] You know generated Applications are managed by their ApplicationSet.
- [ ] You can explain the matrix cardinality and avoid duplicate ownership.
