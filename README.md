# Argo CD + Octopus Deploy demo environment — setup guide

This repository stands up a set of Argo CD applications for demonstrating the
Octopus Deploy Argo CD integration. It covers five scenarios of increasing
complexity, all sharing one small nginx workload, so the moving parts stay
visible.

Read [Part 1](#part-1--prerequisites) through [Part 4](#part-4--octopus-setup)
in order the first time. After that, the
[troubleshooting section](#troubleshooting) is the part worth bookmarking — it
lists every problem we actually hit, with the symptom you'll see rather than
the cause you'd have to guess at.

---

## What you get

| Scenario | What it demonstrates | Argo CD apps |
| --- | --- | --- |
| **Single application** | Plain GitOps. Edit a file, commit, sync. | `argocd-demo` |
| **App of apps** | Argo CD managing its own Application definitions. | `root` |
| **ApplicationSet** | Apps generated from Git directories; fan-out changes. | `demo-development`, `demo-test`, `demo-production` |
| **Octopus image tags** | Release promotion by image tag across environments. | (uses the ApplicationSet apps) |
| **Octopus manifests** | Octopus generating whole manifests from templates. | `octopus-development`, `octopus-test`, `octopus-production` |
| **Helm** | Helm charts, per-environment values, Helm-specific Octopus config. | `helm-development`, `helm-test`, `helm-production` |

Ten Argo CD applications in total, from one manual `kubectl apply`.

### Repository layout

```
bootstrap/
  root-app.yaml              applied by hand, once — bootstraps everything else

argocd/                      synced by root; all Application/ApplicationSet CRs
  application.yaml           the single application
  applicationset.yaml        the tenant ApplicationSet
  applicationset-octopus.yaml  the Octopus-manifests ApplicationSet
  applicationset-helm.yaml   the Helm ApplicationSet

single/                      scenario 1 — kustomize, hand-managed
appset/                      scenario 3 — kustomize base + per-env overlays
  base/
  tenants/development|test|production/
octopus-managed/             scenario 4 — written by Octopus, do not hand-edit
  development|test|production/
templates/                   scenario 4 — input templates for Octopus
helm/demo-web/               scenario 5 — Helm chart + per-env values files
```

### Port map

Every service is a NodePort, so pages stay reachable across rollouts. Bookmark
these once.

| Port | Application | Namespace |
| --- | --- | --- |
| 30080 | single | `argocd-demo` |
| 30081 / 30082 / 30083 | ApplicationSet tenants | `demo-development` / `-test` / `-production` |
| 30091 / 30092 / 30093 | Octopus-managed | `octopus-development` / `-test` / `-production` |
| 30101 / 30102 / 30103 | Helm | `helm-development` / `-test` / `-production` |

---

## Part 1 — Prerequisites

### Cluster

Any Kubernetes cluster where NodePorts are reachable from wherever you'll open
a browser. k3s, k3d (with `-p` port mappings), minikube, and real nodes all
work as-is. **kind and Docker Desktop need port mappings declared at cluster
creation time** — if you're using either, sort that out before going further, 
or none of the demo pages will load.

### Argo CD — including the ApplicationSet controller

Install Argo CD however you normally would, then **verify the ApplicationSet
controller is present**. Not every installation route includes it, and its
absence produces a confusing error much later:

```bash
kubectl get crd applicationsets.argoproj.io
kubectl get pods -n argocd | grep applicationset
```

You need both a CRD and a running pod. If either is missing, see
[ApplicationSet CRD missing](#applicationset-crd-missing) before continuing —
three of the five scenarios depend on it.

### Octopus Deploy

An Octopus instance with an Argo CD instance registered against your cluster.
You'll also need permission to create projects, environments, feeds, and Git
credentials.

---

## Part 2 — Repository setup

### 1. Get your own copy

Fork this repository, or create a new one and upload the contents. **Put the
contents at the repository root** — `single/`, `appset/`, `argocd/` and the
rest should be top-level directories, not nested inside a wrapper folder.

### 2. Replace the repository URL

The placeholder `YOUR-ORG/argocd-demo-app` appears in several files, and
**each ApplicationSet contains two occurrences** — one on the generator, one
on the template. Missing the second one is easy and fails in a confusing way:
the ApplicationSet generates apps successfully, then every generated app
errors on repository access.

```bash
sed -i 's|https://github.com/YOUR-ORG/argocd-demo-app.git|https://github.com/YOUR-ORG/YOUR-REPO.git|g' \
  argocd/*.yaml bootstrap/root-app.yaml
grep -rn "YOUR-ORG" . || echo "all clear"
```

Check `targetRevision` matches your default branch while you're in there.

### 3. Private repository? Add credentials to Argo CD now

```bash
argocd repo add https://github.com/YOUR-ORG/YOUR-REPO.git \
  --username YOUR-USER --password YOUR-PAT
```

Skip this and applications sit in `Unknown` state with a repository access
error rather than anything obviously auth-related.

---

## Part 3 — Bootstrap with app of apps

This is the only manual `kubectl apply` in the whole setup. Everything else
follows from it.

```bash
kubectl apply -f bootstrap/root-app.yaml
```

The `root` application syncs the `argocd/` directory, so all four
Application and ApplicationSet definitions get created for you. Adding a new
scenario later is a matter of committing a file to `argocd/` — no further
manual applies.

### Why `bootstrap/` is separate from `argocd/`

If the root application's own manifest lived in the directory it syncs, it
would manage itself, and a bad edit could leave you unpicking a
self-referential loop by hand. Keeping it one directory over makes it a plain
bootstrap artifact: apply once, forget.

### What to expect

```bash
kubectl get applications -n argocd
```

You should see `root` plus nine child applications. **They will all show
`Missing` health.** This is correct — automated sync is deliberately switched
off so you can demonstrate the `OutOfSync` state rather than having everything
silently reconcile. Sync them when you're ready:

```bash
argocd app sync argocd-demo
argocd app sync demo-development demo-test demo-production
argocd app sync helm-development helm-test helm-production
```

The `octopus-*` applications will show empty or placeholder content until
their first Octopus deployment writes manifests into them.

### About automated sync

`root` runs with `selfHeal: true` so that edits to `argocd/` reach the cluster
without intervention — this is what makes "edit a file on GitHub, watch Argo CD
update its own configuration" work.

Two consequences worth knowing:

- **Editing an Application through the Argo CD UI no longer sticks.** Changes
  get reverted within seconds. Annotations in particular must be committed to
  Git, not added through the UI.
- **Pruning is off by default.** Deleting a file from `argocd/` leaves an
  orphaned Application behind. Turn `prune: true` on once you've decided
  whether you want deletions to cascade — and if you do, add
  `finalizers: [resources-finalizer.argocd.argoproj.io]` to the child
  applications, or you'll get workloads running that no Application claims.

---

## Part 4 — Octopus setup

### Environments and lifecycle

Create three environments: **Development**, **Test**, **Production**.

Then create a lifecycle (Library → Lifecycles) with three phases, one
environment each, in that order. Without it you get the default lifecycle,
which works but gives you no promotion gating — and gating is most of the
story you're demonstrating.

### Feeds and credentials

- **Docker Hub feed** (Library → External Feeds → Docker Container Registry,
  `https://index.docker.io`). Anonymous works for public nginx; authenticating
  avoids rate limits.
- **Git credentials** (Library → Git Credentials) with **write** access to the
  repository. Octopus commits to it — read-only access is not enough.

### Scoping annotations

Octopus finds Argo CD applications by two annotations. Values are **slugs, not
display names** — check the slug field on the project and environment pages.

```yaml
argo.octopus.com/project: your-project-slug
argo.octopus.com/environment: development
```

**Where they go depends on how the application was created:**

| Application created by | Annotations go on |
| --- | --- |
| A hand-written Application CR | `metadata.annotations` on the Application |
| An ApplicationSet | `spec.template.metadata.annotations` on the ApplicationSet |

Putting them on an ApplicationSet's own `metadata.annotations` is valid YAML,
applies cleanly, and does nothing — see
[Annotations on the wrong level](#annotations-on-the-wrong-level).

In this repository, the ApplicationSets template sets the environment from the
element name, so all three environments are handled by one block:

```yaml
  template:
    metadata:
      name: 'demo-{{.env}}'
      annotations:
        argo.octopus.com/project: your-project-slug
        argo.octopus.com/environment: '{{.env}}'
```

You only need to set the project slug. Set it once per ApplicationSet — each
scenario should be its own Octopus project (see below).

### One Octopus project per scenario

Use a separate project for the kustomize/image-tag scenario, the manifests
scenario, and the Helm scenario. Two reasons:

1. **The Helm scenario needs configuration that the others must not have.** The
   image-tag step's Helm image value field is required for Helm sources and
   must be empty for Kustomize and directory sources. Same step, incompatible
   settings — they can't share a deployment process.
2. **The step acts on every application matching the project and environment
   slugs.** Share a slug across scenarios, and one deployment updates all of
   them at once. That's a legitimate pattern to demonstrate deliberately, but a
   confusing accident to stumble into.

---

## The scenarios

### 1. Single application

Plain kustomize, manually synced. The page is served from a ConfigMap
generated from `single/files/index.html`.

**Demo loop:** edit the release string and colour at the top of the HTML file,
commit, refresh in the Argo CD UI, sync, reload the browser tab.

Other changes worth showing:

| Edit | What Argo CD shows |
| --- | --- |
| `replicas` in `deployment.yaml` | pod count changes in the resource tree |
| `image` tag in `deployment.yaml` | rolling update, old ReplicaSet scaled to 0 |
| remove `service.yaml` from `kustomization.yaml` | resource marked for pruning |
| `kubectl scale` the deployment by hand | drift — `OutOfSync` with no commit |

**Why the ConfigMap has a hash suffix.** The kustomization uses
`configMapGenerator`, which appends a content hash to the ConfigMap name.
Changing the HTML changes the name, which changes the Deployment's volume
reference, which forces a rolling update. Without it, Argo CD reports `Synced`
while the running pods keep serving the old page — the single most confusing
failure in a GitOps demo. Do not set `disableNameSuffixHash: true` unless you
want to demonstrate exactly that.

### 2. App of apps

Covered in [Part 3](#part-3--bootstrap-with-app-of-apps). As a demo in its own
right: edit an annotation value in `argocd/application.yaml` on GitHub, and
watch Argo CD update its own Application object without anyone touching the
cluster.

### 3. ApplicationSet

A Git directory generator globs `appset/tenants/*`, so **each directory is an
application**. The directories are named for the Octopus environments, which
lets the ApplicationSet template the environment annotation from the directory
name.

Three demos, different in kind:

1. **Change one tenant.** Edit `appset/tenants/test/files/index.html`. Only
   `demo-test` goes `OutOfSync`.
2. **Change the base.** Edit `appset/base/deployment.yaml`. All three go
   `OutOfSync` at once. This fan-out is the argument for ApplicationSets.
3. **Add or remove a tenant.** Copy a tenant directory, commit, and a fourth
   application appears on its own with its own namespace. Delete it, and it goes
   away. Nothing in scenario 1 can do this.

**NodePorts live in the overlays, not the base.** Node ports are cluster-wide,
so three tenants sharing a base with a fixed `nodePort` would collide — one
binds, the others fail to sync. The base sets `type: NodePort`; each overlay
patches in its own number.

### 4. Octopus image tag promotion

Uses the ApplicationSet applications from scenario 3.

**Octopus setup:** a project on your three-phase lifecycle, with a single
*Update Argo CD Application Image Tags* step and an nginx package reference
from your Docker Hub feed.

- **No deployment targets needed.** The step runs on the Octopus Server and
  finds its work by annotation. If it's asking for a target role, something is
  misconfigured.
- **No environment scoping needed on the step.** It resolves the application
  from the environment the deployment is running in. One step covers all three.

**Use numeric image tags.** `1.27-alpine` isn't valid semver, so Octopus can't
order versions reliably. Each tenant's `kustomization.yaml` carries an images
transformer:

```yaml
images:
  - name: nginx
    newTag: 1.27.4
```

**The tag must live inside each tenant's own path.** Each application's source
path is `appset/tenants/<env>`, so a tag defined in `appset/base/` is outside
the path Octopus is given and won't be updated.

**Demo:** create a release with a new nginx version, deploy to Development,
then promote the same release through Test and Production. Watch the three
pages update one environment at a time.

Add a **Manual Intervention** step scoped to Production only, placed before the
image step. It's skipped in Development and Test, so promotion stays quick
while Production stays governed — the clearest single illustration of what
Octopus adds on top of Argo CD.

### 5. Octopus manifest generation

Octopus renders `templates/app.yaml` with Octopus variables substituted and
commits the result into `octopus-managed/<env>/`.

**Octopus setup:** a project with an *Update Argo CD Application Manifests*
step. Template source is this repository, branch `main`, folder `templates`.
Enable **purge** so that removed resources are removed from the target directory.

Project variables, scoped by environment:

| Variable | development | test | production |
| --- | --- | --- | --- |
| `Replicas` | 1 | 2 | 3 |
| `NodePort` | 30091 | 30092 | 30093 |
| `BandColour` | `#4c9a8f` | `#c2734a` | `#7a6bbd` |
| `ImageTag` | 1.27.4 | 1.27.4 | 1.27.4 |

**Sensitive variables fail the deployment.** Keep everything a plain project
variable.

**This ApplicationSet uses a list generator, not a directory generator.**
Octopus creates the contents of the output directories, so a directory
generator would find nothing before the first deployment ran.

**Nothing forces a pod restart here.** There's no kustomize hash on a plain
directory source, so the template stamps `#{Octopus.Release.Number}` into a pod
annotation. New release, new pod spec, guaranteed rollout. If you write your
own templates, keep that trick or your ConfigMap changes won't take effect.

**Placeholder files.** Each output directory needs a file so Argo CD can
resolve the path before Octopus first writes to it. Use `README.md` — Argo CD's
directory source only reads `.yaml`, `.yml`, and `.json`, so markdown holds the
directory open without being parsed as a manifest. Purge removes it on first
deployment, which is fine.

**Release versioning.** This project has no package reference driving it, so
leave it on Octopus's default incrementing number. Consider starting it at
`1.0.0` so it's visually distinguishable from your other projects on screen —
the release number is rendered in 5rem type on the page.

### 6. Helm

A chart at `helm/demo-web/` with a values file per environment. This is the
scenario where the Octopus integration genuinely behaves differently.

**The one setting that matters:** on the *Update Argo CD Application Image
Tags* step, open the nginx package reference and populate the **Helm image
value** field:

```
image.tag
```

This field is required for Helm sources and must be left empty for Kustomize
and directory sources. Without it the deployment succeeds while doing nothing,
logging a warning about a missing image replace path annotation.

**`values.yaml` has no `tag` key on purpose.** The image path is defined per
chart, not per values file, so Octopus updates `image.tag` wherever it finds it
across that chart's values files — including the shared defaults. That would
leave `values.yaml` defaulting to whatever was last deployed to Development,
which any environment that stopped pinning its own tag would silently inherit.
Removing the key from the base closes it; the template falls back to
`.Chart.AppVersion`:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

**`checksum/config` on the pod template** is the Helm equivalent of the
kustomize ConfigMap hash — when the rendered ConfigMap changes, the checksum
changes, the pod spec changes, and the Deployment rolls.

**`releaseName` is set per environment.** Without it Argo CD uses the
Application name as the Helm release name, which surprises people who then
can't find it.

Worth being able to explain in customer conversations: **Argo CD renders charts
with `helm template`, it doesn't install them.** `helm list` shows nothing, and
there is no Helm release history to roll back to. Rollback comes from Git and
from Octopus. This catches out a lot of teams moving from the Helm CLI.

---

## Troubleshooting

### ApplicationSet CRD missing

**Symptom:** `no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"`,
or via the root application, the more cryptic
`one or more synchronization tasks are not valid: ApplicationSet.argoproj.io "" not found`.

**Cause:** The installation route used didn't include the ApplicationSet
controller. Helm doesn't install or update CRDs by default; `namespace-install.yaml`
omits them entirely.

**Fix:** find your Argo CD version, then install matching CRDs.

```bash
kubectl get deploy argocd-repo-server -n argocd \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

kubectl apply --server-side --force-conflicts \
  -k https://github.com/argoproj/argo-cd/manifests/crds?ref=vX.Y.Z

kubectl rollout restart deployment argocd-applicationset-controller -n argocd
```

`--server-side` is not optional — Argo CD's CRDs exceed the annotation size
limit that client-side apply uses.

If the controller *deployment* doesn't exist either, re-apply the full install
manifest for your version. Note, it will reassert ownership of resources like
`argocd-cm`, so diff first if your install is customized.

**After fixing:** the CRD alone is enough for `kubectl apply` to succeed. An
ApplicationSet will sit there generating nothing if the controller pod isn't
running. Confirm `1/1 Running` before concluding a manifest is wrong.

### Annotations on the wrong level

**Symptom:** Octopus doesn't see the applications. The ApplicationSet looks
correctly annotated.

**Cause:** annotations placed on the ApplicationSet's own
`metadata.annotations` rather than `spec.template.metadata.annotations`. Both
are valid, and both apply cleanly. Octopus only reads Application objects.

**Check what actually reached the generated apps:**

```bash
kubectl get application demo-development -n argocd \
  -o jsonpath='{.metadata.annotations}' | jq
```

### Octopus doesn't pick up new annotations immediately

Octopus caches its view of Argo CD applications. A manual verification can fail
and then succeed a minute later with no change on your side. Refresh the Argo CD
Applications view in Octopus before assuming something is misconfigured.

### Helm: deployment succeeds but nothing changes in Git

**Symptom:** task log warns that a Helm source `is missing an annotation for
the image replace path. It will not be updated.`

**Cause:** No Helm image value configured on the package reference. Octopus
falls back to looking for an annotation and finds none.

**Fix:** set the Helm image value field to `image.tag` on the package
reference, then **create a new release** — the setting is snapshotted at
release creation, so existing releases won't pick it up.

**Note the annotation route takes a different format entirely.** The step field
takes a plain YAML path (`image.tag`). The
`argo.octopus.com/image-replace-paths` annotation takes a Helm-template string
building a *fully qualified* image name, because Octopus needs the registry to
avoid updating images from other registries:

```yaml
argo.octopus.com/image-replace-paths: "docker.io/{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Putting `image.tag` in the annotation silently fails to match. Also, if any
package in a step uses annotations, all of them must.

### Page shows the old version after a successful sync

Work down the chain:

```bash
# 1. What image are the pods actually running?
kubectl get pods -n helm-development \
  -o jsonpath='{.items[*].spec.containers[*].image}{"\n"}'

# 2. What did Octopus commit?
git log --oneline -3 && git show --stat HEAD

# 3. Is it just the browser?
curl -s localhost:30101 | grep -o '1\.27\.[0-9]'
```

If pods are on the old image, Git wasn't updated where Argo CD reads. If pods
are on the new image, and curl shows the new version, it's browser cache.

If pods are on the new image but the page is stale, the ConfigMap didn't
re-render — check the rollout trigger (kustomize hash, Helm `checksum/config`,
or the release-number pod annotation, depending on the scenario).

### Applications show `Missing` health

They've been generated but never synced. Automated sync is off by design. Sync
them.

### `kubectl port-forward` keeps dying

It pins to a single pod even when you target a Service, so it drops every time
a rollout replaces that pod — which is every time you demo anything. Use the
NodePorts instead; that's why they're configured.

### Editing an Application in the Argo CD UI doesn't stick

`root` runs `selfHeal: true`. Commit to Git instead. This is correct behavior
and is itself a good drift-correction demo.

### Argo CD reports `Synced` but the page is unchanged

Almost always a ConfigMap without a content hash. See the rollout-trigger notes
in scenarios 1, 5, and 6.

### `kubectl apply -f <raw GitHub URL>` returns 404

- Private repositories return 404 rather than 403 for unauthenticated raw
  requests.
- Case matters in the path.
- The file may simply not be pushed yet.

Test with a file you know exists (`curl -I .../README.md`). Cloning the
repository and applying from a local path avoids the whole question — and
you'll want it cloned locally for editing demos anyway.

### Sync fails with a node port conflict

Node ports are cluster-wide. Two applications can't share one, regardless of
namespace. Check the [port map](#port-map).

---

## Notes on the images used

`nginx:1.27-alpine` runs as root. That's fine for a demo cluster, but if your
namespaces have restrictive Pod Security admission, swap to
`nginxinc/nginx-unprivileged` and change `containerPort` to `8080`.
