# Deploy BinderHub on top of Kubernetes on Jetstream 2

BinderHub is a kubernetes-based cloud service that allows users to share reproducible interactive computing environments from code repositories. See for example [MyBinder](https://mybinder.org){target=_blank}.

When pointed to a Github repository, it builds a Docker container from the metadata in that repository (for example a `requirements.txt` file), then it gives the user a live Jupyter Notebook session with this custom computing environment and the checked-out repository. The user can then browse the repository and execute Notebooks.

On top of Kubernetes, it also needs a Container Registry to store the Docker containers it builds, in the following tutorial we rely on Google Cloud.

## Setup Kubernetes

The first step is to have a Kubernetes deployment on Jetstream 2, for example deployed with Kubespray using [this tutorial](https://www.zonca.dev/posts/2022-03-30-jetstream2_kubernetes_kubespray){target=_blank}

## Enable the Google Container registry

We first need to activate the Google Container Registry in the Google Cloud account:

* Go to the [API Library](https://console.cloud.google.com/apis/library){target=_blank}
* Search for Google Container Registry API
* Click on Enable
* It will ask to configure billing
* It will ask for credit card information, but you will have $300 free credits for 90 days and the card won't be charged unless you manually switch to a paid account
* Go to the [Google Container registry settings](https://console.cloud.google.com/gcr/settings){target=_blank}, set the images to public, so that they can be pulled back from BinderHub to the container dedicated to execution

## Setup permissions on the Google Container Registry

Follow [the instructions on "Zero to BinderHub" to setup the Google Container Registry](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/setup-registry.html#set-up-google-container-registry){target=_blank}
some notes:

* Better to create a dedicated project just for BinderHub
* "Create key" is under "Actions"->"Manage keys"

## Setup ingress with HTTPS support

On Jetstream 2 we do not have a Load Balancer service like public clouds, therefore we need to use Ingress.

First we need to setup `Cert-Manager` to provide HTTPS certificates via Letsencrypt.

Follow this [other tutorial](https://zonca.dev/2020/03/setup-https-kubernetes-letsencrypt.html){target=_blank} to install a cluster issuer.

## Configure the domains

BinderHub requires 2 subdomains, one for BinderHub and one for JupyterHub.
For the initial testing I recommend to use Jetstream 2 provided domains, once they are working it is easy to replace them with custom domains.

Login to [the Jetstream 2 Horizon instance](https://js2.jetstream-cloud.org/){target=_blank}, choose the right allocation, click on DNS, Zones, choose the Zone which starts with `tg`, then click on "Create Record Set".

Create a "A" record where name is `binder.tg-xxx000000.projects.jetstream-cloud.org` and record is the IP of your master instance. Create a second A record with the name `hub.tg-xxx000000.projects.jetstream-cloud.org` and the same record.

These should be ready to be used in a few minutes.

## Install BinderHub with Helm

Checkout the repository with the configuration files:

    git clone https://github.com/zonca/jupyterhub-deploy-kubernetes-jetstream/

Enter the `binderhub` folder.

Follow the [instructions on how to create just the `secret.yaml` file](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/setup-binderhub.html){target=_blank}.

The repository provides a template named `config_template.yaml` to create `config.yaml`, this file contains all of the configuration items needed to setup the following items, see the links for the relevant documentation:

* [BinderHub with Google Container Registry](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/setup-binderhub.html){target=_blank}
* [Configure HTTPS ingress for BinderHub and JupyterHub](https://binderhub.readthedocs.io/en/latest/https.html){target=_blank}
* [Use Docker-in-Docker to build the containers](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/setup-binderhub.html#use-docker-inside-docker-dind){target=_blank}, which is necessary because the Jetstream 2 Kubernetes deployment uses `containerd`.

Create a file named `config_vars.sh` in the same folder which defines your Jetstream allocation and Google Project IDs:

    export ALLOCATION=xxx000000
    export GOOGLE_PROJECT=binderhub-000000

Then deploy BinderHub running:

    bash install_binderhub.sh

This script first creates `config.yaml` (and overwrites it, so keep your changes in `config_template.yaml`), then deploys or updates the deploy of BinderHub via Helm.

## Test the deployment

Connect to `https://binder.tg-xxx000000.projects.jetstream-cloud.org`, you should see the binder login page:

![Binder login page](../../images/binderloginpage.png)

You can then test with one of the binder examples:

`https://binder.tg-xxx000000.projects.jetstream-cloud.org/v2/gh/binder-examples/requirements/master`

This should build the container using the Docker-in-Docker pod, push it to the registry on Google Cloud, then spawn a Jupyter Notebook instance inside JupyterHub and redirect you to `https://hub.tg-xxx000000.projects.jetstream-cloud.org>.
