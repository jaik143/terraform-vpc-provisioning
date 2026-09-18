# AWS Network Provisioning with Terraform

A Terraform root configuration that provisions a three-tier AWS network - public, private and database subnets across two availability zones - by composing a reusable VPC module.

![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat-square&logo=amazonwebservices&logoColor=white)

---

## Before you clone

`vpc.tf` sources its module from a sibling directory that is **not part of this repository**:

    module "vpc" {
      source = "../terraform-aws-vpc"
      ...
    }

`terraform init` will therefore fail on a fresh clone until that module is available at the relative path. This configuration is the caller; the module holds the resource definitions.

The intended fix is to publish the module and reference it by source rather than by relative path:

    module "vpc" {
      source = "git::https://github.com/jaik143/terraform-aws-vpc.git?ref=v1.0.0"
      ...
    }

Pinning to a tag also means an upstream change cannot silently alter what this configuration deploys.

## Network layout

    VPC
    |
    +-- Public     10.0.1.0/24 (az-a)   10.0.2.0/24 (az-b)    -> Internet Gateway
    +-- Private    10.0.11.0/24 (az-a)  10.0.12.0/24 (az-b)   -> NAT Gateway
    +-- Database   10.0.21.0/24 (az-a)  10.0.22.0/24 (az-b)   -> no internet route

Each tier has progressively less reach: public subnets route to the Internet Gateway, private subnets reach the internet outbound only through the NAT Gateway, and the database tier has no internet route at all.

## Why a separate database tier

Splitting the database subnets from the general private tier means they can carry their own route table and network ACLs without touching application subnets, and an RDS subnet group has a dedicated place to live. It costs nothing to separate now and is disruptive to retrofit later.

## Inputs

| Variable | Default | Purpose |
|---|---|---|
| `project_names` | `expense` | Name prefix applied to every resource |
| `common_tags` | Project / Environment / Terraform | Tags merged onto all resources |
| `public_subnet_cidrs` | `["10.0.1.0/24", "10.0.2.0/24"]` | One CIDR per AZ |
| `private_subnet_cidrs` | `["10.0.11.0/24", "10.0.12.0/24"]` | One CIDR per AZ |
| `database_subnet_cidrs` | `["10.0.21.0/24", "10.0.22.0/24"]` | One CIDR per AZ |
| `is_peering_required` | `true` | Feature flag - skip peering in lower environments |

Every value is overridable from a `.tfvars` file, so the same configuration serves dev, staging and production without edits.

## Outputs

Defined in `output.tf` - VPC and subnet identifiers, so downstream stacks (EKS, RDS, load balancers) consume them via remote state instead of hard-coding IDs that change on every rebuild.

## Usage

    terraform init
    terraform fmt -check
    terraform validate
    terraform plan -var-file=dev.tfvars -out=tfplan
    terraform apply tfplan

## Tear down

    terraform destroy -var-file=dev.tfvars

NAT Gateways bill hourly whether or not traffic flows through them - destroy unused environments rather than leaving them running.

---

**Author** - Kadali Jayanth Kumar . [LinkedIn](https://linkedin.com/in/jayanth-kadali-419798182)
