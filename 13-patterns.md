
      [adriano] In this section we're going to cover the design patterns principles and
      # the related available tools currently available from the ecosystem.
      # Feel free to review. Any comment will be appreciated.
      # Thanks.

# Kuberentes Design Patterns
With the adoption of microservices and containers in the recent years, the way we design, develop and run software applications has changed significantly. Modern software applications are optimised for scalability, elasticity, failure, and speed of change. Driven by these new principles, modern applications require a different set of patterns and practices to be applied in an effective way.

Here what we're going to cover:

 * [Baseline Patterns](#baseline-patterns)
      * [Declarative patterns](#declarative-patterns)
        * [Reconciliation Loop](#reconciliation-loop)
        * [Update strategies](#update-strategies)
        * [Blue/Green strategy](#blue/green-strategy)
        * [Canary strategy](#canary-strategy)
      * [Behavorial patterns](#behavorial-patterns)
        * [Batch Jobs](#batch-jobs)
        * [Scheduled Jobs](#scheduled-jobs)
        * [Daemons](#daemons)
      * [Observability patterns](#observability-patterns)
      * [Life Cycle Conformance patterns](#life-cycle-conformance-patterns)
      * [Structure Patterns](#structure-patterns)
      * [Configuration Patterns](#configuration-patterns)
 * [Advanced patterns](#advanced-patterns)
      * [Kubernetes extension points](#kubernetes-extension-points)
      * [Operator pattern](#operator-patterns)

## Baseline Patterns
Baseline patterns refer to the basic principles for building cloud native applications in Kubernetes. Kubernetes adds a new mindset to the software application design by offering a new set of primitives for creating distributed systems spreading across multiple nodes. Having these new primitives, we add a new set of tools to implements software applications, in addition to the already well known tools offered by programming languages and runtimes.

Containers are building blocks for applications running in Kubernetes. From the technical point of view, a container provides packaging and isolation. However, in the context of a distributed application, the container can be described as:

- It addresses a single concern.
- It is has its own release cycle.
- It defines and carries its own build time dependencies.
- It is immutable and once it is built, it does not change.
- It has a well defined set of APIs to expose its functionality.
- It runs as a single well behaved process.
- It is parameterised for the different environments and different use cases.

Having small and modular reusable containers leads to create a set of standard tools, similarly to a good reusable library provided by a programming language or runtime.

Containers are designed to run only a single process per container, unless the process itself spawns child processes. Running multiple unrelated processes in a single container, leads to keep all those processes up and running, manage their logs, their interactions, and their healtiness. For example, we have to include a mechanism for automatically restarting individual processes if they crash. Also, all those processes would log to the same standard output, so we'll have hard time figuring out which process logged what.

Some wrong practices to avoid:

- Using a process management system such as supervisord to manage multiple processes in the same container.
- Using a bash script to spawn several processes as background jobs in the same container.

Unfortunately, it's not uncommon to find these practices into public images. Please, do not follow them!

In Kuberentes, multiple containers can be grouped into pods. Containers in a pod are deployed together, and are started, stopped, and replicated as a group. When a pod contains multiple containers, all of them are always run on a single node, it never spans multiple nodes.

The simplest pod definition describes the deployment of a single container as in the following descriptor

```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: nginx
        namespace:
        labels:
          run: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:latest
          ports:
          - containerPort: 80
```

All containers inside the same pod can share the same set of resources, e.g. network and process namespaces. This allows the containers in a pod to interact each other through networking via localhost, or inter-process communication mechanisms, if desired. Kubernetes achieves this by configuring all containers in the same pod to use the same set of Linux namespaces, instead of each container having its own set. They can also share the same PID namespace, but that isn’t enabled by default.

On the other side, multiple containers in the same pod cannot share the file system because the container’s filesystem comes from the container image, and by default, it is fully isolated from other containers. However, multiple containers in the same pod can share some host file folders called volumes.

For example, the following code snippet describes a pod with two containers using a shared volume to comminicate each other

```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: nginx
        namespace:
        labels:
          run: nginx
      spec:
        containers:
        - name: main
          image: nginx:latest
          ports:
          - containerPort: 80
          volumeMounts:
          - name: html
            mountPath: /usr/share/nginx/html
        - name: supporting
          image: busybox:latest
          volumeMounts:
          - name: html
            mountPath: /mnt
          command: ["/bin/sh", "-c"]
          args:
            - while true; do
                date >> /mnt/index.html;
                sleep 10;
              done
        volumes:
        - name: html
          emptyDir: {}
```

The first container running a ``nginx`` server, is called ``main`` and it is serving a static webpage created dynamically by a second container called ``supporting``. The main container has a shared volume called ``html`` mounted to the directory ``/usr/share/nginx/html``. The supporting container has the shared volume mounted to the directory ``/mnt``. Every ten seconds, the supporting container adds the current date and time into the ``index.html`` file, which is located in the shared volume. When the user makes an HTTP request to the pod, the nginx server reads this file and transfers it back to the user in response to the request.

All containers in a pod are being started in parallel and there is no way to define that one container must be started after other container. To deal with dependencies and startup order, Kubernetes introduces the Init Containers, which start first and sequentially, before the main and the other supporting containers in the same pod.

For example, the following describes a pod with one main container and an init container using a shared volume to comminicate each other

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace:
  labels:
spec:
  initContainers:
  - name: prepare-html
    image: busybox:latest
    command: ["/bin/sh", "-c", "echo 'Hello World from '$POD_IP'!' > /tmp/index.html"]  
    env:
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    volumeMounts:
    - name: content-data
      mountPath: /tmp
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
    - name: content-data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: content-data
    emptyDir: {}
```

The main requirement of the pod above is to reply to user requests with a greeting message containing the IP address of the pod. Because the IP address of a pod is only known after the pod started, we need to get the IP before the main container. This is the sequence of events happening here:

1. The pod is created and it is scheduled on a given node.
2. The IP address of the pod is assigned.
3. The init container starts and gets the IP address from the APIs server.
4. The init container creates a simple html file containing the pod's IP and places it into the shared volume.
5. The init container exits
6. The main container starts, reads this file and transfers it back to the user in response to requests.

A pod may have any number of init containers. They are executed sequentially and only after the last one completes with success, then the main container and all the other supporting containers are started in parallel.

A reasonably sized microservices based application will consist of multiple containers. Containers, often, have dependencies among themselves, dependencies to the host, and resource requirements. The resources available on a cluster also can vary over time. The way we place containers also impacts the availability, the performances, and the capacity of the distributed systems.

In Kubernetes, assigning pods to nodes is done by the scheduler. Generally, the users leave the scheduler to do its job without constraints. However, it might be required introduce a sort of forcing to the scheduler in order to achieve a better resource usage or meet some application's requirements.

### Declarative patterns
The notion of declarative approach is one of the primary drivers behind the development of Kubernetes. In the declarative way, the user declares the state in order to produce a result. He is not going to to tell the system: "*do this!, do that!*" as in the imperative way. The power of the declarative way is that the user is giving the system a declaration of the desired state of the world. The system, in turn, understand the desired state and takes autonomous actions, independently of the user interaction. This means also that the system can implement self-healing behaviors.

#### Reconciliation loop
The magic behind the declarative approach is based on a set of reconciliation loops implemented through dedicated controllers. The controller is continually repeating the following steps:

1. Get the desired state
2. Observe the actual state
3. Compare matched state with the desired state
4. Take actions

For example, a user might say to Kubernetes, "*I desire to have three replicas of my web application running at all times.*" So, the desired state is to have three pods up and runing. Kubernetes takes that declarative statement and takes responsibility for ensuring that it is always true. It compares the actual number of pods running with the desired one. If just enough, no action is taken. If too many, then it deletes the excess pods, else if too few, it creates additional pods to reconcile the desired state with the actual one.

To make sure your pods are controlled by a reconciliation loop, you need to have the pods managed by a Replication Controller or a Replica Set. For example, the following code snippet describes a set of three pods controlled by a Replica Set 

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  labels:
  namespace:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:latest
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
```

#### Update strategies
Having a growing number of microservices, the continuos delivery process with manual updating and replacing services with newer versions becomes quickly inpractical. Updating an application to a newer version involves activities such as stopping gracefully the old version, starting the new version, waiting and checking if it has started successfully, and, sometimes, rolling-back to the previous version in the case of issues.

This set of operations can be made manually or automatically by Kuberentes itself. The way provided by Kuberentes for support a declarative pattern for software application deployment is called Deployment.

#### Update Strategy
In the declaration of our desired state of the world, we can specify the update strategy:

 * **Rolling:** removes existing pods, while adding new ones at the same time, keeping the application available during the process and ensuring there is no out of service.
 * **Recreate:** all existing pods are removed before new ones are created.

The following snippet reports a rolling update strategy

```yaml
...
strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
...
```

while the following reports a recreate update strategy

```yaml
...
strategy:
    type: Recreate
...
```

We can use the Deployment as a building block, together with other primitives, to implement more advanced release strategies such as *Blue/Green* and *Canary Release* deployments.

#### Blue/Green strategy
The Blue/Green is a release strategy used for deploying software applications in production environment by minimising the downtime. In kuberentes, a Blue/Green can be implemented by creating a new deploy object for the new version of the application (Green) which are not serving any requests yet. At this stage, the old deploy object (Blue) is still running and serving live requests. Once we are confident that the new version is healthy and ready to serve live requests, we switch the traffic from the Blue deploy to the Green. Once the Green deploy has handled all the requests, the Blue deploy can be deleted and resources can be reutilized.

#### Canary strategy
Canary is a release strategy for softly deploying a new version of an application in production by replacing an only small subset of old instances with the new ones. This reduces the risk of introducing a new version into production by letting only a subset of users to reach the new version. After a given time window of observation about how the new version behaves, we can replace all the old instances with the new version.

### Behavorial patterns

#### Batch Jobs
In kubernetes, a Job is an abstraction for create batch processes. A job creates one or more pods and ensures that a given number of them successfully complete. When all pod complete, the job itself is complete. Deleting a job will remove all the pods it created.

#### Scheduled Jobs
In kubernetes, a Cron Job is a time based scheduled job. A cron job runs a job periodically on a given schedule, written in standard Unix cron format.

#### Daemons
In kuberentes, a Daemon Set is a controller type ensuring each node in the cluster runs a pod. As new node is added to the cluster, a new pod is added to the node. As the node is removed from the cluster, the pod running on it is removed and not scheduled on another node. Deleting a Daemon Set will clean up all the pods it created.

### Observability patterns

- Container Healt Check
- Liveness Probe
- Readiness Probe

### Life Cycle Conformance patterns

- Pod temination
- Life cycle hooks: post-start and pre-stop

### Structure Patterns

- Sidecar
- Initializer
- Ambassador
- Adapter

### Configuration Patterns

- Environment variables
- Config Maps
- Secrets

## Advanced patterns

### Kubernetes extension points

- Services Catalog
- APIs aggregation server
- Custom Resources Definition
- Custom Controllers

### Operator pattern

- Platform as Code
- Operator framework
- Operator Lifecycle Manager
- Kubebuilder
- Metacontroller
