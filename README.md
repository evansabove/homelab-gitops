# homelab-gitops

GitOps repository for my homelab k3s cluster, managed by [ArgoCD](https://argo-cd.readthedocs.io/).

## Layout

```
ansible/          One-time bootstrap: installs ArgoCD via Helm and registers the root app.
bootstrap/        ArgoCD's own configuration (Helm values) and the root "app-of-apps".
apps/             ArgoCD Application manifests — one file per workload. The root app watches this dir.
infrastructure/   Cluster-level services (ingress, cert-manager, storage, monitoring) referenced by apps/.
workloads/        Actual manifests/kustomizations for user-facing apps, referenced by apps/.
```

## Bootstrap (run once)

Requires: `ansible`, `helm`, `kubectl` with a kubeconfig pointing at the cluster.

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/bootstrap.yml
```

> **Windows note:** Ansible can't run as a control node on native Windows —
> run the above from WSL with `helm` and the `kubernetes` pip package
> installed there, e.g.:
>
> ```bash
> wsl -e bash -c 'export PATH=$HOME/.local/bin:$PATH KUBECONFIG=/mnt/c/Users/AndyEvans/.kube/config; cd /mnt/c/Projects/homelab-gitops/ansible; ansible-playbook playbooks/bootstrap.yml'
> ```

This installs ArgoCD into the `argocd` namespace and applies
[bootstrap/root-app.yaml](bootstrap/root-app.yaml). From then on, ArgoCD
continuously syncs everything under `apps/` — day-to-day changes are made
by committing to this repo, not by running Ansible again.

## Getting the ArgoCD admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Then port-forward the UI:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

## Adding a new app

1. Put its manifests under `workloads/<name>/` (or `infrastructure/<name>/` for cluster services).
2. Add an ArgoCD `Application` manifest at `apps/<name>.yaml` pointing at that path.
3. Commit and push — ArgoCD picks it up automatically.
