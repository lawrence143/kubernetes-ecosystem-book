**The art of knowing is knowing what to ignore.**

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

# Backup tool comparison

## 1. Ark

Heptio Ark is a utility for managing disaster recovery, specifically for your Kubernetes cluster resources and persistent volumes.

It configures with underlying cloud provider easily and can also  take snapshots of persistent volumes.

If you have stateful applications, ark can be the first  take backups.

### 1.1 Ark - Installation

Ark works in client-server approach.

Install the [Ark-Client](https://github.com/heptio/ark/releases).

Install the server side custom resource definitions as mentioned below or [in this repo](https://github.com/heptio/ark/tree/master/examples).

### 1.2 Ark - How to take a backup

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

## 2. 