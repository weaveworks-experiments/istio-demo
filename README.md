# Weave Cloud Istio demo

Prerequisites:

* GKE > 1.10.0 
* CloudDNS
* Istio > 1.0.0
* Helm and Tiller
* Weave Cloud agents

Setup Istio with Let's Encrypt wildcard certs on Google Cloud by following this [guide](https://github.com/stefanprodan/istio-gke).

### Install Flagger

Flagger is a Kubernetes operator that automates the promotion of canary deployments
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

### GitOps: Istio canary deployments 

Prerequisites:

* Create a new branch from master
* Edit the `workloads/canary.yaml` and change the `podinfo.example.com` with your own domain
* Push the changes to your branch
* Go to Weave Cloud Deploy setup page and add this repo with your own branch
* Go to Weave Cloud web hooks setup page and add this repo

Canary demo:

* Open the podinfo URL and it should display the `1.3.0` version
* In Weave Cloud Deploy do a manual release of `quay.io/stefanprodan/podinfo:1.3.1` 
* The podinfo UI will automatically refresh and display the `1.3.1` version

Automated demo:

* In Weave Cloud Deploy enable automation for `test:deployment/podinfo`
* Create a Flux filter of type semver `~1.3`
