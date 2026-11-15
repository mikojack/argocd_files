# Argo CD training -- exercises

Lab material for the NobleProg **Argo CD** course.

> **Work on your own branch.** Several exercises ask you to commit a change and
> watch Argo CD react to it, so you need somewhere you can push to. In the
> course every participant gets a branch of this repository (e.g. `user3`) and
> a token that can push to it -- work on that branch for the whole day, and
> never push to `main`. A fork of your own works too and is described as an
> option in [exercise 1](exercises/01-install/README.md).

On the lab VM this repository is checked out as **`~/workshop`**
(`/home/dadesktop/workshop`) -- that directory *is* the working copy, which is
the same convention as the other NobleProg labs:

```bash
git clone --branch <your-branch> https://gitlab.com/greenitnet/argocd_files.git ~/workshop
cd ~/workshop
```

It is usually there already. Use that clone rather than making a second one --
[exercise 1, step 1](exercises/01-install/README.md) says what to check before
you do. Every path printed in these exercises is relative to `~/workshop`, so
`kind create cluster --config examples/kind/argocd-cluster.yaml` only works
from there.

## Environment

The lab runs on the virtual machine assigned to you in **DaDesktop** -- you
reach it through the browser and work inside it. Nothing is installed on your
own computer:

| | |
|---|---|
| Working directory | `~/workshop` -- this repository, checked out under that name |
| Clusters | two `kind` clusters running on that VM: `argocd` and `workload` |
| Argo CD | **v3.5.2** -- every manifest and command here is pinned to it |
| Operating system | Debian 13 (trixie), which is what [INSTALL.md](INSTALL.md) is written for |

## Tools

All of these are already installed on the lab VM. The list is here so you can
rebuild the environment on your own machine afterwards -- the exercises work
anywhere these are present.

| Tool | What the labs use it for | Version |
|---|---|---|
| `docker` | runs the `kind` clusters; exercise 4 queries it for the second cluster's address | 28.x, or Debian's `docker.io` 26.1.5 |
| `kind` | creates the two lab clusters | v0.30.0 |
| `kubectl` | everything that is not Argo CD itself | v1.34 |
| `argocd` | the CLI: app, cluster, repo, proj, account | **v3.5.2** |
| `helm` | the chart installation in exercise 1, rendering it yourself in exercise 3 | v3.18.6 |
| `git` | branch, commit, push -- most exercises end in a commit | any recent |
| `jq` | reading JSON out of `kubectl -o json` and `argocd ... -o json` | any recent |
| `yq` | the same for YAML, when a manifest is easier to query than to read | 4.x |
| `tree` | reading the shape of a chart and of a Kustomize base in exercise 3 | any recent |

[INSTALL.md](INSTALL.md) has the commands, one section per tool, written for
Debian 13. It also covers the things that make a day of `kubectl` bearable and
that the exercises do not require: bash completion with a `k` alias, `kubectx`
and `kubens`, `fzf`, and `kube-ps1` to keep the current context in your prompt.
On the lab VM they are installed already.

Four notes worth having before you start:

* **Keep the `argocd` CLI on the same version as the server.** It can be
  downloaded from your own API server, which is the simplest way to guarantee
  that.
* **Keep `kubectl` within one minor of the cluster.** `kind` v0.30.0 builds
  Kubernetes v1.34 clusters, which is why `kubectl` is pinned to v1.34 rather
  than to whatever is current.
* `docker` must be usable by your own user, not only by `root` -- `kind` will
  fail early otherwise.
* Debian's `yq` package is **a different program** from the one the exercises
  assume. If `yq --version` reports 3.x, see the note in
  [INSTALL.md](INSTALL.md#notes).

## Exercises

| # | Directory | Topic |
|---|---|---|
| 1 | [01-install](exercises/01-install/README.md) | Install Argo CD, meet its components, log in |
| 2 | [02-first-app](exercises/02-first-app/README.md) | Your first Application: UI, CLI and YAML |
| 3 | [03-sources](exercises/03-sources/README.md) | Helm and Kustomize as application sources |
| 4 | [04-second-cluster](exercises/04-second-cluster/README.md) | Register a second cluster, change destination |
| 5 | [05-rollback](exercises/05-rollback/README.md) | History, rollback and a deliberately broken deployment |
| 6 | [06-automation](exercises/06-automation/README.md) | Automated sync, self-heal, prune, sync waves |

## Layout

```
~/workshop/           the working copy on the lab VM; everything below is relative to it

README.md           this file -- environment, tool list, exercise index
INSTALL.md          how to install every tool, written for Debian 13
presentation/       the slides for the course, as a PDF
exercises/          one directory per lab, each with a README.md
examples/kind/      cluster definitions for the two lab clusters
examples/apps/      the workloads the labs deploy
                      webapp/            plain manifests
                      helm-webapp/       the same thing as a Helm chart
                      kustomize-webapp/  base + dev and prod overlays
                      waves/             hooks and sync waves
                      empty/             deliberately empty, for exercise 6
examples/argocd/    Application and AppProject manifests
examples/pipeline/  reference CI pipeline for the CI/CD module
```

Every exercise has been run end to end on two `kind` clusters against
Argo CD v3.5.2.
