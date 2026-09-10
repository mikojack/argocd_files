# Argo CD training -- exercises

Lab material for the NobleProg **Argo CD** course.

> **Fork this repository before you start.** Several exercises ask you to commit
> a change and watch Argo CD react to it, so you need a repository you can push
> to. Work on your fork for the whole day.

```bash
git clone https://gitlab.com/<your-user>/argocd_files.git
cd argocd_files
```

## Environment

Everything runs on your own workstation:

| | |
|---|---|
| Clusters | two local `kind` clusters: `argocd` and `workload` |
| Argo CD | **v3.5.2** -- every manifest and command here is pinned to it |
| Tools | `docker`, `kind`, `kubectl`, `argocd`, `helm`, `git` |

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
exercises/    one directory per lab, each with a README.md
examples/     manifests used by the labs (kind configs, sample apps, pipelines)
```
