# How to Setup Rancher (Manually) 13c6c1c8a7198080ae2fffc87b82593c

## How to: Setup Rancher (Manually)

The following instructions are based on successfully deploying a Kubernetes cluster following these instructions: [How to: Using Rancher to deploy Kubernetes on Cudo Compute](https://www.notion.so/How-to-Using-Rancher-to-deploy-Kubernetes-on-Cudo-Compute-1376c1c8a719808dad75d0c18e56a708?pvs=21)

## What is covered here

Having setup the Rancher server and the K8s cluster, there are a set of services which need to be applied to the cluster.

The following needs to be done per cluster to ensure segregation of data and services.

When your cluster is ready, continue

### Order

There is an order to things here. the Storage needs to be setup first as the Monitoring setup will make use of it to have Prometheus and Grafana have configs which survive a reboot. Istio needs Prometheus installed for its observability.

## Setup a Project

### What is a project?

A project is a group of namespaces, and it is a concept introduced by Rancher. Projects allow you to manage multiple namespaces as a group and perform Kubernetes operations in them. You can use projects to support multi-tenancy, so that a team can access a project within a cluster without having access to other projects in the same cluster.What is a project?

https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/manage-clusters/projects-and-namespaces

Within the Rancher interface

1. Navigate to your cluster
2. Click on **Projects/Namespaces**
3. Click on **Create Project**

On the Project Create Screen

Add

1. Name
2. Description

then click on Create

Note: this is not a namespace, a project contains namespaces, this is a method of segregating namespaces.

### Getting kubectl working

Kubectl neds to be installed on your own PC and will provide remote access to the Kubernetes cluster. This is an optional stage however it will help troubleshooting if you have issues.

Open the Rancher Interface

click on Download KubeConfig

On your PC/Jumpbox run

```jsx
mkdir ~/.kube
```

copy the downloaded file into the \~/.kube folder as config

so

```jsx
ls ~/.kube
config
```

Set the Environment Variable

```jsx
KUBECONFIG=~/.kube/config
```

Now run

```jsx
kubectl get ns
```

Will display the following, and kubectl is setup

```jsx
NAME                          STATUS   AGE
calico-system                 Active   169m
cattle-fleet-system           Active   168m
cattle-impersonation-system   Active   168m
cattle-system                 Active   170m
cattle-ui-plugin-system       Active   168m
default                       Active   170m
kube-node-lease               Active   170m
kube-public                   Active   170m
kube-system                   Active   170m
local                         Active   168m
tigera-operator               Active   169m
```

## Storage: Longhorn

### What is Longhorn?

Longhorn is **a free, open-source, cloud-native distributed block storage system for Kubernetes clusters**:

* **Persistent storage**: Longhorn can be used to provide persistent storage for applications in a Kubernetes cluster.
* **Replication**: Longhorn can replicate block storage across multiple nodes and data centers to increase availability.
* **Backups**: Longhorn can store backup data in external storage like NFS or AWS S3.
* **Disaster recovery**: Longhorn can create cross-cluster disaster recovery volumes so that data can be quickly recovered from a backup cluster.
* **Snapshots**: Longhorn can schedule recurring snapshots of a volume.
* **Upgrades**: Longhorn can be upgraded without disrupting persistent volumes.

### Longhorn: Install

1. Within your K8s cluster
2. Click on Apps
3. Charts will be selected as the first entry
4. Click on Longhorn

This will open up the “information” page of the helm chart. Where a description and release notes are held

Click on Install

On the Install screen

1. Select the project created earlier
2. Select “Customize Helm Options before Install”
3. Click Next

This next stage is optional however I’ve had it help in testing

1. Select Longhorn Default settings
2. Select Orphaned Data Cleanup
3. Click on Next

Finally Install

Click on Install

The Helm chart will now start to Install, this will take a few minutes

While this install session is running, the session will display connected

The Output bar can be manipulated

* Close the output by clicking on the X
* Scale the window vertically using these arrows

The Deployment will take a while to install and sit on for a couple of minutes

```jsx
Release "longhorn" does not exist. Installing it now.
```

A successful deployment will look like this

```jsx

Filter
Disconnected
helm upgrade --history-max=5 --install=true --labels=catalog.cattle.io/cluster-repo-name=rancher-charts --namespace=longhorn-system --timeout=10m0s --values=/home/shell/helm/values-longhorn-crd-104.2.1-up1.7.2.yaml --version=104.2.1+up1.7.2 --wait=true longhorn-crd /home/shell/helm/longhorn-crd-104.2.1-up1.7.2.tgz
Release "longhorn-crd" has been upgraded. Happy Helming!
NAME: longhorn-crd
LAST DEPLOYED: Tue Nov 12 13:09:15 2024
NAMESPACE: longhorn-system
STATUS: deployed
REVISION: 2
TEST SUITE: None
---------------------------------------------------------------------
SUCCESS: helm upgrade --history-max=5 --install=true --labels=catalog.cattle.io/cluster-repo-name=rancher-charts --namespace=longhorn-system --timeout=10m0s --values=/home/shell/helm/values-longhorn-crd-104.2.1-up1.7.2.yaml --version=104.2.1+up1.7.2 --wait=true longhorn-crd /home/shell/helm/longhorn-crd-104.2.1-up1.7.2.tgz
---------------------------------------------------------------------
helm upgrade --history-max=5 --install=true --labels=catalog.cattle.io/cluster-repo-name=rancher-charts --namespace=longhorn-system --timeout=10m0s --values=/home/shell/helm/values-longhorn-104.2.1-up1.7.2.yaml --version=104.2.1+up1.7.2 --wait=true longhorn /home/shell/helm/longhorn-104.2.1-up1.7.2.tgz
Release "longhorn" has been upgraded. Happy Helming!
NAME: longhorn
LAST DEPLOYED: Tue Nov 12 13:09:17 2024
NAMESPACE: longhorn-system
```

### Longhorn: Troubleshooting

The Information on the deployment can be found under Apps, Installed Apps

Select All Namespaces and scroll down to the Longhorn Namespace

If the install has failed its usually because `open-iscsi` has not been installed on the nodes

To view the logs within the K8s environment run

To see the state of the pods

```jsx
kubectl get pods -n longhorn-system
```

If you see something like this

```jsx
NAME                                        READY   STATUS             RESTARTS        AGE
longhorn-driver-deployer-5b5bdd484f-np4w4   0/1     Init:0/1           0               18m
longhorn-manager-7kxdx                      1/2     CrashLoopBackOff   8 (2m32s ago)   18m
longhorn-manager-gnf57                      1/2     CrashLoopBackOff   8 (2m35s ago)   18m
longhorn-manager-xqxwd                      1/2     CrashLoopBackOff   8 (2m46s ago)   18m
longhorn-ui-867df7fc6b-bpgcm                1/1     Running            0               18m
longhorn-ui-867df7fc6b-ddbqs                1/1     Running            0               18m
```

you can view the logs of the longhorn-manager with

```jsx
kubectl logs -n longhorn-system longhorn-manager-7kxdx -c longhorn-manager
```

In my case I was getting

```jsx
time="2024-11-12T13:04:28Z" level=fatal msg="Error starting manager: failed to check environment, please make sure you have iscsiadm/open-iscsi installed on the host: failed to execute: /usr/bin/nsenter [nsenter --mount=/host/proc/67016/ns/mnt --net=/host/proc/67016/ns/net iscsiadm --version], output , stderr nsenter: failed to execute iscsiadm: No such file or directory\n: exit status 127" func=main.main.DaemonCmd.func3 file="daemon.go:94"
```

I needed to install open-iscsi and start the iscsid service on each of the nodes

Then delete the longhorn-manager and longhorn-driver-deployer pods

```jsx
kubectl -n longhorn-system delete pod -l app=longhorn-manager
```

```jsx
 kubectl -n longhorn-system delete pod -l app=longhorn-driver-deployer
```

The pods would respawn

```jsx
kubectl get pods -n longhorn-system
```

```jsx
NAME                                        READY   STATUS    RESTARTS      AGE
engine-image-ei-4623b511-64bdz              1/1     Running   0             23s
engine-image-ei-4623b511-qgjmv              1/1     Running   0             23s
engine-image-ei-4623b511-zwg5f              1/1     Running   0             23s
longhorn-driver-deployer-5b5bdd484f-k8pcf   1/1     Running   0             4s
longhorn-manager-hkhfv                      2/2     Running   1 (22s ago)   30s
longhorn-manager-x257h                      2/2     Running   1 (23s ago)   30s
longhorn-manager-x69pj                      2/2     Running   0             30s
longhorn-ui-867df7fc6b-bpgcm                1/1     Running   0             19m
longhorn-ui-867df7fc6b-ddbqs                1/1     Running   0             19m
```

Then rerun the Rancher Install

This fixed this specific issue, which is common

Note: I’ve updated the install guide/terraform to cover this.

### Longhorn: Interface

With Longhorn install

1. Click on the cluster
2. Select Longhorn
3. Launch the Interface

This will take you to this

Clicking on Node at the top will confirm all the nodes are accessable

Longhorn Install is complete

## Monitoring: Prometheus/Grafana

### What is the kube-prometheus-stack?

The [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack/) is meant for `cluster monitoring`, so it is `pre-configured` to collect metrics from all `Kubernetes components`. In addition to that it delivers a default set of `dashboards` and `alerting` rules. Many of the useful dashboards and alerts come from the [kubernetes-mixin](https://github.com/kubernetes-monitoring/kubernetes-mixin/) project.

The `kube-prometheus-stack` consists of three main components:

* `Prometheus Operator`, for spinning up and managing `Prometheus` instances in your `DOKS` cluster.
* `Grafana`, for visualizing metrics and plot data using stunning dashboards.
* `Alertmanager`, for configuring various notifications (e.g. `PagerDuty`, `Slack`, `email`, etc) based on various alerts received from the Prometheus main server.

### Monitoring: Install

Open the Rancher homepage

1. Open the cluster
2. Click on Apps/Charts
3. Click on Monitoring

The install page has a description of the service and release notes.. (worth a quick read)

When ready click on Install

1. Select the project
2. Select Customize Helm Options before Install
3. Click Next

There are some Config changes to make here

1. Select **Prometheus**
2. Select Use to use multiple namespaces
3. Select Persistent Storage for Prometheus
4. Set the size to 5Gb (from 50Gb) ← this depends on the host machine storage availability, set accordingly
5. Select Longhorn as the Storage Class
6. Leave the Access Mode as is.

Select Rancher

1. Select PVC storage for the Grafana Config
2. Change the Size (defaults to 50Gb) ← this depends on the host machine storage availability, set accordingly
3. Choose Longhorn as the Storage Class Name

Click on Install

A Successful install log will look something like this

```jsx
helm upgrade --install=true --labels=catalog.cattle.io/cluster-repo-name=rancher-charts --namespace=cattle-monitoring-system --timeout=10m0s --values=/home/shell/helm/values-rancher-monitoring-crd-104.1.2-up57.0.3.yaml --version=104.1.2+up57.0.3 --wait=true rancher-monitoring-crd /home/shell/helm/rancher-monitoring-crd-104.1.2-up57.0.3.tgz
Release "rancher-monitoring-crd" has been upgraded. Happy Helming!
NAME: rancher-monitoring-crd
LAST DEPLOYED: Tue Nov 12 14:05:59 2024
NAMESPACE: cattle-monitoring-system
STATUS: deployed
REVISION: 3
TEST SUITE: None
---------------------------------------------------------------------
SUCCESS: helm upgrade --install=true --labels=catalog.cattle.io/cluster-repo-name=rancher-charts --namespace=cattle-monitoring-system --timeout=10m0s --values=/home/shell/helm/values-rancher-monitoring-crd-104.1.2-up57.0.3.yaml --version=104.1.2+up57.0.3 --wait=true rancher-monitoring-crd /home/shell/helm/rancher-monitoring-crd-104.1.2-up57.0.3.tgz
---------------------------------------------------------------------
helm upgrade --install=true --labels=catalog.cattle.io/cluster-repo-name=rancher-charts --namespace=cattle-monitoring-system --timeout=10m0s --values=/home/shell/helm/values-rancher-monitoring-104.1.2-up57.0.3.yaml --version=104.1.2+up57.0.3 --wait=true rancher-monitoring /home/shell/helm/rancher-monitoring-104.1.2-up57.0.3.tgz
Release "rancher-monitoring" does not exist. Installing it now.
W1112 14:06:35.104549      45 warnings.go:70] spec.template.spec.containers[2].ports[0]: duplicate port definition with spec.template.spec.containers[1].ports[0]
NAME: rancher-monitoring
LAST DEPLOYED: Tue Nov 12 14:06:08 2024
NAMESPACE: cattle-monitoring-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
rancher-monitoring has been installed. Check its status by running:
  kubectl --namespace cattle-monitoring-system get pods -l "release=rancher-monitoring"
Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
---------------------------------------------------------------------
SUCCESS: helm upgrade --install=true --labels=catalog.cattle.io/cluster-repo-name=rancher-charts --namespace=cattle-monitoring-system --timeout=10m0s --values=/home/shell/helm/values-rancher-monitoring-104.1.2-up57.0.3.yaml --version=104.1.2+up57.0.3 --wait=true rancher-monitoring /home/shell/helm/rancher-monitoring-104.1.2-up57.0.3.tgz
---------------------------------------------------------------------
```

To watch the install using kubectl run

```jsx
watch kubectl get pods -n cattle-monitoring-system
```

Once Complete

```jsx
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-rancher-monitoring-alertmanager-0           2/2     Running   0          4m23s
prometheus-rancher-monitoring-prometheus-0               3/3     Running   0          4m20s
pushprox-kube-controller-manager-client-pfptf            1/1     Running   0          4m32s
pushprox-kube-controller-manager-proxy-c8bb7f747-fs8x8   1/1     Running   0          4m31s
pushprox-kube-etcd-client-z4t75                          1/1     Running   0          4m32s
pushprox-kube-etcd-proxy-889c559bc-4jk9z                 1/1     Running   0          4m31s
pushprox-kube-proxy-client-259lt                         1/1     Running   0          4m32s
pushprox-kube-proxy-client-62sg5                         1/1     Running   0          4m32s
pushprox-kube-proxy-client-854kl                         1/1     Running   0          4m32s
pushprox-kube-proxy-client-9rngz                         1/1     Running   0          4m32s
pushprox-kube-proxy-proxy-58f95c6b87-8242v               1/1     Running   0          4m31s
pushprox-kube-scheduler-client-hh6j5                     1/1     Running   0          4m32s
pushprox-kube-scheduler-proxy-58b9f6557-9wkbb            1/1     Running   0          4m31s
rancher-monitoring-grafana-696db5b885-t2ccb              3/3     Running   0          4m31s
rancher-monitoring-kube-state-metrics-559bbfb984-68ddd   1/1     Running   0          4m31s
rancher-monitoring-operator-d5677454c-wvs55              1/1     Running   0          4m31s
rancher-monitoring-prometheus-adapter-7ccfcd4456-jd6vq   1/1     Running   0          4m31s
rancher-monitoring-prometheus-node-exporter-blw2n        1/1     Running   0          4m32s
rancher-monitoring-prometheus-node-exporter-cwwhj        1/1     Running   0          4m32s
rancher-monitoring-prometheus-node-exporter-n57fs        1/1     Running   0          4m32s
rancher-monitoring-prometheus-node-exporter-w7c4p        1/1     Running   0          4m32s
```

### Monitoring: Longhorn Volumes

### Monitoring: Interfaces

To access the various monitoring Interfaces head to your K8s cluster

Select Monitoring

This will provide access to the WebGUIs for each of the services under Monitoring. These are pre-configured to be pulling information from the cluster.

#### → Grafana

Clicking on Grafana will open the Grafana Interface

#### → Prometheus Targets

This will open the defined targes prometheusis scraping

Monitoring is Installed

## Networking: Istio Mesh

### What is Istio?

Istio is

**an open-source service mesh that helps manage microservices in Kubernetes clusters**

:

* **What it does**Istio helps organizations secure, connect, and monitor microservices in a Kubernetes cluster. It can be used to manage traffic, enforce policies, and more.
* **How it works**Istio works natively with Kubernetes, but can also run on other cluster software. It injects containers into pods to add security, management, and monitoring.
* **What it's useful for**Istio can help with common challenges of distributed architecture, such as:
  * **Traffic management**: Istio can help with inter-service routing, failure recovery, and load balancing.
  * **Security**: Istio can help with encryption, role-based access, and authentication across services.
  * **Observability**: Istio can provide an end-to-end view of traffic flow and service performance.

### Istio: Installation

1. Select your K8s Cluster
2. Select Apps/Charts
3. Click on Istio

The Istio page has a large amount of information about the product and the release notes, its worth a read.

Once Read, click on **Install**

1. Select the project
2. Select Customise Helm Options
3. Click on Next

There is a lot going on on this next page and over time, we may find we need to enable more options, for now I’ve left most as the default

I’ve enabled the Jaeger interface

Click on Next

Click on Install

The Install will start

A successful install log will look something like this

```jsx
helm install --labels=catalog.cattle.io/cluster-repo-name=rancher-charts --namespace=istio-system --timeout=10m0s --values=/home/shell/helm/values-rancher-istio-104.5.0-up1.23.2.yaml --version=104.5.0+up1.23.2 --wait=true rancher-istio /home/shell/helm/rancher-istio-104.5.0-up1.23.2.tgz
W1112 14:38:13.550759      23 warnings.go:70] spec.template.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[0].key: beta.kubernetes.io/arch is deprecated since v1.14; use "kubernetes.io/arch" instead
W1112 14:38:13.550800      23 warnings.go:70] spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[0].preference.matchExpressions[0].key: beta.kubernetes.io/arch is deprecated since v1.14; use "kubernetes.io/arch" instead
W1112 14:38:13.550813      23 warnings.go:70] spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[1].preference.matchExpressions[0].key: beta.kubernetes.io/arch is deprecated since v1.14; use "kubernetes.io/arch" instead
W1112 14:38:13.550823      23 warnings.go:70] spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[2].preference.matchExpressions[0].key: beta.kubernetes.io/arch is deprecated since v1.14; use "kubernetes.io/arch" instead
W1112 14:38:13.550832      23 warnings.go:70] spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[3].preference.matchExpressions[0].key: beta.kubernetes.io/arch is deprecated since v1.14; use "kubernetes.io/arch" instead
NAME: rancher-istio
LAST DEPLOYED: Tue Nov 12 14:38:11 2024
NAMESPACE: istio-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
---------------------------------------------------------------------
SUCCESS: helm install --labels=catalog.cattle.io/cluster-repo-name=rancher-charts --namespace=istio-system --timeout=10m0s --values=/home/shell/helm/values-rancher-istio-104.5.0-up1.23.2.yaml --version=104.5.0+up1.23.2 --wait=true rancher-istio /home/shell/helm/rancher-istio-104.5.0-up1.23.2.tgz
---------------------------------------------------------------------
```

### Istio: Interface

Head to your K8s Cluster in the Rancher Interface

Click on Istio

This will provide access to the interfaces

#### → Jaeger

#### → Kaili

To generate a short lived token run

```jsx
kubectl -n istio-system create token kiali
```

This will generate a long token

```jsx
eyJhbGciOiJSUzI1NiIsImtpZCI6InR6ZmJvWURYN3V0dHcifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJya2UyIl0sImV4NlUFZ2UERrWGg1TVZpamRwa0xOLUdqdXd4WXBHeE6MTczMTQyNjczMiwiaWF0IjoxNzMxNDIzMTMyLCJpc3MiOiJodHRwczovL2t1YmVybmV0ZXMuZGVmYXVsdC5zdmMuY2x1c3Rlci5sb2NhbCIsImp0aSI6IjlmYmVlNzg0LTQxZWItNDc0My1iZDU1LTgwc2VydmljZWFjY291bnQiOnsibmFtZSI6ImtpYWxpIiwidWlkIjoiYzVhMjIwYzQtYzU5ZS00NTgxLWJkMzQtMWI5MTI3ODk0YzUyIn19LCJuYmYiOjE3MzE0MjMxMzIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDppc3Rpby1zeXN0ZW06a2lhbGkifQ.v459POGvk3S6h_u96k1WBH6dhYwbslLT--KjOqhOdmLtG5La6iput_rOXTCCzYWEyODcyZDM5MSIsImt1YmVybmV0ZXMuaW8iOnsibmFtZXNwYWNlIjoiaXN0aW8tc3lzdGVtIiwibJ594Ju7mmQsSJShgz3Nt2qlqjDtwMndoNL4-j9T-JAtOAdU1lVYxNVCLctRvSaM8zZzEYp-0XXJ3gjCrZQneuJXMJVXCU_xq2Q7nKb9gWYHHFekDDwVnRq9_WKTBUZEkDmT8-RRd24v17WRYxZR092d6j20LmTp-dxJIh
```

`(Nope Nathan, its till not a legit token)`\
https://kiali.io/docs/faq/authentication/

Paste the Generated string into the above interface

Mesh is an interesting one to look at

## Security

On Cudo compute the environment is viewable by the internet and we need a firewall in place to stop prying..

Each node needs to be able to talk to each other node on the public address as well as the 10.0.0.0/8 nat interface.

To set up UFW (Uncomplicated Firewall) you'll need to apply similar configurations on all live servers. Here's a step-by-step guide to implement these rules:

Install UFW on all servers:

```jsx
sudo apt update
sudo apt install ufw
```

Set default policies:

```jsx
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow internal communication between all servers and 10.0.0.0/8 network:

```jsx
sudo ufw allow from 185.247.206.142/32
sudo ufw allow from 185.247.206.146/32
sudo ufw allow from 185.247.206.156/32
sudo ufw allow from 185.247.206.149/32
sudo ufw allow from 185.247.206.164/32
sudo ufw allow from 10.0.0.0/8
```

Allow specific ports for 88.98.84.166:

```jsx
sudo ufw allow from 88.98.84.166 to any port 22 proto tcp
sudo ufw allow from 88.98.84.166 to any port 80 proto tcp
sudo ufw allow from 88.98.84.166 to any port 443 proto tcp
```

Enable UFW:

```jsx
sudo ufw enable
```

Verify the rules:

```jsx
sudo ufw status numbered
```

Repeat these steps on all five servers. Make sure to test the connectivity between your servers and from the allowed IP (88.98.84.166) before closing your current SSH session.

## Automation

With the End result of this being to do everything on this page in an automated fashion, it is possible as each item is installed using helm charts which can be downloaded, with this in mind some Ansible should be able to provide a basic K8s setup with services added

## Further Reading

https://docs.digitalocean.com/products/marketplace/catalog/kubernetes-monitoring-stack/
