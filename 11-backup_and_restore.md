* [Overview](#Overview)
* [Backup tool comparison](#backup-tool-comparison)
  * [Ark](#1ark)
  * [Ark Installation](#11-ark-installation)

**The art of knowing is, knowing what to ignore.**

# Overview

When you are in a managed environments such as GKE, EKS, OpenShift etc, you don't have issues around restoring cluster state or upgrading to new Kubernetes version. 

But when you’re operating the cluster yourself, biggest challenge comes to Kube-Admins is how to take a backup and then restore without downtimes from backups.

Moreover, when you  are using Kubernetes in a production setup, you would like to take a backup and restore your entire Kubernetes cluster (automatically) with a single action.

There can be numerous other reasons where taking a backup and then restore from it becomes important from business point of view for an organization.

Below are the main reasons to take backup/restore 

* To recover from disasters
* To test upgrades
* To replicate clusters
* To migrate clusters

We can take Kubernetes DB backup which is etcd or we can take Kubernetes Cluster resources backup in the form of YAML files.

# Backup-tools

There are many tools in market and few of them are listed below.

Usage of the tool(s) changes as the strategy of taking backup changes. The setup will be different depending on  underlying cloud provider such as AWS, GCP, Azure etc. We will cover 3 main Cloud providers setup of Ark tool.


## 1.Ark

Ark is a tool from Heptio and is a utility for managing disaster recovery of your Kubernetes Clusters, specifically  cluster resources and persistent volumes. It configures with underlying cloud provider easily and can also  take snapshots of persistent volumes. If you have stateful applications, Ark should be in the list to take backups.

Ark have below capabilites:

* Can take backups of underlying cluster and restore from backup in case of loss.
* Can copy current cluster resources to other clusters.
* Can replicate your production environment for development and testing environments.

Ark have typical client-server setup and have 2 components:

* A server containing Customr-Rsource-Definitions(CRD's) that runs on the server.
* A command-line client that runs locally.

Ark can run in clusters on a cloud or on-prem clusters. It supports many storage providers for backups and snapshot operations. After adding a plugin system in version 0.6.0, users can create their own plugins to add further complexity and logic in order to be compatible with additional backups and volume storage platforms without modifying the Ark codebase.

Below are the Storage Providers supported by Ark officially.

| Provider                  | Owner    | 
|---------------------------|----------|
| AWS S3                    | Ark      |
| Azure Blob Storage        | Ark      |
| Google Cloud Storage      | Ark      |

Below is the list of Ark Snapshot providers

| Provider                         | Owner           |
|----------------------------------|-----------------|
| AWS EBS                          | Ark             |
| Azure Managed Disks              | Ark             |
| Google Compute Engine Disks      | Ark             |
| Restic                           | Ark             |
| [Portworx](1)                    | Portworx        |
| [DigitalOcean](2)                | StackPointCloud |


### 1.1 Ark Architecture

### 1.2 Ark Installation

Ark works in client-server approach and it can run in Cloud or on-prem both. Compatible Storage providers are listed above.

#### 1.2.1 Ark-Client Installation

This is the easiest part.

Install the [Ark-Client](https://github.com/heptio/ark/releases).

Install the server side custom resource definitions as mentioned below or [in this repo](https://github.com/heptio/ark/tree/master/examples).

#### 1.2.2 Ark-Server Installation

### 1.3 Ark AWS Setup

#### 1.3.1 Ark AWS Configuration

#### 1.3.2 Ark AWS Backup

#### 1.3.3 Ark AWS Restore

### 1.4 Ark GCP Setup

#### 1.4.1 Ark GCP Configuration

#### 1.4.2 Ark GCP Backup

#### 1.4.3 Ark GCP Restore

### 1.5 Ark Azure Setup

#### 1.5.1 Ark Azure Configuration

#### 1.5.2 Ark Azure Backup

#### 1.5.3 Ark Azure Restore

*Pre-requisites*

For storing data in AWS S3 bucket.

1. Create S3 bucket in specific region
2. Create S3 user for bucket that user should have S3 bucket full permission.
3. Create Access key & secret key for that user.
4. Change value of **Access_key** and **Secret_key** in file `00.secret.yaml` in aws.
5. Change bucket name & aws region in file `00-ark-config.yaml`.
6. Change AWS account number & username as your aws account no. and  username in file `02.deployment-kube2iam.yaml`.

**Commands**

```
[slamba ◯  WHM0005395  01.Deployment ] ☘   kubectl apply -f crd/01.ark_pre_requisites.yaml
[slamba ◯  WHM0005395  01.Deployment ] ☘   kubectl apply -f aws/.
[slamba ◯  WHM0005395  01.Deployment ] ☘   kubectl apply -f kubernetes-resources/nginx_deployment.yaml
[slamba ◯  WHM0005395  01.Deployment ] ☘   ark backup create nginx-backup --selector app=nginx
[slamba ◯  WHM0005395  01.Deployment ] ☘   kubectl delete -f kubernetes-resources/nginx_deployment.yaml
[slamba ◯  WHM0005395  01.Deployment ] ☘   ark restore create nginx-backup --from-backup nginx-backup

```

### 1.3 Ark - How to restore from a backup

## 2. etcd


[1]: https://docs.portworx.com/scheduler/kubernetes/ark.html
[2]: https://github.com/StackPointCloud/ark-plugin-digitalocean