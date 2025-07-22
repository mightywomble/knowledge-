# Using Rancher to deploy Kubernetes

## Scope

This document is provided as an example for a PoC to deploy Kubernetes (k8s) clusters on Cudo Compute using Rancher as a Deployment/Management Interface. the outcome of this should be something which is as automated as it can be, and if possible utilising deployment pipelines (Github Actions) where possible.

### **What is Rancher?**

Rancher is a container management platform that can help teams manage Kubernetes clusters and containerized applications. Here are some reasons why you might use Rancher:

* **Centralized management:** Rancher provides a single interface for managing clusters, which can simplify deployment, monitoring, and maintenance. It also centralizes authentication and role-based access control (RBAC).
* **Infrastructure versatility:** Rancher can be used to provision Kubernetes on any infrastructure or cloud, and it supports existing Kubernetes clusters.
* **Pre-configured applications:** Rancher has a catalog of pre-configured applications that can be easily deployed.
* **Resource management:** Rancher offers tools for monitoring performance, logging, and alerts to help ensure optimal resource usage.
* **DevOps tools:** Rancher provides a user interface for DevOps engineers to manage their application workload, and it comes with a catalog of DevOps tools.
* **Cloud-native ecosystem products:** Rancher is certified with a wide selection of cloud native ecosystem products, including security tools, monitoring systems, container registries, and storage and networking drivers.
* **Projects:** Rancher provides a construct called “projects” that group namespaces together to provide a single point of control.
*   **Kubernetes version selection**

    Rancher Desktop allows you to select which Kubernetes version you want to use, which can help your local Kubernetes cluster match the one running in production.

## Rancher Installation

### Server Requirements

The Rancher server is a static server which runs on docker. The deployment of this at this stage does not have any benifit from being automated. this may change moving forwad.

| Item    | Min Value | Notes                                                                                           |
| ------- | --------- | ----------------------------------------------------------------------------------------------- |
| OS      | Debian12  | Will also deploy on Ubuntu 22.04 and above                                                      |
| RAM     | 8Gb       | 12Gb                                                                                            |
| vCPU    | 2         | 4 is preferred                                                                                  |
| Disk    | 50Gb      | 100Gb                                                                                           |
| Network | Default   | Will investigate using the netbird VLAN with this to reduce traffic directly over the internet. |

### Rancher: Setup Server

**Run as ROOT**

Once the Debian server is running, run the following commands

```jsx
apt update && apt upgrade -y
```

#### Docker Install

Setup the environment

```jsx
apt update
apt install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL <https://download.docker.com/linux/debian/gpg> -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \\
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] <https://download.docker.com/linux/debian> \\
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \\
   tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
```

Install docker

```jsx
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

it is recommended to a needed config to sysctl

```
nano /etc/sysctl.d/99-k8s-cni.conf
```

Then add these lines to the file

```
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```

Finally, run the sysctl reload. As recommended by the README in sysctl.d dir.

```
service procps force-reload
```

This part of the server setup is complete, you can reboot, however its not essential.

### Rancher: Install

Rancher resides in the world of docker and as such is installed using the following command

```jsx
docker run -d --restart=unless-stopped \\
  -p 80:80 -p 443:443 \\
  --privileged \\
  -v /opt/rancher:/var/lib/rancher \\
  --name=rancher_server \\
  rancher/rancher:latest
```

To break this command down for some of the bits which could be changed.

the service will be available on these two ports

```jsx

 -p 80:80 -p 443:443 
```

any config will beheld in /opt/rancher on the host server

```jsx
-v /opt/rancher:/var/lib/rancher
```

We are going to make use of the latest published Rancher version

```jsx
 rancher/rancher:latest
```

### Rancher Login

Once the container has spun up, we need to find the One Time password, this is done using the command

```jsx
docker logs rancher_server 2>&1 | grep "Bootstrap Password:"
```

This should output a VERY long text string, copy it, we will need this on the next step

Open the Rancher interface

```jsx
https://<hostservers ip>
```

This will display the following page

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/b1f06df7-4fbf-439a-9f25-2b608c7f56d2/image.png)

Enter that long Bootstrap Password here and press **Login with Local User**

This will take you to the user setup screen

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/557e7df2-71a0-4c60-beae-8d2bf1d01d48/image.png)

I’d make a node of the random password and press continue (then save it in 1password)

This will take you to a screen similar to this

Note: yours will only have a local cluster

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/4ee940f8-b249-4751-8fa4-b77f1971f3e0/image.png)

## K8s Node Deployment

With rancher installed, in this scenario we are going to use Rancher to deploy a Kubernetes cluster to VM’s created on cudo compute using Terraform

[Terraform: Deploy Rancher/K8s Nodes to Cudo Compute](https://www.notion.so/Terraform-Deploy-Rancher-K8s-Nodes-to-Cudo-Compute-1736c1c8a719806887b9f447764ea920?pvs=21)

## Creating a new Cluster

At this point we have a Rancher instance installed and a set of nodes ready to install terraform on. The final stage is to create a cluster and deploy it.

### Rancher Interface

Login to Rancher and Click on **create**

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/d90a7aee-bde6-477b-80fb-dddefca3debc/image.png)

Click on Custom

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/b5f72e7f-e734-48a1-9b19-f0010e3549da/image.png)

This opens the Cluster Creation Page

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/6afe17d3-00af-4f91-977a-ebb4124c1828/image.png)

### **Cluster Options**

At this point, its possible to create the basic cluster by just adding a **Cluster Name**

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/32d21857-3d91-4653-b64b-bcd68947f1bc/image.png)

then clicking on Create

#### Other Options

#### → Container Networks (CNI)

There are several container networks infrastructure options available

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/6f4fc0a9-0923-4c58-a807-8eebb82130de/image.png)

#### → Kubernetes Version

there are several kubernetes versions available, these i belive are updated as the docker image is updated

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/0c98c7a3-2491-4551-be28-8fb4b4ad4213/image.png)

#### → Networking

Its good practice to setup service and cluster networks on a production system, usually using NAT 172. addresses

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/960bf5e1-8e9a-41cb-9132-686931977b29/image.png)

#### → Update Strategies

On a production system being able to smoothly update systems is useful, and can be controlled here.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/a599f1f6-41a0-4a44-9c54-9dbd5c5039a1/image.png)

#### → Warnings

The interface is pretty good at letting you know if something is more advanced

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/d6452589-5455-4ccf-b042-53901779192b/image.png)

#### → Cluster as Code

This will be useful moving forward, as rather than relying on the form for cluster creation, we can template and store as YAML files in git based on a cluster.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/1ec65dee-b023-41b4-862b-da9db31de85b/image.png)

### Deploy Cluster

Once the form has been complete and a name or the cluster provided click on **Create**

this will present the **registration** screen

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/28511cdd-feff-4ec1-9b59-71f5c5ad46cf/image.png)

There is a Strategy here, which is to deploy only what is needed to the nodes we created.

#### → Control Node

We built 1 control node with Terraform

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/5a9edad0-fb73-4333-a5b9-c14362873f7e/image.png)

Under **Node role** select **etcd** and **Control Plane**

deselect **Worker**

Under **step 2** tick **Insecure** (we have not setup any certs for Rancher at this point)

Copy the curl command

Ssh into the control node and paste and run the curl command

It will take about 1 minute to complete.

Its not done though (see below)

#### → Worker Node(s)

We built 3 worker nodes with Terraform, on each of these nodes do the following

Similar process as the control node

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/15396c3b-41a7-4522-b10d-bc8d33415fd0/image.png)

Under **Node role** select **Worker**

deselect **etcd** and **Control Plane**

Under **step 2** tick **Insecure** (we have not setup any certs for Rancher at this point)

Copy the curl command

Ssh into the each worker node and paste and run the curl command

It will take about 1 minute to complete.

Its not done though (see below)

### **The Cluster Build**

Once the Curl commands have neen run, the initial install is pretty quick, however in the back end the server is now building a Kubernetes cluster

This took about 5 minutes on servers with the above specs

The web Interface will keep updating and using the provisioning tab will also show output

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/4caa4fcf-c23e-40c7-acda-a9acae534438/image.png)

The process is

* Build the Control Node
* The worker Nodes wait
* The Control Node competes
* the Worker nodes attempt to communicate with the Control node and complete their setup

After about 5 (might be up to 15) minutes you will see the following

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/677080ef-0c45-4741-879a-796e60b2929f/image.png)

This Kubernetes cluster is now complete

## Using the Cluster

Although managed by Rancher this is a full K8s cluster and can be run as such

### Kubenetes Config File

The K8s deployment can now be used as any Kubernetes cluster, to download the config file click on the three dot menu in the corner and download the KubeConfig

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/76fda721-94ad-40be-a869-211d27f49076/image.png)

**Note: I’ve also highlighted where the YAML for this config can be downloaded. this is useful as it can be used to define builds and redeploy the same build for someone else or in a test environment**

## Final Notes

These are my PoC Learnings, and as such there are many next steps, listed below. this is a live document and will be updated over time.
