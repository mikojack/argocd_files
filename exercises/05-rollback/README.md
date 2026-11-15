# Exercise 5: History, Rollback and a Broken Deployment

**Goal:** use the deployment history, watch a bad manifest reach the cluster,
and see the ordering that phases and waves give you.

Work on the `webapp` application from exercise 2.

## 1. Build up some history

Each of these is a commit to your branch followed by a sync. The replica count
you start from depends on what you did in exercise 2 -- the `sed` below
replaces whatever is there rather than matching a particular number.

```bash
# change 1: scale up
sed -i 's/^  replicas: .*/  replicas: 4/' examples/apps/webapp/deployment.yaml
git commit -am "webapp: scale to 4" && git push
argocd app sync webapp

# change 2: change the page content
sed -i 's|<p>version: .*</p>|<p>version: 2</p>|' examples/apps/webapp/configmap.yaml
git commit -am "webapp: content v2" && git push
argocd app sync webapp

argocd app history webapp
```

Note the numbers in the first column — you will need one in a moment. History
records *deployed revisions*, not every commit you made.

## 2. Break it

```bash
sed -i 's|image: nginx:1.28-alpine|image: nginx:9.99-does-not-exist|' \
  examples/apps/webapp/deployment.yaml
git commit -am "webapp: broken image tag" && git push
argocd app sync webapp
```

The sync **succeeds**. Argo CD applied exactly what Git asked for.

```bash
kubectl -n training-webapp get pods
```

`ErrImagePull`. The old pods are still serving — Kubernetes will not tear them
down for a rollout that never becomes ready.

## 3. Watch the status that surprises everyone

Wait about a minute, then:

```bash
argocd app get webapp
```

```
Sync Status:    Synced
Health Status:  Degraded
```

Say it out loud: the cluster contains exactly what Git asked for, and what Git
asked for is broken. Argo CD did its job; the manifest did not.

> The manifests in this repository set `progressDeadlineSeconds: 60`. With the
> Kubernetes default of 600 you would be waiting ten minutes for `Degraded`.

## 4. Roll back

```bash
argocd app history webapp
argocd app rollback webapp <the id of the last good revision>

argocd app get webapp
kubectl -n training-webapp get deploy webapp \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Healthy again — and `OutOfSync`, deliberately: the cluster no longer matches
`targetRevision`, because you moved the cluster backwards and left Git where it
was.

## 5. Fix forward, which is what you would actually do

```bash
sed -i 's|image: nginx:9.99-does-not-exist|image: nginx:1.28-alpine|' \
  examples/apps/webapp/deployment.yaml
git commit -am "webapp: fix forward" && git push

argocd app get webapp --refresh
argocd app diff webapp
```

**The diff is empty and the application is already `Synced`** -- and you have
not run a sync. Work out why before reading on.

The rollback in step 4 already put the good image into the cluster. This commit
made Git say the same thing. There was never any difference for a sync to
close: the two sides were brought together from opposite directions and met in
the middle.

This is the clearest look you will get at what `Sync Status` actually is. It is
the answer to a comparison, recomputed every time Argo CD refreshes -- not a
record that a sync ran. Step 4 left the cluster right and Git wrong; this
commit made Git right too, and the application went `Synced` without a single
API call against your workloads.

So: does this application still need a sync? Nothing in the cluster would
move. Look at `argocd app history webapp`, decide whether you want this commit
to appear there, then run it or do not:

```bash
argocd app history webapp
argocd app sync webapp
argocd app history webapp
```

The rollback bought you the minutes it took to write that commit. What the
history should say about those minutes afterwards is a question your team gets
to answer, not Argo CD.

## 6. Ordering: phases, hooks and waves

A different application, built to show the sequence:

```bash
cat examples/apps/waves/02-presync-job.yaml
cat examples/apps/waves/03-deployment.yaml

kubectl apply -f examples/argocd/application-waves.yaml
argocd app sync waves
```

Watch the output as it runs. Then read the result:

```bash
argocd app get waves
```

```
batch  Job  waves-migration    Succeeded  PreSync
       ConfigMap  waves-content        Synced
apps   Deployment waves-webapp         Synced  Healthy
batch  Job  waves-smoke-test   Succeeded  PostSync
```

### What ordered this, and how Argo CD knew the hooks were yours

Two mechanisms, independent of each other, and they compose:

**Phases** come from `argocd.argoproj.io/hook` on the resource: `PreSync`,
`Sync`, `PostSync`, plus `SyncFail`, `PreDelete`, `PostDelete` and `Skip`.
Argo CD finishes one phase before it starts the next, and `PostSync` waits for
more than the apply -- upstream defines it as running "after all `Sync` hooks
completed and were successful, a successful application, and all resources in
a `Healthy` state". That is why the smoke test could assume the Service was
answering.

**Waves** come from `argocd.argoproj.io/sync-wave`, an integer that defaults to
`0` and may be negative. Inside one phase Argo CD starts at the lowest number,
applies everything in that wave, waits for it to become healthy, and only then
moves on. In this application the ConfigMap is wave `-1` and the Deployment and
Service are wave `1`, all three in the `Sync` phase.

**And nothing in either Job names the application.** There is no field pointing
back at `waves`. The association is made twice over:

* *Before the hook runs* -- the Job is simply one of the manifests rendered
  from this Application's `source`: `repoURL` + `targetRevision` + `path`.
  Everything under `examples/apps/waves` belongs to the `waves` application
  because that is what the Application points at. A hook is an ordinary
  manifest in that set, singled out by its annotation and pulled out of the
  normal apply.
* *After it runs* -- Argo CD stamps the live object so it can recognise it
  again. Since 3.0 the default tracking method is `annotation`:
  `argocd.argoproj.io/tracking-id`, whose value is
  `<app>:<group>/<Kind>:<namespace>/<name>`. Look at what it put there:

```bash
kubectl -n training-waves get job waves-migration \
  -o jsonpath='{.metadata.annotations}' | jq
```

  The method is set by `application.resourceTrackingMethod` in `argocd-cm`;
  the older `label` method uses `app.kubernetes.io/instance` instead.

The consequence is worth saying out loud: membership is decided by *which
Application's source a manifest came from*, so two applications whose sources
both contain a Job of that name in that namespace will fight over it.

The proof is in the timestamps:

```bash
kubectl -n training-waves get pods --sort-by=.metadata.creationTimestamp \
  -o custom-columns='NAME:.metadata.name,CREATED:.metadata.creationTimestamp'
```

The migration pod, then — about ten seconds later, once it finished — the
application pod, then the smoke test. Nothing overlapped.

```bash
kubectl -n training-waves logs job/waves-smoke-test
```

The PostSync job made a real HTTP request to the Service and printed
`smoke test passed`.

## 7. Break a hook

Edit `examples/apps/waves/02-presync-job.yaml` so the migration
fails — change the command to `exit 1` — then commit, push and sync.

```bash
argocd app sync waves
kubectl -n training-waves get pods
```

The PreSync hook failed, so **nothing else was applied**. That is the whole
point of PreSync: a failed migration must not be followed by a deployment that
expects the new schema.

Undo the change before moving on.

## Questions

**1. Why did the broken sync report success?**

<details>
  <summary>answer</summary>

Because a sync reports whether the manifests were applied, not whether the
workload works. `nginx:9.99-does-not-exist` is a perfectly valid image
reference; the API server accepted the Deployment without complaint, and the
sync had then done its whole job. The failure happened afterwards, in the
kubelet, when the pull failed -- and that is what `Health Status` is for.
`Synced` answers "does the cluster match Git?"; it never answers "is this a good
idea?".

</details>

**2. After the rollback, why was the application `OutOfSync`?**

<details>
  <summary>answer</summary>

Because the rollback moved the cluster and left Git exactly where it was.
Sync status compares the live state against `targetRevision`, and that still
pointed at the commit carrying the broken image. So the cluster was healthy and
deliberately not what Git asked for -- which is precisely the state a rollback
is supposed to produce, and precisely why it is a stopgap rather than a fix.

</details>

**3. In step 5 the application went `Synced` before you ran any sync. What did the two `argocd app history` calls show either side of the sync you ran anyway -- and does that change your answer to whether it was needed?**

<details>
  <summary>answer</summary>

Sync status and deployment history are two different records, which is the
whole point of the question. `Synced` came back on its own because it is the
result of a comparison, recomputed at every refresh, and the fix-forward commit
made the two sides agree. `argocd app history`, on the other hand, lists
completed sync *operations* -- the entries `argocd app rollback` later chooses
between. A refresh never adds a row there.

So the cluster did not need the sync, and the history did -- if you want the
repair to be visible as a deployment of its own rather than inferred from the
absence of one. Compare what the two listings actually printed on your machine
and decide what your team would rather read at three in the morning.

</details>

**4. What would `argocd.argoproj.io/hook-delete-policy: HookSucceeded` have cost you when debugging the failed migration?**

<details>
  <summary>answer</summary>

Not the failed Job -- read the policy carefully. `HookSucceeded` deletes the
hook resource when the sync *succeeds*, and this sync did not, so the broken
migration and its pod would have survived either way.

What it costs you is the run before it. After every green sync the Job and its
pod are removed, so when a later run fails you have the failure and nothing to
compare it against: no previous logs, no previous duration, no evidence of what
"working" looked like. These manifests use `BeforeHookCreation` instead, which
keeps the last run until the next one starts -- which is exactly why
`kubectl logs job/waves-migration` still had something to say in step 6.

</details>

---

[home](../../README.md) | [previous](../04-second-cluster/README.md) | [next](../06-automation/README.md)
