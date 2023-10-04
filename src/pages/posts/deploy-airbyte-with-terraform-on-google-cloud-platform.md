---
description: Deploy Airbyte with Terraform on Google Cloud Platform
slug: deploy-airbyte-with-terraform-on-google-cloud-platform
title: Deploy Airbyte with Terraform on Google Cloud Platform
createdAt: 1695647703625
updatedAt: 1696405383799
tags:
  - Google Cloud
  - Airbyte
  - Terraform
heroImage: /posts/deploy-airbyte-with-terraform-on-google-cloud-platform_thumbnail.png
public: true
layout: ../../layouts/BlogPost.astro
---
Airbyte has quickly become one of the most popular data ingestion tools on the market and it's easy to understand why once you start to get familiar with the tool. Airbyte has hundreds of connectors out of the box, and for the connectors that don't exist, you have the option to create your own. 

In this post I will cover how to deploy Airbyte to a Compute Engine instance in Google Cloud Platform. The goal is to help you get started as seamlessly as possible and we'll work through the practical example of installing Airbyte as explained in their documentation [here](https://docs.airbyte.com/deploying-airbyte/on-gcp-compute-engine/), using Terraform to provision everything in Google Cloud. 

The benefit of using Terraform to provision your deployment is that the work you put in will be easily replicable for anyone else who wants to do the same setup.

If you've ever tried to setup Airbyte you know that it's very easy to do and requires zero effort just to get started if you just follow the documentation from start to finish, no matter what method/ provider you choose (AWS, Azure, Google Cloud Platform, Kubernetes etc.). 

But using a free tool comes with the responsibility of deploying the service to your own environment and Airbytes documentation doesn't go trough all aspects of deploying the tool. Especially the security aspect of running Airbyte on a server which is left to the user to interpret themselves.

If this is your first time setting up Airbyte maybe you will find yourself wondering what you should do next when you see this message at the end of the documentation:

![Screenshot 2023-10-03 at 11.17.22](/posts/deploy-airbyte-with-terraform-on-google-cloud-platform_screenshot-2023-10-03-at-11-17-22.jpg)
The steps shown in the documentation is not meant to show how you get a production ready Airbyte instance up and running. It only goes trough how you get Airbyte up and running on a VM in the cloud and that is exactly why you should proceed with caution before running your Airbyte installation on a publicly exposed VM.

## Architecture

In this setup we will use a shielded VM with no public IP address and set up security measures for both ingress and egress traffic to the instance. To be able to access the VM as a user we will set up the correct IAM roles and tunnel the ssh traffic through Identity Aware Proxy which is a authenticating layer that only lets trough authorised users. 

We will also setup a firewall rule to only let in IAP authorised users to our VPC and Subnet. For egress traffic we will setup Cloud NAT and Cloud Router with a static IP so our VM will be able to make outbound calls for example package downloads and have access to Googles internal services.

![Deploy Airbyte with Terraform on Google Cloud Platform](/posts/deploy-airbyte-with-terraform-on-google-cloud-platform_deploy-airbyte-with-terraform-on-google-cloud-platform.png)

1. The user clicks the SSH button in the Google Cloud console or use the command `gcloud compute ssh <INSTANCE> --zone <ZONE> --tunnel-through-iap --project <PROJECT_ID>`
2. The IAP connection authenticates and authorises the user if they have the IAM role `IAP-secured Tunnel User`.
3. We have created a VPC (Virtual Private Cloud) with a single subnetwork where our instance resides.
4. A shielded VM with no public IP address.
5. Firewall to allow SSH connection from IAP. You can read more about it[here](https://cloud.google.com/iap/docs/using-tcp-forwarding).
6. A Cloud NAT gateway and Cloud router.(It uses the Border Gateway Protocol which advertises fixed IP ranges and serves as a control plane for cloud NAT.)
7. Allow internal access for Google Services like BigQuery and Cloud Storage.
8. Outbound internet access for package downloads.

## Prerequisites

### Create a Gmail or Google Workspace account

To be able to create a Google Cloud Platform account you will need to have a Gmail or Google Workspace account. Get started by signing up [here](https://accounts.google.com/signup).

### Create a Google Cloud account and enable billing

Create your account by signing in to Google Cloud Platform [here](https://cloud.google.com/). All the projects you create in Google Cloud Platform needs to be linked to a billing account. After you have created your account you need to [enable billing](https://cloud.google.com/billing/docs/how-to/modify-project). 

Follow this [guide](https://cloud.google.com/billing/docs/how-to/find-billing-account-id) in the documentation to locate your `Billing account ID` or see my method explained below for the easiest way to find your id. The `Billing account ID` is presented the following format: `010101-F0FFF0-10XX01`. You will need this value in the next step so make sure you save it in a secure place.

The easiest way to find your `Billing account ID` is to go to the go to the [Account management](https://console.cloud.google.com/billing/manage) page for the Cloud Billing account:

![ba-id-account-management-page](/posts/deploy-airbyte-with-terraform-on-google-cloud-platform_ba-id-account-management-page.png)

### Install Google Cloud CLI 

Install Google Cloud SDK following the instructions [here](https://cloud.google.com/sdk/docs/quickstart) for your respective OS. After you have the `gcloud` CLI installed, run the following command in a terminal window and follow the instructions. This will let Terraform use the default credentials for authentication.

```shell
gcloud auth application-default login
```

### Fork or clone this repository locally 

You can fork this repository to your account or clone it to your local machine. To clone the repository, run the following: 

```shell
git clone https://github.com/hampussanden/airbyte-terraform-google-cloud-platform
cd airbyte-terraform-google-cloud-platform
```

To fork this repository follow the steps in the GitHub documentation which you can find [here](https://docs.github.com/en/get-started/quickstart/fork-a-repo).

### Optional: Create a Google Cloud Project and Service account with shell script

If you want to do things step by step you can go ahead and jump to the next section and copy paste the commands into your terminal manually.

If you want to save some time you can run the `service_account.sh` script located in the `/bin` folder of this project.

First start by creating a .env file with the following variables:

```shell
# The Google Cloud project id
PROJECT_ID=""
# The Google Cloud project name
PROJECT_NAME=""
# The Billing account ID from the first step
BILLING_ACCOUNT=""
```

Then run the script:

```shell
sh ./bin/service_account.sh
```

### Create a Google Cloud Project and link your Billing account ID

1. Create a new project
2. Set the project
3. Link your Billing account ID to the project

```shell
# Create the project
gcloud projects create <PROJECT_ID> --name=<PROJECT_NAME>

# Set the correct project variable
gcloud config set project <PROJECT_ID>

# Link your Billing account ID to the project
gcloud billing projects link <PROJECT_ID> --billing-account=<BILLING_ACCOUNT_ID>
```

### Create a Google Cloud Service Account

1. Create a service account.
2. Create a service account key and download the JSON file as service_account.json to your root directory.
4. Grant the role Editor for the service account for now. We will give the service account the right roles later.

```shell
# Create the service account
gcloud iam service-accounts create <PROJECT_NAME> \
  --description="Service Account to use with Terraform"

# Create the key file
gcloud iam service-accounts keys create service_account.json \
  --iam-account=<PROJECT_NAME>@<PROJECT_ID>.iam.gserviceaccount.com

# Grant the Editor role
gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member=serviceAccount:terraform@<PROJECT_ID>.iam.gserviceaccount.com \
  --role=roles/editor
  ```

### Install Terraform

Follow the instructions [here](https://learn.hashicorp.com/tutorials/terraform/install-cli) to install the Terraform CLI locally. Run the following command afterward to check your installation: 

```shell
terraform -v
```

You should see the following output:

```shell
Terraform v1.5.7
on darwin_arm64
+ provider registry.terraform.io/hashicorp/google v4.84.0
```


### Optional: Download and install the IAP Desktop on your local machine. 

It is possible to access your VM via IAP Desktop if you are on Windows or Linux. Click [here](https://github.com/GoogleCloudPlatform/iap-desktop) to download IAP Desktop and follow the steps in the documentation to login via IAP Desktop. 

You could also access your VM via SSH in your terminal which will be the option for everyone who is using MacOS and I will be showing you later in the post how you do this.

### Create a `terraform.tfvars` file

Create a `terraform.tfvars` file in your root directory with the following variables:

```shell
# The Billing account ID from the first step
billing_id = ""
# The Google Cloud project id
project_id = ""
# The Google Cloud project name
project_name = ""
# The Google Cloud region
region = ""
# The Google Cloud zone
zone = ""
```


## Getting started

### Project overview

As a part of this project, the following resource will be deployed with the Terraform.

```
tree

.
├── bin
│   ├── airbyte.sh
│   └── service_account.sh
├── main.tf
├── provider.tf
├── service_account.json
└── variables.tf
```

/bin/airbyte.sh

```shell
#! /bin/bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common wget
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add --
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian buster stable"
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod -a -G docker $USER

sudo apt-get -y install docker-compose-plugin
docker compose version

mkdir airbyte && cd airbyte
wget https://raw.githubusercontent.com/airbytehq/airbyte/master/run-ab-platform.sh
chmod +x run-ab-platform.sh
./run-ab-platform.sh -b

EOF
```

main.tf

```shell
# Enable APIs
resource "google_project_service" "api_services" {
  project = var.project_id
  for_each = toset(
    [
      "compute.googleapis.com",
      "cloudresourcemanager.googleapis.com",
    ]
  )
  service                    = each.key
  disable_on_destroy         = false
  disable_dependent_services = true
}

# Set IAM permissions for the service account
resource "google_project_iam_member" "project" {
  for_each = toset([
    "roles/iam.serviceAccountUser",
    "roles/run.admin",
    "roles/logging.admin",
    "roles/bigquery.jobUser",
    "roles/bigquery.dataEditor",
    "roles/iap.tunnelResourceAccessor"
  ])
  role       = each.value
  project    = var.project_id
  member     = "serviceAccount:${var.project_name}@${var.project_id}.iam.gserviceaccount.com"
  depends_on = [google_project_service.api_services]
}

# Create a VPC
resource "google_compute_network" "vpc" {
  name                    = "airbyte-network"
  auto_create_subnetworks = "false"

}

# Create a Subnet
resource "google_compute_subnetwork" "subnet" {
  name                     = "airbyte-subnet"
  ip_cidr_range            = "10.10.0.0/24"
  network                  = google_compute_network.vpc.name
  region                   = var.region
  private_ip_google_access = true # Allow access to internal Google services
}

## Create a VM in the above subnet

resource "google_compute_instance" "airbyte-instance" {
  project                 = var.project_id
  zone                    = var.zone
  name                    = "airbyte-instance"
  machine_type            = var.machine_type
  metadata_startup_script = file("./bin/airbyte.sh")

  boot_disk {
    initialize_params {
      image = "debian-10-buster-v20230912"
    }
  }

  network_interface {
    network    = "airbyte-network"
    subnetwork = google_compute_subnetwork.subnet.name # Replace with a reference or self link to your subnet, in quotes
  }

  # Enable Shielded VM features
  shielded_instance_config {
    enable_secure_boot = true
    enable_vtpm        = true
  }

  service_account {
    scopes = [
      "cloud-platform",
    ]
    email = "${var.project_name}@${var.project_id}.iam.gserviceaccount.com"
  }
}

# Create a firewall to allow SSH connection from the specified source range
resource "google_compute_firewall" "rules" {
  project = var.project_id
  name    = "allow-ssh"
  network = "airbyte-network"

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }
  source_ranges = ["35.235.240.0/20"]
  depends_on    = [google_compute_network.vpc]
}

## Create Cloud Router

resource "google_compute_router" "router" {
  project    = var.project_id
  name       = "airbyte-router"
  network    = "airbyte-network"
  region     = var.region
  depends_on = [google_compute_network.vpc]
}

## Create Nat Gateway

resource "google_compute_router_nat" "nat" {
  name                               = "airbyte-router-nat"
  router                             = google_compute_router.router.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}
```

variables.tf

```shell
variable "region" {
  default     = "europe-west1"
  description = "Google Cloud Platform Region"
  type        = string
}

variable "zone" {
  default     = "europe-west1-b"
  description = "Google Cloud Platform Zone"
  type        = string
}

variable "machine_type" {
  default = "e2-medium"
  description = "Google Cloud Platform Machine Type"
  type = string
}

variable "project_id" {
  default     = "tcb-project-371706-400508"
  description = "Google Cloud Platform Project ID"
  type        = string
}

variable "project_name" {
  default     = "tcb-project-371706"
  description = "Google Cloud Platform Project Name"
  type        = string
}

variable "billing_account" {
  description = "Google Cloud Platform Billing Account ID"
  type        = string
}
```

provider.tf

```shell
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "4.49.0"
    }
  }
}

provider "google" {
  credentials = file("service_account.json")

  project = var.project_id
  region  = var.region
  zone    = var.zone
}
```

### Deploy your project with Terraform

Before you start make sure once again that you have the right Google Cloud project set.

```shell
# Set the correct project variable
gcloud config set project <PROJECT_ID>
```

Initialize the backend, validate, and format your configuration.

```shell
terraform init

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/google from the dependency lock file
- Using previously-installed hashicorp/google v4.49.0

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

```shell
terraform validate
Success! The configuration is valid.
```

```shell
terraform fmt
main.tf
provider.tf
```

Now go on and create your resources.

```shell
terraform apply

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # google_compute_firewall.rules will be created
  + resource "google_compute_firewall" "rules" {
      + creation_timestamp = (known after apply)
      + destination_ranges = (known after apply)
      + direction          = (known after apply)
      + enable_logging     = (known after apply)
      + id                 = (known after apply)
      + name               = "allow-ssh"
      + network            = "airbyte-network"
      + priority           = 1000
      + project            = "tcb-project-371706-400508"
      + self_link          = (known after apply)
      + source_ranges      = [
          + "35.235.240.0/20",
        ]

      + allow {
          + ports    = [
              + "22",
            ]
          + protocol = "tcp"
        }
    }

  # google_compute_instance.airbyte-instance will be created
  + resource "google_compute_instance" "airbyte-instance" {
      + can_ip_forward          = false
      + cpu_platform            = (known after apply)
      + current_status          = (known after apply)
      + deletion_protection     = false
      + guest_accelerator       = (known after apply)
      + id                      = (known after apply)
      + instance_id             = (known after apply)
      + label_fingerprint       = (known after apply)
      + machine_type            = "e2-medium"
      + metadata_fingerprint    = (known after apply)
      + metadata_startup_script = <<-EOT
            #! /bin/bash
            sudo apt-get update
            sudo apt-get install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common wget
            curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add --
            sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian buster stable"
            sudo apt-get update
            sudo apt-get install -y docker-ce docker-ce-cli containerd.io
            sudo usermod -a -G docker $USER
            
            sudo apt-get -y install docker-compose-plugin
            docker compose version
            
            mkdir airbyte && cd airbyte
            wget https://raw.githubusercontent.com/airbytehq/airbyte/master/run-ab-platform.sh
            chmod +x run-ab-platform.sh
            ./run-ab-platform.sh -b
            
            EOF
        EOT
      + min_cpu_platform        = (known after apply)
      + name                    = "airbyte-instance"
      + project                 = "tcb-project-371706-400508"
      + self_link               = (known after apply)
      + tags_fingerprint        = (known after apply)
      + zone                    = "europe-west1-b"

      + boot_disk {
          + auto_delete                = true
          + device_name                = (known after apply)
          + disk_encryption_key_sha256 = (known after apply)
          + kms_key_self_link          = (known after apply)
          + mode                       = "READ_WRITE"
          + source                     = (known after apply)

          + initialize_params {
              + image  = "debian-10-buster-v20230912"
              + labels = (known after apply)
              + size   = (known after apply)
              + type   = (known after apply)
            }
        }

      + network_interface {
          + ipv6_access_type   = (known after apply)
          + name               = (known after apply)
          + network            = "airbyte-network"
          + network_ip         = (known after apply)
          + stack_type         = (known after apply)
          + subnetwork         = "airbyte-subnet"
          + subnetwork_project = (known after apply)
        }

      + service_account {
          + email  = "tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com"
          + scopes = [
              + "https://www.googleapis.com/auth/cloud-platform",
            ]
        }

      + shielded_instance_config {
          + enable_integrity_monitoring = true
          + enable_secure_boot          = true
          + enable_vtpm                 = true
        }
    }

  # google_compute_network.vpc will be created
  + resource "google_compute_network" "vpc" {
      + auto_create_subnetworks         = false
      + delete_default_routes_on_create = false
      + gateway_ipv4                    = (known after apply)
      + id                              = (known after apply)
      + internal_ipv6_range             = (known after apply)
      + mtu                             = (known after apply)
      + name                            = "airbyte-network"
      + project                         = (known after apply)
      + routing_mode                    = (known after apply)
      + self_link                       = (known after apply)
    }

  # google_compute_router.router will be created
  + resource "google_compute_router" "router" {
      + creation_timestamp = (known after apply)
      + id                 = (known after apply)
      + name               = "airbyte-router"
      + network            = "airbyte-network"
      + project            = "tcb-project-371706-400508"
      + region             = "europe-west1"
      + self_link          = (known after apply)
    }

  # google_compute_router_nat.nat will be created
  + resource "google_compute_router_nat" "nat" {
      + enable_dynamic_port_allocation      = (known after apply)
      + enable_endpoint_independent_mapping = true
      + icmp_idle_timeout_sec               = 30
      + id                                  = (known after apply)
      + name                                = "airbyte-router-nat"
      + nat_ip_allocate_option              = "AUTO_ONLY"
      + project                             = (known after apply)
      + region                              = "europe-west1"
      + router                              = "airbyte-router"
      + source_subnetwork_ip_ranges_to_nat  = "ALL_SUBNETWORKS_ALL_IP_RANGES"
      + tcp_established_idle_timeout_sec    = 1200
      + tcp_transitory_idle_timeout_sec     = 30
      + udp_idle_timeout_sec                = 30

      + log_config {
          + enable = true
          + filter = "ERRORS_ONLY"
        }
    }

  # google_compute_subnetwork.subnet will be created
  + resource "google_compute_subnetwork" "subnet" {
      + creation_timestamp         = (known after apply)
      + external_ipv6_prefix       = (known after apply)
      + fingerprint                = (known after apply)
      + gateway_address            = (known after apply)
      + id                         = (known after apply)
      + ip_cidr_range              = "10.10.0.0/24"
      + ipv6_cidr_range            = (known after apply)
      + name                       = "airbyte-subnet"
      + network                    = "airbyte-network"
      + private_ip_google_access   = true
      + private_ipv6_google_access = (known after apply)
      + project                    = (known after apply)
      + purpose                    = (known after apply)
      + region                     = "europe-west1"
      + secondary_ip_range         = (known after apply)
      + self_link                  = (known after apply)
      + stack_type                 = (known after apply)
    }

  # google_project_iam_member.project["roles/bigquery.dataEditor"] will be created
  + resource "google_project_iam_member" "project" {
      + etag    = (known after apply)
      + id      = (known after apply)
      + member  = "serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com"
      + project = "tcb-project-371706-400508"
      + role    = "roles/bigquery.dataEditor"
    }

  # google_project_iam_member.project["roles/bigquery.jobUser"] will be created
  + resource "google_project_iam_member" "project" {
      + etag    = (known after apply)
      + id      = (known after apply)
      + member  = "serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com"
      + project = "tcb-project-371706-400508"
      + role    = "roles/bigquery.jobUser"
    }

  # google_project_iam_member.project["roles/iam.serviceAccountUser"] will be created
  + resource "google_project_iam_member" "project" {
      + etag    = (known after apply)
      + id      = (known after apply)
      + member  = "serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com"
      + project = "tcb-project-371706-400508"
      + role    = "roles/iam.serviceAccountUser"
    }

  # google_project_iam_member.project["roles/iap.tunnelResourceAccessor"] will be created
  + resource "google_project_iam_member" "project" {
      + etag    = (known after apply)
      + id      = (known after apply)
      + member  = "serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com"
      + project = "tcb-project-371706-400508"
      + role    = "roles/iap.tunnelResourceAccessor"
    }

  # google_project_iam_member.project["roles/logging.admin"] will be created
  + resource "google_project_iam_member" "project" {
      + etag    = (known after apply)
      + id      = (known after apply)
      + member  = "serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com"
      + project = "tcb-project-371706-400508"
      + role    = "roles/logging.admin"
    }

  # google_project_iam_member.project["roles/run.admin"] will be created
  + resource "google_project_iam_member" "project" {
      + etag    = (known after apply)
      + id      = (known after apply)
      + member  = "serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com"
      + project = "tcb-project-371706-400508"
      + role    = "roles/run.admin"
    }

  # google_project_service.api_services["cloudresourcemanager.googleapis.com"] will be created
  + resource "google_project_service" "api_services" {
      + disable_dependent_services = true
      + disable_on_destroy         = false
      + id                         = (known after apply)
      + project                    = "tcb-project-371706-400508"
      + service                    = "cloudresourcemanager.googleapis.com"
    }

  # google_project_service.api_services["compute.googleapis.com"] will be created
  + resource "google_project_service" "api_services" {
      + disable_dependent_services = true
      + disable_on_destroy         = false
      + id                         = (known after apply)
      + project                    = "tcb-project-371706-400508"
      + service                    = "compute.googleapis.com"
    }

Plan: 14 to add, 0 to change, 0 to destroy.
```

Type yes at the confirmation prompt to proceed.

```shell
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

google_project_service.api_services["compute.googleapis.com"]: Creating...
google_project_service.api_services["cloudresourcemanager.googleapis.com"]: Creating...
google_compute_network.vpc: Creating...
google_project_service.api_services["compute.googleapis.com"]: Creation complete after 4s [id=tcb-project-371706-400508/compute.googleapis.com]
google_project_service.api_services["cloudresourcemanager.googleapis.com"]: Creation complete after 4s [id=tcb-project-371706-400508/cloudresourcemanager.googleapis.com]
google_project_iam_member.project["roles/bigquery.jobUser"]: Creating...
google_project_iam_member.project["roles/iap.tunnelResourceAccessor"]: Creating...
google_project_iam_member.project["roles/bigquery.dataEditor"]: Creating...
google_project_iam_member.project["roles/run.admin"]: Creating...
google_project_iam_member.project["roles/logging.admin"]: Creating...
google_project_iam_member.project["roles/iam.serviceAccountUser"]: Creating...
google_compute_network.vpc: Still creating... [10s elapsed]
google_project_iam_member.project["roles/logging.admin"]: Creation complete after 9s [id=tcb-project-371706-400508/roles/logging.admin/serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com]
google_project_iam_member.project["roles/run.admin"]: Creation complete after 9s [id=tcb-project-371706-400508/roles/run.admin/serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com]
google_project_iam_member.project["roles/iap.tunnelResourceAccessor"]: Creation complete after 10s [id=tcb-project-371706-400508/roles/iap.tunnelResourceAccessor/serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com]
google_project_iam_member.project["roles/bigquery.jobUser"]: Still creating... [10s elapsed]
google_project_iam_member.project["roles/iam.serviceAccountUser"]: Still creating... [10s elapsed]
google_project_iam_member.project["roles/bigquery.dataEditor"]: Still creating... [10s elapsed]
google_project_iam_member.project["roles/bigquery.dataEditor"]: Creation complete after 10s [id=tcb-project-371706-400508/roles/bigquery.dataEditor/serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com]
google_project_iam_member.project["roles/bigquery.jobUser"]: Creation complete after 10s [id=tcb-project-371706-400508/roles/bigquery.jobUser/serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com]
google_project_iam_member.project["roles/iam.serviceAccountUser"]: Creation complete after 11s [id=tcb-project-371706-400508/roles/iam.serviceAccountUser/serviceAccount:tcb-project-371706@tcb-project-371706-400508.iam.gserviceaccount.com]
google_compute_network.vpc: Still creating... [20s elapsed]
google_compute_network.vpc: Creation complete after 22s [id=projects/tcb-project-371706-400508/global/networks/airbyte-network]
google_compute_router.router: Creating...
google_compute_subnetwork.subnet: Creating...
google_compute_firewall.rules: Creating...
google_compute_router.router: Still creating... [10s elapsed]
google_compute_subnetwork.subnet: Still creating... [10s elapsed]
google_compute_firewall.rules: Still creating... [10s elapsed]
google_compute_router.router: Creation complete after 12s [id=projects/tcb-project-371706-400508/regions/europe-west1/routers/airbyte-router]
google_compute_router_nat.nat: Creating...
google_compute_firewall.rules: Creation complete after 12s [id=projects/tcb-project-371706-400508/global/firewalls/allow-ssh]
google_compute_subnetwork.subnet: Still creating... [20s elapsed]
google_compute_router_nat.nat: Still creating... [10s elapsed]
google_compute_subnetwork.subnet: Creation complete after 23s [id=projects/tcb-project-371706-400508/regions/europe-west1/subnetworks/airbyte-subnet]
google_compute_instance.airbyte-instance: Creating...
google_compute_router_nat.nat: Creation complete after 12s [id=tcb-project-371706-400508/europe-west1/airbyte-router/airbyte-router-nat]
google_compute_instance.airbyte-instance: Still creating... [10s elapsed]
google_compute_instance.airbyte-instance: Creation complete after 13s [id=projects/tcb-project-371706-400508/zones/europe-west1-b/instances/airbyte-instance]

Apply complete! Resources: 14 added, 0 changed, 0 destroyed.
```

## Connect to your newly created instance

You need to wait for a couple of minutes before trying to connect to the newly created Airbyte instance. It usually takes up to five minutes for the Airbyte installation to complete.

Check the logs to know when Airbyte has started running:

https://console.cloud.google.com/logs/query?project=your-project-id

It should look something like this:

```shell
INFO 2023-10-03T10:21:17.878746610Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-bootloader Exited
INFO 2023-10-03T10:21:17.879152970Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-api-server Starting
INFO 2023-10-03T10:21:17.880451603Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-bootloader Exited
INFO 2023-10-03T10:21:17.905235219Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-server Starting
INFO 2023-10-03T10:21:17.905937244Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-bootloader Exited
INFO 2023-10-03T10:21:17.906478309Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-webapp Starting
INFO 2023-10-03T10:21:17.911063683Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-bootloader Exited
INFO 2023-10-03T10:21:17.911454010Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-connector-builder-server Starting
INFO 2023-10-03T10:21:19.146705888Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-worker Started
INFO 2023-10-03T10:21:19.244694598Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-api-server Started
INFO 2023-10-03T10:21:19.577588680Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-cron Started
INFO 2023-10-03T10:21:19.879973789Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-webapp Started
INFO 2023-10-03T10:21:19.900369538Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-server Started
INFO 2023-10-03T10:21:19.900807719Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-proxy Starting
INFO 2023-10-03T10:21:19.973532894Z [resource.labels.instanceId: airbyte-instance] startup-script: Container airbyte-connector-builder-server Started
```

You can either use the IAP desktop app that we downloaded before or login via SSH in your terminal with the command below:

```bash
gcloud compute ssh <INSTANCE> --zone <ZONE> \
  --tunnel-through-iap --project <PROJECT_ID> \
  -- -L 8000:localhost:8000 -N -f
```

The first time you do this you will be asked to create an SSH key that is stored locally on your computer. Write a password for the key and keep it in a password manager.

```shell
Enter passphrase for key '/Users/hampussanden/.ssh/google_compute_engine':
```

### Log into Airbyte

Go to http://localhost:8000/ and you will get a prompt asking for a user name and password. To login for the first time you use the default user name `airbyte` and password `password` which is explained in the [quickstart](https://github.com/airbytehq/airbyte/blob/master/docs/quickstart/deploy-airbyte.md) documentation.

screenshot

Once you have logged in you need to provide and email address and organisation name for the account. You also have the option to anonymise data collection.

![Screenshot 2023-10-02 at 10.27.58](/posts/deploy-airbyte-with-terraform-on-google-cloud-platform_screenshot-2023-10-02-at-10-27-58.jpg)

And now we have Airbyte running on a secure Compute Engine instance.

![Screenshot 2023-10-02 at 10.28.22](/posts/deploy-airbyte-with-terraform-on-google-cloud-platform_screenshot-2023-10-02-at-10-28-22.jpg)

## Clean up

To avoid any unwanted cost incurred, be sure to clean up the resources created in this project by running.

```shell
terraform destroy
```

**Warning:** This will delete any persisted data and resources in the project. Alternatively, you can turn off the unused Compute Engine instance to save costs as well.

Use the following command to suspend your instance or see the [documentation](https://cloud.google.com/compute/docs/instances/suspend-resume-instance).

```shell
 gcloud compute instances suspend <INSTANCE>
```


## FAQ

### How much will this setup cost?

Here is the [Google Compute Engine pricing](https://cloud.google.com/compute/all-pricing/) for calculation.

### What are the risks of having a VM with public IP?


## References

- https://github.com/danilo-nzyte/airbyte-terraform-gcp/tree/main
- https://registry.terraform.io/providers/hashicorp/google/latest/docs
- https://cloud.google.com/docs/terraform/get-started-with-terraform
- https://dev.to/alvardev/gcp-cloud-functions-with-a-static-ip-3fe9
- https://alphasec.io/3-tips-to-secure-your-gcp-vm-instance/
- https://medium.com/@larry_nguyen/comparing-google-private-access-and-cloud-nat-cc43ddc9ce61
- https://www.seblu.de/2021/12/iap-bypass.html?m=1
- https://johansiebens.dev/posts/2020/12/control-access-to-your-on-prem-services-with-cloud-iap-and-inlets-pro/
- https://cloud.google.com/iap/docs/concepts-overview
- https://medium.com/@larry_nguyen/use-identity-aware-proxy-iap-instead-of-bastion-host-to-connect-to-private-virtual-machines-in-9885bc7c12dd



