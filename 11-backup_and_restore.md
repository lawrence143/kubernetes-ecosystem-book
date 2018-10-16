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

### 1.1 Ark - How to take a backup

### 1.2 Ark - How to restore from a backup

## 2. 