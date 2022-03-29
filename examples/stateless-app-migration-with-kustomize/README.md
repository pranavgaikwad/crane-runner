Stateless Application Migration with Kustomize
==============================================

This tutorial shows you how to migrate a simple stateless
[PHP Guestbook application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)
using Crane Runner, Kustomize, and
[Tekton ClusterTasks](https://tekton.dev/docs/pipelines/tasks/#task-vs-clustertask).
Unlike, in the previous exercise where the application was mirrored from one
namespace in the "source" cluster to the same namespace in the "destination"
cluster, here you will use a Kustomize overlay to import the workload into
the desired namespace.

If you just completed [Stateless Application Mirror](../stateless-app-mirror/),
then you can skip to
[Prepare for Application Migration](#prepare-for-application-migration).

# Roadmap

* Deploy Guestbook application in "source" cluster.
* Prepare for application migration.
* Explore new ClusterTasks.
* Migrate the Guestbook application using kustomize overlay in a
    [Tekton PipelineRun](https://tekton.dev/docs/pipelines/pipelineruns/).

# Before You Begin

You will need a "source" and "destination" Kubernetes cluster with Tekton and
the Crane Runner ClusterTasks installed. Below are the steps required for easy
copy/paste:

```bash
# Start up "source" and "destination" clusters in minikube
curl -s "https://raw.githubusercontent.com/konveyor/crane/main/hack/minikube-clusters-start.sh" | bash

# Install Tekton
# See https://tekton.dev/docs/getting-started/ for help with installing Tekton
kubectl --context dest apply -f "https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml"
kubectl --context dest --namespace tekton-pipelines wait --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=180s

# Install Crane Runner manifests
kustomize build github.com/konveyor/crane-runner/manifests | kubectl --context dest apply -f -
```

# Deploy Guestbook Application in "source" Cluster

You will be deploying
[Kubernetes' stateless guestbook application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)
modified here to be consumable via kustomize.
The guestbook application consisists of:

* redis leader deployment and service
* redis follower deployment and service
* guestbook front-end deployment and service


```bash
kubectl --context src create namespace guestbook
kustomize build github.com/konveyor/crane-runner/examples/resources/guestbook | kubectl --context src --namespace guestbook apply -f -
kubectl --context src --namespace guestbook wait --for=condition=ready pod --selector=app=guestbook --timeout=180s
```

# Prepare for Application Migration

First, create the `hello-kustomize` namespace in the "destination" cluster
where you will migrate the guestbook application from the "source" cluster.

```bash
kubectl --context dest create namespace hello-kustomize
```

You must upload your kubeconfig as a secret. This will be used by the
ClusterTasks to migrate the application.

```bash
kubectl config view --flatten | kubectl --context dest --namespace hello-kustomize create secret generic kubeconfig --from-file=config=/dev/stdin
```

# Explore new ClusterTasks

* The [kustomize-namespace ClusterTask](/manifests/clustertasks/kustomize-namespace.yaml)
    takes a directory storing the result of crane's `apply` step and creates a
    kustomize overlay.
* The [kubectl-apply-kustomize](/manifests/clustertasks/kubectl-apply-kustomize.yaml)
    runs `kubectl apply -k` against a Kustomize directory.

# Create Tekton PipelineRun

Submit the [PipelineRun](/examples/stateless-app-migration-with-kustomize/pipelinerun.yaml):

```bash
kubectl --context dest --namespace hello-kustomize create -f "https://raw.githubusercontent.com/konveyor/crane-runner/main/examples/stateless-app-migration-with-kustomize/pipelinerun.yaml"
```

You can watch what's happening with:

```bash
Every 2.0s: kubectl --context dest --namespace hello-kustomize get pipelineruns,taskruns,pvc,pods

NAME                                           SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
pipelinerun.tekton.dev/hello-kustomize-2fl82   True        Succeeded   107s        54s

NAME                                                               SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
taskrun.tekton.dev/hello-kustomize-2fl82-apply                     True        Succeeded   77s         71s
taskrun.tekton.dev/hello-kustomize-2fl82-export                    True        Succeeded   107s        84s
taskrun.tekton.dev/hello-kustomize-2fl82-kubectl-apply-kustomize   True        Succeeded   64s         54s
taskrun.tekton.dev/hello-kustomize-2fl82-kustomize                 True        Succeeded   71s         65s
taskrun.tekton.dev/hello-kustomize-2fl82-transform                 True        Succeeded   84s         77s

NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/pvc-f8438cd335   Bound    pvc-ab83ac0d-52f1-4f0f-b244-77bf34bcdedb   10Mi       RWO            standard       107s

NAME                                                    READY   STATUS      RESTARTS   AGE
pod/frontend-5fd859dcf6-hhfgs                           1/1     Running     0          57s
pod/frontend-5fd859dcf6-p7qsk                           1/1     Running     0          57s
pod/frontend-5fd859dcf6-szkw6                           1/1     Running     0          57s
pod/hello-kustomize-2fl82-apply-pod                     0/1     Completed   0          77s
pod/hello-kustomize-2fl82-export-pod                    0/1     Completed   0          107s
pod/hello-kustomize-2fl82-kubectl-apply-kustomize-pod   0/1     Completed   0          64s
pod/hello-kustomize-2fl82-kustomize-pod                 0/1     Completed   0          71s
pod/hello-kustomize-2fl82-transform-pod                 0/1     Completed   0          84s
pod/redis-master-55d9747c6c-84p8s                       1/1     Running     0          57s
pod/redis-slave-5c6b4c5b47-cslpt                        1/1     Running     0          57s
pod/redis-slave-5c6b4c5b47-mjslc                        1/1     Running     0          57s
```

A few things for you to note when looking at the PipelineRun:

* Data is shared between the tasks using a
    [`volumeClaimTemplate`](https://tekton.dev/docs/pipelines/workspaces/#volumeclaimtemplate).
    Used in this way, it lives as long as the PipelineRun.
* All but the first task use `runAfter` to make the Tasks run sequentially.

# Verify the Migrated Application is Healthy

You can check that everything survived the move:

```bash
kubectl --context dest --namespace hello-kustomize get all
```

Then, verify the Guestbook application's frontend is running properly:

```bash
kubectl --context dest --namespace hello-kustomize port-forward svc/frontend 8080:80
```

Then navigate to localhost:8080 from your browser.

# What's Next

* Check out [GitOps Integration](../gitops-integration/README.md)
* Read more about [Tekton](https://tekton.dev/docs/getting-started/)
* Read more about [Crane](https://github.com/konveyor/crane)
* Read more about [Kustomize](https://kustomize.io)

# Cleanup

```bash
kubectl --context dest delete namespace hello-kustomize
```
