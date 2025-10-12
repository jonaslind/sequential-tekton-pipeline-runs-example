# sequential-tekton-pipeline-runs-example

At the time of writing, Tekton doesn't have any model for limiting concurrency of pipeline runs (See
https://github.com/tektoncd/pipeline/issues/2828 for more information). This example shows how to work around this
limitation in Tekton and achieve sequential execution of pipeline runs.

Since Tekton is Kubernetes native, we can use the power of the Kubernetes API to poll pipeline run metadata from within
a task run and wait until it's "our turn".

The interesting part is found inside file `tasks/acquire-takeoff-clearance-task.yaml`, the rest of these resources are
only needed if you want to run the full example.

To make this example as self-contained and minimal as possible, the instructions below are written for a standalone
Tekton installation on Podman and Minikube. Linux shell syntax is used in some cases, adjust according to your
environment.

If your Tekton installation is Red Hat OpenShift Pipelines on a Red Hat OpenShift platform, you can use `oc` instead of
`kubectl` in the `acquire-takeoff-clearance-task` task implementation.

## Configure minikube and install tekton

```
minikube start
minikube addons enable registry
minikube addons enable registry-aliases
eval $(minikube -p minikube docker-env)
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
# Wait until all tekton pods are ready
kubectl get pods --namespace tekton-pipelines --watch
```

## Create a container image to be used by tekton tasks

```
podman build --format docker -t localhost/tekton-builder-image-ubuntu-with-kubectl:latest -f builder/Containerfile
podman image save localhost/tekton-builder-image-ubuntu-with-kubectl:latest -o tekton-builder-image-ubuntu-with-kubectl.tar
minikube image load tekton-builder-image-ubuntu-with-kubectl.tar
rm tekton-builder-image-ubuntu-with-kubectl.tar
```

## Create a namespace for our pipeline and configure it

```
kubectl create namespace ci
kubectl apply -f config/builder-role.yaml
kubectl apply -f config/builder-rolebinding.yaml
```

## Create tasks and pipeline

```
kubectl apply -f tasks/my-echo-task.yaml
kubectl apply -f tasks/my-sleep-task.yaml
kubectl apply -f tasks/acquire-takeoff-clearance-task.yaml
kubectl apply -f pipeline/my-pipeline.yaml
```

## Start three pipelineruns and watch their logs

```
yq '.metadata.name = "my-pipeline-run-'$(date +%s%N)'"' pipeline/my-pipeline-run.yaml | kubectl apply -f -
yq '.metadata.name = "my-pipeline-run-'$(date +%s%N)'"' pipeline/my-pipeline-run.yaml | kubectl apply -f -
yq '.metadata.name = "my-pipeline-run-'$(date +%s%N)'"' pipeline/my-pipeline-run.yaml | kubectl apply -f -
watch 'for pipelinerun in $(kubectl get pipelineruns -n ci --sort-by=.metadata.creationTimestamp --no-headers -o custom-columns=":metadata.name") ; do
  echo "### Pipeline Run: $pipelinerun ###"
  echo ""
  for pod in $(kubectl get pods -n ci -l tekton.dev/pipelineRun=$pipelinerun --sort-by=.metadata.creationTimestamp -o name) ; do
    kubectl logs -n ci --timestamps -c step-run $pod 2> /dev/null
  done
  echo ""
done'
```

## Clean up

```
kubectl delete -n ci $(kubectl get pipelineruns -n ci --sort-by=.metadata.creationTimestamp --no-headers -o name)
```
