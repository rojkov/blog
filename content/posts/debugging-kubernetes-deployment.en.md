---
title: "Debugging a Kubernetes Deployment"
date: 2018-09-09T16:23:22+03:00
draft: false
---

When looking for a Kubernetes issue I could investigate I stumbled upon
this [problem](https://github.com/kubernetes/kubernetes/issues/67515).
The title should have been worded as something like &laquo;A new mysterious
`ReplicaSet` replaces the original one soon after its `Deployment` has
been created&raquo;. What's important is that this new `ReplicaSet` fails
to start.

In hindsight the case seems rather trivial, but at the time I saw the
issue it didn't feel like that. The first suspect was the code of
Kubernetes mistakenly firing up an unneeded `ReplicaSet`, thus I needed
a way to hunt down the culprit in the code.

How do people debug such issues? Isn't there a proven methodology, a more
or less systematic approach to solving similar puzzles. I'm not aware
of any. [Not only me](https://github.com/kubernetes/kubernetes/issues/67515#issuecomment-415481314)
is asking such questions it seems. So, I decided to copy my comment
from the issue discussion to this blog.

<!--more-->

My style of debugging is trial and error led by my intuition. But here
is the scheme I usually use:

0. Reproduce the issue locally. Usually this is the most time consuming
   step.

1. Look into all available logs for anything weird or strange (nothing
   interesting here this time).

2. Try to build an older version of Kubernetes, as old as possible. And see
   if the issue is reproducible. Then in case it's not use `git bisect` to
   find the offending commit. Unfortunately the issue is reproducible even
   with K8s 1.8.0.

3. Examine the content of the objects in question (`ReplicaSet`s) using
   `kubectl describe ...`. How objects of the same type differ from each
   other. If some objects are mutable try to find out how their content
   gets updated (`Deployment`). Here I found out that the new replica set lacks
   tolerations and the deployment is not immutable.

4. Find the code which runs upon deployment updates and increase its log
   verbosity. It turned out the relevant code is the deployment controller
   of `kube-contoller-manager`. This environment variable increases log
   verbosity for the code `LOG_SPEC=deployment_controller*=5,replica_set*=5,sync*=5,recreate*=5`.
   The only outcome on this step was the confirmation that the deployment
   gets updated.

5. Modify the sync handler for `Deployment` objects to log what exactly has
   changed (this lib can be used to do the diff https://github.com/d4l3k/messagediff).
   Here we see that the only change was the removal of the tolerations.

6. Look for code which actually does update `Deployment` through the API server
   with this command
   ```shell
   $ git grep Deployments | grep "Update("
   ```
   There are only few places in Kubernetes where `Deployment`s get updated,
   but none looks relevant. This leads to the hypothesis that the source of
   the update is somewhere outside of Kubernetes.

7. Take a closer look at the YAML files. They seem to be quite complex.

8. Try to make the setup as simple as possible where the issue is still
   reproducible. Here I found that small one line changes in different YAML
   files lead to the issue being not reproducible - the setup is too fragile.

9. Then look through the YAMLs line by line in the hope to find something
   interesting. The interesting bit here is that the deployment operates
   on behalf of the `kube-state-metrics` service account which requests
   permission to do `update` on `Deployment`s.

10. Drop this permission. This confirms that the Deployment doesn't
    change any more.

11. Look into the logs of the running pod. Bingo! In `addon-resizer`'s logs
    there is an error about inability to update the `Deployment`.

12. Try to find the sources of `addon-resizer:1.0`. This turned out to be
    not easy task, because the version 1.0 is **very** old. Apparently the
    culprit is K8s API has evolved too much since v1.0. Looking at `PodSpec`
    used in `addon-resizer:1.0` confirms it.

So, it turned out the problem is in the outdated container image
`quay.io/coreos/addon-resizer:1.0` of the deployment.

It runs a client which updates the `kube-state-metrics` deployment. But the
client uses outdated API which lacks `Tolerations` in `PodSpec`. As result
the updated deployment looses tolerations in its pod template. The deployment
controller recreates `ReplicaSet`. The pods controlled by the new `ReplicaSet`
lack tolerations as well and fail to run on the tainted node.
