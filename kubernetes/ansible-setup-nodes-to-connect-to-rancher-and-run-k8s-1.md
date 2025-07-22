# Ansible Setup Nodes to connect to Rancher and run K8s

## Ansible: Setup Nodes to connect to Rancher and run K8s

## Description

This code has been developed for use with deb based Linux distros and is not tested on RPM based distros. It might work, just not tested.

## Code

Note: The Repository is the source of truth and will exist in an updated state. these instructions are for guidelines only, please check code in repository

### Repository

The Code is in the repository directory

```jsx
https://github.com/cudoventures/rancher-deployment/tree/main/ansible
```

### Directory Tree

It uses the following directory structure

```jsx
├── deploy_apps.yaml
├── group_vars
│   └── all.yaml
├── main.yaml
├── README.md
└── roles
    ├── helm_setup
    │   └── tasks
    │       └── main.yml
    ├── rancher_cis
    │   └── tasks
    │       └── main.yml
    ├── rancher_istio
    │   └── tasks
    │       └── main.yml
    ├── rancher_longhorn
    │   └── tasks
    │       └── main.yml
    ├── rancher_monitoring
    │   └── tasks
    │       └── main.yml
    └── rancher_monitoring_crd
        └── tasks
            └── main.yml
```

### File Descriptions

| **Directory**            | **Filename**      | **Purpose**                                                                                                                           | **Notes**                                                                                                                                                                          |
| ------------------------ | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| /                        | main.yaml         | This playbook is run initially to connect the nodes to Rancher and deploy Kubernetes                                                  | `ansible-playbook main.yaml`                                                                                                                                                       |
| /                        | deploy\_apps.yaml | Meta-file which calls the various roles used to deploy the management apps such as cluster storage or monitoring onto the new cluster | `ansible-playbook deploy_apps.yaml`                                                                                                                                                |
| group\_vars              | all.yaml          | when the Ansible is run manually this is where variables are stored for the playbooks and will need to be edited                      | On a pipeline run, I’m hoping to get the variables defined as part of a worker\_dispatch deployment where like Jenkins variables can be defined by the person running the pipeline |
| **ANSIBLE ROLES**        |                   |                                                                                                                                       |                                                                                                                                                                                    |
| helm\_setup              | main.yaml         | installs helm and updates the required repos                                                                                          | the repo details are in group\_vars/all.yaml                                                                                                                                       |
| rancher\_longhorn        | main.yaml         | Deploys Longhorn helm chart to the worker nodes to create a across worker cluster for PV/PVC                                          |                                                                                                                                                                                    |
| rancher\_monitoring\_crd | main.yaml         | Installs a Kubernetes Custom Resource Definition (CRD) to link Monitoring with Rancher                                                |                                                                                                                                                                                    |
| rancher\_monitoring      | main.yaml         | Installs Kube-prometheus-stack integrated into the rancher cluster                                                                    | This play also requires the longhorn role installed as it sets up persistent volumes for Grafana and prometheus on Longhorn. the size of the volume can be adjusted in the play    |
| rancher\_istio           | main.yaml         | Installs the isto cluster network integrated into the rancher cluster                                                                 |                                                                                                                                                                                    |
| rancher\_cis             | main.yaml         | Installs the CIS (security) reporting for the cluster.                                                                                | This can be used to generate CIS compliane security reports.                                                                                                                       |

## Pre-setup

Prior to setup there are some manual items which need to be setup.

Note: These will be automated as this is used more.

### /etc/ansible/hosts

The playbook could (and probably should) be fed using an independent inventory file. I would prefer to use Netbox as a dynamic ansible inventory and have the terraform feed netbox the VM informationa and IP Addresses.

Until that point, the playbooks are setup to use the general hosts file on the server running Ansible.

There are two groups which the IP’s of the Terraform nodes should be updated

As an example

Note: put in your correct IP Addresses

```jsx
[control]
k8scontrol ansible_host=192.168.122.169

[workers]
k8snode01 ansible_host=192.168.122.142
k8snode02 ansible_host=192.168.122.248
k8snode03 ansible_host=192.168.122.153
```

## Ansible code

WARNING: This code is subject to change, the source of truth is the repository at the start of this document

### Playbooks

#### main.yaml

```jsx
---
- hosts: localhost
  connection: local
  gather_facts: true
  become: true 
  tasks:
    - name: Create Rancher RKE2 Cluster
      uri:
        url: "https://{{ rancher.api_url }}/v1/provisioning.cattle.io.clusters"
        method: POST
        headers:
          Authorization: "Bearer {{ rancher.api_token }}"
        body_format: json
        body:
          type: provisioning.cattle.io.cluster
          metadata:
            name: "{{ cluster.name }}"
            namespace: "fleet-default"  # Add this line
          spec:
            rkeConfig:
              machineGlobalConfig:
                cni: "{{ cluster.network }}"
            kubernetesVersion: "{{ cluster.kubernetes_version }}"
            clusterAgentDeploymentCustomization:
              overrideYaml:
                tolerations:
                  - key: "node-role.kubernetes.io/controlplane"
                    operator: "Exists"
                    effect: "NoSchedule"
            chartValues:
              rke2-calico:
                calicoctl:
                  image: rancher/mirrored-calico-ctl
        status_code: 201
      register: create_cluster

    - name: Get Cluster ID
      set_fact:
        cluster_id: "{{ create_cluster.json.id }}"
    
    - name: Show cluster ID  
      debug:
        var: cluster_id

    - name: Cluster Building Wait for 4 seconds
      ansible.builtin.pause:
        seconds: 4

    - name: Get cluster ID
      shell: >
        curl -k -H "Authorization: Bearer {{ rancher.api_token }}" 
        "https://{{ rancher.api_url }}/v3/clusters" | 
        jq '.data[] | select(.name == "{{ cluster.name }}") | .id'
      register: cluster_id_output

    - name: Set cluster ID fact
      set_fact:
        project_id: "{{ cluster_id_output.stdout | trim | replace('\"', '') }}"

    - name: Generate and download kubeconfig
      uri:
        url: "https://{{ rancher.api_url }}/v3/clusters/{{ project_id }}?action=generateKubeconfig"
        method: POST
        headers:
          Authorization: "Bearer {{ rancher.api_token }}"
        status_code: 200
        return_content: yes
      register: kubeconfig_response

    - name: Ensure .kube directory exists
      file:
        path: "/home/{{ ansible_user }}/.kube"
        state: directory
        mode: '0700'
      become: false

    - name: Save kubeconfig to file
      copy:
        content: "{{ kubeconfig_response.json.config }}"
        dest: "/home/{{ ansible_user }}/.kube/config"
        mode: '0600'
      become: false

    - name: Get Cluster Registration Token
      uri:
        url: "https://{{ rancher.api_url }}/v3/clusterregistrationtokens"
        method: GET
        user: "{{ api_user }}"
        password: "{{ api_secret }}"
        force_basic_auth: yes
        return_content: yes
      register: cluster_registration_token_response

    - name: Extract Registration Token
      set_fact:
        registration_token: "{{ cluster_registration_token_response.json | json_query(query) | first | regex_search('--token ([^ ]+)') | regex_replace('--token ', '') }}"
      vars:
        query: "data[?contains(nodeCommand, 'system-agent-install.sh')].nodeCommand"

    - name: Display Registration Token
      debug:
        var: registration_token

    - name: Run rancher-agent on control nodes
      delegate_to: "{{ item }}"
      connection: ssh
      loop: "{{ groups['control'] }}"
      shell: >
        curl -fL "https://{{ rancher.api_url }}/system-agent-install.sh" | sudo sh -s - --server "https://{{ rancher.api_url }}" --label 'cattle.io/os=linux' --token "{{ registration_token }}" --etcd --controlplane

    - name: Run rancher-agent on worker nodes
      delegate_to: "{{ item }}"
      connection: ssh
      loop: "{{ groups['workers'] }}"
      shell: >
        curl -fL "https://{{ rancher.api_url }}/system-agent-install.sh" | sudo sh -s - --server "https://{{ rancher.api_url }}" --label 'cattle.io/os=linux' --token "{{ registration_token }}" --worker

```

#### → What is happening?

Reminder, anything wrapped in \{{ \}} is a variable and will either be **group\_vars/all.yaml** or generated within the play

#### —> Create the cluster

```jsx
    - name: Create Rancher RKE2 Cluster
      uri:
        url: "https://{{ rancher.api_url }}/v1/provisioning.cattle.io.clusters"
        method: POST
        headers:
          Authorization: "Bearer {{ rancher.api_token }}"
        body_format: json
        body:
          type: provisioning.cattle.io.cluster
          metadata:
            name: "{{ cluster.name }}"
            namespace: "fleet-default"  # Change this only if needed
          spec:
            rkeConfig:
              machineGlobalConfig:
                cni: "{{ cluster.network }}"
            kubernetesVersion: "{{ cluster.kubernetes_version }}"
            clusterAgentDeploymentCustomization:
              overrideYaml:
                tolerations:
                  - key: "node-role.kubernetes.io/controlplane"
                    operator: "Exists"
                    effect: "NoSchedule"
            chartValues:
              rke2-calico:
                calicoctl:
                  image: rancher/mirrored-calico-ctl
        status_code: 201
      register: create_cluster
```

This creates the base cluster on the rancher server

#### —> Confirm the cluster has been built

Having created the cluster, the code runs a set of checks to make sure the cluster is up and available to add nodes to it.

```jsx
   - name: Get Cluster ID
      set_fact:
        cluster_id: "{{ create_cluster.json.id }}"
    
    - name: Show cluster ID  
      debug:
        var: cluster_id

    - name: Cluster Building Wait for 4 seconds
      ansible.builtin.pause:
        seconds: 4

    - name: Get cluster ID
      shell: >
        curl -k -H "Authorization: Bearer {{ rancher.api_token }}" 
        "https://{{ rancher.api_url }}/v3/clusters" | 
        jq '.data[] | select(.name == "{{ cluster.name }}") | .id'
      register: cluster_id_output

    - name: Set cluster ID fact
      set_fact:
        project_id: "{{ cluster_id_output.stdout | trim | replace('\"', '') }}"
```

Once the cluster has been created there is a check to make sure the API is accessible

#### —> Pull down a kubeconfig file

Witht he custer up and working, the code pulls down a kubeconfig file into \~/.kube which is useful for remote access for lens or other tools for troubleshooting.

```jsx
  - name: Generate and download kubeconfig
      uri:
        url: "https://{{ rancher.api_url }}/v3/clusters/{{ project_id }}?action=generateKubeconfig"
        method: POST
        headers:
          Authorization: "Bearer {{ rancher.api_token }}"
        status_code: 200
        return_content: yes
      register: kubeconfig_response

    - name: Ensure .kube directory exists
      file:
        path: "/home/{{ ansible_user }}/.kube"
        state: directory
        mode: '0700'
      become: false

    - name: Save kubeconfig to file
      copy:
        content: "{{ kubeconfig_response.json.config }}"
        dest: "/home/{{ ansible_user }}/.kube/config"
        mode: '0600'
      become: false

```

#### —> Find the node registration token

For a node to register against a cluster it needs a registration token which is used as part of the setup command line.

```jsx
   - name: Get Cluster Registration Token
      uri:
        url: "https://{{ rancher.api_url }}/v3/clusterregistrationtokens"
        method: GET
        user: "{{ api_user }}"
        password: "{{ api_secret }}"
        force_basic_auth: yes
        return_content: yes
      register: cluster_registration_token_response

    - name: Extract Registration Token
      set_fact:
        registration_token: "{{ cluster_registration_token_response.json | json_query(query) | first | regex_search('--token ([^ ]+)') | regex_replace('--token ', '') }}"
      vars:
        query: "data[?contains(nodeCommand, 'system-agent-install.sh')].nodeCommand"

    - name: Display Registration Token
      debug:
        var: registration_token
```

This pulls out the variable \{{ registration\_token \}}

#### —> Deploy and Attach the Rancher Nodes

The final tasks Deploy the system-agent string

```jsx
    - name: Run rancher-agent on control nodes
      delegate_to: "{{ item }}"
      connection: ssh
      loop: "{{ groups['control'] }}"
      shell: >
        curl -fL "https://{{ rancher.api_url }}/system-agent-install.sh" | sudo sh -s - --server "https://{{ rancher.api_url }}" --label 'cattle.io/os=linux' --token "{{ registration_token }}" --etcd --controlplane

    - name: Run rancher-agent on worker nodes
      delegate_to: "{{ item }}"
      connection: ssh
      loop: "{{ groups['workers'] }}"
      shell: >
        curl -fL "https://{{ rancher.api_url }}/system-agent-install.sh" | sudo sh -s - --server "https://{{ rancher.api_url }}" --label 'cattle.io/os=linux' --token "{{ registration_token }}" --worker

```

This is the manual page being actioned here

#### —> Wait

The play will complete quite quickly, however the deployment of the nodes will take about 5 to 15 minutes depending on hardware.

This can be tracked in the Rancher interface for the cluster.

Under Machines this is a dynamic screen and should update accordingly.

It does appear to be doing nothing for about 3/4 minutes.. thats ok.

#### deploy\_apps.yaml

Once the cluster has been created and the nodes added. There are a basic set of services supplied by rancher which should be installed

* Longhorn: Cluster storage, this uses free storage space on the worker nodes to create a basic storage cluster.
* Monitoring: kube-prometheus-stack is integrated directly into the rancher cluster interface providing prometheus and grafana
* Istio: this provides visutalisation of network traffic across the cluster and enables the ability to secure and redirect traffic.
* CIS: this addon provides an integrate interface to run CIS security benchmarks/reports across the cluster

```jsx
---
- name: Install Rancher Services
  hosts: localhost
  gather_facts: false
  roles:
    - helm_setup
    - rancher_longhorn
    - rancher_monitoring_crd
    - rancher_monitoring
    - rancher_istio
    - rancher_cis

```

The first role is not teplated, in so much as its designed to install helm repositories.

Each of the following roles follows a similar templated format and is the same helm chars provided within the Rancher interface.

### Roles

#### helm

The helm role adds/updates the repository URL’s on the deployment server. the server will need the **kubeconfig** file as **\~/.kube/config** in place so each of the resulting helm installs knows where to install to.

```jsx
---
- name: Add Rancher Stable Helm repo if not present
  kubernetes.core.helm_repository:
    name: "{{ rancher_name }}"
    repo_url: "{{ rancher_repo_url }}"
  register: rancher_stable_repo
  ignore_errors: true

- name: Add Longhorn Helm repo if not present
  kubernetes.core.helm_repository:
    name: "{{ longhorn_name }}"
    repo_url: "{{ longhorn_repo_url }}"
  register: longhorn_repo
  ignore_errors: true

- name: Add Prometheus Community Helm repo if not present
  kubernetes.core.helm_repository:
    name: "{{ prometheus_name }}"
    repo_url: "{{ prometheus_repo_url }}"
  register: prometheus_community_repo
  ignore_errors: true

- name: Update all Helm repositories
  command: helm repo update

- name: Check for rancher-monitoring-crd chart availability
  command: helm search repo rancher-partner/rancher-monitoring-crd
  register: monitoring_crd_check

- name: Fail if rancher-monitoring-crd chart is not found
  fail:
    msg: "The rancher-monitoring-crd chart is not found in the rancher-partner repository."
  when: monitoring_crd_check.stdout == ""

- name: Check for rancher-monitoring chart availability
  command: helm search repo rancher-partner/rancher-monitoring
  register: monitoring_check

- name: Fail if rancher-monitoring chart is not found
  fail:
    msg: "The rancher-monitoring chart is not found in the rancher-partner repository."
  when: monitoring_check.stdout == ""
```

#### rancher apps

The remaining rancher apps follow the same templated look and feel for each role

```jsx
---
- name: Install Rancher <APP NAME>
  kubernetes.core.helm:
    name: <APP NAME>
    chart_ref: <CHART INFO>
    release_namespace: <INSTALL NAMESPACE>
    create_namespace: true

- name: Wait for 1 minute before next service
  ansible.builtin.pause:
    minutes: 1  # wait 1 min after the helm chart has 
                # complete before doing anything else
    

```

for example Longhorn looks like this

```jsx
---
- name: Install Rancher Longhorn
  kubernetes.core.helm:
    name: longhorn
    chart_ref: longhorn/longhorn
    release_namespace: longhorn-system
    create_namespace: true

- name: Wait for 1 minute before next service
  ansible.builtin.pause:
    minutes: 1

```

The Monitoring role is a good example of adding external values

```jsx
---
- name: Install Rancher Monitoring
  kubernetes.core.helm:
    name: rancher-monitoring
    chart_ref: rancher-stable/rancher-monitoring
    release_namespace: cattle-monitoring-system
    create_namespace: true
    values:
      prometheus:
        prometheusSpec:
          storageSpec:
            volumeClaimTemplate:
              spec:
                storageClassName: longhorn
                accessModes: ["ReadWriteOnce"]
                resources:
                  requests:
                    storage: 10Gi
      grafana:
        persistence:
          enabled: true
          storageClassName: longhorn
          size: 10Gi
      prometheus-adapter:
        enabled: true

- name: Wait for 1 minute before next service
  ansible.builtin.pause:
    minutes: 1
```

The values overwite the default chart and have helm add the persistent volume information for prometheus and grafana so changes hold over a restart.

### Vars

The group\_vars/all.yaml file is the key driver for the playbooks and where changes need to be made

```jsx
# File: group_vars/all.yml

rancher:
  api_url: "rancher.safewebbox.com"
  api_token: "token-x5h52:bmq2ngdn82j6gbx4pd7922tg5xrgl6jp7vf79nqjq7gs7fc7cw8nvb"
  api_user: "token-x5h52" 
  api:secret: "bmq2ngdn82j6gbx4pd7922tg5xrgl6jp7vf79nqjq7gs7fc7cw8nvb"
cluster:
  name: "my-k8s-cluster"
  description: "My Kubernetes Cluster managed by Ansible"
  kubernetes_version: "v1.31.3+rke2r1"
  network: "calico"
  drain_nodes:
    controlplane: yes
    worker: yes

# Additional variables
nodes:
  controlplane:
    hostname: "k8scontrol01"
    user: "david"
  worker:
    - hostname: "k8sworker01"
      user: "david"
    - hostname: "k8sworker02"
      user: "david"
    - hostname: "k8sworker03"
      user: "david"

# Helm repos
rancher_name: "rancher-stable"
rancher_repo_url: "https://charts.rancher.io/"
longhorn_name: "longhorn"
longhorn_repo_url: "https://charts.longhorn.io"
prometheus_name: "prometheus-community"
prometheus_repo_url: "https://prometheus-community.github.io/helm-charts"

# Longhorn configuration (example)
longhorn:
  enabled: true
  version: "105.1.0+up1.7.2)"
  storage_class: "longhorn"

# Monitoring configuration (example)
monitoring:
  enabled: true
  provider: "prometheus"
  version: "105.1.1+up61.3.2"  # Replace with the desired version
  persistent_storage:
    enabled: true
    size: "5Gi"
    storage_class: "{{ longhorn.storage_class }}"

```

As you read through this its easy to see what will need to be changed from the default settings.

For an initial run however the following variables will 100% need to be changed

```jsx
rancher:
  api_url: "rancher.safewebbox.com"
  api_token: "token-x5h52:bmq2ngdn82j6gbx4pd7922tg5xrgl6jp7vf79nqjq7gs7fc7cw8nvb"
  api_user: "token-x5h52" 
  api:secret: "bmq2ngdn82j6gbx4pd7922tg5xrgl6jp7vf79nqjq7gs7fc7cw8nvb"
cluster:
  name: "my-k8s-cluster"
```

as will the `user: david.`

These items are not used, however I have left them in the file for a reference incase I need to use them later.

```jsx
# Longhorn configuration (example)
longhorn:
  enabled: true
  version: "105.1.0+up1.7.2)"
  storage_class: "longhorn"

# Monitoring configuration (example)
monitoring:
  enabled: true
  provider: "prometheus"
  version: "105.1.1+up61.3.2"  # Replace with the desired version
  persistent_storage:
    enabled: true
    size: "5Gi"
    storage_class: "{{ longhorn.storage_class }}"
```

## →Execute Ansible ←

Setup the cluster and add the control and worker nodes

```jsx
ansible-playbook main.yaml
```

Confirm in the rancher interface the machines are all Active (5 to 15 minutes depending on hardware)

Once Active deploy the Rancher apps using

```jsx
ansible-playbook deploy_apps.yaml
```

2 minutes per role (depending on hardware)

## Check and Test

Using the Rancher Web Interface Open the cluster

The Apps Should be available in the side panel to the left.

### Longhorn

Click on Longhorn to open the interface integration

This will Open the Dashboard

click on Node to make sure all your worker nodes are part of the cluster

Click on volume to confirm the Persistent Volumes for monitoring are setup

### Monitoring

Click on Monitoring to open the interface integration

Each of these options should open Web interfaces

Grafana will have predefined dashboards and alerts

Prometheus targets will have the nodes predefined

### Istio

Click on Istio to open the interface integration

I have purposefully not installed Jaeger so expect it to grayed out.

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

### CIS

Click on CIS Benchmark to open the interface integration

This is different as it allows you to create CIS scans (one off or scheduled) in the Rancher interface

## Notes

Rancher appears around version 2.6 to change changed many of the ways it delivers endpoints,urls and API’s which means a lot of the documentation shows the older method of doing this.

A lot of this can and will be paramatised further, at the time of creation this was smaller part of a longer pipeline with the intent to iterate the code with improvements as we use it more.

## References

https://longhorn.io/

https://ranchermanager.docs.rancher.com/integrations-in-rancher/istio

https://ranchermanager.docs.rancher.com/integrations-in-rancher/cis-scans

[https://github.com/rancher/rancher/issues/30130](https://github.com/rancher/rancher/issues/30130)

https://www.reddit.com/r/rancher/comments/1cnkhvs/ranchermonitoring\_without\_manual\_install/

[https://github.com/rancher/rancher/issues/45034](https://github.com/rancher/rancher/issues/45034)

https://www.reddit.com/r/rancher/comments/1hrs7o2/comment/m501804/?context=3
