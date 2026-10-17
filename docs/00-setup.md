# Stage 0 — Prepare the lab

## Goals

Confirm that your existing kind cluster and Argo CD are healthy, make this directory available through Git, and establish the names and access paths used in later stages.

## 0.1 Check local tools and contexts

Run these commands on the machine that administers the existing kind cluster:

```bash
kind get clusters
kubectl config get-contexts
kubectl config current-context
kubectl get nodes -o wide
kubectl -n argocd get pods
argocd version --client
```

Expected shape (names, versions, and pod counts vary):

```text
NAME                         STATUS   ROLES           AGE   VERSION
<kind-control-plane>         Ready    control-plane   ...   v...
<kind-worker>                Ready    <none>          ...   v...
<kind-worker2>               Ready    <none>          ...   v...

NAME                                         READY   STATUS
argocd-application-controller-0              1/1     Running
argocd-repo-server-...                       1/1     Running
argocd-server-...                            1/1     Running
```

If your namespace is not `argocd`, substitute the installed namespace in every command. If the CLI is not installed, use the Argo CD UI for the CLI exercises or install a CLI compatible with your server version.

Check CRDs and server version:

```bash
kubectl get crd applications.argoproj.io applicationsets.argoproj.io appprojects.argoproj.io
argocd version
```

Expected: all three CRDs exist and the Argo CD server reports a version. Client/server version skew can cause CLI option differences; use `argocd <command> --help` when an option differs.

## 0.2 Create and publish the GitOps repository

Create a repository on a Git server reachable from the Argo CD pods, then publish the contents of this lab directory. For a new GitHub remote:

```bash
git init
git add .
git commit -m "Add Argo CD Applications and ApplicationSets lab"
git branch -M main
git remote add origin https://github.com/YOUR_ORG/argocd-applicationset-lab.git
git push -u origin main
```

Replace the example URL and organization. If this directory is already inside a Git repository, use that repository instead; do not initialize a nested repository. Push each manifest change before asking Argo CD to sync it.

Before applying any examples, replace every `YOUR_ORG` placeholder in `bootstrap/project-lab.yaml`, `bootstrap/applications/`, and `bootstrap/applicationsets/` with the same clone URL. Verify none remain:

```bash
grep -R "YOUR_ORG" bootstrap
```

Expected: no output. Commit and push the updated manifests.

In the following examples replace:

```text
https://github.com/YOUR_ORG/argocd-applicationset-lab.git
```

with the exact clone URL. The URL must be usable from the repo-server pod, not just from your workstation.

### Private repository

Use the organization's approved Argo CD repository credentials. One common CLI flow is:

```bash
argocd repo add https://github.com/YOUR_ORG/argocd-applicationset-lab.git \
  --username YOUR_GIT_USERNAME \
  --password
```

Enter a token at the prompt; do not include it in shell history or a committed manifest. CLI flags and SSO/token policies vary by provider and Argo CD version. Prefer a narrowly scoped, read-only deploy key/token. Verify with:

```bash
argocd repo list
```

Expected: the repository is listed and its connection status is successful. If credentials are centrally managed, follow that process rather than adding a second credential.

## 0.3 Confirm Argo CD can fetch the repository

The UI's **Settings → Repositories** page or `argocd repo list` should show the URL with a successful connection. An application sync later is the definitive check because the repo-server must fetch and render the actual revision/path.

Optional network diagnostic (pod names vary):

```bash
kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-repo-server
```

Do not install diagnostic tools into the Argo CD pods as part of this lab.

## 0.4 Set shell variables

The later command examples use Bash:

```bash
export ARGOCD_NS=argocd
export LAB_REPO=https://github.com/YOUR_ORG/argocd-applicationset-lab.git
export LAB_CONTEXT="$(kubectl config current-context)"
```

Replace `LAB_REPO`. Keep the Argo CD context selected for the single-cluster stages. For remote registration, note each remote kubeconfig context separately.

### Checkpoint

- [ ] Existing nodes and Argo CD pods are Ready.
- [ ] Applications, ApplicationSets, and AppProjects CRDs are installed.
- [ ] The lab repository is pushed and Argo CD can authenticate to it.
- [ ] You know the control-cluster context and Argo CD namespace.
