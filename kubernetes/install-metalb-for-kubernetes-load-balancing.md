# Install MetaLB for Kubernetes Load Balancing

The primary difference between a hyperscaler or supplied version of Kubernetes and a self-hosted/onprem kubernetes install is a simple one. the Kubernetes as a service model will provide the entire setup including ingress points, load balancers, persistent data and a bunch of other stuff.

The self hosted config we need to set these up.

These instructions cover how to setup and use MetaLB as an ingress point, this basically means the service’s (nginx, haproxy, grafana, prmetheus etc) setup in Kubernetes can use metaLB to be accessed from the internet/outside world.

## What is MetaLB?

To quote chatGPT

MetalLB is a load-balancer implementation for Kubernetes clusters that runs on bare-metal (as opposed to cloud environments where a load-balancer is typically provided by the cloud provider). MetaLB provides the necessary components to create a load-balancer within your on-premises Kubernetes environment, enabling your services to be exposed outside the cluster in a manner similar to how cloud-based load-balancers work.

#### Key Features of MetalLB

1. **Load-Balancer for Bare-Metal Clusters:** MetaLB provides the capability to use LoadBalancer-type services in Kubernetes clusters that are running on bare-metal hardware or in environments that do not have native load-balancer support.
2. **Support for Multiple Protocols:** MetaLB supports both Layer 2 (data link layer) and BGP (Border Gateway Protocol) modes, giving flexibility in how IP addresses are managed and routed within your network.
3. **Configuration Flexibility:** You can configure MetaLB to suit your specific network requirements. This includes specifying address pools, defining the behavior of the load-balancer, and more.
4. **Integrates with Kubernetes:** MetaLB integrates seamlessly with Kubernetes. It listens for the creation of services of type LoadBalancer and assigns IP addresses to those services from a pre-configured pool.

#### How MetalLB Works

* **Layer 2 Mode:** In this mode, MetaLB uses ARP (Address Resolution Protocol) to announce the IP address to the local network. It essentially makes the Kubernetes service IP appear like a local IP on the network.
* **BGP Mode:** In BGP mode, MetaLB uses the BGP protocol to advertise the IP addresses of your services to your network routers. This allows for more advanced routing configurations and is suitable for larger and more complex network environments.

#### Typical Use Case

1. **Installation:** You install MetaLB in your Kubernetes cluster by applying the necessary manifests or using a package manager like Helm.
2. **Configuration:** Configure MetaLB by creating a `ConfigMap` that defines the address pools and the operating mode (Layer 2 or BGP).
3. **Service Creation:** When you create a Kubernetes service of type LoadBalancer, MetaLB assigns an IP address from the pool and makes it accessible.

## Prerequisites

### IP Subnet

During the config an IP address range accessible on your network will be needed.

For these instructions i will use a 10 IP range on my internal LAN

```jsx
10.10.0.240-10.10.0.250
```

### Kubernetes cluster setup

Instructions to do this are found here

[How to - Build a Kubernetes 3 Node Cluster](https://www.notion.so/How-to-Build-a-Kubernetes-3-Node-Cluster-cc8f6049b6544509b68fb44b90986a78?pvs=21)

## Test Deployment

To test if the ingress from outside of Kubernetes is working correctly we will setup a test nginx service.

### Create NginX Deployment

Create a deployment

```
kubectl create deploy nginx --image=nginx:1.20
```

List the deploy, replicaset and pods as shown below

```
kubectl get deploy,rs,po
```

The expected output is

```jsx
kubectl get deploy,rs,po
```

```
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           39sNAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-6d777db949   1         1         1       38sNAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-6d777db949-sr8x6   1/1     Running   0          38s
```

Scale up the nginx deployment

```
kubectl scale deploy/nginx --replicas=3
```

After scaling up nginx deployment to 3 replicas, the expected output is

```jsx
kubectl get deploy,rs,po
```

```
**NAME                    READY   UP-TO-DATE   AVAILABLE   AGE**
deployment.apps/nginx   3/3     3            3           2m8sNAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-6d777db949   3         3         3       2m8sNAME                         **READY                        STATUS            RESTARTS   AGE**
pod/nginx-6d777db949-jttpw   1/1     Running   0          23s
pod/nginx-6d777db949-qmdk8   1/1     Running   0          23s
pod/nginx-6d777db949-sr8x6   1/1     Running   0          2m8s
```

Create a LoadBalancer service for the above deployment

```
kubectl expose deploy/nginx --type=LoadBalancer --port=80
```

Describe the nginx service we created

```
kubectl get svc
kubectl describe svc/nginx
```

The expected output is

```
**kubectl get svc**

NAME         TYPE           CLUSTER-IP    **EXTERNAL-IP**   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1     **<none>**        443/TCP        28m
nginx        LoadBalancer   10.103.19.189 **<pending>**     80:32019/TCP   5s

**kubectl describe svc/nginx**

Name:                     nginx
Namespace:                default
Labels:                   app=nginx
Annotations:              <none>
Selector:                 app=nginx
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.103.19.189
IPs:                      10.103.19.189
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32019/TCP
Endpoints:                192.168.145.193:80,192.168.145.194:80,192.168.72.129:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Notice the External IP of the nginx LoadBalancer service is `<pending>` . In the absence of MetalLB or similar Load Balancer, the LoadBalancer service will never get an External IP in a bare metal K8s cluster, hence it will end up working exactly like a NodePort Service.

**This isn’t what you would have expected. MetalLB will solve this problem in our local K8s cluster.**

Scale down the nginx deployment to 0.

```
kubectl scale deploy/nginx --replicas=0
```

The Kubernetes NGINX deployment is complete, this will be used later

## Setup MetalLB

**Note:**

**MetaLB has a BGP and a Layer2 mode, this guide will setup the Layer2 mode**

### Create a Namespace

Create a namespace for MetaLB.

```
kubectl apply -fhttps://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
```

### **Note:**

**You need to allocate some IP addresses for the internal use of Metal LB. You need to ensure the IP addresses aren’t already taken.**

**Each time we create a LoadBalancer service, Kubernetes will create an instance of MetalLB LoadBalancer. Kubernetes picks an available IP address from the reserved range of IP addresses specified in our config map and assigns it to the MetalLB LoadBalancer. The MetaLB LoadBalancer, will then Load Balance the bunch of Pod EndPoints associated with the LB Service we created for our application deployment.**

### Create a Config file

**On local machine**

Create a yaml file for the metaLb setup

```jsx
nano metal-lb-cm.yml
```

add the following

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - **10.10.0.240-10.10.0.250**
```

**Note:**

**The section below uses the IP range i selected in the prereqs section above.**

```yaml
      addresses:
      - **10.10.0.240-10.10.0.250**
```

### Setup Firewall

The 3 node cluster doesn’t use firewalls locally, i’ll add this here for reference only

```
sudo firewall-cmd --permanent --add-port=7472/tcp --zone=trusted
sudo firewall-cmd --permanent --add-port=7472/udp --zone=trustedsudo firewall-cmd --permanent --add-port=7946/tcp --zone=trusted
sudo firewall-cmd --permanent --add-port=7946/udp --zone=trustedsudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

The above ports are the default ports used by Metal LB, in case you have modified them, make sure you change the ports accordingly.

### Deploy MetaLB to the Cluster

**On Local machine**

```
kubectl apply -f **metal-lb-cm.yml
kubectl** apply -f <https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml>
```

Lst the pods running inside metallb-system namespace.

```yaml
kubectl get pod  -n metallb-system
```

```
NAME                          READY   STATUS    RESTARTS   AGE
controller-7fbf768f66-m66ph   1/1     Running   0          30s
speaker-hj669                 1/1     Running   0          30s
speaker-l9sbp                 1/1     Running   0          30s
speaker-q9jjf                 1/1     Running   0          30s
```

## Continue NginX test with MetaLB

With MetaLb installed, continue the Nginx test to see how this works.

### Scale Up the Replica Set

Scale up the nginx deployment.

```
kubectl scale deploy nginx --replicas=3
```

outputs

```yaml
deployment.apps/nginx scaled
```

Check if the external IP has been assigned

```yaml
kubectl get svc nginx
```

Should output

```yaml
NAME    TYPE           CLUSTER-IP    **EXTERNAL-IP**       PORT(S)        AGE
nginx   LoadBalancer   10.102.5.84   **10.10.0.240**       80:31829/TCP   5s
```

Note that this time when the service is viewed MetaLB has assigned an IP from our range/pool to the service.

Use the kubectl describe command to see more details

```yaml
kubectl describe svc/nginx
```

```

Name:                     nginx
Namespace:                default
Labels:                   app=nginx
Annotations:              <none>
Selector:                 app=nginx
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.102.5.84
IPs:                      10.102.5.84
**LoadBalancer Ingress:     110.10.0.240**
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31829/TCP
**Endpoints:                192.168.145.195:80,192.168.145.196:80,192.168.72.132:80**
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason        Age   From                Message
  ----    ------        ----  ----                -------
  Normal  IPAllocated   31s   metallb-controller  Assigned IP "192.168.254.240"
  Normal  nodeAssigned  16s   metallb-speaker     announcing from node "master.tektutor.org"
```

As you can see above, the nginx LoadBalancer service is assigned with an ExternalIP from the range we mentioned in the metallb config map.

You may now access the service as shown below.

```yaml
curl <http://10.10.0.240>
```

Displays

```

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p><p>For online documentation and support please refer to
<a href="<http://nginx.org/>">nginx.org</a>.<br/>
Commercial support is available at
<a href="<http://nginx.com/>">nginx.com</a>.</p><p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Done

MetaLb will now assign external IP addresses to any service which is defined as a LoadBalancer as opposed to a ClusterIP, this is important to remember when we deploy the prometheus monitoring.
