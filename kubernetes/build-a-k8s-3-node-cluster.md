# Build a K8s 3 Node Cluster

## Introduction

Everything needs to start somewhere, and having an environment to test/play on with Kubernetes is a good idea. There are third parties such as the hyperscalers who will provide you an environment which has ingress and persistent storage all setup for you. The information below is written however to create a local bare-metal test environment using Debian 12 on a Virtual platform (mine is on Proxmox).

_Note:_

_This should ideally be run as Ansible code, however I’ve put the manual steps here as I’ve not had the time to write the Ansible code._

## Build

### Requirements

3 Servers built and running with the following specs

| Item     | Description                            | Notes         |
| -------- | -------------------------------------- | ------------- |
| OS       | Debian 12                              |               |
| CPU/vCPU | 4                                      | 2 Minimum     |
| RAM      | 4Gb                                    | 2Gb Minimum   |
| Disk     | 100Gb                                  | 20Gb Minimum  |
| IP       | Static                                 | DHCP Reserved |
| Other    | Each node can ping each other nodes IP |               |

### Server Setup

For this environment there are 3 servers setup as follows

| Purpose       | Hostname      | IP          |
| ------------- | ------------- | ----------- |
| Master Node   | kube-master   | 10.10.0.100 |
| Worker Node 1 | kube-worker01 | 10.10.0.101 |
| Worker Node 2 | kube-worker02 | 10.10.0.102 |

### Base OS Setup

The following commands are used to get the base OS setup and communicating with local DNS, if you have an external DNS setup then use this. As this is a test environment I want the servers to be somewhat self sufficient.

#### Set Hostname

| server        | command                                               |
| ------------- | ----------------------------------------------------- |
| kube-master   | `sudo hostnamectl set-hostname "kube-master.local"`   |
| kube-worker01 | `sudo hostnamectl set-hostname "kube-worker01.local"` |
| kube-worker02 | `sudo hostnamectl set-hostname "kube-worker01.local"` |
|               |                                                       |

#### Set /etc/hosts

On all three nodes run

```jsx
sudo nano /etc/hosts
```

Add the following to the end of the file

```jsx
10.10.0.100    kube-master.local    kube-master
10.10.0.101    kube-worker01.local    kube-worker01
10.10.0.102    kube-worker02.local    kube-worker02
```

#### Disable Swap

Kubernetes isn’t a fan of linux swap, disable it on on all the nodes

```jsx
sudo swapoff -a
sudo sed -i '/ swap / s/^\\(.*\\)$/#\\1/g' /etc/fstab
```

#### Firewall

UFW or FirewallD isn’t installed or enabled by default, I have not run it on the test environment, if you want to run a firewall/ufw then run these commands. The rest of this guide will not cover firewall

On Master node, run

```
sudo ufw allow 6443/tcp
sudo ufw allow 2379/tcp
sudo ufw allow 2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10251/tcp
sudo ufw allow 10252/tcp
sudo ufw allow 10255/tcp
sudo ufw reload
```

Worker Nodes,

```
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp
sudo ufw reload
```

#### Install sudo

Debian 12 doesn’t come with sudo installed as default, install it and add your user to the sudo group

```jsx
sudo apt install sudo
sudo usermod -aG sudo <username>
```

#### Install Containerd

Containerd provides the underlying Kubernetes container support

These commands need to be run on all **3 nodes**

Set the following kernel parameters

```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay

sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

Apply these changes

```jsx
sudo sysctl --system
```

Install the containerd packages

```jsx
sudo apt update
sudo apt -y install containerd
```

Enable Kubernetes to use containerd

```jsx
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
```

Change the config.toml created in the last command to use systemd

```jsx
sudo nano /etc/containerd/config.toml
```

change from this (around line 125)

```jsx
 ‘SystemdCgroup = false’
```

to

```jsx
‘SystemdCgroup = true‘
```

Save and exit

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/6b174169-3e69-4eb6-bd30-eb9c30c0a341/Untitled.png)

Restart and enable containerd

```jsx
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Setup Kubernetes

Run on **all Nodes**

#### Install curl and gpg

```jsx
sudo apt install curl -y
sudo apt install gpg -y
```

#### Add Repository

The Kubernetes packages are not available in the default repos, they need to be added

```jsx
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] <https://pkgs.k8s.io/core:/stable:/v1.28/deb/> /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL <https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key> | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

#### Install Kube tools

The purpose of apt-mark hold is to freeze their version to prevent unintended updates (during apt upgrade), helping maintain a stable Kubernetes environment

```jsx
sudo apt update
sudo apt install kubelet kubeadm kubectl -y
sudo apt-mark hold kubelet kubeadm kubectl
```

### Install Kubernetes Cluster

Historically kubelet allowed for command line options, these were removed and YAML files are now used to provide input options

Run **only on the master node**

#### Create kubelet.yaml

Create a file in your home folder

```jsx
nano kubelet.yaml
```

Add the following

```jsx
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "1.28.0" 
controlPlaneEndpoint: "k8s-master"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
```

Note:

With the line **kubernetesVersion: "1.28.0"** there may be more recent versions. I did try with 1.30.0 however got messages about the kube packages not being up to date enough.

Set **controlPlaneEndpoint: “k8s-master”** to the hostname of your master node

#### Initialise the Kubernetes Cluster

These commands will setup the master kubernetes node

Run **only on the master node**

```jsx
sudo kubeadm init --config kubelet.yaml
```

If any errors occur and you need to reinitialise, do this

```jsx
sudo systemctl stop kubelet
sudo systemctl stop containerd
sudo rm -rf /etc/kubernetes /var/lib/etcd /var/lib/kubelet
sudo systemctl restart containerd
kill -9 $(netstat -tunlp | grep 6443 | awk '{print $7}' | cut -d'/' -f1)
sudo kubeadm init --config kubelet.yaml
```

The output should look something like this

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/3887043a-4615-4ce2-9aae-4bfd5ffd56a3/Untitled.png)

Similar output confirms a successful control plane install on the master node.

**Note:**

Copy your screen output, you’ll need to use these commands on your worker nodes later

#### Setup Kubectl access

To enable the kubectl command access to the Kubernetes control plane the following setup needs to complete

Run **only on the master node**

Run this **not as root**

```jsx
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Test Kubectl commands

The kubectl commands can be tested

Run **only on the master node**

```jsx
kubectl get nodes
kubectl cluster-info
```

should show similar output

```jsx
NAME                          STATUS   ROLES           AGE   VERSION
k8s-master.safewebbox.com     Ready    control-plane   24h   v1.28.11
```

and

```jsx
Kubernetes control plane is running at <https://k8s-master:6443>
CoreDNS is running at <https://k8s-master:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy>
```

### Connect Worker Nodes to the cluster

When the kubeadm init command was previously run on the master node, there was a kubectl join command provided which would have unique strings. This kubeadm join command is going to be used now.

#### Join Command

**Run on worker Nodes 01 and 02**

Note

Your command will be different, don’t copy this one, its an example of what you’re looking for from the output on your master node

```jsx
sudo kubeadm join k8s-master:6443 --token 21nm87.x1lgd4jf0lqiiiau \\
--discovery-token-ca-cert-hash sha256:28b503f1f2a2592678724c482776f04b445c5f99d76915552f14e68a24b78009
```

Successful output will look as follows

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/ea88cf37-eadf-4a25-9d05-e6c83a9ec65d/Untitled.png)

#### Test worker nodes have Joined correctly

**Run on Master node**

```jsx
kubectl get nodes
```

this will return something similar

```jsx
NAME                          STATUS   ROLES           AGE   VERSION
k8s-master.safewebbox.com     Ready    control-plane   24h   v1.28.11
k8s-worker01.safewebbox.com   Ready    <none>          23h   v1.28.11
k8s-worker02.safewebbox.com   Ready    <none>          23h   v1.28.11
```

### Pod Networking

We need to install calico to assist with pod networking, proxies etc

#### Install Calico

**Run on Master node**

```jsx
kubectl apply -f <https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml>
```

output

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/6d86fc63-81fe-42dd-a783-00d43ea3fcf2/Untitled.png)

#### Confirm Install of Calico

**Run on Master node**

Run the following command to see the pods (containers) setup for calico

```jsx
kubectl get pods -n kube-system
```

This might take about 5 minutes to see a running status on all the pods, run the command

```jsx
watch kubectl get pods -n kube-system
```

which will update the output of the command if there are any changes

When all pods are **Running**, this is ready

```jsx
NAME                                                READY   STATUS    RESTARTS      AGE
**calico-kube-controllers-7ddc4f45bc-sfjh5            1/1     Running   0             24h
calico-node-r5x4f                                   1/1     Running   0             24h
calico-node-wqmdq                                   1/1     Running   0             24h
calico-node-x6r45                                   1/1     Running   0**             24h
coredns-5dd5756b68-2mkb7                            1/1     Running   0             24h
coredns-5dd5756b68-l4b7j                            1/1     Running   0             24h
etcd-k8s-master.safewebbox.com                      1/1     Running   0             24h
kube-apiserver-k8s-master.safewebbox.com            1/1     Running   0             24h
kube-controller-manager-k8s-master.safewebbox.com   1/1     Running   3 (23h ago)   24h
kube-proxy-5t2sj                                    1/1     Running   0             24h
kube-proxy-89ldw                                    1/1     Running   0             24h
kube-proxy-ckwl2                                    1/1     Running   0             24h
kube-scheduler-k8s-master.safewebbox.com            1/1     Running   3 (23h ago)   24h
```

#### Check nodes

**Run on Master node**

Run

```jsx
kubectl get node
```

```jsx
NAME                          STATUS   ROLES           AGE   VERSION
k8s-master.safewebbox.com     Ready    control-plane   24h   v1.28.11
k8s-worker01.safewebbox.com   Ready    <none>          24h   v1.28.11
k8s-worker02.safewebbox.com   Ready    <none>          24h   v1.28.11
```

#### Complete
