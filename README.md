## Configuring a Group Level Kubernetes cluster for Gitlab Runners

### Create cluster

1. Navigate to the [kubernetes page](https://gitlab.com/groups/digitalmint/-/clusters) for your group.
2. Make sure you select a project that has billing enabled.
3. If doing this in a project for the 1st time make sure the kubernetes API is enabled. [Like this](https://console.cloud.google.com/apis/library/container.googleapis.com?q=kuber&id=1def4230-f361-4931-b386-576c62b90799&project=digitalmint-prices)
4. Create the cluster with the environment scope `*`.


### Configuration

To deploy gitlab runners we will be using the [Gitlab Runner](https://gitlab.com/charts/gitlab-runner/tree/master) helm chart.

You must apply two files to this cluster.

* `values.yaml` contains the values the Helm Chart will use.
    * You set the gitlab url and token values for based on the values from [this page](https://gitlab.com/groups/digitalmint/-/settings/ci_cd).
    * If you do not specify any of these the defaults will be set by the [chart](https://gitlab.com/charts/gitlab-runner/blob/master/values.yaml).
    * `rbac: create: true` will create service account tokens for your projects.
    * the runners must have `privileged: true` so they can run `dind` inside of docker.


* `namespace.yaml` contains the values for the namespace you will create in your project.
    * This isn't really mentioned in the documentation but you **cannot** use the default namespace.


### Deployment

1. First make sure you are in the right gcloud context:

`gcloud container clusters get-credentials YOUR_CLUSTER --zone us-central1-a --project YOUR_PROJECT`

2. Make sure you have helm installed:

`brew install kubernetes-helm`

3. Apply your namespace to the cluster:

`kubectl create -f namespace.yaml`

4. Initialize helm:

`helm init`

5. Give helm tiller access via a service account:

```shell
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

6. Install the helm chart to your cluster:

`helm install --namespace YOUR_NAMESPACE --name YOUR_RUNNER_NAME -f values.yaml gitlab/gitlab-runner`

7. If you need to upgrade the chart:

`helm upgrade --namespace YOUR_NAMESPACE -f values.yaml YOUR_RUNNER_NAME gitlab/gitlab-runner`


### Using with Gitlab CI

In most cases no changes will need to be made for your CI to run. There are two caveats.

1. If you are trying to run `dind` in your runner you must set the following environment variables in your `.gitlab-ci.yml`:

```yaml
variables:
  DOCKER_HOST: tcp://localhost:2375/
  DOCKER_DRIVER: overlay2
services:
  - docker:dind
```

2. If you are deploying to a different kubernetes cluster from your `gitlab-runner`. You must specify the namespace that the deployment is in the `.gitlab-ci.yml`. For example:

`kubectl apply -f cfg/qa/web-deployment.yaml --namespace=default`

This is because the runner assumes you are deploying to the same namespace that it exists in, even though the clusters are different.
