---
title: Getting Started with Terraform
description: Terraform and PingOne
author: Chris Price
tags:
  - terraform
  - pingone
---

[Terraform](https://www.terraform.io) is an Open Source tool from Hashicorp that has become the standard in delivering [Infrastructure-as-Code](https://www.terraform.io/use-cases/infrastructure-as-code).

Through the use of Providers, Terraform allows you to describe a *complete* configuration set, comprised of multiple technologies into a single, standard format [Hashicorp Configuration Language](https://learn.hashicorp.com/collections/terraform/configuration-language)

## Ping and Terraform

There are a few Terraform providers that allow you to configure Ping services.

* [PingOne](https://registry.terraform.io/providers/pingidentity/pingone/latest/docs) (Ping supported)
* [DaVinci](https://registry.terraform.io/providers/samir-gandhi/davinci/latest/docs) (Ping supported)
* [Ping Directory](https://registry.terraform.io/providers/pingidentity/pingdirectory/latest/docs) (Ping supported)
* [PingFederate](https://registry.terraform.io/providers/iwarapter/pingfederate/latest/docs) (Community supported)
* [PingAccess](https://registry.terraform.io/providers/iwarapter/pingaccess/latest/docs) (Community supported)

By combining Providers (along with others, like [Kubernetes](https://registry.terraform.io/providers/hashicorp/kubernetes/latest) and [Helm](https://registry.terraform.io/providers/hashicorp/helm/latest)), you can deploy and manage a the entire configuration needed to support your applications that are integrating with Ping services

## Installing Terraform

Install \ Download the Terraform client from [**here**](https://www.terraform.io/downloads)

On Mac OS, if you have `brew` installed, you can simply type

```zsh
brew install terraform
```

## Start working with Terraform and PingOne

Create a new folder for your configuration

```zsh
mkdir PingOne-Terraform
```

Add a variables file (terraform will detect these and use the values) with something like this:

Filename: `vars.tf`

```hcl
variable "region" {
  type        = string
  description = "Region your P1 Org is in"
}

variable "license_id" {
  type        = string
  description = "License ID to put onto new Environments"
}

variable "env_id" {
  type        = string
  description = "P1 Environment containing the Worker App"
}

variable "worker_id" {
  type        = string
  description = "Worker App ID App - App must have sufficient Roles"
}

variable "worker_secret" {
  type        = string
  description = "Worker App Secret - App must have sufficient Roles"
}

variable "deploy_name" {
  type        = string
  description = "Name used for the deployment"
}
```

Abstract the values of these variables into a `terraform.tfvars` file

```hcl
region = "NorthAmerica"
organization_id = "YourP1OrgId"
license_id = "YourLicenseId"
env_id = "YourWorkerEnvId"
worker_id = "YourWorkerAppId"
worker_secret = "YourWorkerAppSecret"
deploy_name = "YourNewP1EnvName"
```

>**Note:** If you are managing your config in a Git repo, be sure to remove the `terraform.tfvars` from the commit. It contains your **secrets**

Create a new file `pingone.tf` -- this will contain the configuration you want to deploy

```hcl
terraform {
  required_providers {
    pingone = {
      source = "pingidentity/pingone"
      # Set the version you want, or comment \ remove to get the latest
      # version = "~> 0.4"
    }
  }
}

# These values are pulled from the `vars.tf` file if defined
provider "pingone" {
  client_id                    = var.worker_id
  client_secret                = var.worker_secret
  environment_id               = var.env_id
  region                       = var.region
  force_delete_production_type = false
}

# Add a new P1 Environment
resource "pingone_environment" "release_environment" {
  name        = var.deploy_name
  description = "Created by Terraform"
  type        = "PRODUCTION"
  license_id  = var.license_id
  default_population {}
  service {
    type = "SSO"
  }
  service {
    type = "MFA"
  }
  service {
    type = "Risk"
  }
  
  # Comments can be used to document your config, or
  # to remove configuration elements as you test
  
  # service {
  #   type = "Authorize"
  # }
  # service {
  #   type = "DaVinci"
  # }
  # service {
  #   type= "Fraud"
  # }
}

# Add a P1 User
resource "pingone_user" "terraform_user1" {
  environment_id = pingone_environment.release_environment.id
  population_id = pingone_environment.release_environment.default_population_id

  username = "terraform_user1"
  email    = "terraform_user1@mailiantor.com"
}
```

To deploy this configuration, first you need to initilize the folder (this downloads the necessary terraform providers):

```sh
terraform init
```

Then you can ask terraform what it's going to do:

```sh
terraform plan
```

And then you can apply this configuration:

```sh
terraform init
```

You should now see terraform doing _things, and then in PingOne, a new Environment with a User (`terraform_user1`).
