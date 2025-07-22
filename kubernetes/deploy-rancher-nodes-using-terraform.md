# Deploy Rancher Nodes using Terraform

### Node Count

The minimum number of nodes we need to do this is 3, however in this example I’m installing 4 servers.

* 1 x Control Node
* 3 x Worker Nodes

The following steps are for deploying these nodes, then we will deploy K8s using Rancher to these nodes.

### Requirements

| Item    | Min Value | Notes                                                                                              |
| ------- | --------- | -------------------------------------------------------------------------------------------------- |
| OS      | Debian12  | Will also deploy on Ubuntu 22.04 and above or RPM Distros. This terraform is designed for Debian12 |
| RAM     | 4Gb       | 12Gb                                                                                               |
| vCPU    | 2         | 4 is preferred                                                                                     |
| Disk    | 50Gb      | 100Gb                                                                                              |
| Network | Default   | Will investigate using the netbird VLAN with this to reduce traffic directly over the internet.    |

### Terraform Repo

The most up to date code is available in the private Github Repo.

[https://github.com/cudoventures/rancher-deployment/tree/main/terraform](https://github.com/cudoventures/rancher-deployment/tree/main/terraform)

### Terraform Files

I’ve split the terraform Variables into 2 files, the vars file which terrraform uses for the main tf. thereis also a tfvars file which makes it easier for users to update the basic deployment

[**variables.tf**](http://variables.tf)**:** This file sets the variable names used by the [terraform.tf](http://terraform.tf) file beow

```jsx
variable "cudo_platform" {
  description = "The ID of the machine"
  type        = string
#  default     = "cudos-public-testnet"
}

variable "boot_disk_size" {
  description = "The size of the boot disk in GiB"
  type        = number
#  default     = 10
}

variable "vcpus" {
  description = "The number of virtual CPUs"
  type        = number
#  default     = 2
}

variable "memory_gib" {
  description = "The amount of memory in GiB"
  type        = number
#  default     = 4
}

variable "data_center_id" {
  description = "The ID of the data center"
  type        = string
#  default     = "gb-bournemouth-1"
}

variable "api_key" {
  description = "The API key"
  type        = string
#  default     = "42fd6690e650f77d61483a878bc3c184e7fb297122541998e83f6d2302867194"
}

variable "ssh_key_source" {
  description = "The source of the SSH key"
  type        = string
#  default     = "user"
}

variable "image_id" {
  description = "OS in use"
  type        = string
#  default     = "user"
}

variable "instance_names" {
  description = "Type of vm"
  type        = list(string)
#  default     = "user"
}

variable "project_id" {
  description = "The project within Cudos Compute"
  type        = string
#  default     = "user"
}
```

**terraform.tfvars:** These are the variables supplied to the terraform.vars file, using this method makes changes to the deployment easier.

```jsx
instance_names = ["k8scontrol", "k8snode1", "k8snode2", "k8snode3"]
project_id = "cudos-public-testnet"
cudo_platform = "public-testnet"
boot_disk_size = "30"
vcpus = 2
memory_gib = 4
data_center_id = "gb-bournemouth-1"
api_key = "42fd6690e650f77d61483a87fdkldfkldfklfdkl22541998e83f6d2302867194"
ssh_key_source = "project"
image_id = "debian-12"
```

| Variable          | What?                                                                                   | Notes                                                                                                                                                                                                                    |
| ----------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| instance\_names   | This will be used to build the names of the servers                                     | A server will be built for every name in this array, so if we added 2 server names, terraform would build 6 servers                                                                                                      |
| project\_id       | The project within compute the servers are built in                                     |                                                                                                                                                                                                                          |
| cudo\_platform    | This is a prefix used in the machine names                                              | this could be set as test or something to determin what tyope of machine they are.                                                                                                                                       |
| boot\_disks\_size | Self explanatory                                                                        | There are notes to add additional storage Disks [https://registry.terraform.io/providers/CudoVentures/cudo/latest/docs/resources/vm](https://registry.terraform.io/providers/CudoVentures/cudo/latest/docs/resources/vm) |
| vcpus             | Self explanatory                                                                        |                                                                                                                                                                                                                          |
| memory\_gib       | Self explanatory                                                                        |                                                                                                                                                                                                                          |
| data\_center\_id  | the ID of the DC as listed in the compute interface                                     |                                                                                                                                                                                                                          |
| api\_key          | This is generated in the compute interface                                              | No, thats not a legit api key                                                                                                                                                                                            |
| ssh\_key\_source  | I believe setting this as user uses the preloaded ssh keys to gain access to the server |                                                                                                                                                                                                                          |
| image\_id         | Which OS are we using, this needs to match the ID in the compute interface              |                                                                                                                                                                                                                          |

[main.tf](http://main.tf): This is the code Terraform runs to deploy the code

```jsx
terraform {
  required_providers {
    cudo = {
      source  = "CudoVentures/cudo"
      version = "0.5.0"
    }
  }
}

provider "cudo" {
  api_key    = var.api_key
  project_id = var.project_id
}

# Create the VMs
resource "cudo_vm" "rancher-vm" {
  count           = length(var.instance_names)
  id              = "cudos-${var.cudo_platform}-${var.instance_names[count.index]}-${formatdate("YYYYMMDD", timestamp())}"
  machine_type    = "intel-broadwell"
  data_center_id  = var.data_center_id
  memory_gib      = var.memory_gib
  vcpus           = var.vcpus
  boot_disk       = {
    image_id = var.image_id
    size_gib = var.boot_disk_size
  }
  #max_price_hr    = 10.000
  ssh_key_source  = var.ssh_key_source
  start_script    = <<EOF
#!/bin/bash
### This should be an ansible playbook, and willl be changed to it at some point
### each section needs to be its own role in the playbook
# Disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Disable ufw
systemctl stop ufw
systemctl disable ufw

# Add sysctl settings
cat <<EOT >> /etc/sysctl.conf
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
EOT
sysctl -p

# Load kernel modules
modprobe overlay
modprobe br_netfilter

# Add automgmt user
useradd -m -s /bin/bash automgmt
usermod -aG sudo automgmt

# Update sudoers file
echo "automgmt ALL=(ALL) NOPASSWD:ALL" | EDITOR='tee -a' visudo

# Install Docker
apt-get update
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL <https://download.docker.com/linux/debian/gpg> -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \\
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] <https://download.docker.com/linux/debian> \\
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \\
  tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
apt install -y open-iscsi #used by longhorn

# Install btop
apt install -y btop

EOF
}

output "vm_info" {
  value     = {
    for vm in cudo_vm.rancher-vm :
    vm.id => vm
  }
  sensitive = true
}
```

#### A code breakdown

This block of code will pull down the cudo provider from the terraform repos [https://registry.terraform.io/providers/CudoVentures/cudo/latest/docs](https://registry.terraform.io/providers/CudoVentures/cudo/latest/docs)

#### → Provider

Check the version number

```jsx
  required_providers {
    cudo = {
      source  = "CudoVentures/cudo"
      version = "0.5.0"
    }
  }
```

#### → Provider Init

Ensures the provider has access to compute (pulls in from terraform.vars)

```jsx
provider "cudo" {
  api_key    = var.api_key
  project_id = var.project_id
}
```

#### → Main block

This block does the work.

It counts the number of instances we put in the instance\_names variable in terraform.tfvars then runs a loop to create all of the machines

```jsx
# Create the VMs
resource "cudo_vm" "rancher-vm" {
  count           = length(var.instance_names)
  id              = "cudos-${var.cudo_platform}-${var.instance_names[count.index]}-${formatdate("YYYYMMDD", timestamp())}"
  machine_type    = "intel-broadwell"
  data_center_id  = var.data_center_id
  memory_gib      = var.memory_gib
  vcpus           = var.vcpus
  boot_disk       = {
    image_id = var.image_id
    size_gib = var.boot_disk_size
  }
  ssh_key_source  = var.ssh_key_source
```

**Things to note**

I’ve hard coded this line

```jsx
machine_type    = "intel-broadwell"
```

This sets the CPU type based on the **machine-type** identifier i found in the compute interface

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/7e5ac055-9499-4e6a-a399-667fa3f5e691/image.png)

The last block this is designed to help pull the hostname and IP out of the terraform deployment

```jsx
output "vm_info" {
  value     = {
    for vm in cudo_vm.rancher-vm :
    vm.id => vm
  }
  sensitive = true
}
```

#### → Scripts

```jsx
 start_script    = <<EOF
```

Following this line are a set of commands which will be run on each server as part of the setup. this bash ideally should be changed into an ansible Playbook which can be executed, for now however bash works fine.

### Terraform Deployment (Manual)

This is the simple bit, this will need to be automated at some point

There are three steps to this. which should be run in the folder the [terraform.tf](http://terraform.tf) file exists in (to make like simple

Initialize the environment

```jsx
terraform init
```

Check the [terraform.tf](http://terraform.tf) is sane and push the results out to an arbitrarily named file

```jsx
terraform plan --out rancher.out
```

Run the [terraform.tf](http://terraform.tf) and deploy the servers based on the approved plan.

```jsx
terraform apply "rancher.out"
```

### Extract the Hostname and IP

Once the terrqaform apply has complete, run the command

```jsx
echo "[control]" > inventory.ini && \\
terraform output -json vm_info | jq -r 'to_entries[] | select(.key | contains("control")) | "\\(.key) ansible_host=\\(.value.external_ip_address) ansible_user=root"' >> inventory.ini && \\
echo -e "\\n[workers]" >> inventory.ini && \\
terraform output -json vm_info | jq -r 'to_entries[] | select(.key | contains("node")) | "\\(.key) ansible_host=\\(.value.external_ip_address) ansible_user=root"' >> inventory.ini

```

This will pull the Hostname and IP for each of the servers we have built into a file

`inventory.ini`

the output will look like this

```jsx
[control]
cudos-test-k8scontrol-20250107 ansible_host=185.247.206.156 ansible_user=root

[worker]
cudos-test-k8snode1-20250107 ansible_host=185.247.206.155 ansible_user=root
cudos-test-k8snode2-20250107 ansible_host=185.247.206.149 ansible_user=root
cudos-test-k8snode3-20250107 ansible_host=185.247.206.146 ansible_user=root

```

We will use this info to build an inventory file for Ansible later

#### Files

| File              | Purpose                                                                                                                                                                                                                                                                                                                  |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| terraform.tfstate | **Terraform logs information about the resources it has created in a state file**. This enables Terraform to know which resources are under its control and when to update and destroy them. The terraform state file, by default, is named terraform. tfstate and is held in the same directory where Terraform is run. |

**This file needs to he held in a bucket somewhere when we automate this.** | | .terraform.lock.hcl | A Terraform HCL file is **a configuration file that uses HashiCorp Configuration Language (HCL) to define and configure infrastructure resources for a Terraform project**. HCL is a declarative language that's visually similar to JSON, but with additional data structures and capabilities.

This file contains the information of providers pulled down using `terraform init` | | .terraform | This directory holds all the provider information pulled down in a `terraform init` |

#### Burn it all

To tear down the terraform build, use this command. This will only work against the last tfstate file or needs to be directed to the builds tfstate file if in a bucket

```jsx
terraform destroy
```
