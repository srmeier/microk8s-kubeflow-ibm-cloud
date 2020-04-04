# IBM Cloud Microk8s Setup

[Microk8s](https://microk8s.io/) is a light-weight, single node, Kubernetes cluster. One of the add-ons they provide is [Kubeflow](https://www.kubeflow.org/), which is great for any machine learning project you might be working on. Here I'm going to walk through setting up a small VM (big enough to run everything but small enough not to complete destroy your wallet) and help you get familiar with how everything works at a high level.

### Provisioning a Virtual Machine

I recommend getting a public, multi-tenant, variable compute, 4vCPU / 8GB RAM Ubuntu 18.04 LTS VM with 100GB SAN boot disk. Also be sure to select hourly billing (in case you don't need it for the full month they won't keep charging you). In total this will come to about $0.10 per hour (or about $70 a month).
