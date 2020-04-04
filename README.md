# IBM Cloud Microk8s Setup

[Microk8s](https://microk8s.io/) is a light-weight, single node, Kubernetes cluster. One of the add-ons they provide is [Kubeflow](https://www.kubeflow.org/), which is great for any machine learning project you might be working on. Here I'm going to walk through setting up a small VM (big enough to run everything but small enough not to complete destroy your wallet) and help you get familiar with how everything works at a high level.

### Provisioning a Virtual Machine

I recommend getting a public, multi-tenant, variable compute, 4vCPU / 8GB RAM Ubuntu 18.04 LTS VM with 100GB SAN boot disk. Also be sure to select hourly billing (in case you don't need it for the full month they won't keep charging you). In total this will come to about $0.10 per hour (or about $70 a month). Be sure to grab the IP address & password for the VM once it's up and running. The IP address can be found on the "monitoring" tab on the left and the password can be found under the "password" tab.

### Install Microk8s & Kubeflow

Once your VM is up and running SSH into it and running the following commands and be sure to replace `<ip-address>` with the IP for your VM and `<kf-admin-password>` with whatever you want the admin password to be for your Kubeflow cluster.

```sh
apt-get update
sudo snap install microk8s --classic --channel=1.18/stable
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
su - $USER
microk8s status --wait-ready
microk8s.enable dns dashboard storage
export KUBEFLOW_HOSTNAME=<ip-address>.xip.io
export KUBEFLOW_AUTH_PASSWORD=<kf-admin-password>
microk8s.enable kubeflow
```

This might take a little while.
