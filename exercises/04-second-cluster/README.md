# Exercise 4: A Second Cluster

**Goal:** register the `workload` cluster with Argo CD and deploy onto it from
the same repository — and find out what `argocd cluster add` really does.

## 1. The first attempt fails, on purpose

```bash
kubectl config get-contexts -o name
argocd cluster add kind-workload --name workload
```

It fails:

```
error getting server version: Get "https://127.0.0.1:34119/version?timeout=32s":
dial tcp 127.0.0.1:34119: connect: connection refused
```

Read that address. `kind` writes a `127.0.0.1:<random port>` endpoint into your
kubeconfig, which is correct **for the machine `kind` runs on**. Argo CD runs
inside a container in another cluster, and for it `127.0.0.1` is itself.

Both clusters are containers on one Docker network, and the question is only
ever "whose `127.0.0.1`?":

```
                docker network "kind"   172.18.0.0/16

   +--------------------------+        +--------------------------+
   | argocd-control-plane     |        | workload-control-plane   |
   | 172.18.0.2               |        | 172.18.0.3               |
   |                          |        |                          |
   |  argocd-application-     |        |  kube-apiserver :6443    |
   |  controller (a pod)      |        |                          |
   +--------------------------+        +--------------------------+
          |      |                                ^          ^
          |      '--> 127.0.0.1:34119 --> itself  |          |
          |           nothing is listening [X]    |          |
          |                                       |          |
          '--- 172.18.0.3:6443 -------------------'   [OK]   |
                                                             |
   your lab VM                                               |
   kubectl / argocd CLI --- 127.0.0.1:34119 -----------------'
                            a port published on the host by kind
```

The host reaches the API server through a port `kind` published on
`127.0.0.1`; the pod cannot, because that address belongs to its own container.
The address both of them can use is the one on the Docker network -- which is
what the next steps put into the kubeconfig.

## 2. Look at what it created anyway

The command failed at the last step, but it had already done its work on the
target cluster:

```bash
kubectl --context kind-workload -n kube-system get sa argocd-manager
kubectl --context kind-workload get clusterrole argocd-manager-role
kubectl --context kind-workload get clusterrolebinding argocd-manager-role-binding
```

This is what registering a cluster means: a ServiceAccount on the target, a
ClusterRole that by default can manage everything there, and a token that Argo
CD stores. Worth knowing before you run it against production.

## 3. Give it an address that works from inside the cluster

Both clusters sit on the same Docker network, so use the container address:

```bash
WORKLOAD_IP=$(docker inspect workload-control-plane \
  -f '{{.NetworkSettings.Networks.kind.IPAddress}}')
echo $WORKLOAD_IP

kubectl config set-cluster kind-workload --server="https://$WORKLOAD_IP:6443"
kubectl --context kind-workload get nodes
```

The API server certificate already covers that address, so nothing else needs
changing — and your own `kubectl` keeps working, because your machine can reach
the Docker network too.

```bash
argocd cluster add kind-workload --name workload --yes
argocd cluster list
```

Two clusters now, both `Successful`.

> If you restart Docker, the container may get a different IP and the cluster
> goes `Unknown`. Re-run the two commands above. In production this is a stable
> DNS name and the problem does not exist.

## 4. Deploy to the other cluster — and get refused

```bash
argocd app create webapp-workload \
  --project training \
  --repo "$REPO" \
  --path examples/apps/webapp \
  --revision "$BRANCH" \
  --dest-name workload \
  --dest-namespace training-webapp \
  --sync-option CreateNamespace=true
```

```
application destination server 'workload' and namespace 'training-webapp'
do not match any of the allowed destinations in project 'training'
```

The AppProject you wrote in exercise 2 allows exactly one cluster. This is the
control working as designed.

## 5. Widen the project, deliberately

Add the second destination to `examples/argocd/appproject-training.yaml`:

```yaml
  destinations:
    - server: https://kubernetes.default.svc
      namespace: 'training-*'
    - name: workload
      namespace: 'training-*'
```

Commit, push, apply, and retry:

```bash
git add examples/argocd/appproject-training.yaml
git commit -m "training: allow the workload cluster"
git push

kubectl apply -f examples/argocd/appproject-training.yaml
argocd app create webapp-workload \
  --project training \
  --repo "$REPO" \
  --path examples/apps/webapp \
  --revision "$BRANCH" \
  --dest-name workload \
  --dest-namespace training-webapp \
  --sync-option CreateNamespace=true

argocd app sync webapp-workload
```

## 6. Prove it landed somewhere else

```bash
kubectl --context kind-workload -n training-webapp get deploy
kubectl --context kind-argocd  -n training-webapp get deploy
```

Same manifests, same repository, two clusters. The only thing that differs is
four lines of `destination`.

## Questions

**1. What exactly does Argo CD store when you register a cluster, and where?**

<details>
  <summary>answer</summary>

A Secret in the `argocd` namespace, labelled
`argocd.argoproj.io/secret-type: cluster`. It carries the cluster's `name`, its
`server` URL, and a `config` blob holding the bearer token of the
`argocd-manager` ServiceAccount together with the TLS settings and CA data. Not
a database, not Git -- a Secret, readable by anyone with `get secrets` in that
namespace, which is one more reason to take Argo CD's own RBAC seriously.

```bash
kubectl -n argocd get secret -l argocd.argoproj.io/secret-type=cluster
```

</details>

**2. Your colleague ran `argocd cluster add` against production, and it failed. Is the production cluster unchanged?**

<details>
  <summary>answer</summary>

No, and this is the one to remember. Step 2 showed why: the ServiceAccount,
the ClusterRole and the ClusterRoleBinding are created on the *target* cluster
before the command reaches the step that failed. A failed `cluster add` leaves
`argocd-manager` behind with rights over everything in that cluster. Nothing
cleans it up for you -- `kubectl delete` the three objects deliberately, or
decide deliberately to keep them.

</details>

**3. How would you let a team deploy to `workload` but only into their own namespaces?**

<details>
  <summary>answer</summary>

With an AppProject -- the same object you widened in step 5. Give it a
`destinations` entry naming that cluster and a namespace pattern of their own
(`name: workload`, `namespace: 'team-a-*'`), restrict `sourceRepos` to their
repository, and keep `clusterResourceWhitelist` as narrow as they can live with.
Then bind the team to that project in `argocd-rbac-cm`. The boundary is
enforced when an Application is created or updated, which is exactly the error
you triggered in step 4 -- before anything reaches the cluster.

</details>

---

[home](../../README.md) | [previous](../03-sources/README.md) | [next](../05-rollback/README.md)
