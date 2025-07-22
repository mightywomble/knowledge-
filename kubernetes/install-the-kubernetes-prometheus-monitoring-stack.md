# Install the Kubernetes Prometheus monitoring stack

Having created a 3 node cluster, installed MetaLB for Ingress into the cluster and setup remote access to the Cluster from a Local Linux server, the next step is to deploy monitoring.

the Prometheus community supply a full monitoring stack deployed using HELM. the deployment will install Prometheus, AlertManager and Grafana on the Kubernetes cluster and populate these services with Alerts, Dashboards, API Endpoints to be able to get a good immediate overview.

## Prerequisites

### 3 Node Cluster

The 3 Node Kubernetes cluster needs to be installed and working

[How to - Build a Kubernetes 3 Node Cluster](https://www.notion.so/How-to-Build-a-Kubernetes-3-Node-Cluster-cc8f6049b6544509b68fb44b90986a78?pvs=21)

### Kubectl

Access to a working kubectl locally

[How to - Manage your Kubernetes Environment](https://www.notion.so/How-to-Manage-your-Kubernetes-Environment-e9b67d6784414505901cc33de8d0ccfc?pvs=21)

### Helm

Helm Installed

Details of installing helm can be found here

[https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

#### What is Helm?

Helm is a package manager for Kubernetes, which is used to deploy, manage, and configure applications and services on Kubernetes clusters. It simplifies the deployment process by allowing users to define, install, and upgrade complex Kubernetes applications using Helm charts.

#### Key Features of Helm

1. **Package Management:** Helm enables users to package Kubernetes resources into charts, which are collections of files that describe a set of Kubernetes resources.
2. **Versioning:** Helm charts can be versioned, allowing for easy management of application updates and rollbacks.
3. **Templating:** Helm uses templates to define Kubernetes resources, allowing for dynamic and reusable configurations. Templates can be parameterized to accommodate different deployment environments.
4. **Release Management:** Helm manages releases, which are instances of charts running in a Kubernetes cluster. It allows you to install, upgrade, and roll back releases easily.
5. **Dependency Management:** Helm charts can specify dependencies on other charts, enabling complex application stacks to be deployed as a single unit.

## URL

The service is provided via github

* [https://github.com/prometheus-community/helm-charts/tree/6f1bc9ed3f7eb9a8cb4711ca538fd0ddf71fcb96/charts/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/6f1bc9ed3f7eb9a8cb4711ca538fd0ddf71fcb96/charts/kube-prometheus-stack)
* [https://github.com/prometheus-community/helm-charts/tree/6f1bc9ed3f7eb9a8cb4711ca538fd0ddf71fcb96](https://github.com/prometheus-community/helm-charts/tree/6f1bc9ed3f7eb9a8cb4711ca538fd0ddf71fcb96)

## Install the Kube-Prometheus-stack

### Create a Namespace

Create a namespace for the monitoring stack

```yaml
kubectl create ns monitoring
```

### Add a New Helm Repository

Using helm, the repositiry for the prometheus stack needs to be added

```yaml
helm repo add prometheus-community <https://prometheus-community.github.io/helm-charts>

```

```yaml
helm repo update
```

### Edit values.yaml

The Helm installation has a file called values.yaml built into the helm package. Because using this install this install will be using some different configurations we need to create a values.yaml which we can pass to the install command and override the default values

**Note**

\*\*The Default values.yaml can be found on github here: [https://github.com/prometheus-community/helm-charts/blob/f5e395597054cc94ee7d9d92813552501c22266e/charts/kube-prometheus-stack/values.yaml#L4\*\*](https://github.com/prometheus-community/helm-charts/blob/f5e395597054cc94ee7d9d92813552501c22266e/charts/kube-prometheus-stack/values.yaml#L4**)

```yaml
nano values.yaml
```

add the following

```yaml
ruleSelectorNilUsesHelmValues: false
serviceMonitorSelectorNilUsesHelmValues: false
podMonitorSelectorNilUsesHelmValues: false
probeSelectorNilUsesHelmValues: false
scrapeConfigSelectorNilUsesHelmValues: false

	grafana:
	  ingress:
	    enabled: true
	    hosts:
	    - alert.intercloud.cudos.org
```

#### Intercloud Ingress

On the Linode Kubernetes caddy not metalb is used as the ingress, so when running the command

```yaml
kubectl get svc -n monitoring
```

Should output (_note left and right scroll bar_)

```yaml
NAME                                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                            ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   89m
kube-prometheus-stack-alertmanager               ClusterIP   10.128.42.52     <none>        9093/TCP,8080/TCP            90m
kube-prometheus-stack-grafana                    ClusterIP   10.128.212.55    <none>        80/TCP                       90m
kube-prometheus-stack-kube-state-metrics         ClusterIP   10.128.27.158    <none>        8080/TCP                     90m
kube-prometheus-stack-operator                   ClusterIP   10.128.87.50     <none>        443/TCP                      90m
kube-prometheus-stack-prometheus                 ClusterIP   10.128.72.0      <none>        9090/TCP,8080/TCP            90m
kube-prometheus-stack-prometheus-node-exporter   ClusterIP   10.128.21.253    <none>        9100/TCP                     90m
loki-loki-distributed-distributor                ClusterIP   10.128.122.67    <none>        3100/TCP,9095/TCP            85m
loki-loki-distributed-gateway                    ClusterIP   10.128.188.142   <none>        80/TCP                       85m
loki-loki-distributed-ingester                   ClusterIP   10.128.20.231    <none>        3100/TCP,9095/TCP            85m
loki-loki-distributed-ingester-headless          ClusterIP   None             <none>        3100/TCP,9095/TCP            85m
loki-loki-distributed-memberlist                 ClusterIP   None             <none>        7946/TCP                     85m
loki-loki-distributed-querier                    ClusterIP   10.128.202.31    <none>        3100/TCP,9095/TCP            85m
loki-loki-distributed-querier-headless           ClusterIP   None             <none>        3100/TCP,9095/TCP            85m
loki-loki-distributed-query-frontend             ClusterIP   10.128.243.70    <none>        3100/TCP,9095/TCP,9096/TCP   85m
loki-loki-distributed-query-frontend-headless    ClusterIP   None             <none>        3100/TCP,9095/TCP,9096/TCP   85m
prometheus-operated                              ClusterIP   None             <none>        9090/TCP                     89m
```

Each of the accessible services should be assigned as a TYPE = Cluster IP with no external IP Assigned

In the above values.yaml

The section marked **grafana:** is put in place to provide caddy the ingress controller on the Linode

```yaml
grafana:
  ingress:
    enabled: true
    hosts:
    - alert.intercloud.cudos.org
```

Add a DNS record for alert.intercloud in the [cudos.org](http://cudos.org) section of cloudflare pointing at 143.42.255.170

When applied the ingress can be checked using the command

```yaml
kubectl get ingress -n monitoring
```

Expected output

```yaml
NAME                            CLASS    HOSTS                        ADDRESS                                   PORTS   AGE
kube-prometheus-stack-grafana   <none>   alert.intercloud.cudos.org   143-42-255-170.ip.linodeusercontent.com   80      35m
```

This ingress will be processed by Caddy against the URL and forward the traffic to the internal K8s network.

### Values for MetaLB

If you’re using this to setup a test cluster then you will need a values.yaml which looks like this

```yaml
ruleSelectorNilUsesHelmValues: false
serviceMonitorSelectorNilUsesHelmValues: false
podMonitorSelectorNilUsesHelmValues: false
probeSelectorNilUsesHelmValues: false
scrapeConfigSelectorNilUsesHelmValues: false

grafana:
  service:
    type: LoadBalancer
prometheus:
  service:
    type: LoadBalancer
alertmanager:
  service:
    type: LoadBalancer
```

The bottom section will provide an external IP from MetaDB to provide access.

### Helm Deployment

Deploy the kube-prometheus-stack using the **helm update** command

```yaml
helm upgrade --install -f **values.yaml** kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```

After we deploy the Kube-Prometheus stack\*\*,\*\* we get as **default apps**:

* Grafana
* Prometheus
* Alert Manager.

### Check the Deployment

#### Check the Pods are running

```yaml
kubectl get pod -n monitoring
```

Should respond

```yaml
NAME                                                        READY   STATUS    RESTARTS   AGE
alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running   0          21h
kube-prometheus-stack-grafana-76858ff8dd-nnh94              3/3     Running   0          21h
kube-prometheus-stack-kube-state-metrics-7f6967956d-tzrkm   1/1     Running   0          21h
kube-prometheus-stack-operator-79b45fdb47-ccqc6             1/1     Running   0          21h
kube-prometheus-stack-prometheus-node-exporter-bxbtc        1/1     Running   0          21h
kube-prometheus-stack-prometheus-node-exporter-j9gjg        1/1     Running   0          21h
kube-prometheus-stack-prometheus-node-exporter-r2fqw        1/1     Running   0          21h
prometheus-kube-prometheus-stack-prometheus-0               2/2     Running   0          21h
```

If the pods are still spinning up use

```yaml
watch kubectl get pod -n monitoring
```

This will update the above output each time there is an output change

#### Check the services

Run the command

```yaml
kubectl get svc -n monitoring
```

Should output (_note left and right scroll bar_)

```yaml
NAME                                             TYPE           CLUSTER-IP      **EXTERNAL-IP**   PORT(S)         AGE
alertmanager-operated                            ClusterIP      None            <none>        9093/TCP,9094/TCP,9094/UDP      21h
kube-prometheus-stack-alertmanager               LoadBalancer   10.108.18.33    **10.10.0.242**   9093:31456/TCP,8080:31493/TCP   21h
kube-prometheus-stack-grafana                    LoadBalancer   10.105.8.65     **10.10.0.241**   80:31941/TCP    21h
kube-prometheus-stack-kube-state-metrics         ClusterIP      10.97.4.3       <none>        8080/TCP        21h
kube-prometheus-stack-operator                   ClusterIP      10.107.254.83   <none>        443/TCP         21h
kube-prometheus-stack-prometheus                 LoadBalancer   10.108.245.2    **10.10.0.243**   9090:30518/TCP,8080:31169/TCP   21h
kube-prometheus-stack-prometheus-node-exporter   ClusterIP      10.105.24.16    <none>        9100/TCP        21h
prometheus-operated                              ClusterIP      None            <none>        9090/TCP        21h
```

Each of the accessible services should be assigned as a TYPE = Loadbalancer with an IP from the external range/pool

#### Apply a Service Monitor

The ServiceMonitor defines an application that scrapes metrics from Kubernetes.

Create the yaml file

```yaml
nano servicemonitor.yaml
```

Add the following

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus-self
  labels:
    app: kube-prometheus-stack-prometheus
spec:
  endpoints:
  - interval: 30s
    port: web
  selector:
    matchLabels:
      app: kube-prometheus-stack-prometheus
```

Run

```yaml
kubectl apply -f servicemonitor.yaml -n monitoring
```

Expected output

```yaml
servicemonitor.monitoring.coreos.com/prometheus-self created
```

## Fixing some Issues

At this point we have a Kubernetes stack with a Prometheus environment setup, however there are some issues.

Opening up Prometheus and Click on Alerts and the following will be displayed

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/20ceb6b6-546a-46c9-86e1-db12efe582aa/Untitled.png)

These are in the most part an offshoot of how debian deploys a kubernetes cluster and can be resolved with some changes

### Check the port bindings

Run the command

```yaml
sudo ss --plnt
```

on the master server will display something similar to this

```yaml
State                    Recv-Q                   Send-Q                                     Local Address:Port                                      Peer Address:Port                  Process                   
LISTEN                   0                        4096                                           127.0.0.1:9099                                           0.0.0.0:*                                               
LISTEN                   0                        4096                                           127.0.0.1:10257                                          0.0.0.0:*                                               
LISTEN                   0                        4096                                           127.0.0.1:10259                                          0.0.0.0:*                                               
LISTEN                   0                        4096                                           127.0.0.1:45087                                          0.0.0.0:*                                               
LISTEN                   0                        4096                                           127.0.0.1:10248                                          0.0.0.0:*                                               
LISTEN                   0                        4096                                           127.0.0.1:10249                                          0.0.0.0:*                                               
LISTEN                   0                        4096                                           127.0.0.1:2381                                           0.0.0.0:*                                               
LISTEN                   0                        4096                                           127.0.0.1:2379                                           0.0.0.0:*                                               
LISTEN                   0                        8                                                0.0.0.0:179                                            0.0.0.0:*                                               
LISTEN                   0                        128                                              0.0.0.0:22                                             0.0.0.0:*                                               
LISTEN                   0                        4096                                         10.10.0.105:2380                                           0.0.0.0:*                                               
LISTEN                   0                        4096                                         10.10.0.105:2379                                           0.0.0.0:*                                               
LISTEN                   0                        128                                                 [::]:22                                                [::]:*                                               
LISTEN                   0                        4096                                                   *:10256                                                *:*                                               
LISTEN                   0                        4096                                                   *:10250                                                *:*                                               
LISTEN                   0                        4096                                                   *:6443                                                 *:*                                               
LISTEN                   0                        4096                                                   *:9100                                                 *:*
```

These ports being bound to 127.0.0.1 will cause some of the issues highlighted in prometheus

### Gather Data

Compare the things you see there with how the pods are configured:

See the list of pods by calling

```yaml
kubectl -n kube-system get pod
```

This should list something like this

```yaml
NAME                                        READY   STATUS    RESTARTS       AGE
calico-kube-controllers-7ddc4f45bc-rtj4l    1/1     Running   0              101m
calico-node-9fq7p                           1/1     Running   0              101m
calico-node-mj44z                           1/1     Running   0              101m
calico-node-rxrzl                           1/1     Running   0              101m
coredns-5dd5756b68-jlf2r                    1/1     Running   0              107m
coredns-5dd5756b68-r96t6                    1/1     Running   0              107m
etcd-kube-master.local                      1/1     Running   0              107m
kube-apiserver-kube-master.local            1/1     Running   0              107m
kube-controller-manager-kube-master.local   1/1     Running   1 (100m ago)   107m
kube-proxy-7zh27                            1/1     Running   0              102m
kube-proxy-hfnpk                            1/1     Running   0              102m
kube-proxy-qtgfz                            1/1     Running   0              107m
kube-scheduler-kube-master.local            1/1     Running   1 (100m ago)   107m
```

for more information on each pod use this command.

```yaml
kubectl -n kube-system describe pod <podname>
```

This will identify the pod names and the associated manifests

| Needed                  | Pod Name                                  | Manifest                                               |
| ----------------------- | ----------------------------------------- | ------------------------------------------------------ |
| kube-scheduler          | kube-scheduler-kube-master.local          | /etc/kubernetes/manifests/kube-scheduler.yaml          |
| kube-controller-manager | kube-controller-manager-kube-master.local | /etc/kubernetes/manifests/kube-controller-manager.yaml |
| etcd                    | etcd-kube-master.local                    | /etc/kubernetes/manifests/etcd.yaml                    |
| kube-proxy              | kube-proxy-7zh27                          |                                                        |
|                         | kube-proxy-hfnpk                          |                                                        |
|                         | kube-proxy-qtgfz                          |                                                        |

The interesting bits necessary to figure out the correct metrics endpoint configuration are:

* If some liveness or startup probes are present, check which port they are using -> the metrics endpoint will use the same port
* In the command part, check if there's some command line flag for port binding with values like these - if yes, it means that Prometheus will always be refused, because it's not running on the same host:
  * kube-scheduler -> bind-address=127.0.0.1
  * kube-controller-manager -> bind-address=127.0.0.1
  * etcd -> listen-metrics-urls=[http://127.0.0.1](http://127.0.0.1):\<port>

## View The Interfaces

The 3 Interfaces needed are now setup and available on public Interfaces

### Prometheus

#### URL

***

[http://10.10.0.243:9090/](http://10.10.0.243:9090/)

***

#### Homepage

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/518e314d-3776-4cb7-99e5-384f741afe59/Untitled.png)

#### Predefined alerts

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/43e5fcfb-3104-41f4-8d42-cbfa54e4b796/Untitled.png)

### Alert Manager

#### URL

***

[http://10.10.0.242:9093](http://10.10.0.242:9093)

***

#### Home

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/c36e1d7c-7616-4450-81f7-70c23706416b/Untitled.png)

### Grafana

URL

***

[http://10.10.0.241](http://10.10.0.241)

***

#### Default Access

* **Username:** `admin`
* **Password:** `prom-operator`

#### PreInstalled Alert Rules

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/4f33f602-023a-42fa-be6a-b1f41acf90e0/Untitled.png)

#### PreInstalled Dashboards

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/d3311b0d-d4f8-44d1-914c-12ac84103b13/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/41387272-66b5-426d-9e08-447cb2cd10cf/Untitled.png)

## References

[https://groups.google.com/g/prometheus-users/c/\_aI-HySJ-xM?pli=1](https://groups.google.com/g/prometheus-users/c/_aI-HySJ-xM?pli=1)
