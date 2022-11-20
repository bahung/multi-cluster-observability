# Multi-cluster-observability
This document show how to configure and install multi-cluster obsavability solution using Grafana, Prometheus, Thanos, and Loki.
# Background
- We want to set up a central observability solution for all of three clusters in AWS VPC.
- We assume the three clusters' names are development, pre-production, and production.
- The central observer will be installed in the development cluster. It will also monitor the development cluster itself.
# Prerequisites
- IAM with creating role, policy privilege
- Helm
- Helm chart of kube-prometheus, available at [Link](https://github.com/bahung/helm-aws-prometheus-stack)
- Helm chart of thanos, available at [Link](https://github.com/bahung/helm-thanos)
- Helm chart of envoy proxy, available at [Link](https://github.com/bahung/helm-envoy-proxy)
- Helm chart of Grafana-loki
- EKS
- kubectrl
# Steps
Following the steps below:
- [Individual cluster config and installation](./individual-cluster-config.md) (Loki will be updated later.)

- [Observer cluster config and installation](./obeserver-config.md) (Loki will be updated later.)