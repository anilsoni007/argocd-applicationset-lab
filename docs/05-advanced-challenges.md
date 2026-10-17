# Stage 5 — Advanced patterns and practical challenges

These are guided extensions, not prerequisites for the core lab. Use disposable clusters and review generated resources before syncing.

## 5.1 AppProject guardrails

Review [bootstrap/project-lab.yaml](../bootstrap/project-lab.yaml). Extend it to allow only the required namespace and repository. Then attempt each of the following and record the rejection:

- An Application referencing a different source repository.
- An Application targeting a namespace not listed by the project.
- A cluster-scoped resource when the project does not allow it.

Success criteria: the project rejects the out-of-scope application/resource, while the lab workload continues to sync. Do not grant wildcard source/destination/resource permissions merely to make an error disappear.

## 5.2 Git generator with environment folders

Extend a directory generator to include another workload or environment. Use explicit include/exclude rules so shared folders such as `base` are not generated as Applications. Verify:

```bash
kubectl -n argocd get applicationset
kubectl -n argocd get applications -o wide
argocd app list
```

Success criteria: each generated Application has a unique name, one intended path, and one intended destination.

## 5.3 Cluster generator selector challenge

Change the cluster generator selector to include only clusters labeled `lab=true`. Add a third disposable cluster with a different `environment` value and verify it is excluded or explain why a matching environment directory does not exist.

Success criteria: selection is driven by explicit labels; unrelated registered clusters are not targeted.

## 5.4 Matrix cardinality challenge

Before applying a matrix, write down the expected number of Applications for:

- 1 cluster × 1 workload
- 2 clusters × 1 workload each
- 3 clusters × 2 matching workloads each

Inspect the actual generated set and compare. Then make a selector error intentionally in a disposable lab and observe the unexpected expansion; immediately restore the selector.

## 5.5 Progressive delivery design (design-only)

Design a promotion policy that includes a human approval gate, immutable image tags/digests, rollback criteria, and separate credentials/permissions for dev and prod. Explain which controls belong in Git review, CI, Argo CD Projects, and Kubernetes RBAC. Do not claim that an ApplicationSet alone provides an approval gate.

## 5.6 Troubleshooting game

Use [docs/troubleshooting.md](troubleshooting.md) to inject one fault at a time: wrong path, inaccessible repo, invalid Kustomize resource, missing destination namespace, incorrect cluster label, and unreachable remote API. Capture the exact status/error and the command that identifies it. Restore the known-good state after each exercise.

## Completion rubric

You have completed the lab when you can, without copying a recipe:

- Explain how Argo CD calculates desired state and detects drift.
- Create and troubleshoot a manual and automated Application.
- Explain auto-sync, prune, self-heal, hooks, waves, and Application deletion behavior.
- Build an ApplicationSet with a Git generator and explain its generated child Applications.
- Select clusters using registration-secret labels and explain matrix expansion.
- Trace a manifest from Git path to rendered resources, target cluster, and namespace.
- Promote a change through Git review without manually patching production.
