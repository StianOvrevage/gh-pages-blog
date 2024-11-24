---
title:  "Managed Kubernetes on Microsoft Azure (English)"
description: "In this post we will set up a managed Kubernetes cluster from scratch using Azure CLI."
date:   2017-12-29 00:00:00 +0000
image: 2017-12-23-managed-kubernetes-on-azure-eng.png
categories:
    - technology
tags: 
    - kubernetes
    - azure
excerpt_separator: <!--more-->
---

_A few days ago I wrote a walkthrough of [setting up Azure Container Service (AKS) in Norwegian][33]. Someone asked me for an English version of that, and here it is._

Kubernetes(K8s) is becoming the de-facto standard for deploying container-based applications and workloads. Microsoft is currently in preview of their managed Kubernetes offering (Azure Kubernetes Service, AKS) which makes it easy to create a Kubernetes cluster and deploy workloads without the skill and time required to manage day-to-day operations of a Kubernetes-cluster, which today can be complex and time consuming.

In this post we will set up a Kubernetes cluster from scratch using Azure CLI.

<!--more-->

Table of contents

  - [Background](#Background)
    - [Docker containers](#Dockercontainers)
    - [Container orchestration](#Containerorchestration)
  - [Getting started with Azure Kubernetes - AKS](#QuickstartAKS)
    - [Caveats](#Caveats)
    - [Preparations](#Preparations)
    - [Azure login](#AzureLogin)
    - [Activate ContainerService](#ActivateContainerService)
    - [Create a resource group](#CreateResourceGroup)
    - [Create a Kubernetes cluster](#CreateK8sCluster)
    - [Install kubectl](#InstallKubectl)
    - [Inspect cluster](#InspectCluster)
    - [Start some nginx containere](#StartNginx)
    - [Making nginx available with a service](#NginxService)
    - [Scale cluster](#ScaleCluster)
    - [Delete cluster](#DeleteCluster)
  - [Bonus material](#Bonusmaterial)
    - [Deploying services with Helm](#HelmIntro)
      - [Deploy MineCraft with Helm](#HelmMinecraft)
    - [Kubernetes Dashboard](#KubernetesDashboard)
  - [Conclusion](#Conclusion)
  
#### Microsoft Azure

> [If you don't have a Azure subscription already you can try services for $200 for 30 days.][24] The VM size **Standard_B2s** is Burstable, has 2vCPU, 4GB RAM, 8GB temp storage and costs roughly $38 / month. For $200 you can have a cluster of 3-4 B2s nodes plus traffic, loadbalancers and other additional costs.

> _We have no affiliation with Microsoft Azure except their sponsorship of our startup [DataDynamics][26] with cloud services for 24 months in their [BizSpark program][25]._

<a id="Background"></a>
## Background

<a id="Dockercontainers"></a>
###  Docker containers

_We will not do a deep dive on Docker containers in this post, but here is a summary for those who are not familiar with it._

Docker is a way to package software so that it can run on the most popular platforms without worrying about installation, dependencies and to a certain degree, configuration.

In addition, a Docker container uses the operating system of the host machine when it runs. Because of this it's possible to run many more containers on the same host machine compared to running virtual machines.

Here is a incomplete and rough comparison between a Docker container and a virtual machine:

|                     | Virtual machine            | Docker container              |
| ------------------- | --------------             | --------------------          |
| Image size          | from 200MB to many GB      | from 10MB to 3-400MB          |
| Startup time        | 60 seconds +               | 1-10 seconds                  |
| Memory usage        | 256MB-512MB-1GB +          | 2MB +                         |
| Security            | Good isolation between VMs | Not as good isolation between containers |
| Building image      | Minutes                    | Seconds                       |

> **PS** The numbers for virtual machines is taken from memory. I tried starting a MySQL virtual appliance on my laptop but VMware Player refuses to run because of Windows Hyper-V incompatibility. VMware Workstation refuses to run because of license issues and Oracle VirtualBox repeatedly gives me a nasty bluescreen. Hooray!

> **Protip** The smallest and fastest Docker images are built on Alpine Linux. For the webserver Nginx the Alpine-based image is 15MB compared to 108MB for the normal Debian-based image. PostgreSQL:Alpine is 38MB compared to 287MB with "full" OS. Last version of MySQL is 343MB but will in version 8 support Alpine Linux as well.

To recap, some of the advantages of Docker containers are:
 - Compatibility across platforms, Linux, Windows, MacOS.
 - 10-100x smaller size. Faster to download, build and upload.
 - Memory usage only for application and not base OS.
   - Advantage when developing. Ability to run 10-20-30 containers on a development laptop.
   - Advantage in production. Can reduce hardware/cloud costs considerably.
 - Near instant startup. Makes dynamic scaling of applications easier.

[Download Docker for Windows here.][27]

To start a MySQL database container from Windows CMD or Powershell:

```
docker run --name mysql -p 3306:3306 -e MYSQL_RANDOM_ROOT_PASSWORD=true mysql
```

Stop the container with:

```
docker kill mysql
```

You can search for already built Docker images on [Docker Hub][28]. It's also possible to create private Docker repositories for your own software that you don't want to be publicly available.

<a id="Containerorchestration"></a>
#### Container orchestration

Now that Docker container images has become the preferred way to package and distribute software on the Linux platform, there has emerged a need for systems to coordinate running and deploying these containers. Similar to the ecosystem of products VMware has built up around development and operation of virtual machines.

Container orchestration systems have the responsibility for:
 - Load balancing.
 - Service discovery.
 - Health checks.
 - Automatic scaling and restarting of host nodes and containers.
 - Zero downtime upgrades (rolling deploys).

Until recently the ecosystem around container orchestration has been fragmented, and the most popular alternatives have been:
 - [Kubernetes][4] (Originaly from Google, now managed by CNCF, the Cloud Native Computing Foundation)
 - [Swarm][2] (From the maker of Docker)
 - [Mesos][3] (From Apache Software Foundation)
 - [Fleet][1] (From CoreOS)

But the last year there has been a convergence towards Kubernetes as the preferred solution.

  - 7 February
    * [CoreOS announces that they are removing Fleet from Container Linux and recommends Kubernetes][5]
  - 27 July
    * [Microsoft joins the CNCF][9]
  - 9 August
    * [Amazon Web Services join the CNCF][7]
  - 29 August
    * [VMware and Pivotal joins the CNCF][10]
  - 17 September
    * [Oracle joins the CNCF][8]
  - 17 October
    * [Docker announces native support for Kubernetes in addition to it's own Swarm product][6]
  - 24 October
    * [Microsoft Azure announces the managed Kubernetes service AKS][13]
  - 29 November
    * [Amazon Web Services announces the managed Kubernetes service EKS][12]

Especially the last two news items are important. Deploying and running your own Kubernetes-installation requires time and skills ([Read how Stripe used 5 months to trust running Kubernetes in production, just for batch jobs.][15])

Until now the choice has been running your own Kubernetes cluster or using Google Container Engine which has been [using Kubernetes since 2014][14]. Many of us feel a certain discomfort by locking ourselves to one provider. But this is now changing when you can develop infrastructure on Kubernetes and choose between the 3 large cloud providers in addition to running your own cluster if wanted. **\***

**\*** Kubernetes is a fast moving project, and features might be available on the different platforms on different timelines.

<a id="QuickstartAKS"></a>
## Getting started with Azure Kubernetes - AKS

<a id="Caveats"></a>
### Caveats

> This guide is based on the documentation on [Microsoft.com][16]. Setting up a Azure Kubernetes cluster did not work in the beginning of December, but today, 23. December, it seems to work fairly well. But, upgrading the cluster from Kubernetes 1.7 to 1.8 for example does NOT work.
> 
> AKS is in Preview and Azure are working continuously to make AKS stable and to support as many Kubernetes-features as possible. Amazon Web Services has a similar closed invite-only Preview currently while working on stability and features.
> 
> Both Azure and AWS expresses expectations about their Kubernetes offerings will be ready for production in 2018.

<a id="Preparations"></a>
### Preparations

You need Azure-CLI (version 2.0.21 or newer) to execute the `az` commands:
  * [Download Azure-CLI here][17]
  * [Information about Azure-CLI on MacOS and Linux here][18]
  
All commands executed in Windows PowerShell.

<a id="AzureLogin"></a>
### Azure login

Log on to Azure:
```
az login
```
You will get a link to open in your browser together with an authentication code. Enter the code on the webpage and `az login` will save the login information so that you will not have to authenticate again on the same machine.

> **PS** The login information gets saved in `C:\Users\Username\.azure\`. You have to make sure nobody can access these files. They will then have full access to your Azure account.

<a id="ActivateContainerService"></a>
### Activate ContainerService

Since AKS is in Preview/Beta, you explicitly have to activate it in your subscription to get access to the `aks` subcommands.

```
az provider register -n Microsoft.ContainerService
az provider show -n Microsoft.ContainerService
```

<a id="CreateResourceGroup"></a>
### Create a resource group

Here we create a resource group named "my_aks_rg" in Azure region West Europe.

```
az group create --name my_aks_rg --location westeurope
```

> **Protip** 
> To see a list of all available Azure regions, use the command `az account list-locations --output table`. **PS** AKS might not be available in all regions yet!

<a id="CreateK8sCluster"></a>
### Create Kubernetes cluster

```
az aks create --resource-group my_aks_rg --name my_cluster --node-count 3 --generate-ssh-keys --node-vm-size Standard_B2s --node-osdisk-size 128 --kubernetes-version 1.8.2
```

* `--node-count`
  - Number of agent(host) nodes available to run containers
* `--generate-ssh-keys`
  - Creates and prints a SSH key which can be used for SSHing directly to the agent nodes.
* `--node-vm-size`
  - Which size Azure VMs the agent nodes should be created as. To see available sizes use `az vm list-sizes -l westeurope --output table` and [Microsofts webpages][19].
* `--node-osdisk-size`
  - Disk size of the agent nodes in GB. **PS** Containers can be stopped and moved to another host if Kubernetes finds it necessary or if a agent node disappears. All data saved locally in the container will be gone. If saving data permanently use Kubernetes PersistentVolumes and not the local agent node or container disks.
* `--kubernetes-version`
  - Which Kubernetes version to install. Azure does NOT necessarily install the last version by default, and currently upgrading with `az aks upgrade` does not work. Latest version available right now is 1.8.2. It's recommended to use the latest available version since there is a lot of changes from version to version. The documentation is also much better for newer versions.

Save the output of the command in a file in a secure location. It contains keys that can be used to connect to the cluster with SSH. Even though that should not in theory be necessary.

<a id="InstallKubectl"></a>
### Install kubectl

`kubectl` is the client which performs all operations against your Kubernetes cluster. Azure CLI can install `kubectl` for you:

```
az aks install-cli
```

After `kubectl` is installed we need to get login information so that `kubectl` can communicate with the Kubernetes cluster.

```
az aks get-credentials --resource-group my_aks_rg --name my_cluster
```
The login information is saved in `C:\Users\Username\.kube\config`. Keep these files secure as well.

> **Protip** When you have several Kubernetes clusters you can change which one `kubectl` talks to with `kubectl config get-contexts` and `kubectl config set-context my_cluster`.

<a id="InspectCluster"></a>
### Inspect cluster

To check that the cluster and `kubectl` works we start with a couple of commands.

See all agent nodes and status:
```
> kubectl get nodes
NAME                       STATUS    AGE       VERSION
aks-nodepool1-16970026-0   Ready     15m       v1.8.2
aks-nodepool1-16970026-1   Ready     15m       v1.8.2
aks-nodepool1-16970026-2   Ready     15m       v1.8.2
```

See all services, pods and deployments:
```
> kubectl get all --all-namespaces

NAMESPACE     NAME                                          READY     STATUS    RESTARTS   AGE
kube-system   po/kubernetes-dashboard-6fc8cf9586-frpkn      1/1       Running   0          3d

NAMESPACE     NAME                          CLUSTER-IP     EXTERNAL-IP     PORT(S)           AGE
kube-system   svc/kubernetes-dashboard      10.0.161.132   <none>          80/TCP            3d

NAMESPACE     NAME                             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-system   deploy/kubernetes-dashboard      1         1         1            1           3d

NAMESPACE     NAME                                    DESIRED   CURRENT   READY     AGE
kube-system   rs/kubernetes-dashboard-6fc8cf9586      1         1         1         3d
```

This is just some of the output from this command. You do not have to know what the resources in the `kube-system` namespace does. That is part of the intention when Microsoft is managing our cluster for us.

> **Namespaces**
> In Kubernetes there is something called Namespaces. Resources in one namespace does not have automatic access to resources in another namespace. The services that runs Kubernetes itself use the namespace `kube-system`. The `kubectl` command by default only shows you resources in the `default` namespace, unless you specify `--all-namespaces` or `--namespace=xx`.


<a id="StartNginx"></a>
### Start some nginx containers

> An instance of a running container in Kubernetes is called a **Pod**.

> `nginx` is a fast and flexible web server.

Now that the clsuter is up we can start rolling out services and deployments on it.

Lets start with creating a Deployment consiting of 3 containers all running the `nginx:mainline-alpine` image from [Docker hub][31].

**nginx-dep.yaml** looks like this:
```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:mainline-alpine
        ports:
        - containerPort: 80
```

Load this into the cluster with `kubectl create`:

```
kubectl create -f https://raw.githubusercontent.com/StianOvrevage/gh-pages-blog/master/static/attachments/2017-12-23-managed-kubernetes-on-azure/nginx-dep.yaml
```

This command creates the resources described in the file. `kubectl` can read files either from your local disk or from a web URL.

> After making changes to a resource definition (`.yaml` file), you can update the resources in the cluster with `kubetl replace -f resource.yaml`.

We can verify that the Deployment is ready:

```
> kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           10m
```

We can also get the actual Pods that are running:

```
> kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-569477d6d8-dqwx5   1/1       Running   0          10m
nginx-deployment-569477d6d8-xwzpw   1/1       Running   0          10m
nginx-deployment-569477d6d8-z5tfk   1/1       Running   0          10m
```

> **Logger** We can view logs from one pod with `kubectl logs nginx-deployment-569477d6d8-xwzpw`. But since we in this case don't know which Pod ends up getting an incomming request we can view logs from all the Pods which have `app=nginx` label: `kubectl logs -lapp=nginx`. The use of `app=nginx` is our choice in `nginx-dep.yaml` when we configured `spec.template.metadata.labels: app: nginx`.

<a id="NginxService"></a>
### Making nginx available with a service

To send traffic to our new Pods we need to create a **Service**. A service consists of one or more Pods which are chosen based on different criteria, for example which labels they have and whether the Pods are Running and Ready.

Lets create a service which forwards traffic to all Pods with label `app: nginx` and are listening to port 80. In addition we make the service available via a LoadBalancer:

**nginx-svc.yaml** looks like this:

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx
```

We tell Kubernetes to create our service with `kubectl create` as usual:

```
kubectl create -f https://raw.githubusercontent.com/StianOvrevage/gh-pages-blog/master/static/attachments/2017-12-23-managed-kubernetes-on-azure/nginx-svc.yaml
```

We can then wait and see which IP-address Azure assigns our service:
```
> kubectl get svc -w
NAME         CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
nginx        10.0.24.11   13.95.173.255   80:31522/TCP   15m
```

> **PS** It can take a few minutes for Azure to allocate and assign a Public IP for us. In the mean time `<pending>` will appear under EXTERNAL-IP.

A simple **Welcome to nginx** webpage should now be available on http://13.95.173.255 (_remember to replace with your own External-IP_).

We can also delete the service and deployment afterwards:
```
kubectl delete svc nginx
kubectl delete deploy nginx-deployment
```

<a id="ScaleCluster"></a>
### Scaling the cluster

If we want to change the number of agent nodes running Pods we can do that via Azure-CLI:
```
az aks scale --name my_cluster --resource-group my_aks_rg --node-count 5
```

> Currently all nodes will be created with the same size as when we created the cluster. AKS will probably get support for [**node-pools**][20] next year. That will allow for creating different groups of nodes with different size and operating systems, both Linux and Windows.


<a id="DeleteCluster"></a>
### Delete cluster

You can delete the whole cluster like this:
```
az aks delete --name my_cluster --resource-group my_aks_rg --yes
```

<a id="Bonusmaterial"></a>
## Bonus material

Here is some bonus material if you want to go a bit further with Kubernetes.

<a id="HelmIntro"></a>
### Deploying services with Helm

[Helm][21] is a package manager and library of software that is ready to be deployed on a Kubernetes cluster.

Start by downloading the [Helm-client][22]. It will read login information etc. from the same location as `kubectl` automatically.

Install the Helm-server (**Tiller**) on the Kubernetes cluster and update the package library:

```
helm init
helm repo update
```

See available packages (**Charts**) with `helm search`.

<a id="HelmMinecraft"></a>
#### Deploy MineCraft with Helm

Lets deploy a MineCraft server installation on our cluster, just because we can :-)

```
helm install --name stians --set minecraftServer.eula=true stable/minecraft
```

> `--set` overrides one or more of the standard values configured in the package. The MineCraft package is made in a way where it does not start without accepting the user license agreement by setting the variable `minecraftServer.eula`. All the variables that can be set in the MineCraft package are [documented here][23].

Then we wait for Azure to assign us a Public IP:
```
> kubectl get svc -w
stians-minecraft   10.0.237.0   13.95.172.192   25565:30356/TCP   3m
```

Now we can connect to our MineCraft server on `13.95.172.192:25565`!

![Kubernetes in MineCraft on Kubernetes](minecraft-k8s.png)

<a id="KubernetesDashboard"></a>
### Kubernetes Dashboard

Kubernetes also has a graphic web user-interface which makes it a bit easier to see which resources are in the cluster, view logs and even open a remote shell inside a running Pod, among other things.

```
> kubectl proxy
Starting to serve on 127.0.0.1:8001
```

`kubectl` encrypts and tunnels the traffic to the Kubernetes API servers. The dashboard is available on [http://127.0.0.1:8001/ui/][32].

![Kubernetes Dashboard](k8s-dash.png)

<a id="Conclusion"></a>
## Conclusion

I hope you enjoy Kubernetes as much as I have. The learning curve can be a bit steep in the beginning, but it does not take long before you are productive.

Look at the [official guides on Kubernetes.io][29] to learn more about defining different types of resources and services to run on Kubernetes. **PS: There are big changes from version to version so make sure you use the documentation for the correct version!**

Kubernetes also have a very active Slack-community on [kubernetes.slack.com][30] that is worthwhile to check out.


[1]: https://github.com/coreos/fleet
[2]: https://docs.docker.com/engine/swarm/
[3]: http://mesos.apache.org/
[4]: https://kubernetes.io/
[5]: https://coreos.com/blog/migrating-from-fleet-to-kubernetes.html
[6]: https://www.theregister.co.uk/2017/10/17/docker_ee_kubernetes_support/
[7]: https://techcrunch.com/2017/08/09/aws-joins-the-cloud-native-computing-foundation/
[8]: https://techcrunch.com/2017/09/13/oracle-joins-the-cloud-native-computing-foundation-as-a-platinum-member/
[9]: https://azure.microsoft.com/en-us/blog/announcing-cncf/
[10]: https://www.geekwire.com/2017/now-vmware-pivotal-cncf-becoming-hub-enterprise-tech/
[11]: https://cloudplatform.googleblog.com/2014/11/unleashing-containers-and-kubernetes-with-google-compute-engine.html
[12]: https://aws.amazon.com/blogs/aws/amazon-elastic-container-service-for-kubernetes/
[13]: https://azure.microsoft.com/en-us/blog/introducing-azure-container-service-aks-managed-kubernetes-and-azure-container-registry-geo-replication/
[14]: https://cloudplatform.googleblog.com/2014/11/unleashing-containers-and-kubernetes-with-google-compute-engine.html
[15]: https://stripe.com/blog/operating-kubernetes
[16]: https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough
[17]: https://aka.ms/InstallAzureCliWindows
[18]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest
[19]: https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes
[20]: https://cloud.google.com/kubernetes-engine/docs/concepts/node-pools
[21]: https://helm.sh/
[22]: https://github.com/kubernetes/helm/releases
[23]: https://github.com/kubernetes/charts/blob/master/stable/minecraft/values.yaml
[24]: https://azure.microsoft.com/en-us/free/
[25]: https://bizspark.microsoft.com/
[26]: http://www.datadynamics.no/
[27]: https://store.docker.com/editions/community/docker-ce-desktop-windows
[28]: https://hub.docker.com/
[29]: https://v1-8.docs.kubernetes.io/docs/tutorials/
[30]: http://slack.k8s.io/
[31]: https://hub.docker.com/r/_/nginx/
[32]: http://127.0.0.1:8001/ui/
[33]: /2017/12/25/managed-kubernetes-on-azure.html
