# Reference pipelines

Two flavours of the same shape -- `.gitlab-ci.yml` for GitLab CI and
`github-actions/` for GitHub Actions. Both belong in the **application**
repository, not in this one.

## The shape

CI builds an image and writes its tag into the config repository. That commit is
the deployment; Argo CD notices it and syncs. Staging is written automatically on
every merge. Production is a separate, manually triggered promotion that opens a
pull request -- merging it is the production deployment.

Two gates guard production in the GitHub Actions version: the environment
protection rule that releases the promotion job, and the review on the pull
request it opens. Worth knowing before you rely on the first one: an environment
may list several required reviewers, but a single approval releases the job. If
you need two people, express that as a pull request rule.

## GitLab

`.gitlab-ci.yml` belongs in the **application** repository -- the one holding
the Dockerfile and the source code. It builds an image and then commits the new
tag into the **config** repository (this one), which is what Argo CD watches.

Two things make this safe:

* the pipeline holds no cluster credentials at all -- only a token that may push
  to the config repository;
* the config repository has no CI of its own, so a config commit cannot trigger
  another build.

`CONFIG_REPO_TOKEN` is a project access token on the config repository with
`write_repository` scope, stored as a masked CI variable.

If the pipeline has to report on the outcome of the deployment, add
`argocd app wait` at the end, authenticated with an **AppProject role token**
scoped to that one project. Never use the admin account.
