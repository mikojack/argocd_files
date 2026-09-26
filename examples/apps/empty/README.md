# Deliberately empty

No manifests here. Exercise 6 points an application at this directory to trigger
Argo CD's `allowEmpty` protection: automated sync with pruning refuses to run
when manifest generation produces nothing, because that is far more often a
broken path than an intentional deletion.
