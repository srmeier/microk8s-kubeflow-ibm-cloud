# IBM Cloud Microk8s Setup

[Microk8s](https://microk8s.io/) is a light-weight, single node, Kubernetes cluster. One of the add-ons they provide is [Kubeflow](https://www.kubeflow.org/), which is great for any machine learning project you might be working on. Here I'm going to walk through setting up a small VM (big enough to run everything but small enough not to complete destroy your wallet) and help you get familiar with how everything works at a high level.

### Provisioning a Virtual Machine

I recommend getting a public, multi-tenant, variable compute, 4vCPU / 8GB RAM Ubuntu 18.04 LTS VM with 100GB SAN boot disk. Also be sure to select hourly billing (in case you don't need it for the full month they won't keep charging you). In total this will come to about $0.10 per hour (or about $70 a month). Be sure to grab the IP address & password for the VM once it's up and running. The IP address can be found on the "monitoring" tab on the left and the password can be found under the "password" tab.

> :warning: **Using variable compute**: One thing I've noticed is that with 4vCPU / 8GB RAM appears to be right at the limit for running Microk8s + Kubeflow. If possible I would go for a bit more compute. Otherwise, if the website becomes unresponsive reboot the VM and it should fix the issue.

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

Once Kubeflow is finished deploying you will be able to access the login page by going to: `<ip-address>.xip.io`. From here login with username "admin" and use the password which you specified during the installation process. The next thing we'll want to do is write a Dockerfile which we can use to deploy a simple Jupyter notebook server.

### Build a Jupyter Hub Docker Image

The Dockerfile I will be using to build the image is present in the repo, `DockerfileJupyter`. I would not recommend putting any credientials within this Dockerfile. Instead we will later create Kubernetes secrets and inject these secrets into the Jupyter pod using Kubeflow's pod defaults. Here we will inject a GitHub SSH key and credientials for accessing IBM's cloud object storage.

In this tutorial we'll use [Docker Hub](https://hub.docker.com/) to store our Docker image. Depending on what you plan to put in your image you may want to create a customer image repository. To push the image to Docker Hub let's first build the image locally and push the image to Docker Hub. Once in Docker Hub we can reference this image when building our Jupyter notebook using Kubeflow.

Create an account on Docker Hub and run the commands below to build and push your image. Just to emphasize, _do not push your image to Docker Hub if it contains sensitive information_. Also, in the code below replace `<username>` with the username for your Docker Hub account.

```sh
docker login -u <username>
docker build -f DockerfileJupyter -t <username>/kubeflow-notebook-img
docker push <username>/kubeflow-notebook-img
```

Now we have the Docker image ready that we will use to launch our Jupyter notebook from inside Kubeflow. However, before we do that let's setup our SSH Github secret and Cloud Object Storage secret.

[WIP]
