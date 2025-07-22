# Install Application on a K8s cluster

## How to: Install Intercloud on a K8s cluster

There is a lot here, I've tried to break it up into useful sections in the ToC for quicker access. this covers the whole Intercloud deployment and how its manually done.

## Intercloud Deployment

The following instructions are written to create a system which can deploy intercloud to a K8s endpoint, the end goal here is to use this as a stepping off point for a GitOps pipeline. however this document covers the manual steps required

### Assumptions

* A working K8s cluster
* A working kubeconfig to provide access to the K8s cluster.
* Running these instructions on Debian 12
* AutoMgmt user setup using https://github.com/cudoventures/intercloud-blockchain/blob/main/userinstall.tar.gz

### Base Software Install

The following software needs to be installed in order to run the Intercloud deployment scripts

#### git

```jsx
sudo apt install git -y
```

#### Docker

Docker is an **open-source containerization platform** that allows developers to package applications and their dependencies into a standardized unit called a **container**

Containers are lightweight, portable, and isolated from the underlying infrastructure and from each other This makes it possible to run the same container on any machine where Docker is installed, regardless of the operating system.

Add Docker's official GPG key:

```jsx
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL [https://download.docker.com/linux/debian/gpg](https://download.docker.com/linux/debian/gpg) -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the repository to Apt sources:

```jsx
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Install docker

```jsx
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
```

#### Earthly

Earthly is a super simple CI/CD framework that gives you repeatable builds that you write once and run anywhere; has a simple, instantly recognizable syntax; and works with every language, framework, and build tool. With Earthly, you can create Docker images and build artifacts (e.g. binaries, packages, and arbitrary files).

https://earthly.dev/

```jsx
sudo /bin/sh -c 'wget [https://github.com/earthly/earthly/releases/latest/download/earthly-linux-amd64](https://github.com/earthly/earthly/releases/latest/download/earthly-linux-amd64) -O /usr/local/bin/earthly && chmod +x /usr/local/bin/earthly && /usr/local/bin/earthly bootstrap --with-autocomplete'
```

#### Helm

Helm is a package manager for Kubernetes, similar to DEB or YUM files it contains the instructions to deploy an application on the K8s infrastructure

https://helm.sh/

```jsx
curl -fsSL -o get_helm.sh [https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3)
chmod 700 get_helm.sh
./get_helm.sh
```

#### Kubectl

**kubectl** is a command-line tool used to interact with Kubernetes clusters. It acts as a bridge between users and the Kubernetes control plane, allowing users to manage and operate Kubernetes resources such as pods, services, deployments, and more

```jsx
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
```

```jsx

curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```jsx
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring
```

```jsx
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.31/deb/](https://pkgs.k8s.io/core:/stable:/v1.31/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```jsx
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
```

```jsx
sudo apt-get update
sudo apt-get install -y kubectl
```

#### Krew

Krew is the plugin manager for `kubectl` command-line tool.

https://krew.sigs.k8s.io/

```jsx
(
set -x; cd "$(mktemp -d)" &&
OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
KREW="krew-${OS}_${ARCH}" &&
curl -fsSLO "[https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz](https://github.com/kubernetes-sigs/krew/releases/latest/download/$%7BKREW%7D.tar.gz)" &&
tar zxvf "${KREW}.tar.gz" &&
./"${KREW}" install krew
)
```

```jsx
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

```jsx
kubectl krew install slice
```

#### Login to Harbor

The environment needs to login to the Harbor Docker respository, the credentials for this can be found in 1password under **blockchain-development** search for **Harbor (Docker Image Repository)**

```jsx
docker login harbor.infra.compute.cudos.network
```

### Setup Intercloud

There are a set of items in the Intercloud repository which need to be updated as when it was created (and as of writing this) we don’t use Hashicorp Vault for secrets

Commands run as user **automgmt** unless otherwise specified.

#### Pull down the Repo

```jsx
git clone [git@github.com](mailto:git@github.com):cudoventures/intercloud.git
```

#### Create a backend.yaml file

A file containing secrets needs to be created which the helm deployment will use during setup to connect databases and other systems.

```jsx
cd intercloud/deploy/charts/intercloud/templates/
```

Create a copy of the existing template and edit the file.

```jsx
cp backend-template-secret.yaml backend.yaml
```

```jsx
nano backend.yaml
```

The default file looks like this

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-template
  namespace: {{ .Release.Namespace }}
stringData:

  # DB URL for normal operations
  POSTGRES_URL: postgresql://intercloud_user:<password>@postgresql/intercloud

  # DB URL for automated migrations
  POSTGRES_ADMIN_URL: postgresql://intercloud_admin:<password>@postgresql/intercloud

  # Get an API token from compute.cudo.org
  CUDO_COMPUTE_TOKEN: ""

  # The mnemonic for the Osmosis account for swapping tokens to USDC
  OSMOSIS_MNEMONIC: "test test test test test test test test test test test junk"

  # A space-seperated list of keys to use for session encryption
  # You can generate a key with `openssl rand -base64 32` or similar
  SESSION_KEYS: ""
```

The following changes need to be made

change backend-template to backend

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-template
```

to

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend
```

Generate passwords for intercloud\_user and intercloud\_admin

run the following twice

```jsx
tr -dc A-Za-z0-9 </dev/urandom | head -c 13; echo
```

make a note of the new passwords and update the following two lines

Update the following lines

```jsx
POSTGRES_URL: postgresql://intercloud_user:YngwDXgZPkQzE@postgresql/intercloud
```

```jsx
POSTGRES_ADMIN_URL: postgresql://intercloud_admin:Cgf9iWd1J6M4N@postgresql/intercloud
```

Within the Cudo Compute interface under user generate an API key and update

```jsx
CUDO_COMPUTE_TOKEN:"42fd6690e650f77d6137823789235789235789adfdfg541998e83f6d2302867194"
```

Generate a mnemonic to be used within osmosis

Use: https://www.mnemonicgenerator.com/

```jsx
OSMOSIS_MNEMONIC: "**Terrific Tramps Teared Thus Thick Tasteless Toddlers Tasered Terrific Terrifying Toads Joylessly**"
```

Use uuidgen to generate a set of session keys, maybe 3 separated by spaces

```jsx
SESSION_KEYS: "0dc6fcef-ae72-4f18-8b10-62560f5bad3a aa7be55d-3c8f-43df-a6fe-7b1a95a89118 8a3a99cc-0fe3-4bfc-9a1e-39c58214fcc7"
```

This will result in a backend.yaml which looks like this

```jsx
apiVersion: v1
kind: Secret
metadata:
  name: backend
  namespace: {{ .Release.Namespace }}
stringData:

  # DB URL for normal operations
  POSTGRES_URL: postgresql://intercloud_user:YngwDXgZPkQzE@postgresql/intercloud

  # DB URL for automated migrations
  POSTGRES_ADMIN_URL: postgresql://intercloud_admin:Cgf9iWd1J6M4N@postgresql/intercloud

  # Get an API token from compute.cudo.org
  CUDO_COMPUTE_TOKEN:"42fd6690e650f77d6137823789235789235789adfdfg541998e83f6d2302867194"

  # The mnemonic for the Osmosis account for swapping tokens to USDC
  OSMOSIS_MNEMONIC: "**Terrific Tramps Teared Thus Thick Tasteless Toddlers Tasered Terrific Terrifying Toads Joylessly**"

  # A space-seperated list of keys to use for session encryption
  # You can generate a key with `openssl rand -base64 32` or similar
  SESSION_KEYS: "0dc6fcef-ae72-4f18-8b10-62560f5bad3a aa7be55d-3c8f-43df-a6fe-7b1a95a89118 8a3a99cc-0fe3-4bfc-9a1e-39c58214fcc7"
```

***

#### Create a Build endpoint

**computestaging** should be changed for your preferred deployment name

the original code was designed to be pushed to a k8s cluster on Linode with a staging and prod namespaces on the cluster. As we are adding a new cluster(s) they will need to be created as endpoints. Technically they can be called whatever we want, however for the sake of everyone's sanity, we should keep the names the same as the k8s cluster namespace we are deploying to

Note: these commands are run from the root of your git clone location

#### Duplicate the staging target to computestaging

Note: you could duplicate the production environment as well. Staging is safer.

```jsx
cp -r deploy/targets/staging/ deploy/targets/computestaging
```

```jsx
cp -r deploy/clusters/linode deploy/clusters/computestaging
```

This should result in

```jsx
deploy/targets/computestaging
deploy/clusters/computestaging
```

#### Working Kubeconfig

The rancher interface has an option to download the kubeconfig, this is the file which provides access to the kubectl command to your kubernetes deployment.

[https://185.247.206.142/dashboard/c/c-m-6sm924r2/explorer](https://185.247.206.142/dashboard/c/c-m-6sm924r2/explorer)

Download the file and copy the k8s cluster config into **intercloud/deploy/clusters/computestaging/kubeconfig.yaml**

#### Update the Build Env

Edit the [env.sh](http://env.sh) in your new cluster folder

```jsx
nano intercloud/deploy/clusters/computestaging/env.sh
```

Update the full path in your kubeconfig file

```jsx
export KUBECONFIG=**<full path to>**/**intercloud/deploy/clusters/**computestaging**/kubeconfig.yaml**
```

Note: Your setup will be different for /home/david/code

Save and exit

#### Test $KUBECONFIG

```jsx
source intercloud/deploy/clusters/computestaging/env.sh
```

Test this with

```jsx
kubectl get ns
```

This should return a set of namespaces. If it returns a load of [localhost](http://localhost) errors, it’s broken

```jsx
NAME                          STATUS   AGE
calico-system                 Active   6d3h
cattle-dashboards             Active   6d
cattle-fleet-system           Active   6d3h
cattle-impersonation-system   Active   6d3h
cattle-monitoring-system      Active   6d
cattle-system                 Active   6d3h
cattle-ui-plugin-system       Active   6d3h
default                       Active   6d3h
istio-system                  Active   5d23h
kube-node-lease               Active   6d3h
kube-public                   Active   6d3h
kube-system                   Active   6d3h
local                         Active   6d3h
longhorn-system               Active   6d1h
tigera-operator               Active   6d3h
```

#### Update deploy target

The final file to edit is changing the deployment location on the K8s cluster

```jsx
nano deploy/targets/computestaging/deploy.sh 
```

change the cluster name to reflect the namespace the service will be deployed to

```jsx
cluster **computestaging**

repo
build
tag
chart postgresql
chart intercloud
apply
```

Save and exit

#### Update the Cudo Compute project

The Intercloud interface creates its VMs on Cudo Compute. Each Intercloud instance should not be using the same Cudo Compute project as any other instance as this causes issues.

We have created intercloud-example-test as a project on compute which the person whose API keys are being used has access to as well as any other team members.

Edit the intercloud.yaml file

```jsx
nano deploy/targets/computestaging/intercloud.yaml
```

There are 2 changes here

```jsx
host: intercloud.test.cudos.org

config:
  CUDO_COMPUTE_URL: "https://rest.compute.cudo.org"
  CUDO_COMPUTE_PROJECT: "intercloud-blockchain-test"
  SUPPLIER_MARKUP: "0.05"
  CREDIT_LIMIT_RATE: "0.01"
  CREDIT_LIMIT_MAX: "100"
  OSMOSIS_RPC_URL: "https://osmosis-rpc.polkachu.com/"
  OSMOSIS_GAS_PRICE: "0.0025uosmo"
  OSMOSIS_ADDRESS: "osmo10a69lcaarckhcqf0zg04t65fuzlufel5hk448q
```

* set the line starting with `host:` to the external URL the service will use (this is also covered in the HAProxy section below)
* set the `CUDO_COMPUTE_PROJECT:` to the name of the new project

Save and exit

### Deploy to Kubernetes

Create the nameserver on the Kubernetes cluster

```jsx
kubectl create ns computestaging
```

The setup is now ready to deploy, this is done in the intercloud/ folder by running

```jsx
./tool.sh computestaging
```

#### → First run output

The output of the first run should look something like this, once built, subsequent runs will cache the builds unless they are needed to be rebuilt. So the Kubernetes part is at the end and a successful run will look like this

```jsx
========================== 🌍 Earthly Build  ✅ SUCCESS ==========================

🛰️ Reuse cache between CI runs with Earthly Satellites! 2-20X faster than without cache. Generous free tier https://cloud.earthly.dev
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: deploy/clusters/computestaging/kubeconfig.yaml
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: deploy/clusters/computestaging/kubeconfig.yaml
Wrote deploy/targets/computestaging/manifests/networkpolicy-postgresql.yaml -- 689 bytes.
Wrote deploy/targets/computestaging/manifests/poddisruptionbudget-postgresql.yaml -- 585 bytes.
Wrote deploy/targets/computestaging/manifests/serviceaccount-postgresql.yaml -- 387 bytes.
Wrote deploy/targets/computestaging/manifests/secret-postgresql.yaml -- 488 bytes.
Wrote deploy/targets/computestaging/manifests/service-postgresql-hl.yaml -- 1269 bytes.
Wrote deploy/targets/computestaging/manifests/service-postgresql.yaml -- 672 bytes.
Wrote deploy/targets/computestaging/manifests/statefulset-postgresql.yaml -- 5603 bytes.
7 files generated.
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: deploy/clusters/computestaging/kubeconfig.yaml
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: deploy/clusters/computestaging/kubeconfig.yaml
Wrote deploy/targets/computestaging/manifests/secret-backend.yaml -- 1959 bytes.
Wrote deploy/targets/computestaging/manifests/configmap-backend.yaml -- 485 bytes.
Wrote deploy/targets/computestaging/manifests/service-backend.yaml -- 361 bytes.
Wrote deploy/targets/computestaging/manifests/service-frontend.yaml -- 364 bytes.
Wrote deploy/targets/computestaging/manifests/deployment-backend.yaml -- 1369 bytes.
Wrote deploy/targets/computestaging/manifests/deployment-frontend.yaml -- 1192 bytes.
Wrote deploy/targets/computestaging/manifests/ingress-frontend.yaml -- 532 bytes.
7 files generated.
configmap/backend created
deployment.apps/backend created
deployment.apps/frontend created
ingress.networking.k8s.io/frontend created
networkpolicy.networking.k8s.io/postgresql created
poddisruptionbudget.policy/postgresql created
secret/backend created
secret/backend configured
secret/postgresql created
service/backend created
service/frontend created
service/postgresql-hl created
service/postgresql created
serviceaccount/postgresql created
statefulset.apps/postgresql created
```

A Failed Run will look like this

```jsx
========================== 🌍 Earthly Build  ✅ SUCCESS ==========================

🛰️ Reuse cache between CI runs with Earthly Satellites! 2-20X faster than without cache. Generous free tier https://cloud.earthly.dev
Wrote deploy/targets/computestaging/manifests/networkpolicy-postgresql.yaml -- 689 bytes.
Wrote deploy/targets/computestaging/manifests/poddisruptionbudget-postgresql.yaml -- 585 bytes.
Wrote deploy/targets/computestaging/manifests/serviceaccount-postgresql.yaml -- 387 bytes.
Wrote deploy/targets/computestaging/manifests/secret-postgresql.yaml -- 488 bytes.
Wrote deploy/targets/computestaging/manifests/service-postgresql-hl.yaml -- 1269 bytes.
Wrote deploy/targets/computestaging/manifests/service-postgresql.yaml -- 672 bytes.
Wrote deploy/targets/computestaging/manifests/statefulset-postgresql.yaml -- 5603 bytes.
7 files generated.
Wrote deploy/targets/computestaging/manifests/secret-backend.yaml -- 1959 bytes.
Wrote deploy/targets/computestaging/manifests/configmap-backend.yaml -- 485 bytes.
Wrote deploy/targets/computestaging/manifests/service-backend.yaml -- 361 bytes.
Wrote deploy/targets/computestaging/manifests/service-frontend.yaml -- 364 bytes.
Wrote deploy/targets/computestaging/manifests/deployment-backend.yaml -- 1369 bytes.
Wrote deploy/targets/computestaging/manifests/deployment-frontend.yaml -- 1192 bytes.
Wrote deploy/targets/computestaging/manifests/ingress-frontend.yaml -- 532 bytes.
7 files generated.
error validating "deploy/targets/computestaging/manifests/configmap-backend.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/deployment-backend.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/deployment-frontend.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/ingress-frontend.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/networkpolicy-postgresql.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/poddisruptionbudget-postgresql.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/secret-backend.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/secret-postgresql.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/service-backend.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/service-frontend.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/service-postgresql-hl.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/service-postgresql.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/serviceaccount-postgresql.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
error validating "deploy/targets/computestaging/manifests/statefulset-postgresql.yaml": error validating data: failed to download openapi: Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp [::1]:8080: connect: connection refused; if you choose to ignore these errors, turn validation off with --validate=false
```

This will usually be because

1. The KUBECONFIG isn’t set properly in

`deploy/clusters/computestaging/env.sh`\
2\. The name of the namespace has not been entered correctly in

`deploy/targets/computestaging/deploy.sh`\
3\. The namespace has not been created on the K8s cluster

#### → check if Nodes are running

Run a check

```jsx
kubectl get pods -n computestaging
NAME                       READY   STATUS              RESTARTS   AGE
backend-55ffd7bd74-47pwh   0/1     ContainerCreating   0          21s
frontend-8dd85bb89-qz9z4   1/1     Running             0          21s
postgresql-0               0/1     ContainerCreating   0          21s
```

Wait about a minute and run another check

```jsx
kubectl get pods -n computestaging
NAME                       READY   STATUS              RESTARTS   AGE
backend-55ffd7bd74-47pwh   0/1     ContainerCreating   0          21s
frontend-8dd85bb89-qz9z4   1/1     Running             0          21s
postgresql-0               1/1     Running             0          21s
```

The Postgres takes a few seconds to start

#### → 1 Pod will have an error

If you’re watching your pods with

```jsx
kubectl get pods -n computestaging
```

2 of the 3 pods will come up however the third will not, this is because the database can’t be connected as its not setup.

To view this run

```jsx
kubectl logs backend-55ffd7bd74-47pwh -n computestaging
```

you will see

```jsx
PostgresError: password authentication failed for user "intercloud_user"
    at ErrorResponse (file:///app/main.mjs:393980:27)
    at handle (file:///app/main.mjs:393776:703)
    at Socket.data (file:///app/main.mjs:393672:9)
    at Socket.emit (node:events:519:28)
    at addChunk (node:internal/streams/readable:559:12)
    at readableAddChunkPushByteMode (node:internal/streams/readable:510:3)
    at Readable.push (node:internal/streams/readable:390:5)
    at TCP.onStreamRead (node:internal/stream_base_commons:191:23) {
  severity_local: 'FATAL',
  severity: 'FATAL',
  code: '28P01',
  file: 'auth.c',
  line: '323',
  routine: 'auth_failed'
}
```

Indicating an auth failier, this is because the database needs setting up

### Post Setup - Database

While the application has been deployed to K8s, the intercloud database has not been setup. the Database and users need to be created and the initial schema and database setup needs to be imported

#### Install Postgres

The psql command needs to be used, to do this postgres needs to be installed

```jsx
sudo apt install postgresql -y
```

#### Port forwarding

The setup will use port forwarding to gain access to the postgres database, we will forward the database to port 5433 locally

```jsx
kubectl -n computestaging port-forward services/postgresql 5433:5432
```

This will provide access to the database from the build server on port 5433

#### Login to the Postgres database

To extract the password for postgres run

```jsx
kubectl get secret postgresql -n computestaging -o jsonpath='{.data}' | jq -r 'to_entries[] | "\(.key): \(.value | @base64d
)"'
```

In another ssh session to the build server run

```jsx
psql -h localhost -p 5433 -U postgres
```

We now need to create two users, using the same passwords we used in backend.yaml

```jsx
create user intercloud_admin password '<password>';
create user "intercloud_user" password '<password>';
```

Create the database and assign the admin user to it

```jsx
create database intercloud with owner intercloud_admin;
```

exit out of postgres

```jsx
\q
```

#### Import schemas and database

Under the intercloud folder is a folder named db/ this contains the databases needed, using the newly created intercloud\_admin user (and the password from backend.yaml) use the psql command to push the data to the database

```jsx
psql -h localhost -p 5433 -U intercloud_admin -d intercloud < db/schema.sql
psql -h localhost -p 5433 -U intercloud_admin -d intercloud < db/initial.sql
```

#### Check Pods

Running the command

```jsx
kubectl get pods -n computestaging
```

Will show

```jsx
NAME                       READY   STATUS             RESTARTS       AGE
backend-55ffd7bd74-47pwh   0/1     CrashLoopBackOff   9 (100s ago)   23m
frontend-8dd85bb89-qz9z4   1/1     Running            0              23m
postgresql-0               1/1     Running            0              23m
```

The backend pod needs to be recreated, we do this by deleting it. Kubernetes will auto rebuild it..

```jsx
kubectl delete pod backend-55ffd7bd74-47pwh -n computestaging
```

This will create a new pod

```jsx
kubectl get pods -n computestaging
NAME                       READY   STATUS    RESTARTS   AGE
**backend-55ffd7bd74-hcdjn   1/1     Running   0          3s**
frontend-8dd85bb89-qz9z4   1/1     Running   0          23m
postgresql-0               1/1     Running   0          23m
```

This is now running.

### Service URL - [intercloud.test.cudos.org](http://intercloud.test.cudos.org)

The service needs to be accessed via the outside world, while we have Istio installed on Rancher, for the purposes of simplicity I’m going to use HAProxy as the external ingress point, this will redirect to one of the 3 nodes to load balance the traffic.

Keeping it simple

#### Update the Egress URL

#### → As part of the deployment

Update the file

```jsx
nano deploy/targets/computestaging/intercloud.yaml   
```

Update the host: line to the URL you wish to use

```jsx
                                   
host: staging.intercloud.cudos.org

config:
  CUDO_COMPUTE_URL: "https://rest.compute.cudo.org"
  CUDO_COMPUTE_PROJECT: "intercloud-staging"
  SUPPLIER_MARKUP: "0.05"
  CREDIT_LIMIT_RATE: "0.01"
  CREDIT_LIMIT_MAX: "100"
  OSMOSIS_RPC_URL: "https://osmosis-rpc.polkachu.com/"
  OSMOSIS_GAS_PRICE: "0.0025uosmo"
  OSMOSIS_ADDRESS: "osmo10a69lcaarckhcqf0zg04t65fuzlufel5hk448q"
```

when generated using ./tool.sh the manifest file at deploy/targets/computestaging/manifests/ingress-frontend.yaml will use this as the ingress url

#### → Post deployment

Note: this is only really ideal for testing. As each time ./tool.sh is run the manifests are regenerated.

Edit the egress domain in **targets/computestaging/manifests/ingress-frontend.yaml**

```jsx
kubectl apply -f deploy/targets/computestaging/manifests/ingress-frontend.yaml
```

#### → Check the Egress

```jsx
kubectl get ingress -n computestaging

NAME       CLASS   HOSTS                          ADDRESS                                           PORTS   AGE
frontend   nginx   staging.compute.cudos.org   185.247.206.149,185.247.206.156,185.247.206.164   80      27m
```

#### Setup Cloudflare & Cert

This is not the best way to do do this, I’ve noted below how I will do it moving forward. However for this PoC it works.

Simply

* Create the DNS in Cloudflare
* Create an origin cert

Update

```jsx
/opt/cloudflare/certs/wildcard/cert.pem
/opt/cloudflare/certs/wildcard/cert.pem.key
```

Restart haproxy.

#### Update the HAProxy Config

NOTE: I need to change how this works, at present it is using an origin certificate, which is not transferrable between the modular back end approach so as it stands this haproxy will only work with [intercloud.test.cudos.org](http://intercloud.test.cudos.org) where realistically it should use a wildcard \*.cudos.org certificate so we can add multiple back end services. I’ve documented how I do this personally here: https://www.daveknowstech.com/Dave-Knows-Tech-8a5a612e3f694755ae9022cce6975a69?p=2006861b18ee40d8a031d0760b0cb7f8\&pm=c on my own website.

This is a far more scalable approach, as we can scale up services quickly by adding a line and a cfg file and not worry about certs

#### → HAproxy setup - 185.247.206.141

with a standard HAproxy setup, its not very modular and leads to some very large cfg files, what I've done on is change this approach and broken the haproxy.fcg out so it also uses a conf.d folder to store a single frontend (ingress) config file called 00-frontend.cfg and multiple backend config files. this

* Allows a modular approach
* Allows us to take a single service out of scope quickly
* Allows services to listen on 80/443 ingress and redirect based on ACL
* It makes troubleshooting easier as you’re only editing what you need to edit.

#### → Systemd file

To make this work i’ve needed to edit the systemd service file

```jsx
/lib/systemd/system/haproxy.service
```

which now looks as follows

```jsx
[Unit]
Description=HAProxy Load Balancer
Documentation=man:haproxy(1)
Documentation=file:/usr/share/doc/haproxy/configuration.txt.gz
After=network-online.target rsyslog.service
Wants=network-online.target

[Service]
EnvironmentFile=-/etc/default/haproxy
EnvironmentFile=-/etc/sysconfig/haproxy
BindReadOnlyPaths=/dev/log:/var/lib/haproxy/dev/log
Environment="CONFIG=/etc/haproxy/haproxy.cfg" "CONFIGDIR=/etc/haproxy/conf.d/" "PIDFILE=/run/haproxy.pid" "EXTRAOPTS=-S /run/haproxy-master.sock"
ExecStart=/usr/sbin/haproxy -Ws -f $CONFIG -f $CONFIGDIR -p $PIDFILE $EXTRAOPTS
ExecReload=/usr/sbin/haproxy -Ws -f $CONFIG -f $CONFIGDIR -c -q $EXTRAOPTS
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed
Restart=always
SuccessExitStatus=143
Type=notify

[Install]
WantedBy=multi-user.target
```

whats essentially changed is

```jsx
Environment="CONFIG=/etc/haproxy/haproxy.cfg" "CONFIGDIR=/etc/haproxy/conf.d/" "PIDFILE=/run/haproxy.pid" "EXTRAOPTS=-S /run/haproxy-master.sock"
ExecStart=/usr/sbin/haproxy -Ws -f $CONFIG -f $CONFIGDIR -p $PIDFILE $EXTRAOPTS
ExecReload=/usr/sbin/haproxy -Ws -f $CONFIG -f $CONFIGDIR -c -q $EXTRAOPTS
```

Here i’ve created vars for $CONFIG and $CONFIGDIR and in the ExecStart and ExecReload statements used these as variables for haproxy to look for files.

#### → haproxy.cfg

```jsx
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 50000

    # Default ciphers to use on SSL-enabled listening sockets
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

    tune.ssl.default-dh-param 2048

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 10s
    timeout client  1m
    timeout server  1m
    option http-keep-alive
    option http-server-close
    compression algo gzip
    compression type text/html text/plain text/css application/javascript
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:bky9DHZ@pjk_aqe4efd

cache static-cache
    total-max-size 256    # Cache size in megabytes
    max-age 240           # Maximum age of cached objects in seconds
```

This is a base haproxy.cfg with all references to front end and back end taken out of the file.

The only “difference” is the section

```jsx
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:bky9DHZ@pjk_aqe4efd
```

This could allow a stats page (which in all honesty isn’t highly useful) be shown on 185.247.206.141:8404

#### → 00-frontend.cfg

```jsx
frontend https_frontend
    bind *:443 ssl crt /opt/cloudflare/certs/wildcard/cert.pem alpn h2,http/1.1

    # Use the real client IP from Cloudflare
    http-request set-src req.hdr(CF-Connecting-IP)

    # ACLs to match hostnames
    acl host_netdata hdr(host) -i netdata.cudos.org
    acl host_graylog hdr(host) -i graylog.cudos.org
    acl host_intercloud hdr(host) -i intercloud.test.cudos.org

    # Check if the request is coming from Cloudflare
    acl cloudflare_ips src -f /etc/haproxy/cloudflare_ips.lst

    # Allow if from Cloudflare, deny otherwise
    http-request deny unless cloudflare_ips

    # Use backends based on hostname
    use_backend netdata_backend if host_netdata
    use_backend graylog_backend if host_graylog
    use_backend intercloud_backend if host_intercloud

    # Default backend if no match
    default_backend default_backend
```

This file handles all the ingress into the proxy. It allows the proxy to listen on ports 80/443 and redirect traffic depending on the URL which hits the access list. If traffic doesn’t match it redirects to the default\_backend

#### → 03-intercloudtest.cfg

```
backend intercloud_backend
    mode http
    balance roundrobin
    option forwardfor
    http-reuse safe

    # Enable cookie-based session persistence
    cookie SERVERID insert indirect nocache

    # Define servers with unique cookie values
    server k8s1 185.247.206.156:80 check cookie k8s1
    server k8s2 185.247.206.149:80 check cookie k8s2
    server k8s3 185.247.206.164:80 check cookie k8s3

    timeout connect 10s
    timeout server 30s
    retries 3

```

This backend uses a round robin approach to hit one of the 3 K8s nodes which the intercloud is running on. Its using server persistence using cookies, which might cause a problem and can be removed if it does.

#### → Testing

the command to test the haproxy config is now

```jsx
haproxy -c -f haproxy.cfg -f conf.d
```

in /etc/haproxy

## Troubleshooting

Some pointers, commands and tools to help troubleshooting. these can ALL be run on or from your local PC. You don’t need to run them on the build server (and to be honest i’d prefer you didn’t)

### Tools

#### Install Lens

There are plenty of tools out there to wrap a Gui around Kubernetes, My preferred tooling is Lens. Which itself comes in 2 flavours. the Mirantis Lens or OpenLens. the latter is not updated, it works. I use and setup an account (free) with Lens

https://k8slens.dev/

[https://github.com/MuhammedKalkan/OpenLens](https://github.com/MuhammedKalkan/OpenLens)

#### → Install Lens

Head to https://k8slens.dev/download

Choose your favorite flavour

Install

Note: I’ll be referring to a linux version for the remainder of the instructions.

#### → Setup Kubeconfig

Lens makes use of the kubeconfig file in the location .kube in your home folder

```jsx
mkdir ~/.kube
```

Download the KubeConfig from Rancher, select the cluster to manage. (computestaging is in cudo-private-testing)

copy the dowloaded kubeconfig.yaml into your new directory

```jsx
cp kubeconfig.yaml ~/.kube/config
```

```jsx
KUBECONFIG=~/.kube/config
```

Open Lens

Click on the Kubernetes cluster

#### → Homescreen

The Homescreen of Lens works on a fairly simple system.

Select the thing you want to see on the left. This will Open as a tab across the top

Choose the namespcae or other items from the dropdown

#### → Useful Things

**Workloads → Overview**

A useful overview of a pod, select the namespace from the dropdown at the top

Workloads → Pods

When you’re deploying this will display the status of pods and the errors if there is a problem creating the pod.

**Config → Secrets**

Useful for quickly pulling out passwords stored in Kubernetes

#### Storage Mesh

The Postgres Database is held on persisten storage using a Longhorn storage mesh. To view this information Head here

https://185.247.206.142/dashboard/c/c-m-6sm924r2/longhorn

Login to Rancher

Select the Cluster

Select Longhorn

Open Longhorn

Rancher Will Launch

the initial dashboard will show any errors, its realtime

Select Volumes

You will see the Postgres Volume

click on the Name

This allows you to drill down further into the Detail of the underlying Mesh volume. See where its stored, Event Logs and take snapshots and replicas.

#### Rancher Interface

[https://185.247.206.142/](https://185.247.206.142/)

Login creds in 1Passwords

Click on the Cluster

This will present an interface similar to Lens which you can drill down into different areas of the cluster.

* Nodes: this will display the status of the nodes
* Monitoring: this is enabled and interfaces available for Grafana are accessed here
* Longhorn: Storage mesh prometheus sits on
* Istio: Im not using this, we use haproxy for ingress currently.

#### Postgres Commands

#### → Make users Admins

To make users admins directly within the database

Port forward the Postgres DB

Login to the intercloud db as intercloud\_admin

Note: Use Lens to see the password for Intercloud admin

Login to the build server

```jsx
ssh root@185.247.206.169
```

switch to automgmt

```jsx
su - automgmt
```

Setup Kubectl

```jsx
source /home/automgmt/code/intercloud/deploy/clusters/computestaging/env.sh
```

Enable port forwarding

```jsx
kubectl -n computestaging port-forward services/postgresql 5433:5432
```

Open a new Tab and ssh to the build server in a new tab

```jsx
ssh root@185.247.206.169
```

Login to the database

```jsx
psql -h localhost -p 5433 -U intercloud_admin -d intercloud
```

the Password is found by running

```jsx
kubectl get secret backend -n computestaging -o jsonpath='{.data.POSTGRES_ADMIN_URL}' | base64 --decode
```

View the database

```jsx
SELECT * FROM "user";
```

you will see something like this

```jsx
            id            |                                                                                                                                          data                                                                                                                                           
--------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 nhtvl2n3jkashjkawm9b0ae | {"id": "nhtvl2n3jkashjkawm9b0ae", "admin": false, "debit": "0", "stake": "0", "credit": "0", "frozen": false, "created": "2024-11-19T14:08:35.175Z", "modified": "2024-11-19T14:08:35.175Z", "eligibleCredit": "0", "referralCredit": "0"}
```

Update the admin field from false to true

```jsx
UPDATE "user"
SET data = jsonb_set(data::jsonb, '{admin}', 'true'::jsonb, true);
```

This will change to

```jsx
            id            |                                                                                                                                          data                                                                                                                                           
--------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 nhtvl2n3jkashjkawm9b0ae | {"id": "nhtvl2n3jkashjkawm9b0ae", "admin": true, "debit": "0", "stake": "0", "credit": "0", "frozen": false, "created": "2024-11-19T14:08:35.175Z", "modified": "2024-11-19T14:08:35.175Z", "eligibleCredit": "0", "referralCredit": "0"}
```

There is an Admin Button now in the Intercloud Interface

#### Kubectl commands

Pod = A container

Namespace = a sandbox for a pod to go in

Node = a server used to run pods on

Monitoring
