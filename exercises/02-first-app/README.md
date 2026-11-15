# Exercise 2: Your First Application

**Goal:** create the same application three different ways, learn to read the
three statuses apart, and see what a change in Git looks like from Argo CD's
side.

Everything below uses `$BRANCH` and `$REPO` from `~/.workshop-env`
(exercise 1, step 1). If `echo $BRANCH` prints nothing, run
`source ~/.workshop-env` first.

## 1. A project, and manifests that follow your branch

### Put your branch into the manifests

The manifests in `examples/argocd/` carry a placeholder, `<your-branch>`, where
`targetRevision` goes. Fill it in once for all of them, commit and push:

```bash
cd ~/workshop
sed -i "s|<your-branch>|$BRANCH|" examples/argocd/*.yaml
grep -n 'repoURL\|targetRevision' examples/argocd/*.yaml

git add examples/argocd/
git commit -m "training: track branch $BRANCH"
git push
```

Every `Application` you apply from now on follows *your* branch, and nothing
anybody else pushes today can move it.

> On a fork? Add a second expression so the manifests point at it too:
> `sed -i -e "s|<your-branch>|$BRANCH|" -e "s|https://gitlab.com/greenitnet/argocd_files.git|$REPO|" examples/argocd/*.yaml`

### The AppProject

`default` allows everything. Real installations do not use it. Apply the
project from the file you have just edited:

```bash
kubectl config use-context kind-argocd
kubectl apply -f examples/argocd/appproject-training.yaml

argocd proj get training
```

Read what it allows: one repository, namespaces matching `training-*`, and
`Namespace` as the only cluster-scoped resource. Everything else is refused.

The course repository is public, so Argo CD clones it without a credential --
there is nothing to register before you create an application. The token you
were given is for *your* `git push`, not for Argo CD.

<details>
  <summary>A private repository -- how would Argo CD get in?</summary>

Register a credential once, before the first application, and everything
after it is unchanged. Without one the application never leaves `Unknown`:
Argo CD cannot read the repository, so it has nothing to compare the cluster
against, and the condition on the application names the repository rather
than the manifests.

```bash
argocd repo add "$REPO" \
  --project training \
  --username "$GITLAB_USER" \
  --password "$GITLAB_TOKEN"

argocd repo list
```

`--project training` scopes the credential to the AppProject, so applications
in other projects cannot borrow it. Leave the flag out and the repository is
available installation-wide.

Since Argo CD 3.0 repository credentials live in Secrets, and it is the label
that makes a Secret a repository. `argocd repo add` writes one:

```bash
kubectl -n argocd get secret -l argocd.argoproj.io/secret-type=repository
```

The declarative form is the one you would actually keep:

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
  url: https://gitlab.com/greenitnet/argocd_files.git
  project: training
  username: <the-username>
  password: <the-token>
```

`kubectl apply -f` it from a file you keep **out of Git** -- a token in a
committed Secret is a token you have published. Getting that Secret into the
cluster without committing the token is what Sealed Secrets, External Secrets
and SOPS are for; that is its own topic, later.

Do not hand Argo CD the token you push with: it can **write**, and Argo CD
only ever reads. Give it a credential that cannot do more than that -- in
GitLab a **deploy token**, *Settings -> Repository -> Deploy tokens -> Add
token*, scope `read_repository`. It gives
you a username (`gitlab+deploy-token-<n>`, unless you set your own) and a
token shown exactly once, and `argocd repo add` takes them the same way.

Over SSH instead, with a deploy key on the project:

```bash
argocd repo add git@gitlab.com:greenitnet/argocd_files.git \
  --project training \
  --ssh-private-key-path ~/.ssh/id_ed25519
```

Then every `--repo` and every `repoURL` has to use the SSH form too. Argo CD
matches credentials to applications by repository URL, and
`https://gitlab.com/...` and `git@gitlab.com:...` are two different
repositories as far as that matching is concerned.

</details>

## 2. From the web UI

In the UI: **+ NEW APP**.

| Field | Value |
|---|---|
| Application Name | `webapp-ui` |
| Project | `training` |
| Repository URL | `https://gitlab.com/greenitnet/argocd_files.git` (`echo $REPO`) |
| Revision | your branch (`echo $BRANCH`) |
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
  --repo "$REPO" \
  --path examples/apps/webapp \
  --revision "$BRANCH" \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace training-webapp-cli \
  --sync-option CreateNamespace=true

argocd app sync webapp-cli
```

`--revision` is the **Revision** field you filled in by hand in the UI. Leave
it out and Argo CD reads it as `HEAD` -- whatever the repository's *default*
branch currently points at, which here is `main`, not yours. Your commits would
never reach the application, and nothing would tell you so: it would sit
`Synced` to somebody else's idea of the repository. Always name the branch.

<details>
  <summary>I want to track a branch instead -- what else does --revision take?</summary>

Four kinds of value, and the difference between them is whether the
application keeps moving:

| `--revision` | Argo CD syncs to | Moves when |
|---|---|---|
| `HEAD` | the tip of the repository's default branch | someone pushes to that branch |
| `user3`, `user3-experiment` | the tip of that branch | you push to that branch |
| `v1.2.0` (tag) | the commit the tag names | you retag |
| `9f2a1c4` (SHA) | that commit | never -- it is pinned |

Branches are the useful one here: you can try a change against a real cluster
without touching the branch your other applications are watching. Prefix it
with your own branch name, so it is obvious in the shared repository whose it
is:

```bash
git switch -c "$BRANCH-experiment"
# edit examples/apps/webapp/deployment.yaml -- replicas: 5
git commit -am "experiment: five replicas"
git push -u origin "$BRANCH-experiment"

argocd app set webapp-cli --revision "$BRANCH-experiment"
argocd app get webapp-cli --refresh
argocd app diff webapp-cli
argocd app sync webapp-cli
```

`argocd app get webapp-cli` now shows `Target: user3-experiment` where it
said `user3` before. Put both sides back when you are done -- the application
*and* your own working copy, because everything from exercise 4 onward commits
to the branch the applications track -- and delete the experiment from the
shared repository:

```bash
argocd app set webapp-cli --revision "$BRANCH"
argocd app sync webapp-cli

git switch "$BRANCH"
git push origin --delete "$BRANCH-experiment"
```

Declaratively this is the `targetRevision` field -- every manifest in
`examples/argocd/` now carries your branch name since step 1, and step 4 is where you start editing
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

The manifest is already in Git -- step 1 filled in your branch and pushed it.
Read it, then apply it:

```bash
cat examples/argocd/application-webapp.yaml
```

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

Edit `examples/apps/webapp/deployment.yaml`, change `replicas` from `2` to
`3`, commit and push to your branch:

```bash
sed -i 's/^  replicas: .*/  replicas: 3/' examples/apps/webapp/deployment.yaml
git commit -am "webapp: three replicas"
git push
```

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
  --repo "$REPO" \
  --path examples/apps/webapp \
  --revision "$BRANCH" \
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
