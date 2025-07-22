# Fix Issues

This is a live document which will contain resolutions to issues seen on the Linode and Test K8s cluster

## Fixing Error in Slack

### Diagnosis

If you see this error in slack, it usually happens every month or so when Linode respawns the kubeproxy services.

```jsx

Intercloud Monitoring
APP  09:16
[FIRING:3] Monitoring Event Notification
Alert: Target disappeared from Prometheus target discovery. - critical
 Description: KubeProxy has disappeared from Prometheus target discovery.
 Graph: :chart_with_upwards_trend: Runbook: <|:spiral_note_pad:>
 Details:
  • alertname: KubeProxyDown
```

Or any message stating that there is an issue with kube-proxy

Open

```jsx
<http://139.144.159.244:9090/targets?search=>
```

Scroll down to

#### [\*\*serviceMonitor/cudo-monitoring/kube-prometheus-stack-kube-proxy/0 (4/4 up)](http://139.144.159.244:9090/targets?search=#pool-serviceMonitor/cudo-monitoring/kube-prometheus-stack-kube-proxy/0)show less\*\*

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/9a3e2b23-40bc-4970-9106-8b42eee86d54/image.png)

The State of this will be red

### Get the kubeconfig File

If you don’t have the kubectl setup then download the Kubeconfig by heading to

[https://cloud.linode.com/kubernetes/clusters](https://cloud.linode.com/kubernetes/clusters) ← this is Linode Kubernetes cluster

Note: Nathan or Joan can setup a login

Click on the Download Kubeconfig

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/03cece61-ea7e-4409-b99e-ff134302194a/image.png)

the resulting file needs to go into /home/\<your user>/.kube as the filename config

run the command

```jsx
kubectl get ns
```

should result in somthing like this

```jsx
NAME                    STATUS   AGE
atlas                   Active   69d
caddy                   Active   255d
computron               Active   55d
cudo-monitoring         Active   143d
default                 Active   618d
echo                    Active   613d
fetchcloud              Active   256d
frontend                Active   613d
gretchen                Active   5d16h
harbor                  Active   613d
intercloud-monitoring   Active   290d
kube-node-lease         Active   618d
kube-public             Active   618d
kube-system             Active   618d
mainnet                 Active   610d
metrics-server          Active   46d
seahorse                Active   89d
staging                 Active   291d
testnet                 Active   610d
v2                      Active   305d
```

### Fix the issue

On the self hosted nodes we can edit the manifests to resolve this, however these are not available as we have no access to the control plane server. However we can edit the ConfigMap using kubectl

Find the nameserver to work on

```yaml
kubectl get namespace
```

will list something like this

```yaml
NAME                    STATUS   AGE
caddy                   Active   117d
cudo-monitoring         Active   4d23h
default                 Active   479d
echo                    Active   474d
fetchcloud              Active   117d
frontend                Active   474d
harbor                  Active   474d
intercloud-monitoring   Active   151d
kube-node-lease         Active   479d
kube-public             Active   479d
kube-system             Active   479d
mainnet                 Active   471d
staging                 Active   152d
testnet                 Active   471d
v2                      Active   166d
```

on Linode (and most setups) we are interested in **kube-system**

Knowing the namespace we now need to list the config maps

```yaml
kubectl get configmap -n kube-system
```

will list something similar

```yaml
NAME                                                   DATA   AGE
calico-birdmod                                         2      479d
calico-config                                          4      479d
cluster-autoscaler-status                              1      4d16h
coredns                                                1      479d
coredns-base                                           1      109d
extension-apiserver-authentication                     6      479d
get-linode-id                                          1      479d
kube-apiserver-legacy-service-account-token-tracking   1      216d
**kube-proxy                                             2      479d**
kube-root-ca.crt                                       1      479d
kubeadm-config                                         1      479d
kubelet-config                                         1      479d
kubernetes-dashboard-settings                          0      479d
```

And there is a **kube-proxy** config map

to view this use the command

```yaml
kubectl get configmap -n kube-system kube-proxy -o yaml
```

This will display the config map in YAML format

```yaml
apiVersion: v1
data:
config.conf: |-
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
acceptContentTypes: ""
burst: 0
contentType: ""
kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
qps: 0
clusterCIDR: 10.2.0.0/16 # This must be the PodCIDR for LKE!
configSyncPeriod: 0s
conntrack:
maxPerCore: null
min: null
tcpCloseWaitTimeout: null
tcpEstablishedTimeout: null
enableProfiling: false
healthzBindAddress: ""
hostnameOverride: ""
iptables:
masqueradeAll: false
masqueradeBit: null
minSyncPeriod: 0s
syncPeriod: 0s
ipvs:
excludeCIDRs: null
minSyncPeriod: 0s
scheduler: ""
strictARP: false
syncPeriod: 0s
kind: KubeProxyConfiguration
metricsBindAddress: ""
mode: ""
nodePortAddresses: null
oomScoreAdj: null
portRange: ""
winkernel:
enableDSR: false
networkName: ""
sourceVip: ""
kubeconfig.conf: |-
apiVersion: v1
kind: Config
clusters:
- cluster:
certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
server: <https://398c0e3e-5eb8-4b88-bdb2-ef9d9c304e2a.eu-west-1.linodelke.net:443>
name: default
contexts:
- context:
cluster: default
namespace: default
user: default
name: default
current-context: default
users:
- name: default
user:
tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
kind: ConfigMap
metadata:
annotations:
kubectl.kubernetes.io/last-applied-configuration: |
{"apiVersion":"v1","data":{"config.conf":"apiVersion: kubeproxy.config.k8s.io/v1alpha1\\nbindAddress: 0.0.0.0\\nclientConnection:\\n  acceptContentTypes: \\"\\"\\n  burst: 0\\n  contentType: \\"\\"\\n  kubeconfig: /var/lib/kube-proxy/kubeconfig.conf\\n  qps: 0\\nclusterCIDR: 10.2.0.0/16 # This must be the PodCIDR for LKE!\\nconfigSyncPeriod: 0s\\nconntrack:\\n  maxPerCore: null\\n  min: null\\n  tcpCloseWaitTimeout: null\\n  tcpEstablishedTimeout: null\\nenableProfiling: false\\nhealthzBindAddress: \\"\\"\\nhostnameOverride: \\"\\"\\niptables:\\n  masqueradeAll: false\\n  masqueradeBit: null\\n  minSyncPeriod: 0s\\n  syncPeriod: 0s\\nipvs:\\n  excludeCIDRs: null\\n  minSyncPeriod: 0s\\n  scheduler: \\"\\"\\n  strictARP: false\\n  syncPeriod: 0s\\nkind: KubeProxyConfiguration\\nmetricsBindAddress: \\"\\"\\nmode: \\"\\"\\nnodePortAddresses: null\\noomScoreAdj: null\\nportRange: \\"\\"\\nwinkernel:\\n  enableDSR: false\\n  networkName: \\"\\"\\n  sourceVip: \\"\\"","kubeconfig.conf":"apiVersion: v1\\nkind: Config\\nclusters:\\n- cluster:\\n    certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt\\n    server: <https://398c0e3e-5eb8-4b88-bdb2-ef9d9c304e2a.eu-west-1.linodelke.net:443>\\n  name: default\\ncontexts:\\n- context:\\n    cluster: default\\n    namespace: default\\n    user: default\\n  name: default\\ncurrent-context: default\\nusers:\\n- name: default\\n  user:\\n    tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token"},"kind":"ConfigMap","metadata":{"annotations":{"lke.linode.com/caplke-version":"v1.28.11-001"},"labels":{"app":"kube-proxy"},"name":"kube-proxy","namespace":"kube-system"}}
lke.linode.com/caplke-version: v1.28.11-001
creationTimestamp: "2023-03-03T17:25:45Z"
labels:
app: kube-proxy
name: kube-proxy
namespace: kube-system
resourceVersion: "38758167"
uid: c95fdaf2-a9bd-43da-af4b-a8cadac93771
```

The line we need to change is

```yaml
metricsBindAddress: ""
```

To edit this configmap use the command

```yaml
kubectl -n kube-system edit configmap kube-proxy
```

This will open your default terminal text editor

Navigate to

```yaml
metricsBindAddress: ""
```

and change it to

```yaml
metricsBindAddress: "0.0.0.0:10249"
```

Save and exit the file

#### kube-proxy - Regenerate pods

With the config changed, unlike a change to the manifest files directly, this configMap change doesn’t regenerate the pod (container) to do this run

```jsx
kubectl delete pod -l k8s-app=kube-proxy -n kube-system
```

This will delete and respawn (in about 3 seconds) the proxy pods

check they are running with

```yaml
kubectl get pod -n kube-system
```

This will return something like

```yaml
NAME                                      READY   STATUS    RESTARTS      AGE
calico-kube-controllers-dfdc77f47-qd5nx   1/1     Running   7 (16h ago)   49d
calico-node-pt8gl                         1/1     Running   0             98d
calico-node-r5qkd                         1/1     Running   0             98d
calico-node-rhbpt                         1/1     Running   2 (21d ago)   98d
coredns-7cdf8b5459-g42rt                  1/1     Running   0             26d
coredns-7cdf8b5459-rhl8v                  1/1     Running   0             49d
csi-linode-controller-0                   4/4     Running   0             4d3h
csi-linode-node-h8kmx                     2/2     Running   0             4d3h
csi-linode-node-lcf96                     2/2     Running   0             4d3h
csi-linode-node-nbs5q                     2/2     Running   0             4d3h
**kube-proxy-4vfjf                          1/1     Running   0             52m
kube-proxy-7fvkh                          1/1     Running   0             52m
kube-proxy-bz2pj                          1/1     Running   0             52m**
metrics-server-84989b68d9-sb5b8           1/1     Running   0             4d20h
```

The age of the kub-proxy pods should indicate they have been recreated.

Check this has worked

Head back to the prometheus screen and refresh it a few times, it should all be green

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/9a3e2b23-40bc-4970-9106-8b42eee86d54/image.png)

### Fixing Errors - General Kubernetes

#### Wrong port number queried by Prometheus

Update the values.yaml created earlier from this

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

to this

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
    
kubeEtcd:
  enabled: true
  service:
    port: 2381
    targetPort: 2381
    clusterIP: None

kubeProxy:
  config:
    metricsBindAddress: "0.0.0.0:10249"
  service:
    clusterIP: None
  serviceMonitor:
    https: true
    insecureSkipVerify: true

kubelet:
  serviceMonitor:
    https: true
    insecureSkipVerify: true
  metrics:
    enabled: true
    resourcePath: "/metrics/resource"
  service:
    clusterIP: None

kubeControllerManager:
  enabled: true
  service:
    port: 10257
    targetPort: 10257
    clusterIP: None
  serviceMonitor:
    https: true
    insecureSkipVerify: true

kubeScheduler:
  enabled: true
  service:
    port: 10259
    targetPort: 10259
    clusterIP: None
  serviceMonitor:
    https: true
    insecureSkipVerify: true

```

Save and exit the file

Run

```yaml
 helm upgrade --install -f **values.yaml** kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```

Run

```yaml
get pods -n monitoring
```

this should return

```yaml
NAME                                                        READY   STATUS    RESTARTS   AGE
alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running   0          30m
kube-prometheus-stack-grafana-76858ff8dd-88v4m              3/3     Running   0          30m
kube-prometheus-stack-kube-state-metrics-7f6967956d-4x7lx   1/1     Running   0          30m
kube-prometheus-stack-operator-7fcbcf6c8f-fcvrx             1/1     Running   0          30m
kube-prometheus-stack-prometheus-node-exporter-g4g5m        1/1     Running   0          30m
kube-prometheus-stack-prometheus-node-exporter-gqwq5        1/1     Running   0          30m
kube-prometheus-stack-prometheus-node-exporter-tpqgd        1/1     Running   0          30m
prometheus-kube-prometheus-stack-prometheus-0               2/2     Running   0          30m

```

Its quite possible this doesn’t solve any issues.

#### Port binding refusing Prometheus by encountering one of the flags

**On Master Server**

If the above did not remove the errors in prometheus we need to change the underlying kubernetes manifest files located in /etc/kubernetes/manifests

**Note:**

**What is supposed to happen is as soon as you save and exit a manifest file, kubernetes has a watcher which rebuilds the pod for you.**

**Backup**

**Trust me on this, you’ll thank me when you have issues.**

```yaml
sudo cp -rf /etc/kubernetes /opt
```

**kube-scheduler**

Edit the Manifest file

```yaml
sudo nano /etc/kubernetes/manifests/kube-scheduler.yaml
```

Find

```yaml
bind-address=127.0.0.1 
```

change to

```yaml
bind-address=0.0.0.0 
```

**kube-controller-manager**

Edit the Manifest file

```yaml
sudo nano /etc/kubernetes/manifests/kube-controller-manager.yaml
```

Find

```yaml
bind-address=127.0.0.1 
```

change to

```yaml
bind-address=0.0.0.0 
```

**etcd**

Edit the Manifest File

```yaml
sudo nano /etc/kubernetes/manifests/etcd.yaml
```

change

```yaml
listen-metrics-urls=[<http://127.0.0.1>](<http://127.0.0.1/>):<port> 
```

change to

```yaml
listen-metrics-urls=[<http://127.0.0.1>](<http://127.0.0.1/>):<port>,**http**://<cluster IP>:2381
```

you can find the cluster IP in the other settings in that same file, note that the change here is adding an **HTTP** not an HTTPS endpoint

### Resolving issues on Linode

While the above will work and resolve alerting and metrics issues in a self hosted environment, in Linode there is no specific access via ssh to the nodes, when access is obtained there are no avilable manifest files (because these are worker nodes) and there is no access to the control plane.

While the issues are the same, the method to resolve them is a little different

#### Access to nodes

While ssh access to the nodes is not a thing for linode, there is a tool (documented in the Technical FAQ) called node-shell which will give access

[https://www.notion.so/cudo/Connecting-to-a-k8s-node-using-kubectl-e7a7dd947aa5455cb1993d334df01ef3](https://www.notion.so/cudo/Connecting-to-a-k8s-node-using-kubectl-e7a7dd947aa5455cb1993d334df01ef3)

Install this an run

```yaml
kubectl get node
```

which will return something this

```yaml
NAME                           STATUS   ROLES    AGE   VERSION
lke95955-144973-64022d535c2b   Ready    <none>   98d   v1.28.3
lke95955-144973-6405fc3131c8   Ready    <none>   98d   v1.28.3
lke95955-144973-6405fc31914d   Ready    <none>   98d   v1.28.3
```

attach to any of the nodes using

```yaml
kubectl node-shell lke95955-144973-64022d535c2b
```

this will display the following

```yaml
spawning "nsenter-87v96q" on "lke95955-144973-64022d535c2b"
If you don't see a command prompt, try pressing enter.
root@lke95955-144973-64022d535c2b:/#
```

To see why the kube-proxy isn’t connecting run

```yaml
ss -plnt
```

This will display

```yaml
State       Recv-Q      Send-Q               Local Address:Port              Peer Address:Port      Process
LISTEN      0           4096                     127.0.0.1:38789                  0.0.0.0:*          users:(("containerd",pid=688,fd=13))
LISTEN      0           4096                     127.0.0.1:10248                  0.0.0.0:*          users:(("kubelet",pid=1453,fd=26))
LISTEN      0           8                  192.168.165.206:179                    0.0.0.0:*          users:(("bird",pid=4728,fd=7))
LISTEN      0           128                        0.0.0.0:22                     0.0.0.0:*          users:(("sshd",pid=519,fd=3))
LISTEN      0           4096                     127.0.0.1:9099                   0.0.0.0:*          users:(("calico-node",pid=3243,fd=9))
LISTEN      0           4096                             *:10256                        *:*          users:(("kube-proxy",pid=1160267,fd=14))
LISTEN      0           128                           [::]:22                        [::]:*          users:(("sshd",pid=519,fd=4))
**LISTEN      0           4096                     127.0.0.1:10249                        *:*          users:(("kube-proxy",pid=1160267,fd=13))**
LISTEN      0           4096                             *:10250                        *:*          users:(("kubelet",pid=1453,fd=20))
LISTEN      0           4096                             *:9100                         *:*          users:(("node_exporter",pid=181231,fd=3))
```

The offending line is

```yaml
LISTEN      0           4096                     127.0.0.1:10249                        *:*          users:(("kube-proxy",pid=1160267,fd=13))
```

This has the metric interface listening on [localhost](http://localhost) and prometheus can’t access it

This needs to be changed to listen on 0.0.0.0:10249

Exit out of the node-shell

```yaml
exit
```

#### kube-proxy - Edit Config Map

On the self hosted nodes we can edit the manifests to resolve this, however these are not available as we have no access to the control plane server. However we can edit the ConfigMap using kubectl

Find the nameserver to work on

```yaml
kubectl get namespace
```

will list something like this

```yaml
NAME                    STATUS   AGE
caddy                   Active   117d
cudo-monitoring         Active   4d23h
default                 Active   479d
echo                    Active   474d
fetchcloud              Active   117d
frontend                Active   474d
harbor                  Active   474d
intercloud-monitoring   Active   151d
kube-node-lease         Active   479d
kube-public             Active   479d
kube-system             Active   479d
mainnet                 Active   471d
staging                 Active   152d
testnet                 Active   471d
v2                      Active   166d
```

on Linode (and most setups) we are interested in **kube-system**

Knowing the namespace we now need to list the config maps

```yaml
kubectl get configmap -n kube-system
```

will list something similar

```yaml
NAME                                                   DATA   AGE
calico-birdmod                                         2      479d
calico-config                                          4      479d
cluster-autoscaler-status                              1      4d16h
coredns                                                1      479d
coredns-base                                           1      109d
extension-apiserver-authentication                     6      479d
get-linode-id                                          1      479d
kube-apiserver-legacy-service-account-token-tracking   1      216d
**kube-proxy                                             2      479d**
kube-root-ca.crt                                       1      479d
kubeadm-config                                         1      479d
kubelet-config                                         1      479d
kubernetes-dashboard-settings                          0      479d
```

And there is a **kube-proxy** config map

to view this use the command

```yaml
kubectl get configmap -n kube-system kube-proxy -o yaml
```

This will display the config map in YAML format

```yaml
apiVersion: v1
data:
config.conf: |-
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
acceptContentTypes: ""
burst: 0
contentType: ""
kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
qps: 0
clusterCIDR: 10.2.0.0/16 # This must be the PodCIDR for LKE!
configSyncPeriod: 0s
conntrack:
maxPerCore: null
min: null
tcpCloseWaitTimeout: null
tcpEstablishedTimeout: null
enableProfiling: false
healthzBindAddress: ""
hostnameOverride: ""
iptables:
masqueradeAll: false
masqueradeBit: null
minSyncPeriod: 0s
syncPeriod: 0s
ipvs:
excludeCIDRs: null
minSyncPeriod: 0s
scheduler: ""
strictARP: false
syncPeriod: 0s
kind: KubeProxyConfiguration
metricsBindAddress: ""
mode: ""
nodePortAddresses: null
oomScoreAdj: null
portRange: ""
winkernel:
enableDSR: false
networkName: ""
sourceVip: ""
kubeconfig.conf: |-
apiVersion: v1
kind: Config
clusters:
- cluster:
certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
server: <https://398c0e3e-5eb8-4b88-bdb2-ef9d9c304e2a.eu-west-1.linodelke.net:443>
name: default
contexts:
- context:
cluster: default
namespace: default
user: default
name: default
current-context: default
users:
- name: default
user:
tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
kind: ConfigMap
metadata:
annotations:
kubectl.kubernetes.io/last-applied-configuration: |
{"apiVersion":"v1","data":{"config.conf":"apiVersion: kubeproxy.config.k8s.io/v1alpha1\\nbindAddress: 0.0.0.0\\nclientConnection:\\n  acceptContentTypes: \\"\\"\\n  burst: 0\\n  contentType: \\"\\"\\n  kubeconfig: /var/lib/kube-proxy/kubeconfig.conf\\n  qps: 0\\nclusterCIDR: 10.2.0.0/16 # This must be the PodCIDR for LKE!\\nconfigSyncPeriod: 0s\\nconntrack:\\n  maxPerCore: null\\n  min: null\\n  tcpCloseWaitTimeout: null\\n  tcpEstablishedTimeout: null\\nenableProfiling: false\\nhealthzBindAddress: \\"\\"\\nhostnameOverride: \\"\\"\\niptables:\\n  masqueradeAll: false\\n  masqueradeBit: null\\n  minSyncPeriod: 0s\\n  syncPeriod: 0s\\nipvs:\\n  excludeCIDRs: null\\n  minSyncPeriod: 0s\\n  scheduler: \\"\\"\\n  strictARP: false\\n  syncPeriod: 0s\\nkind: KubeProxyConfiguration\\nmetricsBindAddress: \\"\\"\\nmode: \\"\\"\\nnodePortAddresses: null\\noomScoreAdj: null\\nportRange: \\"\\"\\nwinkernel:\\n  enableDSR: false\\n  networkName: \\"\\"\\n  sourceVip: \\"\\"","kubeconfig.conf":"apiVersion: v1\\nkind: Config\\nclusters:\\n- cluster:\\n    certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt\\n    server: <https://398c0e3e-5eb8-4b88-bdb2-ef9d9c304e2a.eu-west-1.linodelke.net:443>\\n  name: default\\ncontexts:\\n- context:\\n    cluster: default\\n    namespace: default\\n    user: default\\n  name: default\\ncurrent-context: default\\nusers:\\n- name: default\\n  user:\\n    tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token"},"kind":"ConfigMap","metadata":{"annotations":{"lke.linode.com/caplke-version":"v1.28.11-001"},"labels":{"app":"kube-proxy"},"name":"kube-proxy","namespace":"kube-system"}}
lke.linode.com/caplke-version: v1.28.11-001
creationTimestamp: "2023-03-03T17:25:45Z"
labels:
app: kube-proxy
name: kube-proxy
namespace: kube-system
resourceVersion: "38758167"
uid: c95fdaf2-a9bd-43da-af4b-a8cadac93771
```

The line we need to change is

```yaml
metricsBindAddress: ""
```

To edit this configmap use the command

```yaml
kubectl -n kube-system edit configmap kube-proxy
```

This will open your default terminal text editor

Navigate to

```yaml
metricsBindAddress: ""
```

and change it to

```yaml
metricsBindAddress: "0.0.0.0:10249"
```

Save and exit the file

#### kube-proxy - Regenerate pods

With the config changed, unlike a change to the manifest files directly, this configMap change doesn’t regenerate the pod (container) to do this run

```yaml
kubectl delete pod -n kube-system -l component=kube-proxy
```

This will delete and respawn (in about 3 seconds) the proxy pods

check they are running with

```yaml
kubectl get pod -n kube-system
```

This will return something like

```yaml
NAME                                      READY   STATUS    RESTARTS      AGE
calico-kube-controllers-dfdc77f47-qd5nx   1/1     Running   7 (16h ago)   49d
calico-node-pt8gl                         1/1     Running   0             98d
calico-node-r5qkd                         1/1     Running   0             98d
calico-node-rhbpt                         1/1     Running   2 (21d ago)   98d
coredns-7cdf8b5459-g42rt                  1/1     Running   0             26d
coredns-7cdf8b5459-rhl8v                  1/1     Running   0             49d
csi-linode-controller-0                   4/4     Running   0             4d3h
csi-linode-node-h8kmx                     2/2     Running   0             4d3h
csi-linode-node-lcf96                     2/2     Running   0             4d3h
csi-linode-node-nbs5q                     2/2     Running   0             4d3h
**kube-proxy-4vfjf                          1/1     Running   0             52m
kube-proxy-7fvkh                          1/1     Running   0             52m
kube-proxy-bz2pj                          1/1     Running   0             52m**
metrics-server-84989b68d9-sb5b8           1/1     Running   0             4d20h
```

The age of the kub-proxy pods should indicate they have been recreated.

#### Give Prometheus a nudge

Prometheus may not update immediatly, it will wait till its next scrape time

Give it a nudge with

```yaml
curl -X POST 143.42.254.168:9090/-/reload
```

#### Resolve kube-proxy http over https issue

Now the proxy nodes are visible to proetheus it will start complaining that it cant communicate because of a mixed http/https issue. To resolve this issue..

edit your values.yaml to turn off the default strict https verification

Add the following section to the end.

```yaml
kubeProxy:
  service:
    targetPort: 10249
  serviceMonitor:
    https: true
  serviceMonitor:
    insecureSkipVerify: true
```

Update the helm install with

```yaml
helm upgrade --install -f values.yaml kube-prometheus-stack prometheus-community/kube-prometheus-stack -n cudo-monitoring
```

Either run the above **curl** command or wait 10 minutes and the alert should disapper and under Targets in the prometheus interface the kube-proxy should be happy

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/f366bcbc-1c6e-437f-b1bc-e196f05219f6/Untitled.png)

#### Disable controller and Schedular metrics

there are two final errors, these relate to the kubeControllerManager and kubeSchedule not being available, i think this is because we don’t have access to the control plane, so i’ve disabled them

Add the following to the end of the **values.yaml**

```yaml
kubeControllerManager:
  enabled: false

kubeScheduler:
  enabled: false
```

Apply this with

```yaml
helm upgrade --install -f values.yaml kube-prometheus-stack prometheus-community/kube-prometheus-stack -n cudo-monitoring
```

Run this

```yaml
curl -X POST 143.42.254.168:9090/-/reload
```

The only remaining alert should be watchdog which is a good thing..

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/913a676d-de41-497f-95f1-eb68fcef75ef/Untitled.png)

#### Watchdog

The watchdog will send alerts out to the remote services (in our case slack) to confirm that route is running

```yaml

Intercloud Monitoring
APP  13:21
[FIRING:1] Monitoring Event Notification
Alert: An alert that should always be firing to certify that Alertmanager is working properly. - none
 Description: This is an alert meant to ensure that the entire alerting pipeline is functional.
This alert is always firing, therefore it should always be firing in Alertmanager
and always fire against a receiver. There are integrations with various notification
mechanisms that send a notification when this alert is not firing. For example the
Show more
```

z
