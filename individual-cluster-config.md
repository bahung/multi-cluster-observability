- [Introduction](#introduction)
- [Architecture](#architecture)
- [Kube Prometheus Stack](#kube-prometheus-stack)
  - [Create a new `shared-values.yaml`](#create-a-new-shared-valuesyaml)
  - [Configure Thanos side car using `thanos-config.yaml`](#configure-thanos-side-car-using-thanos-configyaml)
  - [Configure secret for sidecar](#configure-secret-for-sidecar)
  - [Create ingress to expose sidecar](#create-ingress-to-expose-sidecar)
  - [Configure dns name, https certificate, and external label for each cluster](#configure-dns-name-https-certificate-and-external-label-for-each-cluster)
- [Installation](#installation)

# Introduction
Using kube-prometheus stack toghether with Thanos, we will setup multi-cluster observability solution into AWS VPC composes of three EKS clusters. This document shows how to configure the monitored cluster.

# Architecture
Consider we have multiple kubernetes clusters (e.g. Cluster A, Cluster B etc.) that we would like to monitor (node, pod metrics etc.) from a central Observer cluster.

Below is a reference architecture in AWS showcasing how we could achieve it with Thanos:

# Kube Prometheus Stack
Kube Prometheus is an open source project that provides easy to operate end-to-end Kubernetes cluster monitoring with Prometheus using the Prometheus Operator.

It packages below components:

- Prometheus Operator
- Highly available Prometheus
- Prometheus Node Exporter
- Kube State Metrics
- other components such as grafana, alert manager and so forth.

For our setup, we will install Kube Prometheus Stack in each our clusters that we would like to monitor.

Since we are planning for monitoring from a central observer cluster, we do not want to install some tools e.g. grafana in each of the clusters but only the central observer cluster.

<!-- We make use of the helm chart repo [https://github.com/HarshadRanganathan/helm-aws-prometheus-stack] and override the values for easy installation. --->

## Create a new `shared-values.yaml`
- We would want to give the stack a shorter name so let’s override few helm variables in the chart to achieve that.
```sh
nameOverride: prometheus
fullnameOverride: prometheus
```
- Edit the compenents that we would like to have in the monitored clusters:
```sh
grafana:
  enabled: false

# disable prometheus service monitors for below components
kubelet:
  enabled: false
kubeControllerManager:
  enabled: false
coreDns:
  enabled: false
kubeApiServer:
  enabled: false
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
kubeProxy:
  enabled: false

# enable kubeStateMetrics and nodeExporter 
kubeStateMetrics:
  enabled: true
nodeExporter:
  enabled: true

# Prometheus operator settings. PO provides Kubernetes native deployment and management of Prometheus and related monitoring components
prometheusOperator:
  kubeletService:
    enabled: false
  resources:
    requests:
      cpu: 500m
      memory: 100Mi
    limits:
      cpu: 700m
      memory: 200Mi

# Prometheus settings
prometheus:
  serviceAccount:
    create: false
    name: prometheus  #This service account will be linked to an IAM role which has permissions to write the metric data to S3.
  prometheusSpec:
    scrapeInterval: "1m"
    scrapeTimeout: "60s"
    evaluationInterval: "1m"
    retention: 7d
    resources:
      requests:
        cpu: 200m
        memory: 1000Mi
      limits:
        cpu: 2500m
        memory: 20000Mi
    storageSpec:
      volumeClaimTemplate:
        metadata:
          name: prometheus
        spec:
          storageClassName: gp2
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 30Gi

    # thanos sidecar configure, refer to thanos-config.yaml
    thanos:
      baseImage: quay.io/thanos/thanos
      version: v0.18.0
      objectStorageConfig:
        name: thanos-objstore-config
        key: thanos.yaml

```

## Configure Thanos side car using `thanos-config.yaml`
Thanos Sidecar performs following actions:
- Serves as a sidecar container alongside Prometheus
- Uploads Prometheus chunks to an object storage
- Serves as an entry point for PromQL queries

File `thanos-config.yaml`
```sh
type: s3
config:
  bucket: prod-metrics-storage
  endpoint: s3.ap-northeast-1.amazonaws.com
  sse_config:
    type: "SSE-S3"
```

## Configure secret for sidecar
This secret stores sensitive info so that Thanos sidecar can get the object configure
```sh
kubectl -n platform create secret generic thanos-objstore-config --from-file=thanos.yaml=thanos-config.yaml
```

## Create ingress to expose sidecar
Add the below configurations to `shared-value.yaml`
```sh
  service:
    type: NodePort
  thanosIngress:
    enabled: true
    paths:
    - /*
    annotations:
      external-dns/zone: private
      kubernetes.io/ingress.class: alb
      alb.ingress.kubernetes.io/scheme: internal
      alb.ingress.kubernetes.io/backend-protocol-version: GRPC
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
      alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
```

## Configure dns name, https certificate, and external label for each cluster
Create `dev-prod-values.yaml` to provide development cluster specific values:
```sh
prometheus:
  thanosIngress:
    hosts:
    - clusterA.prometheus.example.net
    annotations:
      external-dns.alpha.kubernetes.io/hostname: clusterA.prometheus.example.net
      alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1::certificate/<arn>
      alb.ingress.kubernetes.io/load-balancer-attributes: access_logs.s3.enabled=true,access_logs.s3.bucket=,access_logs.s3.prefix=awselasticloadbalancing/clusterA
  # external lables to distinguish that our metrics are coming from several clusters
  prometheusSpec:
    externalLabels:
      cluster: clusterA
      stage: prod
      region: us-east-1

```

# Installation

```sh
helm upgrade -i prometheus . -n platform --values=shared-values.yaml --values=cluster-a-prod-values.yaml
```