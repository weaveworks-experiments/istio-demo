# Weave Cloud Istio demo

Prerequisites:

* GKE > 1.10.0 
* CloudDNS
* Istio > 1.0.0
* Helm and Tiller
* Weave Cloud agents

Setup Istio with Let's Encrypt wildcard certs on Google Cloud by following this [guide](https://github.com/stefanprodan/istio-gke).

### Install Flagger

[Flagger](https://github.com/stefanprodan/flagger) is a Kubernetes operator that automates the promotion of canary deployments
using Istio routing for traffic shifting and Prometheus metrics for canary analysis.

Deploy Flagger and Grafana in the `istio-system` namespace using Helm:

```bash
helm repo add flagger https://flagger.app

helm upgrade -i flagger flagger/flagger \
--namespace=istio-system \
--set metricsServer=http://prometheus.istio-system:9090 \
--set controlLoopInterval=1m

helm upgrade -i flagger-grafana flagger/flagger-grafana \
--namespace=istio-system
```

### Install Flux

Prerequisites:

* Create a new branch from master
* Edit the `workloads/canary.yaml` and change the `podinfo.example.com` with your own domain
* Push the changes to your branch

Before installing Flux scale down the weave agent anf flux deployments that are running in `weave` namespace:

```bash
kubectl -n weave scale --replicas=0 deployment/weave-agent
kubectl -n weave scale --replicas=0 deployment/weave-flux-agent
```

Install Flux using Helm (make sure to use a unique Flux git tag):

```bash
export GIT_BRANCH=your-branch
export GIT_TAG=your-git-tag
export WC_TOKEN=your-weave-token

helm upgrade -i flux weaveworks/flux \
--namespace flux \
--set rbac.create=true \
--set git.path=cluster \
--set git.pollInterval=2m \
--set registry.pollInterval=30s \
--set registry.cacheExpiry=24h \
--set helmOperator.create=true \
--set helmOperator.createCRD=false \
--set helmOperator.chartsSyncInterval=2m \
--set git.url=ssh://git@github.com/weaveworks-experiments/istio-demo \
--set git.branch=${GIT_BRANCH} \
--set git.label=${GIT_TAG} \
--set token=${WC_TOKEN}
```

At startup Flux generates a SSH key and logs the public key. Find the SSH public key with:

```bash
kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
```

In order to sync your cluster state with GitHub you need to copy the public key and create a deploy key 
with write access on your GitHub repository. Go to _Settings > Deploy keys_ click on _Add deploy key_, check 
_Allow write access_, paste the Flux public key and click _Add key_.

### GitOps: Istio canary deployments 

Canary demo:

* Open the podinfo URL and it should display the `1.3.0` version
* In Weave Cloud Deploy do a manual release of `quay.io/stefanprodan/podinfo:1.3.1` 
* The podinfo UI will automatically refresh and display the `1.3.1` version

Automated demo:

* In Weave Cloud Deploy enable automation for `test:deployment/podinfo`
* Create a Flux filter of type semver `~1.3`
