# IBM Cloud Microk8s Setup

[Microk8s](https://microk8s.io/) is a light-weight, single node, Kubernetes cluster. One of the add-ons they provide is [Kubeflow](https://www.kubeflow.org/), which is great for any machine learning project you might be working on. Here I'm going to walk through setting up a small VM (big enough to run everything but small enough not to complete destroy your wallet) and help you get familiar with how everything works at a high level.

### Provisioning a Virtual Machine

I recommend getting a public, multi-tenant, 4vCPU / 16GB RAM Ubuntu 18.04 LTS VM with 100GB SAN boot disk. Also be sure to select hourly billing (in case you don't need it for the full month they won't keep charging you). In total this will come to about $0.21 per hour (or about $150 a month). Be sure to grab the IP address & password for the VM once it's up and running. The IP address can be found on the "monitoring" tab on the left and the password can be found under the "password" tab.

> :warning: **Using variable compute**: One thing I've noticed is that with 4vCPU / 8GB RAM appears to be right at the limit for running Microk8s + Kubeflow. If you plan on running any sort of workload I _highly_ recommend provisioning a larger VM.

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

Now we have the Docker image ready that we will use to launch our Jupyter notebook from inside Kubeflow. If you aren't planning to use cloud object storage or Git to store your notebooks you can stop here and start making use of Kubeflow!

### Creating Github & Cloud Object Storage Secrets

In order to inject the SSH keys into our Jupyter notebook pod we will need to create a Kuberenetes secret and Kubeflow PodDefault. We will do a similar step for injecting our cloud storage creds as well.

#### 1. Github SSH Secrets

To create the SSH secret first place the RSA keys on your VM by connecting over FTP. Once these files are on your server (can place them in the root directory) run the following command.

```sh
microk8s.kubectl create secret generic kf-ssh-secret --from-file=id_rsa=/root/id_rsa --from-file=id_rsa.pub=/root/id_rsa.pub --from-file=known_hosts=/root/known_hosts -n admin
```

If you currently do not have SSH files on your local machine you use to access GitHub you can create them by following this [tutorial](https://help.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh). Make sure this secret is created in the namespace of the user you plan to give access to it (i.e. here we assume that's the `admin` user). Once you have created the secret be sure to _delete_ the SSH key files from your root directory.

#### 2. Cloud Object Storage Secrets

If you don't already have an IBM Cloud Object Storage service setup go ahead and create one. Next create a new standard bucket, name it something reasonable, and pay attention to the location where this bucket will be created (`us-east` in my case). The default service credientials they create for you are no good because we require HMAC creds. Create a new service crediential with HMAC enabled. View these credentials after they have been created and write down the `access_key_id` & `secret_access_key`. These are the variables we will save as a new secret within our Microk8s cluster.

Update the `kf-cos-secret.yaml` file found in this repo with the `access_key_id` & `secret_access_key` with the credientials for your newly created bucket and run the command below to create a new secret (the YAML in this repo assumes the default username of `admin` and creates the secret in that namespace).

```sh
microk8s.kubectl apply -f kf-cos-secret.yaml
```

### Deploying PodDefaults

To automatically inject the secrets into the Jupyter notebook pod we must next create PodDefaults. Run the following commands to create these PodDefaults in the `admin` namespace.

```sh
microk8s.kubectl apply -f kf-ssh-poddefault.yaml
microk8s.kubectl apply -f kf-cos-poddefault.yaml
```

The YAMLs above can be found in this repo.

### Creating our Jupyter Notebook Server

Now we are ready to create out Jupyter notebook server. Make sure to use the custom Jupyter notebook image we created before, mine is `docker.io/srmeier/kubeflow-notebook-img`. You will also see now in the configuation section the ability to add our two PodDefaults. Let's do that.

![m8s_pic_02](https://stephenmeier.net/files/pics/m8s_pic_02.png)

Once your notebook is up and running open a terminal shell from the Jupyter Lab interface and run the following lines to finish setting up GitHub SSH.

```sh
eval $(ssh-agent -s)
chmod 600 ~/.ssh/id_rsa
ssh-add ~/.ssh/id_rsa
```

> :warning: **Secrets not being injected**: When using Microk8s I've noticed that sometimes the kuberenetes secrets are not injected properly when creating the Jupyter notebook's pod. If this happens you can manually upload the SSH keys and environmental variables.
