# Stage 4 — Multi-cluster and environment promotion

This stage assumes the existing EC2 kind cluster hosts the Argo CD control plane. The remote clusters can run on your other machines. Each target Kubernetes API must be reachable from the Argo CD controller pod over a private, restricted network.

## 4.1 Design the topology first

Example:

```text
Admin workstation
  ├── kubectl/argocd CLI ──> control kind API on EC2
  ├── kubectl context ─────> dev kind API on machine A
  └── kubectl context ─────> prod kind API on machine B

Argo CD application-controller on EC2
  ├── Git server (outbound HTTPS/SSH)
  ├── dev kind API (private address and port)
  └── prod kind API (private address and port)
```

Use private IPs/VPC routing/VPN and restrict firewall/security-group rules to the EC2 host or its security group. Do not expose API server ports publicly. Since each kind cluster is nested inside a host, the host must forward the chosen port to kind's API server, and the endpoint in the kubeconfig must be reachable from the controller pod. A successful `kubectl` from your laptop alone does not prove this.

## 4.2 Expose a kind API only on a private interface

On each remote Linux machine, choose an unused port and a private host address. For example, create a kind config such as:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: lab-dev
apiServerAddress: "10.0.1.25"
apiServerPort: 6445
nodes:
  - role: control-plane
  - role: worker
```

Replace `10.0.1.25` with that machine's private address. Create the cluster:

```bash
kind create cluster --config kind-dev.yaml
kubectl cluster-info --context kind-lab-dev
kubectl get nodes --context kind-lab-dev
```

Configure the host firewall and cloud security group so only the Argo CD host/network can reach TCP 6445. Confirm the generated kubeconfig's server endpoint is the private address and port. For prod, use another cluster name, host, and port. Kind configuration capabilities can vary by version; validate against your installed kind version before creating clusters.

Do not change the existing control cluster's kind config. If a remote host is behind NAT or cannot route from EC2, stop here and establish VPN/private connectivity rather than opening the API to the internet.

## 4.3 Register each remote cluster

From an admin machine with the remote kubeconfig contexts available:

```bash
kubectl config get-contexts
kubectl --context kind-lab-dev get nodes
argocd cluster add kind-lab-dev --name lab-dev
```

Repeat for prod when ready. `argocd cluster add` uses the selected context and creates registration credentials/RBAC in the Argo CD namespace. Review the permissions and generated Secret. Use disposable lab clusters; do not reuse broad administrator kubeconfigs in production.

List registered destinations:

```bash
argocd cluster list
kubectl -n argocd get secrets -l argocd.argoproj.io/secret-type=cluster
```

Expected: the cluster list includes the remote destination and reports it as reachable. If connection fails, test connectivity from the Argo CD controller's network namespace or a temporary diagnostic pod approved for the lab; the workstation's `kubectl` is not sufficient evidence.

Label the registration Secrets. Find the correct secret by inspecting its `name` label and endpoint (do not print the `config` data field):

```bash
kubectl -n argocd get secrets -l argocd.argoproj.io/secret-type=cluster \
  -L name,environment,lab
argocd cluster list
```

Then add labels to the Secret for each registered cluster. Use the server address from `argocd cluster list` to map each registered name to its Secret; do not print or share the Secret's `config` data:

```bash
kubectl -n argocd label secret <dev-cluster-secret> lab=true environment=dev --overwrite
kubectl -n argocd label secret <prod-cluster-secret> lab=true environment=prod --overwrite
```

Replace the secret names with the names you found. Confirm labels:

```bash
kubectl -n argocd get secrets -l argocd.argoproj.io/secret-type=cluster \
  -L name,environment,lab
```

Treat cluster Secrets as sensitive. Never paste their contents into tickets or commit them. Avoid editing the `argocd.argoproj.io/secret-type` label.

## 4.4 Deploy dev to its matching cluster

The matrix manifest uses `{{ .server }}` (the registered cluster endpoint) as its destination and derives the namespace from the `environment` label. Before applying it, extend `bootstrap/project-lab.yaml` with a destination entry for each remote cluster's exact server URL as reported by `argocd cluster list`, allowing only `argocd-lab-dev` for the dev cluster and `argocd-lab-prod` for prod. Apply the updated project to Argo CD and push the Git change before creating the matrix ApplicationSet. Do not use a wildcard destination.

Apply the matrix only after the labels and AppProject destinations are correct:

```bash
kubectl apply -n argocd -f bootstrap/applicationsets/cluster-matrix.yaml
kubectl -n argocd get applications
argocd app list
```

Expected: the dev workload Application targets the registered dev cluster, uses `gitops/envs/dev/hello`, and deploys into `argocd-lab-dev`. Wait for health:

```bash
argocd app wait hello-lab-dev --sync --health --timeout 240
```

Generated names use workload plus cluster name in this lab. If your actual cluster name differs, use the name shown by `kubectl -n argocd get applications`.

Verify against the *remote* cluster context:

```bash
kubectl --context kind-lab-dev get deploy,svc -n argocd-lab-dev
```

## 4.5 Promote by Git, not by copying live state

Promotion is a reviewable source change. The prod overlay references the shared base and sets three replicas. Change the prod overlay in Git, push it, inspect the prod Application diff, then sync/allow automation. Do not use `kubectl get -o yaml` from dev as a promotion mechanism.

For a controlled challenge:

1. Make a harmless change to the base (for example, add a label).
2. Confirm dev receives it.
3. Promote the same change to prod only after reviewing the diff and expected workload count.
4. Confirm dev and prod Applications point to their respective cluster/environment pairs.

If you want promotion by Git branch or tag, create a separate Application/ApplicationSet source revision per environment and document who can change it. A `targetRevision` branch is a deployment control, not merely a convenience.

## 4.6 Tear down remote access

Delete the cluster-targeting ApplicationSet first and inspect what happens to generated Applications and workloads. Then remove individual cluster registrations only when no Applications target them:

```bash
argocd cluster rm lab-dev
```

Check `argocd cluster rm --help` for your CLI version. Deleting the kind cluster is separate and must be done on its host. Do not remove the control cluster or the existing Argo CD namespace.

### Checkpoint

- [ ] Remote API endpoint is reachable privately from the EC2/Argo CD network.
- [ ] Each remote cluster is registered and visible to Argo CD.
- [ ] Cluster Secrets carry only the intended `lab` and `environment` labels.
- [ ] The generated Application targets the expected cluster and environment path.
- [ ] You can explain Git-based promotion and cluster-registration privilege.
