# Argo CD Applications and ApplicationSets Lab

A hands-on, progressive lab for learning Argo CD from the first `Application` through multi-environment and multi-cluster `ApplicationSet` patterns. The lab is designed for an existing kind cluster with Argo CD installed; it does not reinstall or upgrade Argo CD.

> **Lab topology note:** Three kind nodes inside one t2.medium EC2 instance are useful for learning scheduling and Kubernetes behavior, but they are not three independent machines. They share the EC2 host's CPU, memory, disk, and failure domain. Use your other machines for the multi-cluster labs, and do not expose kind API ports to the public internet.

## What you will build

- A GitOps repository with a Kustomize base and dev/prod overlays.
- Manual and automated Argo CD Applications; sync, prune, self-heal, health, and drift exercises.
- ApplicationProjects, directory and cluster generators, and a cluster/environment matrix.
- A promotion workflow in which a reviewed Git change, rather than a `kubectl` command, promotes configuration.
- Troubleshooting drills and challenges with success criteria.

## Lab map

| Stage | Workbook | Result |
|---|---|---|
| 0 | [docs/00-setup.md](docs/00-setup.md) | Verify the existing cluster, Argo CD, CLI, and Git repository |
| 1 | [docs/01-applications.md](docs/01-applications.md) | Deploy and inspect a manually synced Application |
| 2 | [docs/02-sync-and-health.md](docs/02-sync-and-health.md) | Practice automated sync, prune, self-heal, health, and sync waves |
| 3 | [docs/03-applicationsets.md](docs/03-applicationsets.md) | Generate Applications from Git directories and cluster metadata |
| 4 | [docs/04-multicluster-promotion.md](docs/04-multicluster-promotion.md) | Register remote kind clusters and promote dev to prod |
| 5 | [docs/05-advanced-challenges.md](docs/05-advanced-challenges.md) | Apply Projects, generators, and production-minded design checks |
| Reference | [docs/troubleshooting.md](docs/troubleshooting.md) | Diagnose common Git, sync, generator, and network failures |

## Before you begin

1. Complete Stage 0 and confirm you have permission to create namespaces and register clusters.
2. Push this repository to a Git server reachable from the Argo CD application-controller. GitHub, GitLab, and an internally hosted Git server are all suitable. A local folder on your laptop is not a Git URL the controller can fetch.
3. Replace `https://github.com/YOUR_ORG/argocd-applicationset-lab.git` in the example Argo CD resources with your repository's clone URL. Do not commit credentials or tokens.
4. For a private repository, add credentials to Argo CD using the approved repository credential mechanism for your installation. See Stage 0. Do not put credentials in an `Application`, `ApplicationSet`, or this repository.

The examples target current Argo CD APIs (`argoproj.io/v1alpha1`) and use ApplicationSet Go templating where noted. Check your installed version and CRDs before using newer options. The resources do not pin an Argo CD version or require changing the existing installation.

## Recommended sequence

Run the stages in order. Stages 1–3 work with the existing cluster. Stage 4 is optional until you have at least one remote kind cluster reachable from the EC2 host. Keep the cluster-specific ApplicationSet disabled until you have registered and labeled clusters.

At the end of each stage, remove only the resources created by that stage if you want a clean rerun. Avoid deleting the `argocd` namespace or the existing Argo CD installation.

## Repository layout

```text
gitops/apps/hello/base/                 Shared workload
gitops/envs/dev/hello/                   Dev overlay
gitops/envs/prod/hello/                  Prod overlay
bootstrap/project-lab.yaml               Restricted lab AppProject
bootstrap/applications/                  Explicit Application examples
bootstrap/applicationsets/               Git and matrix ApplicationSet examples
docs/                                    Guided exercises and troubleshooting
```

## Safety and cost

- Use a lab-only namespace and AppProject. These manifests are educational examples, not production hardening.
- Do not expose Argo CD, the Kubernetes API, or kind's API port to `0.0.0.0/0`. Restrict access to a private network, VPN, or the specific source host/security group.
- `argocd cluster add` grants Argo CD access to the target cluster. Use disposable lab clusters and review the created service account/RBAC.
- A t2.medium is resource-constrained. Deploy one workload at a time, watch node pressure, and use the remote machines for separate clusters rather than running many kind clusters on the EC2 host.
- Never commit kubeconfigs, bearer tokens, private keys, or Secret values.
