---
description: >-
  The purpose of using the automation is to reduce errors, the purpose of using
  the code is to be able to setup a small DevOps style pipeline for testing.
---

# How to Setup an automated GKE deployment using Terraform

{% embed url="https://github.com/mightywomble/gke-deployment?tab=readme-ov-file" %}

## How to - Setup an automated GKE deployment using Github Actions and Terraform

## What is this?

The ask of this guide is to create a GKE Cluster on Google Cloud using code and automation.

The purpose of using the automation is to reduce errors, the purpose of using the code is to be able to setup a small DevOps style pipeline for testing.

While the production GKE cluster will be on all the time, to keep costs down, the staging cluster will be an on the fly build either via manually launching a github action to deploy a test cluster or deploying during testing and then tearing down once testing is complete and the PR approved for production deployment.

For the purposes of this guide we have

| **Item**      | **Purpose**                                                                                                    | **Notes**                                                                                                           |
| ------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| jumpbox       | this is the location GitHub actions runs from. this guide uses a separate Linux box, this could be your laptop | This can be on any internet facing device, in testing I’ve run this on a raspberry pi running raspbian/manjaro\_arm |
| GitHub runner | This is located on the jumpbox and will orchestrate all the actions centrally                                  |                                                                                                                     |
| Google Cloud  | Where the magic will happen                                                                                    | I think google are still doing a level of free accounts for new users                                               |
| A PC          | To create code for Terraform and git hub                                                                       | Can be any OS, needs a code editor and git installed.                                                               |
| project name  | This is the project name of the Google Cloud project                                                           | I have used the gke-home-xxxx format, this will be different for the Cudo GKE project                               |

## Pre Setup

While the GKE deployment will be automated, there is some work setting up several components which needs to be done to get the deployment environment setup and working. Each of these tasks is a one off, and while I could automate them, none of the tasks is arduous enough to create automated deployment for a one off install.

### Create a Google cloud Project

Navigate to [https://cloud.google.com/](https://cloud.google.com/) and click on **consle** in the top right

Untitled

On the Top Left click on the project Dropdown

Untitled

Click on **New Project** in the top right of the box

Untitled

Give the project a name and click on create

Untitled

#### Link Project to Billing

This is a requirement for the Terraform to work and is covered using these links

CyberITHub: [https://www.cyberithub.com/how-to-link-google-cloud-project-to-billing-account-in-4-easy-steps/](https://www.cyberithub.com/how-to-link-google-cloud-project-to-billing-account-in-4-easy-steps/)

### Install Gcloud

**On the Jumpbox**

**As: user**

**Reference: https://cloud.google.com/sdk/docs/install**

***

This is a well documented procedure I’d recommend following the link above, however at the time of writing this was the process to follow

| Step                    | Command                                                                                              | Notes                                                                                                                |
| ----------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Download tar.gz file    | curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-linux-arm.tar.gz | On the link above there are download links for x64/x86 and arm64 tar.gz files, use the appropriate one for you setup |
| un-compress file        | tar -xzvf google-cloud-cli-linux-arm.tar.gz                                                          | Change the file name to match the one you downloaded                                                                 |
| go to the new directory | cd google-cloud-sdk                                                                                  |                                                                                                                      |
| run the cli installer   | ./install.sh                                                                                         |                                                                                                                      |
| Answer the questions    | - Do you want to help improve the Google Cloud CLI (y/N)? N                                          |                                                                                                                      |

* List of installed items: Do you want to continue (Y/n)? Y
* Y

```jsx
Update the rc file: Enter a path to an rc file to update, or leave blank to use [/home/david/.bashrc]: Enter
| |
| Exit the shell/ssh session and reconnect | | This will update the settings applied by the installer |
| Run Gcloud init | gcloud init | |
| Sign into Google using your account | You must sign in to continue. Would you like to sign in (Y/n)? Y | |
| Open Link | a long link will be provided | Either copy and past the link into your browser or click on it if your terminal supports it. |
| Verification code | Copy the verification code and paste it into the terminal | the above link should open a page with a verification code and a copy button from Google |
| Confirm the project you created previously | Select the project from the available project list | |
| Default compute region and Zone | Select N here | Your terraform code will define where the GKE will install |
```

Complete

### Gcloud Service account

Because I want the process to run independently of my gmail account, further setup is needed to create a service account (called **automgmt)** which will run the interactions between the jumpbox and GCP. This will also be used later for some ansible we can use to deploy to the cluster.

**On the Jumpbox**

**As: user**

From: https://alex.dzyoba.com/blog/gcp-ansible-service-account/

***

| **Step**                         | **Command**                                                                                                                              | **Notes**                                             |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| Create a GCP IAM Service Account | `gcloud iam service-accounts create automgmt --display-name "Service account for Automation"`                                            | Expected Output: Created service account \[automgmt]. |
| Enable OSLOGIN for your project  | `gcloud compute project-info add-metadata --metadata enable-oslogin=TRUE`                                                                | Expected Output: Updated                              |
| Add roles                        | \`for role in 'roles/compute.instanceAdmin' 'roles/compute.instanceAdmin.v1' 'roles/compute.osAdminLogin' 'roles/iam.serviceAccountUser' |                                                       |
| do                               |                                                                                                                                          |                                                       |

```
gcloud projects add-iam-policy-binding cudos-intercloud --member='serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com' --role="${role}"
```

done\` | the Service account Information (highlighted in green) can be found under IAM → Service Account in the GCP Console

[https://console.cloud.google.com/iam-admin/serviceaccounts](https://console.cloud.google.com/iam-admin/serviceaccounts)\
|\
\| Create a key for the service account | `gcloud iam service-accounts keys create .gcp/gcp-key-automgmt.json [--iam-account=](mailto:--iam-account=automgmt@gke-home-431708.iam.gserviceaccount.com)automgmt@cudos-intercloud.iam.gserviceaccount.com` | This will create GCP key, not the SSH key. This key is used for interacting with Google Cloud API – tools like gcloud, gsutil and others are using it. We will need this key for gcloud to add SSH key to the service account.Should create a key on the local machine under \~/.gcp/gcp-key-automgmt.json |\
\| Create SSH key for the service account | `ssh-keygen -f **ssh-key-automgmt**` | Press Enter twice for passwordless keys |\
\| Add the GCP SSH account to the OSLOGIN service account | `gcloud auth activate-service-account --key-file=.gcp/gcp-key-automgmt.json` | to allow service account to access instances via SSH it has to have SSH key added to it. To do this, first, we have to activate service account in gcloud:

Success: `Activated service account credentials for: [automgmt@cudos-intercloud.iam.gserviceaccount.com]` |\
\| Add the SSH key | `gcloud compute os-login ssh-keys add --key-file=**ssh-key-automgmt.pub**` | Success: `loginProfile`: |\
\| | | Note, that you don’t need to add SSH key to compute metadata, authentication works via OS login. But this means that you need to know a special user name for the service account. |\
\| Find Service Account ID | gcloud iam service-accounts describe \ [ansible-sa@my-gcp-project.iam.gserviceaccount.com](mailto:automgmt@gke-home-431708.iam.gserviceaccount.com) \ --format='value(uniqueId)' | Success: ID Returned `106627723496398399336`Make a note of thisThis id is the OSLOGIN username in the format `sa_<serviceid>` |\
\| Test | `ssh -i **.ssh/ssh-key-automgmt** **sa_106627723496398399336@10.0.0.44**` | Use the ssh private key you generated, the service id in the sa\_ format and a local ip of a GCP VM. |

Complete

### Github Runner Deployment

To have Github Actions run our pipelines we need to install a Self Hosted Github Runner on the jumpbox. this again, is a well defined process.

Access the Github Repository

Untitled

### Terraform Install

As our pipeline will need to run Terraform, Terraform needs to be installed on the jumbox

**On the Jumpbox**

**As: user**

**Reference**: https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli

If your Jumbox is using apt or rpm, then this link will provide instructions to get Terraform added to the repository. These instructions are for the RPI and unsupported (ARCH) installs

| Step                   | Command                                                | Notes |
| ---------------------- | ------------------------------------------------------ | ----- |
| Find the latest Binary | Open https://developer.hashicorp.com/terraform/install |       |

Download the Binary for the architecture you are using

Untitled

| Step                               | Command                                                                                   | Notes                                                 |
| ---------------------------------- | ----------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| Download the latest Binary         | wget -c https://releases.hashicorp.com/terraform/1.9.4/terraform\_1.9.4\_linux\_arm64.zip | Right click on the Download link and choose copy link |
| Unzip the zip file                 | unzip terraform\_1.9.4\_linux\_arm64.zip                                                  |                                                       |
| Move Terraform to a $PATH location | sudo mv terraform /usr/local/bin/                                                         | use show $PATH to see what your path looks like       |
| Test                               | terraform version                                                                         | Success:                                              |
| \`Terraform v1.9.4                 |                                                                                           |                                                       |
| on linux\_arm64\`                  |                                                                                           |                                                       |

Complete

### Install Required Software

**On the Jumpbox**

**As: user**

Install using your package manager the following

* ansible
* git
* wget
* curl
* btop ← (Optional)

### Configure Ansible

Ansible needs to know it should be communicating using the service id account we created earlier `sa_<service id>`

**On the Jumpbox**

**As: user**

**From**: https://alex.dzyoba.com/blog/gcp-ansible-service-account/

***

There is a special variable `ansible_user` that sets user name for SSH when Ansible connects to the host.

In this case, I we could have a group `gcp` where all GCP instances are added, and so can set `ansible_user` in group\_vars like this:

```
# File inventory/dev/group_vars/gcp
ansible_user: sa_106627723496398399336

```

And check it:

```
$ ansible -i inventory/dev gcp -m ping
10.0.0.44 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
10.0.0.43 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

```

And now we have Ansible configured to access GCP instances via OS Login.

### Complete: Pre-Setup

This is the setup which needs to be done, we can now start looking at automation

## Automation: Terraform

### Create a GitHub repo

This process is reliant on a GitHub repo, this can be public or private its up to you, personally i’d make it private.

Open GitHub and head to your repository list

Untitled

Click on **New**

Untitled

* Give the Repo a name (and a description)
* Choose Private (unless you want the repo public)
* Click on Create Repo

On the Jumbox (or wherever you’re going to code this from, laptop?)

Create a location for your code

```
mkdir -p ~/code/home/intercloud-gke-deployment/terraform
cd  ~/code/home/intercloud-gke-deployment
```

Init the Git commands here

```
echo "# gke_deploy" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:mightywomble/gke_deploy.gitgit push -u origin main
```

Remember to change the git remote line to the one on the github page

This will commit your [README.md](http://readme.md/) and future code to git

```jsx
cd  ~/code/home/intercloud-gke-deployment/terraform
```

This is where we will create the code

### Create the Code

Our Terraform code is going to be broken down to 3 files

```jsx
terraform
├── main.tf
├── outputs.tf
└── variables.tf
```

The next section will explain what is going on with this code.

**Note:**

**Terraform is not YAML, so the tabs don’t cause the same level of headache as Ansible YAML if the spacing is out.. the tabs in the code are there to help read the code. Brackets however, they are a different thing.** 😃

#### main.tf

This is the main Terraform file, the one which does all the orchestration and creates everything.

```yaml
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
      version = "5.40.0"
    }
  }
}

module "gke_auth" {
  source = "terraform-google-modules/kubernetes-engine/google//modules/auth"
  version = "31.1.0"
  depends_on   = [module.gke]
  project_id   = var.project_id
  location     = module.gke.location
  cluster_name = module.gke.name
}

resource "local_file" "kubeconfig" {
  content  = module.gke_auth.kubeconfig_raw
  filename = "kubeconfig-${var.env_name}"
}

module "gcp-network" {
  source       = "terraform-google-modules/network/google"
  version      = "9.1.0"
  project_id   = var.project_id
  network_name = "${var.network}-${var.env_name}"

  subnets = [
    {
      subnet_name   = "${var.subnetwork}-${var.env_name}"
      subnet_ip     = "10.10.0.0/16"
      subnet_region = var.region
    },
  ]

  secondary_ranges = {
    "${var.subnetwork}-${var.env_name}" = [
      {
        range_name    = var.ip_range_pods_name
        ip_cidr_range = "10.20.0.0/16"
      },
      {
        range_name    = var.ip_range_services_name
        ip_cidr_range = "10.30.0.0/16"
      },
    ]
  }
}

data "google_client_config" "default" {}

provider "kubernetes" {
  host                   = "https://${module.gke.endpoint}"
  token                  = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(module.gke.ca_certificate)
}

module "gke" {
  source                 = "terraform-google-modules/kubernetes-engine/google//modules/private-cluster"
  version                = "31.1.0"
  project_id             = var.project_id
  name                   = "${var.cluster_name}-${var.env_name}"
  regional               = true
  region                 = var.region
  network                = module.gcp-network.network_name
  subnetwork             = module.gcp-network.subnets_names[0]
  ip_range_pods          = var.ip_range_pods_name
  ip_range_services      = var.ip_range_services_name
  
  node_pools = [
    {
      name                      = "node-pool"
      machine_type              = "e2-medium"
      node_locations            = "europe-west1-b,europe-west1-c,europe-west1-d"
      min_count                 = 1
      max_count                 = 2
      disk_size_gb              = 30
    },
  ]
}

```

#### variables.tf

This is where we declare the variables we use in the main.tf

```yaml
variable "project_id" {
description = "The project ID to host the cluster in"
default     = "cudos-intercloud"
}
variable "cluster_name" {
description = "The name for the GKE cluster"
default     = "home-cluster"
}
variable "env_name" {
  description = "The environment for the GKE cluster"
  default     = "prod"
}
variable "region" {
  description = "The region to host the cluster in"
  default     = "europe-west1"
}
variable "network" {
  description = "The VPC network created to host the cluster in"
  default     = "gke-network"
}
variable "subnetwork" {
  description = "The subnetwork created to host the cluster in"
  default     = "gke-subnet"
}
variable "ip_range_pods_name" {
  description = "The secondary ip range to use for pods"
  default     = "ip-range-pods"
}
variable "ip_range_services_name" {
  description = "The secondary ip range to use for services"
  default     = "ip-range-services"
}
```

#### outputs.tf

This is going to pull down the config file we use with kubectl to manage the cluster

```yaml
output "cluster_name" {
  description = "Cluster name"
  value       = module.gke.name
}
```

This it the code we need, what does it all mean?

### What is the code doing?

#### What is going on here?

#### → main.tf

```yaml
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
      version = "5.40.0"
    }
  }
}
```

**Source**: https://registry.terraform.io/providers/hashicorp/google/latest/docs

This block lets Terraform know how to communicate with Google Cloud, without this the rest of the code wouldn’t function.

The version number in this block can be updated as bugs and functionality are released, the latest version number can be found in the Source link above.

```yaml
module "gke_auth" {
  source = "terraform-google-modules/kubernetes-engine/google//modules/auth"
  version = "24.1.0"
  depends_on   = [module.gke]
  project_id   = var.project_id
  location     = module.gke.location
  cluster_name = module.gke.name
}

resource "local_file" "kubeconfig" {
  content  = module.gke_auth.kubeconfig_raw
  filename = "kubeconfig-${var.env_name}"
}
```

**Source**: [https://github.com/terraform-google-modules/terraform-google-kubernetes-engine/tree/master/modules/auth](https://github.com/terraform-google-modules/terraform-google-kubernetes-engine/tree/master/modules/auth)

In this block you configure the authentication and authorization module for access to the cluster, as well as how to fetch the details for the kubeconfig file which `kubectl` will use.

The items starting with var. refer to the variables presented in **variables.tf**

The version number in this block can be updated as bugs and functionality are released, the latest version number can be found in the Source link above.

```yaml
module "gcp-network" {
  source       = "terraform-google-modules/network/google"
  version      = "6.0.0"
  project_id   = var.project_id
  network_name = "${var.network}-${var.env_name}"

  subnets = [
    {
      subnet_name   = "${var.subnetwork}-${var.env_name}"
      subnet_ip     = "10.10.0.0/16"
      subnet_region = var.region
    },
  ]

  secondary_ranges = {
    "${var.subnetwork}-${var.env_name}" = [
      {
        range_name    = var.ip_range_pods_name
        ip_cidr_range = "10.20.0.0/16"
      },
      {
        range_name    = var.ip_range_services_name
        ip_cidr_range = "10.30.0.0/16"
      },
    ]
  }
}

data "google_client_config" "default" {}
```

**Source**: [https://github.com/terraform-google-modules/terraform-google-network](https://github.com/terraform-google-modules/terraform-google-network)

This module configures the allocation of subnets to the GKE cluster and nodes within the cluster

The GCP networking module creates a separate VPC dedicated to the cluster.

It also sets up separate subnet ranges for the pods and services.

The items starting with var. refer to the variables presented in **variables.tf**

The version number in this block can be updated as bugs and functionality are released, the latest version number can be found in the Source link above.

```yaml
provider "kubernetes" {
  host                   = "https://${module.gke.endpoint}"
  token                  = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(module.gke.ca_certificate)
}

```

While the google provider in the first block provides access to google Cloud,

This code is configuring a Kubernetes provider for Terraform, which allows you to interact with a

Kubernetes cluster using the Kubernetes API.

This code configures a Kubernetes provider for Terraform that allows you to interact with a GKE

cluster using the Kubernetes API.

```yaml
module "gke" {
  source                 = "terraform-google-modules/kubernetes-engine/google//modules/private-cluster"
  version                = "31.1.0"
  project_id             = var.project_id
  name                   = "${var.cluster_name}-${var.env_name}"
  regional               = true
  region                 = var.region
  network                = module.gcp-network.network_name
  subnetwork             = module.gcp-network.subnets_names[0]
  ip_range_pods          = var.ip_range_pods_name
  ip_range_services      = var.ip_range_services_name
  
  node_pools = [
    {
      name                      = "node-pool"
      machine_type              = "e2-medium"
      node_locations            = "europe-west1-b,europe-west1-c,europe-west1-d"
      min_count                 = 1
      max_count                 = 2
      disk_size_gb              = 30
    },
  ]
}

```

**Source**: [https://github.com/terraform-google-modules/terraform-google-kubernetes-engine](https://github.com/terraform-google-modules/terraform-google-kubernetes-engine)

this block is where the magic happens

This GKE module is pulling in the `var.<variable>` data from [`variables.tf`](http://variables.tf/) and using it to build a profile of the type of GKE Cluster to build and where to build it. this data is used to build the plan (of maximum success) which terraform will action to create a GKE Cluster

The items starting with var. refer to the variables presented in **variables.tf**

The version number in this block can be updated as bugs and functionality are released, the latest version number can be found in the Source link above.

There is also nothing to stop the node\_pools section to be variable based

```yaml
  node_pools = [
    {
      name                      = "node-pool"
      machine_type              = "e2-medium"
      node_locations            = "europe-west1-b,europe-west1-c,europe-west1-d"
      min_count                 = 1
      max_count                 = 2
      disk_size_gb              = 30
    },
  ]
```

The reason its not is its possible to add extra node pools in the code

Perhaps you want to add another - CPU optimized node pool to your cluster for your compute hungry applications.\
You can edit the file, and the new node pool as follows:

```yaml
If you want to add a second pool of nodes to your cluster.

module "gke" {
# ...
ip_range_services      = var.ip_range_services_name
node_pools = [
{
name                      = "node-pool"
machine_type              = "e2-medium"
node_locations            = "europe-west1-b,europe-west1-c,europe-west1-d"
min_count                 = 1
max_count                 = 2
disk_size_gb              = 30
},
**{
name                      = "high-cpu-pool"
machine_type              = "n1-highcpu-4"
node_locations            = "europe-west1-b"
min_count                 = 1
max_count                 = 1
disk_size_gb              = 100
}**
]
}
```

**Remember with Terraform if you do this, you don’t need to destroy the existing platform**

**run**

#### → variables.tf

The [va](http://varible.tf/)riables.tf is referenced by the [`main.tf`](http://main.tf/) and provides the user defined information common for the terraform plan to complete.

```yaml
variable "project_id" {
description = "The project ID to host the cluster in"
default     = "cudos-intercloud"
}
variable "cluster_name" {
description = "The name for the GKE cluster"
default     = "intercloud-cluster"
}
variable "env_name" {
  description = "The environment for the GKE cluster"
  default     = "prod"
}
variable "region" {
  description = "The region to host the cluster in"
  default     = "europe-west1"
}
variable "network" {
  description = "The VPC network created to host the cluster in"
  default     = "gke-network"
}
variable "subnetwork" {
  description = "The subnetwork created to host the cluster in"
  default     = "gke-subnet"
}
variable "ip_range_pods_name" {
  description = "The secondary ip range to use for pods"
  default     = "ip-range-pods"
}
variable "ip_range_services_name" {
  description = "The secondary ip range to use for services"
  default     = "ip-range-services"
}
```

Variables here which will need changing

```yaml
variable "project_id" {
description = "The project ID to host the cluster in"
default     = **"cudos-intercloud"**
}
variable "cluster_name" {
description = "The name for the GKE cluster"
default     = **"home-cluster"**
}
```

These variables will provide access to your project and name the cluster.

### Create Workspaces (tfstate)

In order to store a tfstate file for each deployment type centrally on a google Bucket,

We will run our terraform in workspaces

Create and use Terraform workspaces for each environment:

```jsx
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
```

Switch to the appropriate workspace before applying:

```jsx
terraform workspace select dev

```

This will allow the code block below to create tfstate files and store them in a Google Cloud bucket

```jsx
/*
In this resource block, this defines where the tfstate file will be held
This is your tfstate file for the cluster ans will be named dev, staging or prod accordingly.
*/

terraform {
  backend "gcs" {
    bucket = "intercloud-terraform-state"
    prefix = "terraform/state"
  }
}
```

The Bucket needs to be created manually

Create a bucket, called intercloud-terraform-state and use the cheapest options on the wizard

This will create the tfstate files on Google Cloud

### Run the code

**NOTE:** Before you run this make sure you are authenticated to gcloud

```jsx
gcloud auth application-default login
```

Terraform has a process for deployment

It needs to pull down the modules into a local environment which the code (main.tf) states are required, so it can communicate with the right servers

It then needs to check the code will actually build something, this is done by creating a build plan..

If that build plan looks good, then this plan is applied by terraform to build the remote environment.

The commands to do this are:

#### → terraform init

```yaml
terraform init
```

Pulls down the provider and modules needed to plan the build and execute it. when run a folder named `.terraform` is created under the directory the tf code is in, this will contain the `modules` and `providers`

Example:

The providers directory looks like this

```bash
└── providers
└── registry.terraform.io
└── hashicorp
├── google
│   └── 5.40.0
│       └── linux_amd64
├── google-beta
│   └── 5.40.0
│       └── linux_amd64
├── kubernetes
│   └── 2.31.0
│       └── linux_amd64
├── local
│   └── 2.5.1
│       └── linux_amd64
└── random
└── 3.6.2
└── linux_amd64
```

#### → terraform validate

```yaml
terraform validate
```

**Source:** https://myrestraining.com/blog/terraform/what-is-terraform-validate/

Terraform Validate is a built-in command in Terraform.

It checks your Terraform configuration files for syntax errors and invalid values.

It uses the Terraform engine to parse the configuration files and checks for errors that would prevent Terraform from successfully deploying the infrastructure.

The command validates the configuration files in a directory, referring only to the configuration and not accessing any remote services such as remote state, provider APIs, etc

#### → terraform plan

Switch to the appropriate workspace before applying:

```jsx
terraform workspace select dev
```

Then run

```yaml
terraform plan -out=deploy.out
```

**Source:** https://spacelift.io/blog/terraform-plan

Terraform plan is a Terraform CLI command that previews the changes that will be made to the infrastructure based on the current code configuration. It generates and displays an execution plan, detailing which actions will be taken on which resources, allowing for a review before actual application.

This step is crucial for understanding the potential impact of changes and ensuring that they align with intentions, thereby preventing unintended modifications. With `terraform plan`, you will always have a summary at the end, in which you observe the number of resources that will be created/modified/destroyed.

The `-out=` string creates a binary output of the plan which can be used at a later date

#### → terraform apply

```yaml
terraform apply deploy.out
```

**Source**: https://developer.hashicorp.com/terraform/cli/commands/apply

This is where the rubbwer hits the road and the plan is applied to the remote server

When you run `terraform apply` without passing a saved plan file, Terraform automatically creates a new execution plan as if you had run [`terraform plan`](https://developer.hashicorp.com/terraform/cli/commands/plan), prompts you to approve that plan, and takes the indicated actions. You can use all of the [planning modes](https://developer.hashicorp.com/terraform/cli/commands/plan#planning-modes) and [planning options](https://developer.hashicorp.com/terraform/cli/commands/plan#planning-options) to customize how Terraform will create the plan.

You can pass the `-auto-approve` option to instruct Terraform to apply the plan without asking for confirmation.

When you pass a [saved plan file](https://developer.hashicorp.com/terraform/cli/commands/plan#out-filename) to `terraform apply`, Terraform takes the actions in the saved plan without prompting you for confirmation. You may want to use this two-step workflow when [running Terraform in automation](https://developer.hashicorp.com/terraform/tutorials/automation/automate-terraform?utm_source=WEBSITE\&utm_medium=WEB_IO\&utm_offer=ARTICLE_PAGE\&utm_content=DOCS).

Use [`terraform show`](https://developer.hashicorp.com/terraform/cli/commands/show) to inspect a saved plan file before applying it.

When using a saved plan, you cannot specify any additional planning modes or options. These options only affect Terraform’s decisions about which actions to take, and the plan file contains the final results of those decisions.

#### Extra files

Once the plan is run, there will be several other files in the directory with the code.

#### **→ deploy.out**

This was the plan file created when terraform plan was run.

#### **→ terraform.tfstate**

When you run **terraform apply** command to create an infrastructure on cloud, Terraform creates a state file called “**terraform.tfstate**”. This State File contains full details of resources in our terraform code. When you modify something on your code and apply it on cloud, terraform will look into the **state file**, and compare the changes made in the code from that state file and the changes to the infrastructure based on the state file.

**Further Reading:** https://www.easydeploy.io/blog/terraform-state-file/

#### → t**erraform.tfstate.backup**

The terraform.tfstate.backup file is a **backup of the terraform.tfstate file**. Terraform automatically creates a backup of the state file before making any changes to the state file. This ensures that you can recover from a corrupted or lost state file. The terraform.tfstate.backup file is stored in the same directory as the terraform.tfstate file.

**Further Reading:** [https://www.devopsschool.com/blog/what-is-terraform-tfstate-backup-file-in-terraform/](https://www.devopsschool.com/blog/what-is-terraform-tfstate-backup-file-in-terraform/)

#### **→ .terraform.lock.hcl**

This is a dependency lock file created when `terraform init` is run

**Further Reading:** https://developer.hashicorp.com/terraform/language/files/dependency-lock

#### → .gitignore

This file isn’t created by terraform, you will however need one to ensure that you are not pushing the above files to your git repo

The format of the `.gitignore` will look like this

```bash
# Ignore .terraform directories
**/.terraform/*

# Ignore files with .hcl extension
*.hcl

# Ignore Terraform state files
*.tfstate
*.tfstate.backup
```

###

### Push to Git

With the code in place and tested, it should be pushed to the git repository

```bash
cd  ~/code/home/intercloud-gke-deployment/
git add .
git commit -m "Initial Commit"git push
```

This will push your code `terraform/*.tf` files up to your git repo

This will not include any state files, provider locks so the repo will need to have `terraform init, plan` and `apply` run from scratch when pulled down.

## How to manually deploy dev, staging and prod

The [main.tf](http://main.tf) file is paramatised to be a single deployment file, it is fed by Variable files which provide the specifics of each cluster.

Terraform workspaces are used to split the states up when running

Reading: https://developer.hashicorp.com/terraform/language/state/workspaces

```jsx
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
```

There are variable files in the folder vars which are labelled in accordance with the environment they will setup

If we want to setup a dev cluster

Switch to the appropriate workspace before applying:

```jsx
terraform workspace select **dev**
```

Then run

```yaml
terraform plan --var-file=vars/variables-**dev**.tf -out=deploy.out
```

```jsx
terraform apply --var-file=vars/variables-**dev**.tf -out=deploy.out
```

Change Dev accordingly.

## Automation: Github Actions

Github actions allow us to automate the Terraform code (in this example) in a pipeline, our initial pipeline is a simple one, where either a push of code to the git repo or a manual button click will run the Terraform deployment and create or destroy the GKE cluster on Google Cloud

We will then expand the code to have a staging area, which will

* Push the code to Github
* PR the Code
* Create a staging (temporary) GKE Cluster
* Deploy our application to GKE
* Test it
* Push the code to prod with a review
* Tear down the staging GKE
* Deploy the code to prod.

This ensures the code we have written deploys in a like for like environment to production (live) and this is manually confirmed to be working right (automated testing later) as the same terraform code used to create production is used to create the staging environment.

### Setup Github Runner Deployment

To have Github Actions run our pipelines we need to install a Self Hosted Github Runner on the jumpbox. this again, is a well defined process.

Access the Github Repository

Untitled

Click on

1. Settings
2. Actions
3. Runners
4. New Self Hosted Runner

Untitled

Select

1. Linux as your OS
2. Choose the architecture (i’ve used arm64 for the raspberry pi)
3. Follow the download instructions however, we will modify them

**On the Jumpbox**

**As: user**

#### → Install

| **Step**                              | **Command**                                                                                                                                                                                                                | **Notes**                                                                         |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Create a place to run the runner from | `sudo mkdir /opt/actions-gkecd /opt/action-gke`                                                                                                                                                                            | I run multiple runers on this raspberry PI and have them run out of /opt/actions- |
| Download the runner tar.gz            | `sudo curl -o actions-runner-linux-arm64-2.317.0.tar.gz -L [https://github.com/actions/runner/releases/download/](https://github.com/actions/runner/releases/download/)v2.317.0/actions-runner-linux-arm64-2.317.0.tar.gz` | Check the latest version on the above page                                        |
| Decompress the file                   | `sudo tar xzvf actions-runner-linux-arm64-2.317.0.tar.gz`                                                                                                                                                                  |                                                                                   |
| Change ownership                      | `cd ..sudo chown **david:david** -R /opt/actions-gke/cd /opt/action-gke`                                                                                                                                                   | remember to us your username                                                      |

#### → Configure

**On the Jumpbox**

**As: user**

| **Step**                                                                                                                                             | **Command**                                                                                                                                                               | **Notes**                                                                                                           |
| ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Run the config                                                                                                                                       | `./config.sh --url [https://github.com/mightywomble/intercloud-gke-deploymentment](https://github.com/mightywomble/gke-deployment) --token AFA3ARH2HDFRKH3RBSNEZ23GWTOHO` | Run the config command exactly as shown in your GitHub, the key is unique and a 1 time key.                         |
| Enter the name of the runner group to add this runner to: \[press Enter for Default]                                                                 | Press Enter                                                                                                                                                               |                                                                                                                     |
| Enter the name of runner: \[press Enter for manjaro]                                                                                                 | Press Enter                                                                                                                                                               | Change if you feel the need to                                                                                      |
| This runner will have the following labels: 'self-hosted', 'Linux', 'ARM64'Enter any additional labels (ex. label-1,label-2): \[press Enter to skip] | enter `manjaro,gke`Press Enter                                                                                                                                            | Enter the hostname of the server and any other labelsSuccess:√ Runner successfully added√ Runner connection is good |
| Enter name of work folder: \[press Enter for \_work]                                                                                                 | Press Enter                                                                                                                                                               | Success:√ Settings Saved.                                                                                           |
| Install the Service                                                                                                                                  | `sudo ./svc.sh install`                                                                                                                                                   |                                                                                                                     |
| Start the Service                                                                                                                                    | `sudo ./svc.sh start`                                                                                                                                                     | SuccessRun \`systemctl                                                                                              |

Complete

Click on **Settings → runners** and you will see the runner which was just installed, the labels and the **Status** as green

### Create Workflow Identity Pool Permissions

For Github and Google Cloud to play nicely, we need to create a Workflow Identity Pool and associate service accounts with it in order for the Terraform code to run as expected.

I’ve included this as a manual step however its possible to run this as Terraform code from the Jumpbox manually to setup an environment. I’ll include some links at the end to provide guidance on this

These instructions will follow the Google guidelines and create what we need using gcloud-cli

**Note:**

There are a LOT of poorly written guides and videos out there with people assuming (or not understanding themselves) this link is one of the better ones to get this working.

**Source**: https://www.linkedin.com/pulse/deploy-terraform-code-using-github-actions-openid-connect-chandio-brwbf/

**On the Jumpbox**

**As: user**

#### Things we need to know

| What            | Example                                                                                                       | Notes                                     |
| --------------- | ------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| PROJECT\_ID     | cudo-intercloud                                                                                               | This is the ID with the number at the end |
| Service Account | [automgmt@cudos-intercloud.iam.gserviceaccount.com](mailto:automgmt@cudos-intercloud.iam.gserviceaccount.com) | We set this up previously                 |
|                 |                                                                                                               |                                           |
|                 |                                                                                                               |                                           |
|                 |                                                                                                               |                                           |

→ Check the Service Account Exists

```bash
gcloud iam service-accounts list
```

displays (it might display others)

```jsx
DISPLAY NAME                                                           EMAIL                                                                DISABLED
Service account for Automation                                         automgmt@cudos-intercloud.iam.gserviceaccount.com                     False

```

#### Add IAM Roles to the Service Account

There are several Roles we need to add to the service account

```bash
gcloud projects add-iam-policy-binding cudos-intercloud \
  --member="serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com" \
  --role="roles/compute.admin"

gcloud projects add-iam-policy-binding cudos-intercloud \
  --member="serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com" \
  --role="roles/container.admin"

gcloud projects add-iam-policy-binding cudos-intercloud \
  --member="serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com" \
  --role="roles/storage.admin"

gcloud projects add-iam-policy-binding cudos-intercloud \
  --member="serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountAdmin"

gcloud projects add-iam-policy-binding cudos-intercloud \
  --member="serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com" \
  --role="roles/iam.roleAdmin"

gcloud projects add-iam-policy-binding cudos-intercloud \
  --member="serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com" \
  --role="roles/resourcemanager.projectIamAdmin"

gcloud projects add-iam-policy-binding cudos-intercloud \
  --member="serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountTokenCreator"

gcloud projects add-iam-policy-binding cudos-intercloud \
  --member="serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser" 
```

Once done check this with

```bash
gcloud projects get-iam-policy cudos-intercloud
```

This will return the roles bound to the account

```bash
bindings:- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/artifactregistry.admin
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/cloudbuild.builds.editor
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/compute.admin
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/compute.instanceAdmin
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/compute.instanceAdmin.v1
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/compute.osAdminLogin
- members:
- serviceAccount:gke-home-service-account@cudos-intercloud.iam.gserviceaccount.com
- serviceAccount:service-840306196601@compute-system.iam.gserviceaccount.com
role: roles/compute.serviceAgent
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/container.admin
- members:
- serviceAccount:tf-gke-intercloud-clus-09r1@cudos-intercloud.iam.gserviceaccount.com
- serviceAccount:tf-gke-intercloud-clus-49at@cudos-intercloud.iam.gserviceaccount.com
role: roles/container.defaultNodeServiceAccount
- members:
- serviceAccount:gke-home-service-account@cudos-intercloud.iam.gserviceaccount.com
- serviceAccount:service-840306196601@container-engine-robot.iam.gserviceaccount.com
role: roles/container.serviceAgent
- members:
- serviceAccount:service-840306196601@containerregistry.iam.gserviceaccount.com
role: roles/containerregistry.ServiceAgent
- members:
- serviceAccount:840306196601-compute@developer.gserviceaccount.com
- serviceAccount:840306196601@cloudservices.gserviceaccount.com
role: roles/editor
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/iam.roleAdmin
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/iam.serviceAccountAdmin
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/iam.serviceAccountTokenCreator
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/iam.serviceAccountUser
- members:
- serviceAccount:tf-gke-intercloud-clus-09r1@cudos-intercloud.iam.gserviceaccount.com
- serviceAccount:tf-gke-intercloud-clus-49at@cudos-intercloud.iam.gserviceaccount.com
role: roles/monitoring.metricWriter
- members:
- user:fieldymac@gmail.com
role: roles/owner
- members:
- serviceAccount:service-840306196601@gcp-sa-pubsub.iam.gserviceaccount.com
role: roles/pubsub.serviceAgent
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/resourcemanager.projectIamAdmin
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/run.admin
- members:
- serviceAccount:tf-gke-intercloud-clus-09r1@cudos-intercloud.iam.gserviceaccount.com
- serviceAccount:tf-gke-intercloud-clus-49at@cudos-intercloud.iam.gserviceaccount.com
role: roles/stackdriver.resourceMetadata.writer
- members:
- serviceAccount:automgmt@cudos-intercloud.iam.gserviceaccount.com
role: roles/storage.admin
version: 1
```

In the GCP Consule under IAM this looks as follows

image.png

#### Create a Workload Identity Pool

**Pool Name: ghactionspool**

We need to create a Pool for the OIDC Connector to frequent

```bash
gcloud iam workload-identity-pools create ghactionspool \
    --project="cudos-intercloud" \
    --location="global" \
     --display-name="GitHub Actions Pool" \
    --description="An Identity Pool forGithub Action For GKE"
```

Check this with

```bash
gcloud iam workload-identity-pools list --location="global"
```

This should return similar to

```bash
description: The pool to authenticate GitHub actions.
displayName: GitHub Actions Pool
name: projects/84030342246601/locations/global/workloadIdentityPools/ghactionspool
state: ACTIVE
```

In the Console

image.png

#### Full Identity of the Workload Identity Pool

**Keep a note of the result of this, we will use it later**

```bash
gcloud iam workload-identity-pools describe "ghactionspool" --project=cudos-intercloud --location="global" --format="value(name)"
```

displays

```bash
projects/84030342246601/locations/global/workloadIdentityPools/ghactionspool
```

#### Create a Workload Identity Provider

**Provider Name: ghactionsoidc**

Run:

```bash
gcloud iam workload-identity-pools providers create-oidc **ghactionsoidc** \
  --project="cudos-intercloud" \
  --location="global" \
  --workload-identity-pool="ghactionspool" \
  --display-name="My GitHub repo Provider for GKE" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.aud=assertion.aud,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

In the Console, this will look as follows

Edit the OIDC Entry

image.png

Will Show

image.png

and

image.png

#### Allow authentications from the Workload Identity Pool to your Google Cloud Service Account

Prior to starting this, we are going to create some variables in Bash to make life easier

```bash
export PROJECT_ID="cudos-intercloud"
export REPO="cudoventures/intercloud-intercloud-gke-deploymentment"
export WORKLOAD_IDENTITY_POOL_ID="projects/467575788303/locations/global/workloadIdentityPools/ghactionspool"
```

Run the following command

```bash
gcloud iam service-accounts add-iam-policy-binding "automgmt@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${WORKLOAD_IDENTITY_POOL_ID}/attribute.repository/${REPO}"
```

Head to the Google console (IAM)

Click on the **automgmt** Service account

Head over to Permissions and the data will be present here

image.png

#### Extract the Workload Identity Provider resource name

Run the command

```bash
gcloud iam workload-identity-pools providers describe **ghactionsoidc** --location="**global**" --workload-identity-pool="**ghactionspool**"
```

this will display

```bash
attributeCondition: assertion.repository_owner=='cudoventures'
attributeMapping:
  attribute.actor: assertion.actor
  attribute.aud: assertion.aud
  attribute.repository: assertion.repository
  google.subject: assertion.sub
displayName: My GitHub repo Provider for GKE
name: **projects/467575788303/locations/global/workloadIdentityPools/ghactionspool/providers/ghactionsoidc**
oidc:
  issuerUri: https://token.actions.githubusercontent.com
state: ACTIVE
```

Keep a note of the data next to name:

```bash
**projects/467575788303/locations/global/workloadIdentityPools/ghactionspool/providers/ghactionsoidc**
```

This will be the **workload\_identity\_provider** in GitHub Actions

At this point, OIDC should be set up

### Create Github Action to Deploy terraform on the Github runner

Everything is now in place to create the automation which will run the terraform code on our github runner and deploy GKE in our project

#### Create the Workflow folder

The code for Github Actions is contained in a YAML file which is located in a specfici directory which will need to be created

```bash
cd  ~/code/home/intercloud-gke-deployment/
```

create the directory

```bash
mkdir -p .github/workflows
```

#### Workflow code

create a yaml file

```bash
nano ~/code/home/intercloud-gke-deployment/.github/workflows/deployterraform.yml
```

Add the following code

```bash
name: Deploy GKE Production
on:
#  pull_request:
#    # paths: # setup paths if necessary
#    branches:
#      - main
#    types:
#      - opened # default
#      - synchronize # default
#      - reopened # default
#      - closed
  workflow_dispatch:

env:
  WORKING_DIR: terraform/  # relative path under which your terraform codes are

jobs:
  deploy-prod:
    runs-on: gke
    # These permissions are needed too interact with GitHub's OIDC Token endpoint. New
    defaults:
      run:
        working-directory: ${{ env.WORKING_DIR }}
    permissions:
      id-token: write 
      contents: read         
    steps:
      - name: "Checkout"
        uses: actions/checkout@v3
      - name: Configure GCP credentials
        id: auth
        uses: google-github-actions/auth@v2
        with:
          # Value from command: gcloud iam workload-identity-pools providers describe github-actions --workload-identity-pool="github-actions-pool" --location="global"
          workload_identity_provider: "projects/**84030342246601**/locations/global/workloadIdentityPools/ghactionspool/providers/ghactionsoidc"
          create_credentials_file: true
          service_account: "automgmt@cudos-intercloud.iam.gserviceaccount.com"        
          token_format: "access_token"
          access_token_lifetime: "120s"
      - name: Echo stuff
        run: printenv
        
      - name: Setup Node for Terraform
        uses: actions/setup-node@v2
        with:
          node-version: '20'
        
      - name: Setup Terraform with specified version on the runner
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.9.4
          terraform_wrapper: true

      # Initialize a new or existing Terraform working directory by creating initial files, loading any remote state, downloading mo
      - name: Terraform init
        id: init
        run: terraform init
      
      # Checks that all Terraform configuration files adhere to a canonical format
      - name: Terraform Format
        id: fmt
        run:  terraform fmt -recursive -write=true 

      # Validate Code
      - name: Terraform validate
        id: validate
        run: terraform validate

      # Generates an execution plan for Terraform
      - name: Terraform plan
        id: plan
        run: terraform plan 
        continue-on-error: true

      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1

      # On push to "main", build or change infrastructure according to Terraform configuration files
      # Note: It is recommended to set up a required "strict" status check in your repository for "Terraform Cloud". See the documentation on "strict" required status checks for more information: https://help.github.com/en/github/administering-a-repository/types-of-required-status-checks
      - name: Terraform Apply
        run: terraform apply -auto-approve  
```

#### What is this doing?

To breakdown this code

```bash
name: Deploy GKE Production
on:
#  pull_request:
#    # paths: # setup paths if necessary
#    branches:
#      - main
#    types:
#      - opened # default
#      - synchronize # default
#      - reopened # default
#      - closed
  workflow_dispatch:
```

When this code goes live, we want it to run each time code is pushed into the main branch, however while testing `workflow_dispatch` is enabled, this requires a button press in the github actions interface

```bash
env:  
  WORKING_DIR: terraform/  # relative path under which your terraform codes are
```

I’ve set some variables, check the terraform version on the runner using the command `terraform verison` and update this accordingly.

```bash
jobs:
  deploy-prod:
    runs-on: gke
    # These permissions are needed too interact with GitHub's OIDC Token endpoint. New
    defaults:
      run:
        working-directory: ${{ env.WORKING_DIR }}
    permissions:
      id-token: write 
      contents: read      
```

Our first job (things to do) is setting the terraform environment on the runner, specifically which runner to action this workflow on `runs-on` and where the terraform code is `working-directory`

```bash
    steps:
      - name: "Checkout"
        uses: actions/checkout@v3 
```

First step is to check the code in this git repository out on the runner

```bash
      - name: "Checkout"
        uses: actions/checkout@v3
      - name: Configure GCP credentials
        id: auth
        uses: google-github-actions/auth@v2
        with:
          # Value from command: gcloud iam workload-identity-pools providers describe github-actions --workload-identity-pool="github-actions-pool" --location="global"
          workload_identity_provider: "projects/**84030342246601**/locations/global/workloadIdentityPools/ghactionspool/providers/ghactionsoidc"
          create_credentials_file: true
          service_account: "automgmt@cudos-intercloud.iam.gserviceaccount.com"        
          token_format: "access_token"
          access_token_lifetime: "120s"
      - name: Echo stuff
        run: printenv
```

Authenticate the runner using the github cert with the `automgmt` sa we setup

There are two secrets which need to be setup in github (I’ve just not done it here)

WORKLOAD\_IDENTITY\_PROVIDER

SERVICE\_ACCOUNT\_EMAIL

Which we will setup below, its best to have these in the secret vault than in the code.

```bash
      - name: Setup Node for Terraform
        uses: actions/setup-node@v2
        with:          node-version: '20'      - name: Setup Terraform with specified version on the runner
        uses: hashicorp/setup-terraform@v2
        with:          terraform_version: 1.9.4
          terraform_wrapper: true
```

Setting up a node environment and using the hashicorp terraform provider

```bash
# Initialize a new or existing Terraform working directory by creating initial files, loading any remote state, downloading mo
      - name: Terraform init
        id: init
        run: terraform init
      
      # Checks that all Terraform configuration files adhere to a canonical format
      - name: Terraform Format
        id: fmt
        run:  terraform fmt -recursive -write=true 

      # Validate Code
      - name: Terraform validate
        id: validate
        run: terraform validate

      # Generates an execution plan for Terraform
      - name: Terraform plan
        id: plan
        run: terraform plan 
        continue-on-error: true

      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1

      # On push to "main", build or change infrastructure according to Terraform configuration files
      # Note: It is recommended to set up a required "strict" status check in your repository for "Terraform Cloud". See the documentation on "strict" required status checks for more information: https://help.github.com/en/github/administering-a-repository/types-of-required-status-checks
      - name: Terraform Apply
        run: terraform apply -auto-approve  
```

Install our Terraform code

#### GitHub Secrets (Optional)

**Note: while testing, i’ve not implemented this to hide the repo details out of the code. do this if this goes on a public repo**

Open your GitHub repository

Untitled

Click on

→ Settings

→ Secrets and Variables

→ Actions

→ Secrets

→ New Repository Secret

Add the secrets

Untitled

Untitled

Once added these should be listed and ready to run

Untitled

Everything is now in place.

#### Push

Push the code

```bash
cd ~/code/home/intercloud-gke-deployment/
git add .
git commit -m "updated git workflow"git push
```

#### Test

Head to the Actions Tab in the GitHub Repo

Untitled

## Post Deploy

### Kubectl Install

`kubectl` is the command used to interact with the kubernetes interface from the command line, its available in most linux package managers and can be installed using these. I’ve added the commands below for consistency as we have used gcloud to managing GKE

**Note**

this process will download (and overwrite any existing) a file called config to \~/.kube if you have existing kube.config files to reach other K8s clusters, i’d strongly suggest renaming them BEFORE you do this.

#### Install Kubectl

This is covered here

Google Page:https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#install\_plugin

| Step            | Command                           | Notes |
| --------------- | --------------------------------- | ----- |
| Install kubectl | gcloud components install kubectl |       |
| test install    | kubectl version –client           |       |

#### Add gcloud-cli Kubernetes Modules

for the **gcloud-cli** to make use of the GKE Environment, additional modules need to be installed **gke-gcloud-auth-plugin**

This is covered here

Google Page:https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#install\_plugin

The basic steps are:

| **Step**                     | **Command**                                                                                                                                                         | **Notes**                                                                                                                                                                |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Check if module is installed | `gke-gcloud-auth-plugin --version`                                                                                                                                  | This should return not found                                                                                                                                             |
| Install the module           | `gcloud components install gke-gcloud-auth-plugin`                                                                                                                  | This will display the version and size and ask for confirmation (**y**)                                                                                                  |
| Download and install         |                                                                                                                                                                     | This will take about 2 minutes on a RPI                                                                                                                                  |
| Check if Module is installed | `gke-gcloud-auth-plugin --version`                                                                                                                                  | should return with something like Kubernetes v1.28.2-alpha+2291a60496d419da95186fa76128c72fa8e3410d                                                                      |
| Update                       | `gcloud container clusters get-credentials CLUSTER_NAME \ --region=COMPUTE_REGION`Example`gcloud container clusters get-credentials gke-home --region=europe-west2` | Replace the following:CLUSTER\_NAME: the name of your cluster.COMPUTE\_REGION: the Compute Engine region for your cluster. For zonal clusters, use --zone=COMPUTE\_ZONE. |
| Verify the install           | `kubectl get namespaces`                                                                                                                                            | `NAME STATUS AGEdefault Active 51dkube-node-lease Active 51dkube-public Active 51dkube-system Active 51d`                                                                |

There are additional instructions for using kubectl here:

https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#interact\_kubectl

## References

| Site           | Link                                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------- |
| Terraform Code | https://learnk8s.io/terraform-gke                                                                       |
| OIDC           | https://www.linkedin.com/pulse/deploy-terraform-code-using-github-actions-openid-connect-chandio-brwbf/ |
