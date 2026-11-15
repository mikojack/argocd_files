# Exercise 3: Helm and Kustomize Sources

**Goal:** deploy the same workload from a Helm chart and from a Kustomize
overlay, and see where the settings in the `Application` win over the settings
in the repository.

The `Application` manifests below already track your branch -- exercise 2,
step 1 filled it in.

## 1. Helm

Look at the chart first — it is deliberately tiny:

```bash
tree examples/apps/helm-webapp
cat examples/apps/helm-webapp/values.yaml
cat examples/apps/helm-webapp/values-dev.yaml
```

A `Chart.yaml`, three templates and two value files -- nothing else. Argo CD
needs no more than that to treat a directory as a chart.

`values.yaml` says one replica and tag `1.28-alpine`. `values-dev.yaml`
overrides the replica count and the message.

Now read the Application before you apply it:

```bash
cat examples/argocd/application-helm-webapp.yaml
```

It selects `values-dev.yaml` **and** sets `image.tag` as a parameter. Both
change the same chart. Predict which tag ends up running.

You can check your prediction without touching the cluster. Rendering the chart
is precisely what Argo CD's repo-server does with it -- so do it yourself
first, on the chart's own terms:

```bash
helm template webapp -f examples/apps/helm-webapp/values.yaml examples/apps/helm-webapp/.
```

One replica, `nginx:1.28-alpine`. Now render it the way the `Application` asks
for it: the release name comes from `releaseName`, the values file and the
parameter from the same `helm` block you just read.

```bash
helm template helm-webapp \
  -f examples/apps/helm-webapp/values-dev.yaml \
  --set image.tag=1.29-alpine \
  examples/apps/helm-webapp/. | grep -E 'replicas:|image:'
```

There is your answer, before a single object exists in the cluster.

<details>
  <summary>One thing your local render does not know: the cluster</summary>

`helm template` is offline by design. It never contacts a cluster, so it has to
invent the two things a chart can ask about the cluster it is heading for:

* `.Capabilities.KubeVersion` is the Kubernetes version **compiled into your
  helm binary**, not the version of your cluster. Helm 3.18.6 answers
  `v1.33.0` here whatever you are pointing at.
* `.Capabilities.APIVersions` is a minimal default list. Ask it whether the
  cluster has `networking.k8s.io/v1/Ingress` and a bare `helm template` says
  **false** -- for a cluster that certainly has it.

So a chart that guards an object with
`{{ if .Capabilities.APIVersions.Has "..." }}` renders *fewer* objects on your
laptop than it will in the cluster. You can supply both by hand:

```bash
helm template webapp examples/apps/helm-webapp/. \
  --kube-version v1.34.0 \
  --api-versions networking.k8s.io/v1/Ingress
```

**Argo CD does this for you.** When its repo-server renders a chart it passes
the kube version and the API versions of the *destination cluster*, which it
already knows from its cluster cache. The CLI says so plainly -- `argocd app
set --helm-kube-version` and `--helm-api-versions` both document themselves as
"If not set, use the ... from the destination cluster". Those flags exist to
*override* that, typically to make a render reproducible in CI where no cluster
is reachable.

Two consequences worth carrying out of the room:

1. When a local render and Argo CD disagree about which objects exist, this is
   the first thing to check -- it is not necessarily a bug.
2. `argocd app manifests <app>` is the authoritative answer to "what will
   actually be applied", because it is rendered the way Argo CD renders it.

The chart in this exercise asks about neither, so both renders agree here. The
day it does not, you will know why.

</details>

Now let Argo CD reach the same one:

```bash
kubectl apply -f examples/argocd/application-helm-webapp.yaml
argocd app sync helm-webapp

kubectl -n training-helm get deploy helm-webapp \
  -o jsonpath='{.spec.replicas} {.spec.template.spec.containers[0].image}{"\n"}'
```

You should see `2 nginx:1.29-alpine` -- the same two values you rendered
locally a moment ago. The replica count came from the values file, the tag from
`parameters`. **Parameters win over value files.**

## 2. What Argo CD did not do

```bash
helm list -n training-helm
```

Nothing. There is no Helm release, no release history, and `helm rollback` is
not available for this application. Argo CD ran `helm template` and took over
from there — the lifecycle belongs to Argo CD.

Confirm what it rendered:

```bash
argocd app manifests helm-webapp | head -30
```

## 3. Kustomize

```bash
tree examples/apps/kustomize-webapp
```

Read the shape before the contents. The four real manifests live in `base/`
and each overlay is a single `kustomization.yaml` -- no copies, no duplicated
Deployment. That is the whole idea, and it is visible in the directory listing
before you open a single file.

```bash
cat examples/apps/kustomize-webapp/overlays/dev/kustomization.yaml
cat examples/apps/kustomize-webapp/overlays/prod/kustomization.yaml
```

Two overlays over one base: different name prefixes, different replica counts,
and `prod` pins a different image tag.

```bash
kubectl apply -f examples/argocd/application-kustomize-webapp.yaml
argocd app sync kustomize-webapp

kubectl -n training-kustomize get deploy
```

Everything is prefixed `dev-`, and there is one replica.

## 4. Override the image the way a pipeline would

The Application also carries a `kustomize.images` entry. Check what actually
runs:

```bash
kubectl -n training-kustomize get deploy dev-webapp \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

`nginx:1.29-alpine`. Now render the same overlay yourself and compare:

```bash
kubectl kustomize examples/apps/kustomize-webapp/overlays/dev | grep 'image:'
```

`nginx:1.28-alpine` — the base pins that tag and the `dev` overlay does not
touch the image at all, so **nothing in Git under that path asks for 1.29**.
The cluster runs 1.29 because the `Application` said so: `kustomize.images` is
applied on top of whatever the overlay renders, and it wins. This is the field a
CI pipeline usually writes to, except that a pipeline writes it into Git rather
than into the `Application` -- where you can see it in a diff.

Worth noticing before the next step: the `prod` overlay *does* pin
`newTag: 1.29-alpine`, the same value. Switch the path to prod and the override
stops being visible, because both sides now ask for the same thing. An override
you cannot see is still an override.

## 5. Point it at prod instead

```bash
argocd app set kustomize-webapp \
  --path examples/apps/kustomize-webapp/overlays/prod
argocd app diff kustomize-webapp
```

Read the diff before syncing: every object gets a new name, because the prefix
changed. Syncing this creates a second set of objects and orphans the first —
renaming is not the same as updating.

Sync it, then look at what is in the namespace:

```bash
argocd app sync kustomize-webapp
kubectl -n training-kustomize get deploy
```

Both `dev-webapp` and `prod-webapp` exist. The `dev-` one is no longer in the
target state and nothing removed it — we will fix that with pruning in
exercise 6.

## Questions

**1. Which of these three settings wins: chart `values.yaml`, the `valueFiles` entry, or `parameters`?**

<details>
  <summary>answer</summary>

`parameters`. They are applied last, in that order: the chart's own
`values.yaml` first, then each file listed in `valueFiles`, then
`helm.parameters` from the `Application`. Step 1 showed both halves of it at
once -- the replica count came from `values-dev.yaml` and the tag from the
parameter, giving `2 nginx:1.29-alpine`. The practical consequence: a parameter
set in the `Application` cannot be overridden by editing the chart, which is
what makes it the right place for a CI pipeline to write an image tag.

</details>

**2. Why can you not use `helm rollback` on `helm-webapp`?**

<details>
  <summary>answer</summary>

Because there is no Helm release to roll back. Argo CD runs `helm template`
and applies the output itself, so Helm is never asked to install anything: no
release Secret is written, no release history exists, and `helm list -n
training-helm` comes back empty. The lifecycle belongs to Argo CD, and the
equivalents are `argocd app history` and `argocd app rollback` -- which is
exercise 5.

</details>

**3. What happened to `dev-webapp` when you switched the path, and why?**

<details>
  <summary>answer</summary>

Nothing happened to it -- that is the answer, and the problem. The prod
overlay sets a different name prefix, so the rendered objects have different
names, and different names mean different resources as far as the comparison is
concerned. The `dev-` objects simply fell out of the target state. Argo CD does
not delete what it no longer manages unless you ask it to, so they stayed behind
as orphans, still running, still serving. Exercise 6 turns pruning on and they
disappear.

</details>

---

[home](../../README.md) | [previous](../02-first-app/README.md) | [next](../04-second-cluster/README.md)
