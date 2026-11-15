# Exercise 1: Installing Argo CD

**Goal:** get a working Argo CD on a local cluster, recognise every component
that the standard installation brings up, and log in from both the CLI and the
web UI.

## 1. Get the exercise repository

Everything below is run from inside a clone of this repository. The `kind`
configs, the manifests and the applications are all referenced by relative path
-- `kind create cluster --config examples/kind/...` only works when that is
your working directory. From exercise 2 onward you also *push* to it, and
Argo CD reads back what you pushed.

The whole group works in **one** repository, each participant on **a branch of
their own**. Your trainer gives you three things:

| | |
|---|---|
| a branch name | e.g. `user3` -- yours for the day, nobody else pushes to it |
| a GitLab username | the user the token belongs to |
| a token | read and write access to this repository, and nothing else |

The repository is public, so anyone -- Argo CD included -- can read it without
a credential. The token is what lets **you push** to it.

### Put the three values where every later step can find them

```bash
cat > ~/.workshop-env <<'EOF'
export BRANCH=user3                  # your branch
export GITLAB_USER=<the-username>    # from the trainer
export GITLAB_TOKEN=<the-token>      # from the trainer
export REPO=https://gitlab.com/greenitnet/argocd_files.git
EOF
chmod 600 ~/.workshop-env
echo 'source ~/.workshop-env' >> ~/.bashrc
source ~/.workshop-env
```

Every command from here to the end of exercise 6 uses `$BRANCH` and `$REPO`
rather than a value you have to type, which is why they go into a file and not
only into this shell. Any terminal you open later reads it from `~/.bashrc`.

> A token in a file in your home directory is a lab shortcut. It is acceptable
> on a VM that is thrown away tonight and on a token that expires with the
> course; it is not how you would keep a credential at work.

### Teach git who you are and how to log in

```bash
git config --global user.name  "$GITLAB_USER"
git config --global user.email "$GITLAB_USER@example.com"

git config --global credential.helper store
printf 'protocol=https\nhost=gitlab.com\nusername=%s\npassword=%s\n' \
  "$GITLAB_USER" "$GITLAB_TOKEN" | git credential approve
```

The first two lines are what `git commit` asks for -- without them the first
commit in exercise 2 stops with `Author identity unknown`. The last two hand
the token to git's credential store (`~/.git-credentials`), so no `clone`,
`pull` or `push` today asks you for a password.

### Check out your branch

**If the repository is already on your lab VM**, use that clone -- point it at
the course repository and switch to your branch:

```bash
cd ~/workshop
git remote set-url origin "$REPO"
git fetch origin
git switch "$BRANCH"
```

**If it is not there**, clone it straight onto your branch:

```bash
git clone --branch "$BRANCH" "$REPO" ~/workshop
cd ~/workshop
```

`git switch` with a name that exists only on the remote creates a local branch
tracking `origin/$BRANCH`, so a bare `git push` later goes to the right place.

<details>
  <summary><code>fatal: invalid reference</code> / <code>Remote branch ... not found</code> -- my branch does not exist</summary>

The trainer creates the branches before the course, so ask first -- a typo in
`~/.workshop-env` is the usual cause. If you are told to create it yourself,
start it from `main`:

```bash
git switch -c "$BRANCH" origin/main
git push -u origin "$BRANCH"
```

</details>

Then confirm you are standing in the right directory, on the right branch:

```bash
ls examples/kind/                  # argocd-cluster.yaml, workload-cluster.yaml, ...
git remote -v                      # origin is https://gitlab.com/greenitnet/argocd_files.git
git status -sb                     # ## user3...origin/user3
```

> **Stay on your branch, and never push to `main`.** Every `Application` you
> create today tracks `targetRevision: $BRANCH`, so a commit pushed anywhere
> else is a commit Argo CD will not look at. `main` is the course material for
> the next group -- and the token can write to every branch, so the only thing
> keeping the group out of each other's branches is that you do not
> push to them.

<details>
  <summary>Optional: work on a fork of your own instead</summary>

Everything in these exercises also works on a fork -- useful if you repeat the
course on your own afterwards, or want to keep your work after the token
expires. Fork <https://gitlab.com/greenitnet/argocd_files> in GitLab, then set `REPO` to your fork and `BRANCH` to the
branch you want Argo CD to follow -- usually `main`, since the fork is yours:

```bash
cat > ~/.workshop-env <<'EOF'
export BRANCH=main
export GITLAB_USER=<your-gitlab-user>
export GITLAB_TOKEN=<a-personal-access-token>   # scopes read_repository, write_repository
export REPO=https://gitlab.com/<your-namespace>/argocd_files.git
EOF
source ~/.workshop-env

git clone --branch "$BRANCH" "$REPO" ~/workshop
```

The git identity and credential store steps above are the same. Two
differences to carry through the rest of the day:

* In exercise 2, step 1, where the manifests get your branch name, they also
  need your fork's URL instead of the course repository's -- the step shows the
  extra `sed` expression.
* Keep the fork **public**. Argo CD is given no repository credential in
  these exercises; a private fork needs one, and exercise 2, step 1 has a
  folded section on how to register it.

</details>

## 2. Create the lab clusters

```bash
kind create cluster --name argocd --config examples/kind/argocd-cluster.yaml
kind create cluster --name workload --config examples/kind/workload-cluster.yaml

kubectl config get-contexts -o name
```

`kind` leaves you on the context of the cluster it created last, so switch back
to the one that will host Argo CD:

```bash
kubectl config use-context kind-argocd
```

## 3. Install Argo CD

The installation is a single manifest. We pin it to **v3.5.2** rather than
using `stable`, so that everyone in the room runs the same version:

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.2/manifests/install.yaml
```

> `--server-side` is what the official getting-started guide uses. The manifest
> contains CRDs large enough to blow past the annotation size limit of a
> client-side `kubectl apply`.

### The way you would do it at work

The manifest above is the quickest way into a lab, and it is what the rest of
this exercise assumes. It is not what you should run on a cluster you have to
maintain: it is an imperative `kubectl apply`, so the installation exists only
in the cluster and nowhere in Git.

The chart takes values, and values can be committed:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --version 10.9.0 \
  --values argocd-values.yaml
```

`argocd-values.yaml` is **your** file, not something you will find in this
repository and not a copy of the chart's defaults. It holds the handful of
settings that make this instance yours -- the ingress hostname, RBAC,
an SSO connector, resource requests -- and everything you leave out keeps the
chart's default. That is exactly why it is worth committing: it is a short,
readable description of how your Argo CD differs from a stock one. Start by
reading what you are overriding:

```bash
helm show values argo/argo-cd --version 10.9.0
```

Chart **10.9.0** ships Argo CD **v3.5.2** -- the same version this course is
pinned to. Note that the two version numbers move independently: several earlier
chart releases also ship v3.5.2, and the next chart release ships a newer
Argo CD. `--version` pins the chart; `appVersion` tells you what is inside it.

Both paths produce the same workload names, the same ConfigMaps and the same
`argocd-initial-admin-secret`, so everything below works either way -- but pick
**one**. Running the chart into a namespace that already holds the manifest
installation leaves you with two owners of the same objects and a day of
confusing diffs.

Once the values file is in Git, the next step is to let Argo CD manage its own
installation from it -- we come back to that in the Getting Started module.

Wait until everything is up:

```bash
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd get pods
```

## 4. Find the components you have just met on the slides

```bash
kubectl -n argocd get deploy,statefulset
```

You should be able to name what each one does:

| Workload | Role |
|---|---|
| `argocd-server` | API server: gRPC/REST, web UI, RBAC enforcement, webhook receiver |
| `argocd-repo-server` | keeps a cached clone of the Git repo and renders manifests |
| `argocd-application-controller` | the reconcile loop: compares live state with target state |
| `argocd-redis` | cache shared by the components above |
| `argocd-dex-server` | SSO connector (unused until you configure a provider) |
| `argocd-applicationset-controller` | bundled since v2.3, templates Applications |
| `argocd-notifications-controller` | sends triggers to Slack/Teams/webhooks |

And the custom resources it registered:

```bash
kubectl get crd | grep argoproj.io
```

## 5. Install the CLI

Download it from the GitHub release matching the server, or straight from your
own API server later on:

```bash
curl -fsSLo argocd \
  https://github.com/argoproj/argo-cd/releases/download/v3.5.2/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/argocd

argocd version --client
```

## 6. Reach the API server

The service is `ClusterIP` by default. For the lab, port-forward it:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Leave that running in its own terminal and open <https://localhost:8080>
(accept the self-signed certificate).

> If port 8080 is already taken on your lab machine, use another one --
> `kubectl port-forward svc/argocd-server -n argocd 8091:443` -- and adjust the
> login command below to match.

## 7. Log in

```bash
argocd admin initial-password -n argocd
argocd login localhost:8080 --username admin --insecure --grpc-web
```

<details>
  <summary>Where did that login go? The CLI just wrote a token somewhere</summary>

Not into your kubeconfig, and not into the cluster. `argocd login` writes to the
CLI's own configuration file:

```bash
cat ~/.config/argocd/config
```

Inside are the server address, a context name, and a **JWT bearer token**. Every
later `argocd` command reads it from there -- which is why `argocd app list`
needs no credentials, and why copying that file to another machine copies your
access with it. Treat it as a credential, not as settings.

The CLI picks the path in this order:

| | |
|---|---|
| `--config <path>` | explicit flag, wins over everything |
| `$XDG_CONFIG_HOME/argocd/config` | when that variable is set |
| `$HOME/.config/argocd/config` | the default you have |

One file can hold several servers at once, exactly like a kubeconfig holds
several clusters -- `argocd context` lists them and switches between them. That
is what makes exercise 4 painless: you will be talking to one Argo CD about two
clusters, but if you ever run two Argo CDs, this is where they live side by
side.

Two ways to skip the file entirely, which is what a pipeline does:

* `--auth-token <token>`, or the `ARGOCD_AUTH_TOKEN` environment variable --
  a token passed in directly, with no login step and nothing written to disk.
* `--core` -- the CLI stops talking to the API server and talks to Kubernetes
  directly, so there is no Argo CD token at all; your kubeconfig authenticates
  you. Useful on a cluster where `argocd-server` is not even exposed.

</details>

Change the password -- the initial one is stored in a Secret that you are meant
to delete afterwards:

```bash
argocd account update-password
kubectl -n argocd delete secret argocd-initial-admin-secret
```

Check that the CLI is talking to the server:

```bash
argocd app list
argocd cluster list
```

An empty application list and one cluster (`https://kubernetes.default.svc`,
the cluster Argo CD runs in) is exactly what you should see.

## 8. Look around the configuration

Nothing here needs changing yet -- just learn where things live:

```bash
kubectl -n argocd get cm
```

| ConfigMap | Holds |
|---|---|
| `argocd-cm` | resource customisations, config management plugins, SSO connectors |
| `argocd-cmd-params-cm` | component flags (e.g. `server.insecure`) |
| `argocd-rbac-cm` | `policy.csv`, `policy.default` |

Repository credentials are **not** here -- since Argo CD 3.0 they must be
Kubernetes Secrets.

---

## Optional: reach Argo CD through an ingress

`kubectl port-forward` is fine for a lab and artificial everywhere else. It
needs its own terminal, it dies when the connection hiccups, and it teaches you
nothing you will use at work. This section replaces it with the real access
path. Do it if you finish early, or afterwards at home.

> **Steps 1 to 8 are the exercise; this part is extra.** It throws the `argocd`
> cluster away and builds a new one, so anything you deployed into it goes with
> it -- which is why it belongs at the end. Every command you need is repeated
> below, so you can work through the section on its own without jumping back up
> the page. The only thing it assumes is that the `argocd` CLI is already
> installed (step 5): that lives on your machine, not in the cluster, so it
> survives.

### 1. A cluster with ports 80 and 443 mapped

The ingress controller has to be reachable from your browser, which needs a
cluster built differently. Start by throwing the current one away:

```bash
kind delete cluster --name argocd
kind create cluster --name argocd --config examples/kind/argocd-cluster-ingress.yaml
kubectl config use-context kind-argocd
```

That config maps the host's ports 80 and 443 into the node and labels it
`ingress-ready=true`. **If anything on your machine already listens on 80 or
443, cluster creation fails outright** -- check with `ss -ltn` first.

### 2. An ingress controller

Not `ingress-nginx`: the Kubernetes community retired it in March 2026 and the
repository is archived, with no further security fixes. We use Traefik, which is
maintained, documented by Argo CD, and has one property that matters here -- it
serves HTTP and gRPC on the same port, so the UI and the CLI can share a single
hostname.

```bash
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik -n traefik --create-namespace \
  --version 41.6.0 \
  --set service.type=ClusterIP \
  --set ports.web.hostPort=80 \
  --set ports.websecure.hostPort=443 \
  --set-string nodeSelector."ingress-ready"=true

kubectl -n traefik rollout status deploy/traefik
```

### 3. Argo CD, again

The same install as step 3, on the new cluster:

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.2/manifests/install.yaml

kubectl -n argocd rollout status deploy/argocd-server
```

### 4. Stop Argo CD doing TLS itself

Traefik terminates TLS at the edge. If Argo CD also served HTTPS behind it you
would have two layers of TLS and confusing gRPC failures. This is the trap the
slides warn about:

```bash
kubectl -n argocd patch cm argocd-cmd-params-cm --type merge \
  -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deploy/argocd-server
kubectl -n argocd rollout status deploy/argocd-server
```

### 5. The ingress

```bash
kubectl apply -f examples/argocd/ingress-traefik.yaml
```

Two details in that file are worth reading. The host is
`argocd.127.0.0.1.nip.io` -- `nip.io` resolves any `*.127.0.0.1.nip.io` name to
127.0.0.1, so you get a real hostname without editing `/etc/hosts` and without
root. And the two annotations bind the route to Traefik's TLS entrypoint;
without them the router answers on plain HTTP only and HTTPS returns 404.

**Note what we just did, though.** That `kubectl apply` is imperative, and so
was the `patch` in the step before it: both changes now exist in the cluster and
nowhere else. Rebuild this cluster and they are gone -- which is the same
criticism this exercise made of installing Argo CD by manifest.

Argo CD's own configuration can produce the Ingress for you. The
[argo-cd chart](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd)
renders it from values, and the same file carries the `server.insecure`
parameter, so both of the last two steps collapse into something you commit:

```yaml
configs:
  params:
    server.insecure: true

server:
  ingress:
    enabled: true
    ingressClassName: traefik
    hostname: argocd.127.0.0.1.nip.io
    annotations:
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      traefik.ingress.kubernetes.io/router.tls: "true"
```

That is the `argocd-values.yaml` from step 3 doing its job: the hostname your
Argo CD answers on, and the fact that something else terminates TLS for it, are
exactly the kind of per-instance decisions that belong in Git rather than in a
`kubectl` command someone ran once. We do it imperatively here because it is one
`apply` against a cluster that already exists, and because seeing the Ingress on
its own makes the two annotations easier to talk about.

### 6. Use it

Open <https://argocd.127.0.0.1.nip.io/> and accept the certificate warning --
Traefik is serving its own self-signed certificate, which is exactly what you
would replace with a real one in production.

This is a new cluster, so it has a new initial password -- read it again rather
than reaching for the one you set earlier:

```bash
argocd admin initial-password -n argocd
argocd login argocd.127.0.0.1.nip.io --username admin --insecure --grpc-web
argocd cluster list
```

`--insecure` skips verification of that self-signed certificate. `--grpc-web`
tunnels gRPC over HTTP/1.1, which is what makes the CLI work through a proxy.
And as in step 7, change the password and delete the Secret once you are in:

```bash
argocd account update-password
kubectl -n argocd delete secret argocd-initial-admin-secret
```

No port-forward, no spare terminal, and the URL survives a disconnect.

### Questions

**1. Why does `server.insecure` make things *more* correct here rather than less?**

<details>
  <summary>answer</summary>

TLS is not being removed, it is being moved. Traefik terminates TLS at the
edge and talks plain HTTP to `argocd-server` over the cluster network. Leave
`argocd-server` serving HTTPS as well and you have two TLS layers, with the
proxy having to trust a self-signed certificate it was never given -- which
surfaces as confusing gRPC errors rather than a clean failure. `server.insecure`
says "something in front of me is doing TLS", which is the normal arrangement
behind any ingress controller.

</details>

**2. What would change if you put a real certificate in front of this?**

<details>
  <summary>answer</summary>

Only Traefik's side of it. You would give the ingress a real certificate --
cert-manager, or a Secret referenced by the route -- and then drop `--insecure`
from `argocd login`, because the CLI would finally have something it can verify.
`server.insecure: true` stays exactly as it is, and so do the ingress
annotations, the hostname and every Application manifest.

</details>

**3. The UI worked on HTTP before you added the two annotations, but HTTPS returned 404. What were they doing?**

<details>
  <summary>answer</summary>

Binding the route to Traefik's TLS entrypoint. Without
`router.entrypoints: websecure` and `router.tls: "true"` Traefik publishes the
router on the plain-HTTP entrypoint only, so a request arriving on 443 matches
no router at all -- and a request that matches no router is a 404, not a TLS
error. The host and the path were right the whole time; nothing was listening
for them on that entrypoint.

</details>

---

[home](../../README.md) | [next: exercise 2](../02-first-app/README.md)
