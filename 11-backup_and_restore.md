**[WIP]**

* [Overview](#Overview)
* [Backup in Kubernetes](#backup-tools)
  * [Ark](#1ark)
    * [Ark Architecture](#11-ark-architecture)
      * [Ark Backup Workflow](#111-ark-backup-workflow)
      * [Ark Scheduled Backups](#112-ark-scheduled-backups)
      * [Ark Restores](#113-ark-backup-restores)
      * [Ark-Sync](#114-ark-sync)
    * [Ark Installation](#12-ark-installation)
      * [Ark Client](#121-ark-client-installation)
      * [Ark Server](#122-ark-server-installation)
    * [Ark AWS Setup](#12-ark-installation)
      * [Ark AWS Configuration](#131-ark-aws-configuration)
      * [Ark AWS Backup](#132-ark-aws-backup)
      * [Ark AWS Restore](#133-ark-aws-restore)
    * [Ark GCP Setup](#12-ark-installation)
      * [Ark GCP Configuration](#141-ark-gcp-configuration)
      * [Ark GCP Backup](#142-ark-gcp-backup)
      * [Ark GCP Restore](#143-ark-gcp-restore)
    * [Ark Azure Setup](#12-ark-installation)
      * [Ark Azure Configuration](#151-ark-azure-configuration)
      * [Ark Azure Backup](#152-ark-azure-backup)
      * [Ark Azure Restore](#153-ark-azure-restore)
  * [etcd](#2-etcd)

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

Ark can back up or restore all objects in the cluster, and can also filter objects by type, namespace, and/or label. Ark is ideal for the disaster recovery use case, as well as for snapshotting application state, before performing system operations on your cluster (e.g. upgrades).

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
| [Portworx][1]                    | Portworx        |
| [DigitalOcean][2]                | StackPointCloud |


### 1.1 Ark Architecture

Ark tool consists of custom-resources like backup,restore, schedules and defined in Kubernetes as Custom-Resource-Definitions(CRD) and then stored in etcd.

Some of the CRD's are 

1. Config
2. Backup
3. Restore
4. Schedule

Config CRD provides core information and options such for cloud provider settings.
Schedule CRD allows to back up data at recurring intervals.
Backup CRD allows to take backup with help of BackupController.
Restore CRD allows to restore all of the objects and persistent volumes from a previously created backup.

During a backup operation:

1. Ark created a tarball of all resources and uploads the Kubernetes objects into cloud object storage.
2. Ark calls the cloud provider API to make disk snapshots of persistent volumes, if specified.

Hooks can also be executed during the backup process. Hooks can work pre and post backups and can do certain tasks before and after taking backups. For example, we might need to tell database to flush in-memory buffers of disk before taking a snapshot. We will take one example of hooks below.

Point to note here is that Ark backups are not strictly atomic. In cases where Kubernetes objects are being created or edited at the time of backup, those objects/resources might not be included in the backup. Though such odds are pretty low, but quite possible in very active clusters.

#### 1.1.1 Ark Backup Workflow

Architecture is shown below. Backup Contoller is created while creating the Ark Server side component using CRDs. We will see this part in the installation section.

<p align="center">
  <img src="img/ark.svg" width="585"> </image>
</p>

Flow of events during backup process:

1. Ark client sends a call to the Kubernetes API server to create a Backup object.
2. Ark Server component BackupController notices creation of new Backup object and performs validation.
3. BackupController begins backup process and collects the data to backup by querying the API server for resources.
4. BackupController makes a call to the object storage service – e.g, AWS S3 – to upload the backup file.

By default, ark backup create makes disk snapshots of any persistent volumes. You can adjust the snapshots by specifying additional flags. See the CLI help for more information. Snapshots can be disabled with the option --snapshot-volumes=false.

#### 1.1.2 Ark Scheduled Backups

We can scedule a backup using a cron and backup can be created at a specified time. Scheduled backups are saved with the name <SCHEDULE NAME>-<TIMESTAMP>, where <TIMESTAMP> is formatted as YYYYMMDDhhmmss.

During creation of backup, we can specify a TTL by adding the flag --ttl <DURATION>. If Ark sees that an existing backup resource is expired, it removes:

1. The backup resource
2. The backup file from cloud object storage
3. All PersistentVolume snapshots
4. All associated Restores

#### 1.1.3 Ark Backup Restores

Restore operation will restore all objects and persistent volumes from a previously created backup. We can also restore only a filtered subset of objects and persistent volumes.

The default name of a restore is <BACKUP NAME>-<TIMESTAMP>, where <TIMESTAMP> is formatted as YYYYMMDDhhmmss. Custom name can also be given to backups. A restored object also includes a label with key ark-restore and value <RESTORE NAME>.

Ark can also be run in restore-only mode, which then disables backup, schedule, and garbage collection functionalities during disaster recovery.

#### 1.1.4 Ark Sync

Ark treats target object storage as the source of truth. It checks continuouslyif the correct backup resources are always present. If there is a properly formatted backup file in the storage bucket, but no corresponding backup resource in the Kubernetes API, Ark synchronizes the information from object storage to Kubernetes.

This allows restore functionality to work in a cluster migration scenario, where the original backup objects do not exist in the new cluster.

### 1.2 Ark Installation

Ark works in client-server approach and it can run in Cloud or on-prem both. Compatible Storage providers are listed above.

#### 1.2.1 Ark-Client Installation

This is the easiest part.

Install the [Ark-Client](https://github.com/heptio/ark/releases) directly as pre-compiled library as per the environment.
And put the client in your PATH environment variable.

You can run the `ark` command to check the client version.

```
[ark@k8s] $ which ark
/usr/local/bin/ark
[ark@k8s] $
[ark@k8s] $ ark version
Version: v0.9.6
Git commit: v0.9.6
Git tree state: clean
[ark@k8s] $ 
```

If you are able to run above command, you have successfully installed the ark client.

#### 1.2.2 Ark-Server Installation

Ark server side installation comprised of CRD's which are present at [ark-github-repo](https://github.com/heptio/ark/tree/master/examples/common)

By default, CRD's will be created in namespace **heptio-ark**. But namespace is configurable. Alongwith CRDs, a namespace, service-account and ClusterRoleBinding resources will also be created.

There is support for [restic][3] configurations also from ark version 0.9.0.

Change the namespace name, ClusterRoleBindings configuration or Service Acccount details for custom installation. Default installation is straight forward. Below is an example for default installation.

```
[ark@k8s] $ kubectl apply -f https://raw.githubusercontent.com/heptio/ark/master/examples/common/00-prereqs.yaml
[ark@k8s] $ kubectl apply -f 00-prereqs.yaml
customresourcedefinition.apiextensions.k8s.io "backups.ark.heptio.com" created
customresourcedefinition.apiextensions.k8s.io "schedules.ark.heptio.com" created
customresourcedefinition.apiextensions.k8s.io "restores.ark.heptio.com" created
customresourcedefinition.apiextensions.k8s.io "configs.ark.heptio.com" created
customresourcedefinition.apiextensions.k8s.io "downloadrequests.ark.heptio.com" created
customresourcedefinition.apiextensions.k8s.io "deletebackuprequests.ark.heptio.com" created
customresourcedefinition.apiextensions.k8s.io "podvolumebackups.ark.heptio.com" created
customresourcedefinition.apiextensions.k8s.io "podvolumerestores.ark.heptio.com" created
customresourcedefinition.apiextensions.k8s.io "resticrepositories.ark.heptio.com" created
customresourcedefinition.apiextensions.k8s.io "backupstoragelocations.ark.heptio.com" created
namespace "heptio-ark" created
serviceaccount "ark" created
clusterrolebinding.rbac.authorization.k8s.io "ark" created
```

As the Ark CRD's are created as above. You can check the CRD resources created using below command.

```
[ark@k8s] $ kubectl get crd -o go-template='{{range .items}}{{if eq .spec.group "ark.heptio.com"}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}'
backups.ark.heptio.com
backupstoragelocations.ark.heptio.com
configs.ark.heptio.com
deletebackuprequests.ark.heptio.com
downloadrequests.ark.heptio.com
podvolumebackups.ark.heptio.com
podvolumerestores.ark.heptio.com
resticrepositories.ark.heptio.com
restores.ark.heptio.com
schedules.ark.heptio.com
```

If you see all the resources above, Ark server side component is installed properly.

With both Ark client and server components installed, let's setup the storage in different cloud vendors to store the backups.

### 1.3 Ark AWS Setup

To do the setup for Ark in AWS, we need to do the follwing steps. 

1. Create S3 Bucket in chosen resion.
2. Create AWS IAM user/role/policies for Ark.
3. Configure Ark CRD resource name Config with details from point 1 and 2.
4. Create Kubernetes Secrete resource for user credentials or configure [Kube2iam][4]

Note: The best practices to keep secrets in Kubernetes can be seen in Security Chapter. We highly recommend to read that chapter and use the tricks here in the examples below.

#### 1.3.1 Ark AWS Configuration

Lets perform the above 4 tasks.

##### 1.3.1.1 Create AWS S3 Bucket

Ark needs a storage to store the backups. S3 is object based storage in AWS and best choice to store backups for Kubernetes cluster resources.

```
[ark@k8s] $ aws s3api create-bucket --bucket heptio-ark-kubernetes-demo --region us-east-1
{
    "Location": "/heptio-ark-kubernetes-demo"
}
[ark@k8s] $ 
[ark@k8s] $ aws s3 ls
2018-10-22 23:53:06 heptio-ark-kubernetes-demo
2018-06-12 15:06:41 elasticbeanstalk-eu-west-2-136335740207
018-10-11 21:13:08 spinnakerforpractice-us-east-1
[ark@k8s] $ 
[ark@k8s] $ aws s3 ls s3://heptio-ark-kubernetes-demo
[ark@k8s] $ 
```

##### 1.3.1.2 Create AWS IAM User

This step is divided in 4 parts and we will perform them one by one.

1. Create Ark User

```
[ark@k8s] $ aws iam create-user --user-name heptio-ark
{
    "User": {
        "UserName": "heptio-ark",
        "Path": "/",
        "CreateDate": "2018-10-22T22:15:34Z",
        "UserId": "AAAAAAAAAAAAAAAAAAAAA",
        "Arn": "arn:aws:iam::xxxxxxxxxxxx:user/heptio-ark"
    }
}
[ark@k8s] $ 
[ark@k8s] $ 
[ark@k8s] $ aws iam list-users | jq -r '.Users[] | select(.UserName=="heptio-ark")'
{
  "UserName": "heptio-ark",
  "Path": "/",
  "CreateDate": "2018-10-22T22:15:34Z",
  "UserId": "AAAAAAAAAAAAAAAAAAAAA",
  "Arn": "arn:aws:iam::xxxxxxxxxxxx:user/heptio-ark"
}
[ark@k8s] $
[ark@k8s] $  
```

2. Attach Policy to Ark user

The ark user needs permissions to get, put, delete, abort and list multipart-uploads on S3.
And also to describe,create tags,volumes and snapshots and to delete snapshots on EC2.

The policy document will like below.

```
[ark@k8s] $ cat heptio-ark-user-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::heptio-ark-kubernetes-demo/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::heptio-ark-kubernetes-demo"
            ]
        }
    ]
}
```

Make sure the arn for s3 bucket i.e. **arn:aws:s3:::heptio-ark-kubernetes-demo** is correct.
**heptio-ark-kubernetes-demo** is S3 bucket, we created in step-1.

We now need to apply this policy to our user.

```
[ark@k8s] $ aws iam put-user-policy --user-name heptio-ark --policy-name heptio-ark --policy-document file://heptio-ark-user-policy.json
[ark@k8s] $ aws iam get-user-policy --policy-name heptio-ark  --user-name heptio-ark
{
    "UserName": "heptio-ark",
    "PolicyName": "heptio-ark",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "ec2:DescribeVolumes",
                    "ec2:DescribeSnapshots",
                    "ec2:CreateTags",
                    "ec2:CreateVolume",
                    "ec2:CreateSnapshot",
                    "ec2:DeleteSnapshot"
                ],
                "Resource": "*",
                "Effect": "Allow"
            },
            {
                "Action": [
                    "s3:GetObject",
                    "s3:DeleteObject",
                    "s3:PutObject",
                    "s3:AbortMultipartUpload",
                    "s3:ListMultipartUploadParts"
                ],
                "Resource": [
                    "arn:aws:s3:::heptio-ark-kubernetes-demo/*"
                ],
                "Effect": "Allow"
            },
            {
                "Action": [
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::heptio-ark-kubernetes-demo"
                ],
                "Effect": "Allow"
            }
        ]
    }
}
```

3. Create Keys for user.

```
[ark@k8s] $ aws iam create-access-key --user-name heptio-ark
{
    "AccessKey": {
        "UserName": "heptio-ark",
        "Status": "Active",
        "CreateDate": "2018-10-22T23:17:36Z",
        "SecretAccessKey": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxcx",
        "AccessKeyId": "yyyyyyyyyyyyyyyyyyyyyyy"
    }
}
[ark@k8s] $ aws iam list-access-keys --user-name heptio-ark
{
    "AccessKeyMetadata": [
        {
            "UserName": "heptio-ark",
            "Status": "Active",
            "CreateDate": "2018-10-22T23:17:36Z",
            "AccessKeyId": "AKIAJUY5QTZGUOQ3IH6Q"
        }
    ]
}
```

4. Create AWS Credentials for user

Create the aws specfic credentials file. We will create Kubernetes secret resource of this file.

```
[ark@k8s] $ cat heptio-ark-credentials
[default]
 aws_access_key_id=yyyyyyyyyyyyyyyyyyyyyyy
 aws_secret_access_key=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

where the access key id and secret are the values returned from the step 3.


##### 1.3.1.3 Create Credentials and configuration

As we already created the CRD's, last step we need to do is to create the secret and then configure AWS Deployment file with correct details

Creating the secrets for aws is straight-forward

```
[ark@k8s] $ kubectl create secret generic cloud-credentials --namespace heptio-ark --from-file heptio-ark-credentials -o yaml --dry-run
apiVersion: v1
data:
  heptio-ark-credentials: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxnNnRnJGWlV1ZE9xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxnhVpCg==
kind: Secret
metadata:
  creationTimestamp: null
  name: cloud-credentials
  namespace: heptio-ark
[ark@k8s] $ 
[ark@k8s] $ kubectl create secret generic cloud-credentials --namespace heptio-ark --from-file heptio-ark-credentials
secret "cloud-credentials" creat
[ark@k8s] $ 
[ark@k8s] $ kubectl get secrets cloud-credentials --namespace heptio-ark
NAME              TYPE      DATA      AGE
cloud-credentials   Opaque    1         1m
```

Now change the Correct Bucket and Region name in the ark config crd and create it.

```
[ark@k8s] $ cat 01.ark-config.yaml
---
apiVersion: ark.heptio.com/v1
kind: Config
metadata:
  namespace: heptio-ark
  name: default
persistentVolumeProvider:
  name: aws
  config:
    region: /<YOUR_REGION>
backupStorageProvider:
  name: aws
  bucket: <YOUR_BUCKET>
  config:
    region: /<YOUR_REGION>
[ark@k8s] $ sed -i '' 's/<YOUR_BUCKET>/heptio-ark-kubernetes-demo/g' 05-ark-backupstoragelocation.yaml
[ark@k8s] $ sed -i '' 's/<YOUR_REGION>/us-east-1/g' 05-ark-backupstoragelocation.yaml
[ark@k8s] $ cat 01.ark-config.yaml
---
apiVersion: ark.heptio.com/v1
kind: Config
metadata:
  namespace: heptio-ark
  name: default
persistentVolumeProvider:
  name: aws
  config:
    region: us-east-1
backupStorageProvider:
  name: aws
  bucket: heptio-ark-kubernetes-demo
  config:
    region: us-east-1
[ark@k8s] $ 
[ark@k8s] $ kubectl  apply -f 01.ark-config.yaml
config.ark.heptio.com "default" created
[ark@k8s] $ kubectl get config
NAME      AGE
default   13s
```

Last step is to get the Deploymenet Resource and create it.

```
[ark@k8s] $ curl -s https://raw.githubusercontent.com/heptio/ark/master/examples/aws/10-deployment.yaml -o 10-deployment.yaml
[ark@k8s] $ kubectl apply -f 10-deployment.yaml
deployment.apps "ark" created
[ark@k8s] $
[ark@k8s] $ kubectl get deployments --namespace heptio-ark
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
ark       1         1         1            1           29s
[ark@k8s] $
[ark@k8s] $ kubectl get pods
NAME                   READY     STATUS    RESTARTS   AGE
ark-5d4bcbdcb7-jvnr2   1/1       Running   0          2m
```

And thats it, our Ark configuration is done.
Now time to use it.

##### 1.3.1.3 Create Roles for Kube2IAM Deployment

[Kube2iam][4] is a Kubernetes application that allows managing AWS IAM permissions for pod via annotations rather than operating on API keys.

To use this backup policy, please install [Kube2iam][4] in your cluster first. Access to the AWS API will be brokered through kube2iam. 

All traffic from pods destined for the AWS API will be redirected to kube2iam. Based on annotations in the pod configurations, kube2iam will make a call to the AWS API to retrieve temporary credentials matching the specified role in the annotation and return these to the caller. All other AWS API calls will be proxied through kube2iam to ensure the principle of least privilege is enforced and policy cannot be bypassed.

These are steps, we need to take before running kube2iam Deployment

1. Create a Trust Policy document to allow the role being used for EC2 management & assume kube2iam role:

```
[ark@k8s] $ cat heptio-ark-trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/<ROLE_CREATED_WHEN_INITIALIZING_KUBE2IAM>"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
[ark@k8s] $ 
```

2. Create IAM Role.

```
[ark@k8s] $ aws iam create-role --role-name heptio-ark --assume-role-policy-document file://./heptio-ark-trust-policy.json
[ark@k8s] $ 
```

3. Create Policy and attach to heptio-ark role.

The ark user needs permissions to get, put, delete, abort and list multipart-uploads on S3.
And also to describe,create tags,volumes and snapshots and to delete snapshots on EC2.

The policy document will like below.

```
[ark@k8s] $ cat heptio-ark-role-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::heptio-ark-kubernetes-demo/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::heptio-ark-kubernetes-demo"
            ]
        }
    ]
}
```


Make sure the arn for s3 bucket i.e. **arn:aws:s3:::heptio-ark-kubernetes-demo** is correct.
**heptio-ark-kubernetes-demo** is S3 bucket, we created in previous section step-1.

We now need to apply this policy to our user.

```
[ark@k8s] $ aws iam put-user-policy --role-name heptio-ark --policy-name heptio-ark-role --policy-document file://heptio-ark-role-policy.json
[ark@k8s] $ 
```

#### 1.3.2 Ark AWS Backup

##### 1.3.2.1 Ark Kuberentes Resources backup

To start the backup, lets create a dummy deployment and take a backup of it. 
```
[ark@k8s] $ kubectl run dummy-nginx-deployment --image=nginx --port 80 --labels=app=nginx,ver=0.1 --namespace heptio-ark --expose=true --port 80
service "dummy-nginx-deployment" created
deployment.apps "dummy-nginx-deployment" created
[ark@k8s] $ 
[ark@k8s] $ kubectl get deployments,pods,services
NAME                                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/ark                      1         1         1            1           8m
deployment.extensions/dummy-nginx-deployment   1         1         1            1           48s

NAME                                          READY     STATUS    RESTARTS   AGE
pod/ark-5d4bcbdcb7-jvnr2                      1/1       Running   0          8m
pod/dummy-nginx-deployment-54b584546f-sk722   1/1       Running   0          48s

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/dummy-nginx-deployment   ClusterIP   172.20.103.43   <none>        80/TCP    48s

```

Dummy resource is created, so lets take a backup.
But before running ark command, lets check if there is something in my bucket.

```
[ark@k8s] $ aws s3 ls s3://heptio-ark-kubernetes-demo
[ark@k8s] $ 
```
There are no resources in the bucket. Lets create a backup.

```
[ark@k8s] $ ark create backup dummy-nginx-backup --selector app=nginx
Backup request "dummy-nginx-backup" submitted successfully.
Run `ark backup describe dummy-nginx-backup` for more details.

[ark@k8s] $  ark backup describe dummy-nginx-backup
Name:         dummy-nginx-backup
Namespace:    heptio-ark
Labels:       <none>
Annotations:  <none>

Phase:  Completed

Namespaces:
  Included:  *
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        <none>
  Cluster-scoped:  auto

Label selector:  app=nginx

Snapshot PVs:  auto

TTL:  720h0m0s

Hooks:  <none>

Backup Format Version:  1

Started:    2018-10-26 12:00:23 +0200 CEST
Completed:  2018-10-26 12:00:31 +0200 CEST

Expiration:  2018-11-25 11:00:23 +0100 CET

Validation errors:  <none>

Persistent Volumes: <none included>
```

```
[ark@k8s] $ aws s3 ls s3://heptio-ark-kubernetes-demo
                           PRE dummy-nginx-backup/
                           PRE nginx6/
2018-10-26 10:32:53        816 05-ark-backupstoragelocation.yaml
[ark@k8s] $ 
```

So far so good. We can create the backup and our backup is present in the S3 bucket. We can list the backup as below.

```
[ark@k8s] $ ark get backups
NAME                 STATUS      CREATED                          EXPIRES   SELECTOR
dummy-nginx-backup   Completed   2018-10-26 12:26:44 +0200 CEST   29d       app=nginx
nginx6               Completed   2018-10-26 12:21:57 +0200 CEST   29d       app=nginx
[ark@k8s] $ 
```

##### 1.3.2.2 Ark Volume Resources backup

To start the volume snapshot backup, we will re-use a dummy deployment and take a backup of it.

First lets create a Storage Class and then a Persistent Volume Claim. We will attach the PVC to our nginx Deployment then.

Create storage class.

```
[ark@k8s] $ cat create_storage_class.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp2
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
mountOptions:
  - debug
[ark@k8s] $ 
[ark@k8s] $ kubectl apply -f create_storage_class.yaml
storageclass "gp2" created
[ark@k8s] $ 
[ark@k8s] $ kubectl get storageclasses
NAME      PROVISIONER             AGE
gp2       kubernetes.io/aws-ebs   83d
```

Create Persistent Volume Claim

```
[ark@k8s] $ cat pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ark-nginx-pvc
spec:
  storageClassName: gp2
  resources:
    requests:
      storage: 1Gi
  accessModes:
    - ReadWriteOnce
[ark@k8s] $ 
[ark@k8s] $ kubectl apply -f pvc.yaml
persistentvolumeclaim "ark-nginx-pvc" created
[ark@k8s] $ 
[ark@k8s] $ kubectl get pvc
NAME            STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ark-nginx-pvc   Bound     pvc-9a9dd9ed-db89-11e8-b2ad-0e44a7170cb2   1Gi        RWO            gp2            30m
[ark@k8s] $ 
```

Use Persistent Volume Claim
```
[ark@k8s] $ kubectl run dummy-nginx-deployment --image=nginx --port 80 --labels=app=nginx,ver=0.1 --namespace heptio-ark --expose=true --port 80 > nginx-deploy.yaml
[ark@k8s] $ 
```

Now edit the file as below to add volume template. You can use any location to mount in container. I took /data for simplicity.

```
[ark@k8s] $ cat nginx-deploy.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
    ver: "0.1"
  name: dummy-nginx-deployment
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
    ver: "0.1"
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
    ver: "0.1"
  name: dummy-nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
      ver: "0.1"
  strategy: {}
  template:
    metadata:
      labels:
        app: nginx
        ver: "0.1"
    spec:
      containers:
      - image: nginx
        name: dummy-nginx-deployment
        ports:
        - containerPort: 80
        volumeMounts:
        - name: gp2-ark
          mountPath: /data/db
      volumes:
      - name: gp2-ark
        persistentVolumeClaim:
          claimName: ark-nginx-pvc

[ark@k8s] $ kubectl apply -f nginx-deploy.yaml
[ark@k8s] $ kubectl get deployments,pods,services,pvc,pv
NAME                                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/ark                      1         1         1            1           3d
deployment.extensions/dummy-nginx-deployment   2         2         2            2           3d

NAME                                          READY     STATUS    RESTARTS   AGE
pod/ark-5d4bcbdcb7-f62xk                      1/1       Running   0          3d
pod/dummy-nginx-deployment-6df6f94d65-dxv2f   1/1       Running   0          32m
pod/dummy-nginx-deployment-6df6f94d65-mbfkm   1/1       Running   0          32m

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/dummy-nginx-deployment   ClusterIP   172.20.103.43   <none>        80/TCP    3d

NAME                                  STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/ark-nginx-pvc   Bound     pvc-9a9dd9ed-db89-11e8-b2ad-0e44a7170cb2   1Gi        RWO            gp2            35m

NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                       STORAGECLASS   REASON    AGE
pvc-9a9...   1Gi        RWO            Retain           Bound     heptio-ark/ark-nginx-pvc    gp2                      35m

```

You can also check aws details. I use awless client to check aws resources.

```
[ark@k8s] $ awless ls volumes --sort NAME
|          ID           |                     NAME ▲                   | TYPE |   STATE   | SIZE | ENCRYPTED | CREATED  |    ZONE    |       INSTANCES       |
|-----------------------|----------------------------------------------|------|-----------|------|-----------|----------|------------|-----------------------|
| vol-07c23f5b83b7d20c6 | kubernetes-dynamic-pvc-9a9dd9ed-db89-.....   | gp2  | in-use    | 1G   | false     | 38 mins  | us-east-1c | [i-06ca939c85e86c353] |
```

Dummy resource and persistent volume are created is created, so lets take a backup.
But before running ark command, lets check if there is something in my bucket.

```
[ark@k8s] $ aws s3 ls s3://heptio-ark-kubernetes-demo
                           PRE dummy-nginx-backup/
                           PRE nginx6/
2018-10-26 10:32:53        816 05-ark-backupstoragelocation.yaml
[ark@k8s] $ 
```
There are no resources in the bucket. Lets create a backup.

```
[ark@k8s] $ ark create backup dummy-nginx-backup --selector app=nginx
Backup request "dummy-nginx-backup" submitted successfully.
Run `ark backup describe dummy-nginx-backup` for more details.

[ark@k8s] $  ark create backup dummy-nginx-backup-1 --selector app=nginx --snapshot-volumes=true
Backup request "dummy-nginx-backup-1" submitted successfully.
Run `ark backup describe dummy-nginx-backup-1` for more details.
[ark@k8s] $ 
[ark@k8s] $ ark backup describe dummy-nginx-backup-1
Name:         dummy-nginx-backup
ark backup describe dummy-nginx-backup-1
Name:         dummy-nginx-backup-1
Namespace:    heptio-ark
Labels:       <none>
Annotations:  <none>

Phase:  Completed

Namespaces:
  Included:  *
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        <none>
  Cluster-scoped:  auto

Label selector:  app=nginx

Snapshot PVs:  true

TTL:  720h0m0s

Hooks:  <none>

Backup Format Version:  1

Started:    2018-10-29 15:53:07 +0100 CET
Completed:  2018-10-29 15:53:15 +0100 CET

Expiration:  2018-11-28 15:53:07 +0100 CET

Validation errors:  <none>

Persistent Volumes:
  pvc-9a9dd9ed-db89-11e8-b2ad-0e44a7170cb2:
    Snapshot ID:        snap-084a86e98cb6ea6bd
    Type:               gp2
    Availability Zone:  us-east-1c
    IOPS:               <N/A>
```

```
[ark@k8s] $ aws s3 ls s3://heptio-ark-kubernetes-demo
                           PRE dummy-nginx-backup-1/
                           PRE dummy-nginx-backup/
                           PRE nginx6/
2018-10-26 10:32:53        816 05-ark-backupstoragelocation.yaml
[ark@k8s] $ 
```

Check AWS for snapshots.
```
[ark@k8s] $ awless ls snapshots
|          ID ▲          |        VOLUME         | ENCRYPTED |    OWNER     |   STATE   | PROGRESS | CREATED | SIZE |
|------------------------|-----------------------|-----------|--------------|-----------|----------|---------|------|
| snap-084a86e98cb6ea6bd | vol-07c23f5b83b7d20c6 | false     | 136335740207 | completed | 100%     | 37 mins | 1G   |
```

So far so good. We can create the backup and our backup is present in the S3 bucket. We can list the backup as below.

```
[ark@k8s] $ ark get backups
NAME                   STATUS      CREATED                          EXPIRES   SELECTOR
dummy-nginx-backup     Completed   2018-10-26 12:26:44 +0200 CEST   26d       app=nginx
dummy-nginx-backup-1   Completed   2018-10-29 15:53:07 +0100 CET    29d       app=nginx
nginx6                 Completed   2018-10-26 12:21:57 +0200 CEST   26d       app=nginx
[ark@k8s] $ 
```

##### 1.3.2.3 Ark Resources backup using kube2iam

In case of kube2iam, we need to install a new deployment for ark as given below.

```
[ark@k8s] $ cat ark-deployment-kube2iam.yaml
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  namespace: heptio-ark
  name: ark
spec:
  replicas: 1
  template:
    metadata:
      labels:
        component: ark
      annotations:
        iam.amazonaws.com/role: arn:aws:iam::<AWS-CUST-ID>:role/<Role-Created-Above-i.e.heptio-ark>
        prometheus.io/scrape: "true"
        prometheus.io/port: "8085"
        prometheus.io/path: "/metrics"
    spec:
      restartPolicy: Always
      serviceAccountName: ark
      containers:
        - name: ark
          image: gcr.io/heptio-images/ark:latest
          ports:
            - name: metrics
              containerPort: 8085
          command:
            - /ark
          args:
            - server
          volumeMounts:
            - name: plugins
              mountPath: /plugins
      volumes:
        - name: plugins
          emptyDir: {}
```

Once deployed, the backup strategy will work the same way as in step 1 and 2.

#### 1.3.3 Ark AWS Restore

##### 1.3.3.1 Ark Resources/Kube2iam restore

To restore from the backup created is very easy. But lets delete the deployment created before and see if restore can put it back.

Delete the deployment

```
[ark@k8s] $ kubectl get deployments,pods
NAME                                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/ark                      1         1         1            1           4h
deployment.extensions/dummy-nginx-deployment   1         1         1            1           2h

NAME                                          READY     STATUS    RESTARTS   AGE
pod/ark-5d4bcbdcb7-f62xk                      1/1       Running   0          2h
pod/dummy-nginx-deployment-54b584546f-sk722   1/1       Running   0          2h

[ark@k8s] $ 
[ark@k8s] $ kubectl delete deployment dummy-nginx-deployment
deployment.extensions "dummy-nginx-deployment" deleted
[ark@k8s] $ 
[ark@k8s] $ kubectl get deployments,pods
NAME                        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/ark   1         1         1            1           4h

NAME                       READY     STATUS    RESTARTS   AGE
pod/ark-5d4bcbdcb7-f62xk   1/1       Running   0          2h

```

Restore the deployment

```
[ark@k8s] $ ark restore create dummy-nginx-backup --from-backup dummy-nginx-backup
Restore request "dummy-nginx-backup" submitted successfully.
Run `ark restore describe dummy-nginx-backup` for more details.
[ark@k8s] $ 
[ark@k8s] $ ark restore describe dummy-nginx-backup
Name:         dummy-nginx-backup
Namespace:    heptio-ark
Labels:       <none>
Annotations:  <none>

Backup:  dummy-nginx-backup

Namespaces:
  Included:  *
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        nodes, events, events.events.k8s.io, backups.ark.heptio.com, restores.ark.heptio.com
  Cluster-scoped:  auto

Namespace mappings:  <none>

Label selector:  <none>

Restore PVs:  auto

Phase:  Completed

Validation errors:  <none>

Warnings:
  Ark:        <none>
  Cluster:    <none>
  Namespaces:
    heptio-ark:   not restored: endpoints "dummy-nginx-deployment" already exists and is different from backed up version.
                  not restored: services "dummy-nginx-deployment" already exists and is different from backed up version.
    kube-system:  not restored: pods "nginx-wh-repo-5cd8ffb865-7qndl" already exists and is different from backed up version.
                  not restored: services "nginx-wh-repo" already exists and is different from backed up version.

Errors:
  Ark:        <none>
  Cluster:    <none>
  Namespaces: <none>
[ark@k8s] $ 
[ark@k8s] $ kubectl get deployments,pods
NAME                                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/ark                      1         1         1            1           4h
deployment.extensions/dummy-nginx-deployment   1         1         1            1           39s

NAME                                          READY     STATUS    RESTARTS   AGE
pod/ark-5d4bcbdcb7-f62xk                      1/1       Running   0          2h
pod/dummy-nginx-deployment-54b584546f-sk722   1/1       Running   0          39s
```

##### 1.3.3.2 Ark Persistent Volume restore

To restore from the backup created is very easy. But lets delete the deployment created before and see if restore can put it back.

Delete the deployment, persistent volume and persistent volume claims.

```
[ark@k8s] $ kubectl delete -f nginx-deploy.yaml
service "dummy-nginx-deployment" deleted
deployment.extensions "dummy-nginx-deployment" deleted
[ark@k8s] $ 
[ark@k8s] $ kubectl delete pvc ark-nginx-pvc
persistentvolumeclaim "ark-nginx-pvc" deleted
[ark@k8s] $ 
[ark@k8s] $ kubectl delete pvc ark-nginx-pvc
[ark@k8s] $ 
[ark@k8s] $ delete pv pvc-9a9dd9ed-db89-11e8-b2ad-0e44a7170cb2
persistentvolume "pvc-9a9dd9ed-db89-11e8-b2ad-0e44a7170cb2" deleted
[ark@k8s] $ 
[ark@k8s] $ kubectl get pods
NAME                   READY     STATUS    RESTARTS   AGE
ark-5d4bcbdcb7-f62xk   1/1       Running   0          4d
[ark@k8s] $ 
[ark@k8s] $ kubectl get pvc
No resources found.
[ark@k8s] $ 
[ark@k8s] $ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                                       STORAGECLASS   REASON    AGE
pvc-81a6e506-9a1b-11e8-95fa-12e0cecfa7b2   1Gi        RWO            Retain           Bound     kube-system/elasticsearch-logging-elasticsearch-logging-0   gp2                      84d
pvc-8dccbbca-9a1b-11e8-95fa-12e0cecfa7b2   1Gi        RWO            Retain           Bound     kube-system/elasticsearch-logging-elasticsearch-logging-1   gp2                      84d
[ark@k8s] $ 
```

Restore the deployment

```
[ark@k8s] $ ark restore create dummy-nginx-backup-2 --from-backup dummy-nginx-backup-1
Restore request "dummy-nginx-backup-2" submitted successfully.
Run `ark restore describe dummy-nginx-backup-2` for more details.
[ark@k8s] $ ark restore describe dummy-nginx-backup-2
Name:         dummy-nginx-backup-2
Namespace:    heptio-ark
Labels:       <none>
Annotations:  <none>

Backup:  dummy-nginx-backup-1

Namespaces:
  Included:  *
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        nodes, events, events.events.k8s.io, backups.ark.heptio.com, restores.ark.heptio.com
  Cluster-scoped:  auto

Namespace mappings:  <none>

Label selector:  <none>

Restore PVs:  auto

Phase:  Completed

Validation errors:  <none>

Warnings:
  Ark:        <none>
  Cluster:    <none>
  Namespaces:
    heptio-ark:   not restored: replicasets.apps "dummy-nginx-deployment-6df6f94d65" already exists and is different from backed up version.
    kube-system:  not restored: pods "nginx-wh-repo-5cd8ffb865-7qndl" already exists and is different from backed up version.
                  not restored: services "nginx-wh-repo" already exists and is different from backed up version.

Errors:
  Ark:        <none>
  Cluster:    <none>
  Namespaces: <none>

```

Check the resources after restoring.

```
[ark@k8s] $ kubectl get pods,svc,pvc,pv
NAME                                          READY     STATUS    RESTARTS   AGE
pod/ark-5d4bcbdcb7-f62xk                      1/1       Running   0          4d
pod/dummy-nginx-deployment-6df6f94d65-dxv2f   1/1       Running   0          11m
pod/dummy-nginx-deployment-6df6f94d65-mbfkm   1/1       Running   0          11m

NAME                             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/dummy-nginx-deployment   ClusterIP   172.20.18.19   <none>        80/TCP    11m

NAME                                  STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/ark-nginx-pvc   Bound     pvc-9a9dd9ed-db89-11e8-b2ad-0e44a7170cb2   1Gi        RWO            gp2            11m

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                                       STORAGECLASS   REASON    AGE
persistentvolume/pvc-81a6e506-9a1b-11e8-95fa-12e0cecfa7b2   1Gi        RWO            Retain           Bound     kube-system/elasticsearch-logging-elasticsearch-logging-0   gp2                      84d
persistentvolume/pvc-8dccbbca-9a1b-11e8-95fa-12e0cecfa7b2   1Gi        RWO            Retain           Bound     kube-system/elasticsearch-logging-elasticsearch-logging-1   gp2                      84d
persistentvolume/pvc-9a9dd9ed-db89-11e8-b2ad-0e44a7170cb2   1Gi        RWO            Retain           Bound     heptio-ark/ark-nginx-pvc                                                             11m
[ark@k8s] $ 
```



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


### 1.3 Ark - How to restore from a backup

## 2. etcd


[1]: https://docs.portworx.com/scheduler/kubernetes/ark.html
[2]: https://github.com/StackPointCloud/ark-plugin-digitalocean
[3]: https://restic.net/
[4]: https://github.com/jtblin/kube2iam