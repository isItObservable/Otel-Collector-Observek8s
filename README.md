# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## Episode : how to Observe your K8s cluster using OpenTelemetry
This repository contains the files utilized during the tutorial presented in the dedicated IsItObservable episode related to observing your k8s cluster using OpenTelemetry
What you will learn
* How to build a metric pipeline using the OpenTelemetry Collector
* Few useful OpenTelemetry collector Receivers, Processors



This repository showcase the usage of the OpenTelemtry Collector with :
* the HipsterShop
* K6 ( to generate load in the background)
* The OpenTelemetry Operator
* Nginx ingress controller
* Prometheus
* Loki
* Grafana

## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm

## Deployment Steps in GCP

You will first need a Kubernetes cluster with 2 Nodes.
You can either deploy on Minikube or K3s or follow the instructions to create GKE cluster:
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
    cloudtrace.googleapis.com \
    clouddebugger.googleapis.com \
    cloudprofiler.googleapis.com \
    --project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create onlineboutique \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```

### 3.Clone the Github Repository
```
git clone https://github.com/isItObservable/Otel-Collector-Observek8s.git
cd Otel-Collector-Observek8s
```
### 4.Deploy Nginx Ingress Controller with the Prometheus exporter
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
#### 1. get the ip adress of the ingress gateway
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* hipstershop
```
IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -ojson | jq -j '.status.loadBalancer.ingress[].ip')
```
#### 2. Update the manifest files

update the following files to update the ingress definitions :
```
sed -i "s,IP_TO_REPLACE,$IP," kubernetes-manifests/k8s-manifest.yaml
sed -i "s,IP_TO_REPLACE,$IP," grafana/ingress.yaml
```


### 5.Deploy OpenTelemetry Operator

#### 1. Cert-Manager
The OpenTelemetry operator requires to deploy the Cert-manager :
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```
#### 2. OpenTelemetry Operator
```
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

### 6.Deploy the demo application
```
kubectl create ns hipster-shop
kubectl apply -f kubernetes-manifests/k8s-manifest.yaml -n hipster-shop
```

### 7.Prometheus without any exporters

#### 1. Deploy Prometheus
 ```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack --set kubeStateMetrics.enabled=false --set nodeExporter.enabled=false --set kubelet.enabled=false --set kubeApiServer.enabled=false --set kubeControllerManager.enabled=false --set coreDns.enabled=false --set kubeDns.enabled=false --set kubeEtcd.enabled=false --set kubeScheduler.enabled=false --set kubeProxy.enabled=false --set sidecar.datasources.label=grafana_datasource --set sidecar.datasources.labelValue="1" --set sidecar.dashboards.enabled=true 

```
#### 2. Enable remote Writer
To be able to send the Collector metrics to Prometheus , we need to enable the remote writer feature.

To enable this feature we will need to edit the CRD containing all the settings of promethes: prometehus

To get the Prometheus object named use by prometheus we need to run the following command:
```
kubectl get Prometheus
```
here is the expected output:
```
NAME                                    VERSION   REPLICAS   AGE
prometheus-kube-prometheus-prometheus   v2.32.1   1          22h
```
We will need to add an extra property in the configuration object :
```
enableFeatures:
- remote-write-receiver
```
so to update the object :
```
kubectl edit Prometheus prometheus-kube-prometheus-prometheus
```
After the update your Prometheus object should look  like :
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  generation: 2
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 30.0.1
    chart: kube-prometheus-stack-30.0.1
    heritage: Helm
    release: prometheus
  name: prometheus-kube-prometheus-prometheus
  namespace: default
spec:
  alerting:
  alertmanagers:
  - apiVersion: v2
    name: prometheus-kube-prometheus-alertmanager
    namespace: default
    pathPrefix: /
    port: http-web
  enableAdminAPI: false
  enableFeatures:
  - remote-write-receiver
  externalUrl: http://prometheus-kube-prometheus-prometheus.default:9090
  image: quay.io/prometheus/prometheus:v2.32.1
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  podMonitorNamespaceSelector: {}
  podMonitorSelector:
  matchLabels:
  release: prometheus
  portName: http-web
  probeNamespaceSelector: {}
  probeSelector:
  matchLabels:
  release: prometheus
  replicas: 1
  retention: 10d
  routePrefix: /
  ruleNamespaceSelector: {}
  ruleSelector:
  matchLabels:
  release: prometheus
  securityContext:
  fsGroup: 2000
  runAsGroup: 2000
  runAsNonRoot: true
  runAsUser: 1000
  serviceAccountName: prometheus-kube-prometheus-prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
  matchLabels:
  release: prometheus
  shards: 1
  version: v2.32.1
```
#### 2. Deploy the Grafana ingress
```
kubectl apply -f grafana/ingress.yaml
```
#### 3. Get The Prometheus service
```
PROMETHEUS_SERVER=$(kubectl get svc -l app=kube-prometheus-stack-prometheus -o jsonpath="{.items[0].metadata.name}")
sed -i "s,PROMETHEUS_SERVER,$PROMETHEUS_SERVER," otelemetry/openTelemetry.yaml
```

### 8. Deploy Loki without any log agents
```
kubectl create ns loki
helm upgrade --install loki grafana/loki --namespace loki
kubectl wait pod -n loki -l  app=loki --for=condition=Ready --timeout=2m
LOKI_SERVICE=$(kubectl  get svc -l app=loki  -n loki -o jsonpath="{.items[0].metadata.name}")
sed -i "s,LOKI_TO_REPLACE,$LOKI_SERVICE," otelemetry/openTelemetry.yaml
```


### 9. Collector pipeline 

#### 1. Create the service Account
```
kubectl apply -f otelemetry/rbac.yaml
```
#### 2. Update Collector pipeline
```
CLUSTERID=$(kubectl get namespace kube-system -o jsonpath='{.metadata.uid}')
CLUSTERNAME="YOUR OWN NAME"
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," otelemetry/openTelemetry.yaml
sed -i "s,CLUSTER_NAME_TO_REPLACE,$CLUSTERNAME," otelemetry/openTelemetry.yaml
```
#### 3. Deploy the collector pipeline
```
kubectl apply -f otelemetry/openTelemetry.yaml
```






