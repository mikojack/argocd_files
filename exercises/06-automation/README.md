# Exercise 6: Automated Sync, Self-Heal and Prune

**Goal:** hand the wheel to Argo CD, then find every safety catch by walking
into it deliberately.

## 1. Turn automation on

```bash
argocd app set webapp --sync-policy automated
argocd app get webapp | grep "Sync Policy"
```

Now make a change in Git and do **not** sync:

```bash
sed -i 's|<p>version: .*</p>|<p>version: 3</p>|' examples/apps/webapp/configmap.yaml
git commit -am "webapp: content v3" && git push
```

Wait. Detection is 120 seconds plus up to 60 seconds of jitter — so up to three
minutes. Watch it happen:

```bash
watch -n5 'argocd app get webapp | head -14'
```

Impatient? `argocd app get webapp --refresh` forces the comparison now. In
production you would configure a webhook instead.

## 2. Rollback is no longer available

```bash
argocd app history webapp
argocd app rollback webapp <any earlier id>
```

```
rollback cannot be initiated when auto-sync is enabled
```

The controller would immediately sync it forward again, so Argo CD refuses
rather than fight itself. To roll back you must first
`argocd app set webapp --sync-policy none`.

**This is why exercise 5 came before this one.**

## 3. Drift, without self-heal

```bash
kubectl -n training-webapp scale deploy/webapp --replicas=1
sleep 20
argocd app get webapp | grep "Sync Status"
kubectl -n training-webapp get deploy webapp
```

`OutOfSync`, one replica. Automation applies what is in Git when Git changes; it
does not undo what you do to the cluster. The drift is visible and waiting for a
human.

## 4. Drift, with self-heal

```bash
argocd app set webapp --self-heal

kubectl -n training-webapp scale deploy/webapp --replicas=1
sleep 25
kubectl -n training-webapp get deploy webapp
```

Back to four. Tell your team before you enable this: their debugging edits will
disappear under them, which is the point and is still surprising the first time.

## 5. Delete something from Git

```bash
git rm examples/apps/webapp/service.yaml
git commit -m "webapp: drop the service" && git push
```

Wait for the sync, then:

```bash
kubectl -n training-webapp get svc
```

The Service is still there. Pruning is **off by default** — GitOps will create
and update without being asked, but it will not delete.

## 6. Enable pruning

```bash
argocd app set webapp --auto-prune
kubectl -n training-webapp get svc
```

Still there. The application is already `Synced`, so nothing triggers a new
comparison. Force one:

```bash
argocd app get webapp --refresh
sleep 15
kubectl -n training-webapp get svc
argocd app get webapp | tail -5
```

Now the Service shows as `Pruned` and it is gone. Turning on pruning does not
retroactively clean up — it applies at the next comparison.

Restore the file afterwards. `git checkout <path>` on its own will not do it
-- you deleted the file *and committed the deletion*, so there is no version of
it in `HEAD` or in the index to check out, and git answers `did not match any
file(s) known to git`. You have to say which commit to take it from: the one
before the commit that removed it.

```bash
git checkout "$(git rev-list -n 1 HEAD -- examples/apps/webapp/service.yaml)^" \
  -- examples/apps/webapp/service.yaml
git commit -m "webapp: restore the service" && git push
```

`git rev-list -n 1 HEAD -- <path>` is the last commit that touched that file --
here, the one that deleted it -- and `^` is its parent, where the file was
still alive. If the deletion is genuinely your most recent commit,
`git checkout HEAD~1 -- examples/apps/webapp/service.yaml` says the same thing
and is easier to read; the form above keeps working after you have committed
something else in between.

The checked-out file is already staged, so the `git commit` above needs no
`-a`.

## 7. Walk into the last safety catch

What happens if manifest generation suddenly produces nothing — a typo in the
path, a branch that does not exist, a chart that fails to render?

```bash
argocd app set webapp --path examples/apps/empty
sleep 30
argocd app get webapp
```

```
SyncError  Skipping sync attempt to [...]: auto-sync will wipe out all resources
```

```bash
kubectl -n training-webapp get deploy
```

Everything is still running, and it is worth being exact about what stopped it.
`allowEmpty` is not a global setting and not one of the `syncOptions`: it is a
field of the **automated** sync policy, `spec.syncPolicy.automated.allowEmpty`,
and it defaults to `false`. You will see it in the manifest you read in step 8:

```yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
```

On the CLI it is `argocd app set webapp --allow-empty`, which the help text
describes as "Set allow zero live resources for automated sync policy" -- and
that last phrase is the part to notice. Because the field lives inside
`automated`, it guards *automatic* syncs only. Read the error again: "auto-sync
will wipe out all resources". A manual `argocd app sync` walks straight past
this catch and applies the empty set.

Do not set it to `true` to "fix" a sync error — the error is the feature.

Put the path back:

```bash
argocd app set webapp --path examples/apps/webapp
```

## 8. The full policy, declaratively

Everything you just did with `argocd app set` belongs in Git:

```bash
cat examples/argocd/application-webapp-automated.yaml
```

Apply it and you are back where you started, except the policy is now version
controlled like everything else.

## Questions

**1. You push a commit and nothing happens for two minutes. Name three possible reasons.**

<details>
  <summary>answer</summary>

Any three of these, and the order is roughly how often they turn out to be
the answer:

* **Argo CD has not looked yet.** Without a webhook it polls, and step 1 showed
  the window. Two minutes of nothing is the normal case, not a fault.
* **The commit is not where the application is looking** -- a different branch
  from `targetRevision`, or a path outside `source.path`. The application is
  `Synced` and correct; it is simply synced to something else.
* **Automation is off.** The application goes `OutOfSync` and stays there,
  waiting for a human. Check `Sync Policy` before blaming the refresh.
* **The comparison is failing rather than idling** -- repository credentials,
  an unreachable repo, a chart that will not render. Look for a
  `ComparisonError` condition instead of assuming silence means inaction.

</details>

**2. Why does enabling `prune` not delete anything until the next comparison?**

<details>
  <summary>answer</summary>

Because `prune` is not an action, it is a permission that applies while a
sync runs. Setting it rewrites the sync policy and nothing else; the application
was already `Synced`, so there was no sync queued and no comparison pending, and
a permission with nothing to authorise does nothing. The deletion happens the
first time Argo CD compares afterwards and finds a live resource with no
counterpart in the target state -- which is why `--refresh` made it immediate.
The same reasoning explains the more alarming version of this: turning pruning
on does not retroactively tidy a namespace, so the damage, when it comes, is
always on someone's next commit.

</details>

**3. Which of `prune`, `selfHeal` and `allowEmpty` would you enable on a production cluster tomorrow, and which would you argue about first?**

<details>
  <summary>answer</summary>

`allowEmpty` is not a candidate: leave it off. It is the catch that saved the
application in step 7, and the only time anyone wants to turn it on is while
staring at the sync error it exists to produce.

`prune` is usually fine, on one condition -- that the repository really is the
only thing writing to those namespaces. Audit what is living there that Git has
never heard of *before* you enable it, not after.

`selfHeal` is the one to argue about, and the argument is not technical. It is
right in principle and it will silently undo a colleague's `kubectl edit` while
they are still looking at it. Enable it once the team knows it is on, never as a
quiet improvement.

</details>

---

[home](../../README.md) | [previous](../05-rollback/README.md)
