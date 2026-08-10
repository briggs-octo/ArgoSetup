# Argo CD + Octopus Deploy demo environment

Five GitOps scenarios sharing one small nginx workload, built for demonstrating
the Octopus Deploy Argo CD integration.

**Start with [SETUP.md](SETUP.md).** It covers prerequisites, bootstrap, each
scenario, and a troubleshooting section indexed by the error message you'll
actually see.

## Layout

```
bootstrap/          applied by hand once — bootstraps everything else
argocd/             synced by root; all Application/ApplicationSet CRs
single/             scenario 1 — kustomize, hand-managed
appset/             scenario 3 — kustomize base + per-environment overlays
octopus-managed/    scenario 4 — written by Octopus, do not hand-edit
templates/          scenario 4 — input templates for Octopus
helm/demo-web/      scenario 5 — Helm chart + per-environment values
```

## Quick start

```bash
# 1. Replace the placeholder repo URL (two occurrences per ApplicationSet)
sed -i 's|https://github.com/YOUR-ORG/argocd-demo-app.git|https://github.com/YOUR-ORG/YOUR-REPO.git|g' \
  argocd/*.yaml bootstrap/root-app.yaml

# 2. Confirm the ApplicationSet controller exists — three scenarios need it
kubectl get crd applicationsets.argoproj.io
kubectl get pods -n argocd | grep applicationset

# 3. Bootstrap. This is the only manual apply.
kubectl apply -f bootstrap/root-app.yaml
```

Applications appear with `Missing` health because automated sync is off by
design. Sync them when ready:

```bash
argocd app sync argocd-demo
argocd app sync demo-development demo-test demo-production
argocd app sync helm-development helm-test helm-production
```

## Ports

| Port | Application |
| --- | --- |
| 30080 | single |
| 30081 / 30082 / 30083 | ApplicationSet tenants (dev / test / prod) |
| 30091 / 30092 / 30093 | Octopus-managed (dev / test / prod) |
| 30101 / 30102 / 30103 | Helm (dev / test / prod) |

## Octopus projects

Use a **separate Octopus project per scenario**. The Helm scenario requires a
Helm image value on the step's package reference (`image.tag`) that the
Kustomize scenarios must not have, so they cannot share a deployment process.
Set each ApplicationSet's `argo.octopus.com/project` annotation accordingly.
