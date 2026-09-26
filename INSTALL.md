# Installing the lab tools

The DaDesktop VM already has all of this. Use these commands when you want to
rebuild the environment on your own machine after the course.

Written for **Debian 13 (trixie)**, which is what the lab VM runs. Package
versions below were read from the trixie archive; every download URL was
checked. Where a version is pinned, it is pinned on purpose -- see
[Notes](#notes).

## What Debian 13 gives you, and what it does not

| Tool | In the archive? | How to install |
|---|---|---|
| `git` | yes -- 2.47.3 | `apt-get` |
| `jq` | yes -- 1.7.1 | `apt-get` |
| `tree` | yes -- 2.2.1 | `apt-get` |
| `bash-completion` | yes -- 2.16.0 | `apt-get` |
| `fzf` | yes -- 0.60.3 | `apt-get`, or upstream for a current release |
| `kubectx`, `kubens` | yes -- 0.9.5 | `apt-get`, or upstream for a current release |
| `docker` | yes -- `docker.io` 26.1.5 | `apt-get`, or upstream for a current release |
| `yq` | **the wrong one** -- 3.4.3, a different program | upstream binary |
| `kind` | no | upstream binary |
| `kubectl` | no | upstream binary |
| `argocd` | no | upstream binary |
| `helm` | no | upstream tarball |
| `kube-ps1` | no | shell script from GitHub |

## From the distribution

```bash
sudo apt-get update
sudo apt-get install -y git jq tree curl ca-certificates bash-completion
```

## docker

Debian's own package is enough for the lab:

```bash
sudo apt-get install -y docker.io
```

For a current Docker release instead (28.x at the time of writing):

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Either way, let your user talk to the socket:

```bash
sudo usermod -aG docker "$USER"
newgrp docker            # or log out and back in
```

## kind

```bash
curl -fsSLo ./kind https://kind.sigs.k8s.io/dl/v0.30.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

## kubectl

```bash
curl -fsSLO "https://dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

## kubectl completion and the `k` alias

Two days of typing `kubectl` in full is two days you do not get back. This is
the [upstream recipe][kubectl-optional], with Debian's paths filled in.

Completion ships inside `kubectl` itself, but it needs `bash-completion`
underneath it. Check whether you already have that loaded:

```bash
type _init_completion
```

If it prints a shell function, move on. If it says "not found":

```bash
sudo apt-get install -y bash-completion
echo 'source /usr/share/bash-completion/bash_completion' >>~/.bashrc
```

Then the kubectl script and the alias:

```bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
exec bash
```

Now `k get pods` works, and so do `k get po<TAB>`, `k rollout <TAB>` and the
rest. The third line is the one people leave out -- without it the alias still
runs, it just stops completing, because completion is registered against the
name `kubectl` and bash matches on the name you typed.

The `source <(...)` line regenerates the script on every new shell. To pay that
cost once instead, write the script out and drop that line:

```bash
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl >/dev/null
```

The file then goes stale when you upgrade `kubectl`, so regenerate it then.
Keep the `alias` and `complete` lines in `~/.bashrc` either way.

[kubectl-optional]: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#optional-kubectl-configurations-and-plugins

## argocd

```bash
curl -fsSLo ./argocd \
  https://github.com/argoproj/argo-cd/releases/download/v3.5.2/argocd-linux-amd64
chmod +x ./argocd
sudo mv ./argocd /usr/local/bin/argocd
```

## helm

```bash
curl -fsSLO https://get.helm.sh/helm-v3.18.6-linux-amd64.tar.gz
tar -zxf helm-v3.18.6-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
rm -rf linux-amd64 helm-v3.18.6-linux-amd64.tar.gz
```

## yq

Not from `apt` -- see the note below.

```bash
curl -fsSLo ./yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
chmod +x ./yq
sudo mv ./yq /usr/local/bin/yq
```

## kubectx and kubens

[`kubectx`](https://github.com/ahmetb/kubectx) switches the current context and
`kubens` switches the current namespace. Both replace a `kubectl config
set-context --current --namespace=...` you would otherwise retype all day, and
they start paying for themselves at exercise 4, where a second cluster appears.

```bash
sudo apt-get install -y kubectx
```

Debian ships 0.9.5 and installs the bash completions with it, so `kubectx
<TAB>` offers your contexts with nothing further to configure. For the current
upstream release (0.11.0), which is two separate downloads:

```bash
curl -fsSL https://github.com/ahmetb/kubectx/releases/download/v0.11.0/kubectx_v0.11.0_linux_x86_64.tar.gz \
  | sudo tar -xz -C /usr/local/bin kubectx
curl -fsSL https://github.com/ahmetb/kubectx/releases/download/v0.11.0/kubens_v0.11.0_linux_x86_64.tar.gz \
  | sudo tar -xz -C /usr/local/bin kubens
```

What you will actually use:

```bash
kubectx                  # list contexts, current one marked
kubectx kind-argocd      # switch to it
kubectx -                # switch back to the previous one
kubens argocd            # later kubectl calls default to this namespace
kubens -
```

## fzf

[`fzf`](https://github.com/junegunn/fzf) is a general fuzzy finder, and it
earns its place next to the two above: when `fzf` is on `$PATH`, `kubectx` and
`kubens` with no arguments turn into interactive pickers instead of printing a
list.

```bash
sudo apt-get install -y fzf
```

Debian installs the binary but does not wire up the shell integration. Add it:

```bash
echo 'eval "$(fzf --bash)"' >>~/.bashrc
exec bash
```

That is <kbd>Ctrl-R</kbd> over shell history and <kbd>Ctrl-T</kbd> over file
paths -- both worth having during a lab full of long `kubectl` lines. For a
current upstream release (0.74.4):

```bash
curl -fsSL https://github.com/junegunn/fzf/releases/download/v0.74.4/fzf-0.74.4-linux_amd64.tar.gz \
  | sudo tar -xz -C /usr/local/bin fzf
```

If you want the plain listing back from `kubectx` for one command, pipe it:
`kubectx | cat`. To opt out of the picker permanently, set
`KUBECTX_IGNORE_FZF=1`.

## kube-ps1

[`kube-ps1`](https://github.com/jonmosco/kube-ps1) puts the current context and
namespace into the shell prompt, which is the cheapest way to stop running
exercise commands against the wrong cluster. It is not packaged for Debian --
it is one shell script:

```bash
sudo curl -fsSLo /usr/local/share/kube-ps1.sh \
  https://raw.githubusercontent.com/jonmosco/kube-ps1/v1.0.0/kube-ps1.sh
```

Then in `~/.bashrc`:

```bash
source /usr/local/share/kube-ps1.sh
PS1='[\u@\h \W $(kube_ps1)]\$ '
```

Single quotes around that `PS1`, always: `$(kube_ps1)` has to run at every
prompt, not once while `.bashrc` is being read. The prompt then carries
`(⎈|kind-argocd:argocd)`. `kubeoff` hides it in the current shell,
`kubeon` brings it back.

## macOS

```bash
brew install kind kubectl helm git jq yq tree argocd kubectx fzf kube-ps1 bash-completion
```

Docker Desktop, or another container runtime, has to be running before `kind`
will do anything.

Homebrew installs `kube-ps1` as
`$HOMEBREW_PREFIX/opt/kube-ps1/share/kube-ps1.sh` -- that is the path to
`source`. The `.bashrc` lines for completion, the `k`
alias, `fzf` and the prompt are the same as above; on a stock macOS shell they
belong in `~/.zshrc`, with `kubectl completion zsh` in place of `bash`.

## Check that it worked

```bash
docker version --format '{{.Server.Version}}'
kind version
kubectl version --client -o yaml | grep gitVersion
argocd version --client --short
helm version --short
git --version
jq --version
yq --version
tree --version
kubectx --version
kubens --version
fzf --version
```

The shell pieces only exist in an interactive shell, so check them by using
them: `k get<TAB>` should complete, `kubectx` with no arguments should open the
fzf picker, and the prompt should show a context.

```bash
type _init_completion     # bash-completion is loaded
type __start_kubectl      # kubectl completion is loaded
complete -p k             # complete -o default -F __start_kubectl k
type kube_ps1             # the prompt function exists
```

Then the real test -- this is exercise 1, step 2:

```bash
kind create cluster --name smoke
kubectl --context kind-smoke get nodes
kind delete cluster --name smoke
```

## Notes

**`docker` must work without `sudo`.** `kind` talks to the Docker socket as your
own user and fails early otherwise. That is what the `usermod` line is for, and
it only takes effect in a new login session.

**Debian's `yq` is a different program.** The archive ships version 3.4.3 of
kislyuk's `yq`, a Python wrapper around `jq`. The tool the examples assume is
Mike Farah's Go implementation, which is on version 4 and has different syntax.
Installing the Debian package will produce errors that look like your commands
are wrong when the problem is that you have another tool of the same name.

**Keep `kubectl` close to the cluster.** Kubernetes supports a skew of one minor
version either side. `kind` v0.30.0 creates Kubernetes v1.34 clusters, so the
command above pins kubectl to v1.34 rather than fetching whatever the current
stable release happens to be -- which is already several minors ahead.

**Keep `argocd` on the server's version.** The course pins Argo CD to
**v3.5.2**. Once a server is running, the CLI can also be downloaded straight
from it, which removes the question entirely.

**The alias is a local convenience, not course notation.** Every exercise and
slide spells out `kubectl`, so that the commands work on a machine where none
of this section has been done. Read `kubectl` as `k` wherever you like.

**`kube-ps1` hooks `PROMPT_COMMAND`.** It refreshes the context and namespace
just before each prompt is drawn. If you set `PROMPT_COMMAND` yourself further
down in `~/.bashrc`, append to it rather than overwriting it, or the prompt
will go blank.

**ARM machines.** Replace `amd64` with `arm64` in the download URLs. On Apple
Silicon the Homebrew commands above are the easier route.
