# Exercise 2: Your First Application

**Goal:** create the same application three different ways, learn to read the
three statuses apart, and see what a change in Git looks like from Argo CD's
side.

Throughout this exercise, replace `<your-fork>` with your own GitLab namespace.
The fork is assumed to be public: Argo CD clones it itself, and nothing below
hands it a credential.

<details>
  <summary>My fork is private -- how does Argo CD get in?</summary>

Register the credential once, before step 2, and the rest of the exercise is
unchanged. Without it the application never leaves `Unknown`: Argo CD cannot
read the repository, so it has nothing to compare the cluster against, and the
condition on the application names the repository rather than the manifests.

GitLab's read-only credential for exactly this is a **deploy token** --
*Settings -> Repository -> Deploy tokens -> Add token*, scope
`read_repository`. It gives you a username (`gitlab+deploy-token-<n>`, unless
you set your own) and a token shown exactly once.

```bash
argocd repo add https://gitlab.com/<your-fork>/argocd_files.git \
  --project training \
  --username gitlab+deploy-token-123456 \
  --password <the-token>

argocd repo list
```

`--project training` scopes the credential to the AppProject from step 1, so
applications in other projects cannot borrow it. Leave the flag out and the
repository is available installation-wide.

The declarative form is the one you would actually keep -- since Argo CD 3.0
repository credentials live in Secrets, and it is the label that makes a Secret
a repository:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-files-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://gitlab.com/<your-fork>/argocd_files.git
  project: training
  username: gitlab+deploy-token-123456
  password: <the-token>
```

`kubectl apply -f` it from a file you keep **out of Git** -- a token in a
committed Secret is a token you have published. Getting that Secret into the
cluster without committing the token is what Sealed Secrets, External Secrets
and SOPS are for; that is its own topic, later.

Over SSH instead, with a deploy key on the project:

```bash
argocd repo add git@gitlab.com:<your-fork>/argocd_files.git \
  --project training \
  --ssh-private-key-path ~/.ssh/id_ed25519
```

Then every `--repo` and every `repoURL` below has to use the SSH form too.
Argo CD matches credentials to applications by repository URL, and
`https://gitlab.com/...` and `git@gitlab.com:...` are two different
repositories as far as that matching is concerned.

</details>

## 1. A project to put it in

`default` allows everything. Real installations do not use it. Edit
`examples/argocd/appproject-training.yaml`, replace `<your-fork>`, and apply it:

```bash
kubectl config use-context kind-argocd
kubectl apply -f examples/argocd/appproject-training.yaml

argocd proj get training
```

Read what it allows: one repository, namespaces matching `training-*`, and
`Namespace` as the only cluster-scoped resource. Everything else is refused.

## 2. From the web UI

In the UI: **+ NEW APP**.

| Field | Value |
|---|---|
| Application Name | `webapp-ui` |
| Project | `training` |
| Repository URL | `https://gitlab.com/<your-fork>/argocd_files.git` |
| Revision | `main` |
| Path | `examples/apps/webapp` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `training-webapp-ui` |
| Sync Options | tick **AUTO-CREATE NAMESPACE** |

Create it, but **do not sync yet**.

Look at the application tile. It is `OutOfSync` and `Missing`: Argo CD knows what
should exist and can see that none of it does.

Now sync it, and watch the tree fill in.

## 3. From the CLI

```bash
argocd app create webapp-cli \
  --project training \
  --repo https://gitlab.com/<your-fork>/argocd_files.git \
  --path examples/apps/webapp \
  --revision HEAD \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace training-webapp-cli \
  --sync-option CreateNamespace=true

argocd app sync webapp-cli
```

`--revision` is the **Revision** field you filled in by hand in the UI. Leave
it out and Argo CD reads it as `HEAD`, so the command above spells out the
default rather than changing it: `HEAD` is whatever your fork's default branch
currently points at. The UI form said `main`; on a fresh fork the two are the
same commit, which is exactly why it is worth knowing they are not the same
*setting*.

<details>
  <summary>I want to track a branch instead -- what else does --revision take?</summary>

Four kinds of value, and the difference between them is whether the
application keeps moving:

| `--revision` | Argo CD syncs to | Moves when |
|---|---|---|
| `HEAD` | the tip of the repository's default branch | you push to that branch |
| `main`, `feature/x` | the tip of that branch | you push to that branch |
| `v1.2.0` (tag) | the commit the tag names | you retag |
| `9f2a1c4` (SHA) | that commit | never -- it is pinned |

Branches are the useful one here: you can try a change against a real cluster
without touching the branch everyone else is watching.

```bash
git switch -c experiment
# edit examples/apps/webapp/deployment.yaml -- replicas: 5
git commit -am "experiment: five replicas"
git push -u origin experiment

argocd app set webapp-cli --revision experiment
argocd app get webapp-cli --refresh
argocd app diff webapp-cli
argocd app sync webapp-cli
```

`argocd app get webapp-cli` now shows `Target: experiment` where it said
`HEAD` before. Put both sides back when you are done -- the application *and*
your own working copy, because everything from exercise 4 onward commits to the
branch the applications track:

```bash
argocd app set webapp-cli --revision HEAD
argocd app sync webapp-cli

git switch main
```

Declaratively this is the `targetRevision` field -- every manifest in
`examples/argocd/` pins it to `main`, and step 4 is where you start editing
those instead of passing flags. `argocd app sync --revision <x>` exists too,
but it syncs *once* to `<x>` without changing what the application tracks, so
the next automatic refresh pulls the application back to its target revision.

</details>

Same result, one command. Now look at what Argo CD actually created:

```bash
argocd app get webapp-cli -o yaml | head -40
```

That is an ordinary Kubernetes resource. Which means it belongs in Git.

## 4. From a manifest — the only one that is GitOps

Delete both of the previous applications:

```bash
argocd app delete webapp-ui --yes
argocd app delete webapp-cli --yes
```

Edit `examples/argocd/application-webapp.yaml`, replace `<your-fork>`, commit and
push it to your fork:

```bash
git add examples/argocd/application-webapp.yaml
git commit -m "training: webapp application"
git push
```

Then apply it:

```bash
kubectl apply -f examples/argocd/application-webapp.yaml
argocd app sync webapp
```

The difference matters: rebuilding this cluster from scratch now means applying
one file, not clicking through a form.

## 5. Read the three statuses

```bash
argocd app get webapp
```

Find these three, and say out loud what each one means:

* **Sync Status** — does the live state match the target state?
* the last **Operation** — did the sync itself succeed?
* **Health Status** — is the application working?

They are independent. You will see `Synced` + `Degraded` before the day is over.

## 6. Make a change in Git

Edit `examples/apps/webapp/deployment.yaml` in your fork, change `replicas` from
`2` to `3`, commit and push.

```bash
argocd app get webapp
```

Still `Synced` — Argo CD has not looked yet. Force it:

```bash
argocd app get webapp --refresh
argocd app diff webapp
```

The diff shows exactly one line. Apply it:

```bash
argocd app sync webapp
kubectl -n training-webapp get pods
```

## 7. Test the boundary

Try to deploy the same application into a namespace the project does not allow:

```bash
argocd app create webapp-forbidden \
  --project training \
  --repo https://gitlab.com/<your-fork>/argocd_files.git \
  --path examples/apps/webapp \
  --revision HEAD \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace kube-system
```

Read the error. This is the AppProject doing its job — and it is the reason not
to leave everything in `default`.

## Questions to answer before moving on

**1. Why was the application `OutOfSync` immediately after you created it in the UI?**

<details>
  <summary>answer</summary>

Because sync status is the result of a comparison, not a record of what you
did. Argo CD had rendered the manifests from Git and looked for them in the
cluster: the target state named four objects, the live state had none, so the
answer was `OutOfSync` -- and `Missing` for each resource. Creating an
Application never applies anything. It only writes down what ought to be
true.

</details>

**2. What would happen to the applications you created in steps 2 and 3 if this cluster were deleted and rebuilt?**

<details>
  <summary>answer</summary>

They would be gone, and nothing in Git would bring them back. The UI form and
the `argocd app create` command both wrote an `Application` object straight into
the cluster; that object was the only place the definition existed. The one from
step 4 survives, because its manifest is committed -- you rebuild it with one
`kubectl apply`. That difference is the entire argument of this exercise.

</details>

**3. Where is the state of `webapp` actually stored?**

<details>
  <summary>answer</summary>

In two places that are easy to confuse. The *desired* state is in Git: the
manifests under `examples/apps/webapp`, plus the `Application` manifest itself.
The *Argo CD* state is a Kubernetes object -- an `applications.argoproj.io`
resource in the `argocd` namespace -- and its `status` is where sync status,
health and deployment history live. There is no database behind Argo CD; delete
that object and you delete everything Argo CD knows, which is why it belongs in
Git too.

</details>

---

[home](../../README.md) | [previous](../01-install/README.md) | [next](../03-sources/README.md)
