# Troubleshooting reference

Start with the Application/ApplicationSet condition message. Avoid deleting resources or widening permissions before understanding the error.

| Symptom | Inspect | Likely cause / exercise |
|---|---|---|
| Application `ComparisonError` / manifest generation error | `argocd app get <app>`, `argocd app manifests <app>`, repo-server logs | Wrong path, invalid YAML, missing Kustomize file, or unsupported renderer |
| `repository not found` / authentication failure | `argocd repo list`; repository settings; repo-server logs | Wrong clone URL, credentials unavailable to Argo CD, or Git server not reachable |
| Application stays `OutOfSync` after push | `argocd app get <app> --hard-refresh`; `argocd app diff <app>` | Wrong branch/path, push did not complete, or render failure |
| Auto-sync does not happen | `kubectl -n argocd get application <app> -o yaml` | No `automated` policy, sync error, or auto-sync retry behavior/version differences |
| Resource remains after removal from Git | Check Application's `automated.prune`; `argocd app resources <app>` | Prune is disabled; deletion of an Application object is not the same as pruning |
| Live edit is not reverted | Check `selfHeal`; inspect sync status and Application events | Self-heal is disabled or controller has not reconciled |
| `InvalidSpecError` on project/destination | `kubectl -n argocd get appproject lab -o yaml`; Application conditions | Source URL, namespace, or destination is not allowed by the AppProject |
| ApplicationSet creates no Applications | `kubectl -n argocd describe applicationset <name>`; inspect labels and generator paths | No directory match, selector mismatch, invalid template, or controller/CRD version issue |
| Matrix creates too many Applications | Inspect cluster-secret labels and each generator input | Selector too broad; cardinality is clusters × matching Git directories |
| Remote cluster `Unknown` / connection failure | `argocd cluster list`; check endpoint route/firewall from EC2 | Workstation can reach API but controller network cannot; wrong endpoint or CA credentials |
| `SharedResourceWarning` | `argocd app resources <app>` and compare all Applications | Multiple Applications claim one object; stop sync and resolve ownership |
| Deployment not Healthy | `kubectl describe deploy,pod -n <namespace>`; events and image pull status | Insufficient capacity, image pull failure, bad readiness, or pending pods |

## Useful diagnostics

```bash
argocd app get <app> --hard-refresh
argocd app diff <app>
argocd app history <app>
argocd app resources <app>
argocd app manifests <app>
kubectl -n argocd describe application <app>
kubectl -n argocd describe applicationset <appset>
kubectl -n argocd get applications,applicationsets,appprojects
kubectl get events -A --sort-by=.lastTimestamp
```

For controller logs, first discover pod names:

```bash
kubectl -n argocd get pods
kubectl -n argocd logs deploy/argocd-applicationset-controller --since=10m
kubectl -n argocd logs statefulset/argocd-application-controller --since=10m
kubectl -n argocd logs deploy/argocd-repo-server --since=10m
```

Workload names and deployment types vary with installation. If a named deployment does not exist, list pods and their owner references, then use the actual workload name. Logs may contain repository paths or cluster metadata; redact sensitive values before sharing.

## Troubleshooting exercises

1. Change an Application path to a nonexistent directory. Observe the condition; restore the correct path.
2. Break one Kustomize resource name. Find the rendering error; fix and push.
3. Remove the `environment` label from a disposable cluster Secret. Observe the matrix generator output; restore the label.
4. Make the remote endpoint unreachable in a controlled test. Compare `kubectl` from your admin host with Argo CD's cluster status, then restore connectivity.
5. Remove the allowed repo URL from the lab AppProject. Observe the rejection; restore the exact URL.

For every exercise, record: expected state, observed state, diagnostic command, root cause, and recovery change.
