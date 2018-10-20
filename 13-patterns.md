
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
        * [Container Healt Check](#container-healt-check)
        * [Liveness Probe](#liveness-probe)
        * [Readiness Probe](#readiness-probe)
        * [Self Awareness](#self-awarness)
      * [Life Cycle Conformance patterns](#life-cycle-conformance-patterns)
        * [Pod temination](#pod-temination)
        * [Life cycle hooks](#self-awarness)
      * [Structure Patterns](#structure-patterns)
        * [Sidecar](#sidecar)
        * [Initializer](#initializer)
        * [Ambassador](#ambassador)
        * [Adapter](#adapter)
      * [Configuration Patterns](#configuration-patterns)
        * [Environment variables](#environment-variables)
        * [Config Maps](#config-maps)
        * [Secrets](#secrets)
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
      [more to add here]

#### Batch Jobs
In kubernetes, a Job is an abstraction for create batch processes. A job creates one or more pods and ensures that a given number of them successfully complete. When all pod complete, the job itself is complete. Deleting a job will remove all the pods it created.

#### Scheduled Jobs
In kubernetes, a Cron Job is a time based scheduled job. A cron job runs a job periodically on a given schedule, written in standard Unix cron format.

#### Daemons
In kuberentes, a Daemon Set is a controller type ensuring each node in the cluster runs a pod. As new node is added to the cluster, a new pod is added to the node. As the node is removed from the cluster, the pod running on it is removed and not scheduled on another node. Deleting a Daemon Set will clean up all the pods it created.

### Observability patterns
Today, it's an accepted concept that software applications can have failures and the chances for failure increases even more when working with distributed applications. The modern approach shifted from "*be obssesed by preventing failures*" to "*detect failures and take correttive actions*". 

To support this pattern, Kubernetes provides:

  * Container Healt Check
  * Liveness Probe
  * Readiness Probe
  * Self Awarness

#### Container Healt Check
The container health check is the check that the Kuberentes agent running on the nodes (called *kubelet*) constantly performs on the containers. The ``restartPolicy`` property of the pod controls how kubelet behaves when a container exits

  * **Always**: always restart an exited container (default)
  * **OnFailure**: restart the container only when exited with failure
  * **Never**: never restart the container

#### Liveness Probe
When an application runs into some deadlock or out-of-memory conditions, it is still be considered healthy from the container health check, so the agent is not taking any action. To detect this kind of issues and any other failures more related to the application logic, kubernetes introduces the **Liveness Probe**.

A liveness probe is a regular checks performed by the kubelet on the container to confirm it is still healthy. We can specify a liveness probe for each container in the pod’s specification. Kubernetes will periodically execute the probe and restart the container if the probe fails.

Kubernetes probes a container liveness using one of the three ways:

  * **HTTP**: performs an http request on the container’s IP address, a port and a path. If the probe receives a response, and the response is not an http error, the probe is considered successful.
  * **TCP**: tries to open a tcp socket on the container’s IP address and a port. If the connection is established successfully, then the probe is considered successful.
  * **EXEC**: execs an arbitrary command against the container and checks the exit status code. If the status code is 0, then the probe is considered successful.
  
For example, the following pod descriptor defines a liveness probe for a ``nginx`` container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace:
  labels:
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
        scheme: HTTP
      initialDelaySeconds: 30
      timeoutSeconds: 10
      periodSeconds: 5
      failureThreshold: 1
```

The pod descriptor above defines an HTTP liveness probe, which tells Kubernetes to periodically perform a http requests on the root path and port 80 to check if the container is still healthy. These requests start after 30 seconds after the container is running. The frequency of the probe is set to 5 seconds and the timout is set to 10 seconds before to declare the probe unsuccessful.

#### Readiness Probe
Pods are included as endpoints of a service. As soon as a new pod is created, it becomes part of the service and requests start to be sent to the pod. The pod may need time to load configuration and data, or it may need some time to perform a startup procedure before the first user request can be served. It makes sense to not forward user's requests to a pod that is in still in the process of starting up until it is fully ready.

To detect if a pod is ready to serve user's requests, kubernetes introduces the **Readiness Probe**. The readiness probe is invoked periodically and determines whether the specific pod should receive user's requests or not. When a readiness probe returns success, it is meaning that the container is ready to accept requests and then kuberentes add the pod as endpoint to the service.

Kubernetes probes a container readiness using one of the three ways:

  * **HTTP**: performs an http request on the container’s IP address, a port and a path. If the probe receives a response, and the response is not an http error, the probe is considered successful.
  * **TCP**: tries to open a tcp socket on the container’s IP address and a port. If the connection is established successfully, then the probe is considered successful.
  * **EXEC**: execs an arbitrary command against the container and checks the exit status code. If the status code is 0, then the probe is considered successful.
  
For example, the following pod descriptor defines a liveness probe for a ``mysql`` container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  namespace:
  labels:
    run: mysql
spec:
  containers:
  - name: mysql
    image: mysql:5.6
    env:
    - name: MYSQL_ALLOW_EMPTY_PASSWORD
      value: "1"
    ports:
    - name: mysql
      protocol: TCP
      containerPort: 3306
    readinessProbe:
      exec:
        # Check we can execute queries over TCP
        command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
      initialDelaySeconds: 30
      timeoutSeconds: 10
      periodSeconds: 5
      failureThreshold: 1
```

The pod descriptor above defines an exec readiness probe, which tells Kubernetes to periodically perform a SQL query against the ``mysql`` server to check if the container is ready to serve requests. These requests start after 30 seconds after the container is running. The frequency of the probe is set to 5 seconds and the timout is set to 10 seconds before to declare the probe unsuccessful.

To check how a readiness probe affects services, create a ``mysql`` service as in the following descriptor 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace:
spec:
  ports:
  - port: 3306
    protocol: TCP
    targetPort: 3306
  type: ClusterIP
  selector:
    run: mysql
```

and check the endpoints update on the pod creation.

#### Self Awareness
There are many situations where applications need to know information about the environment where they are running into. That may include information that is known only at runtime such as the pod name, pod IP, namespace, the host name or other metadata.

Such information can be required in many scenarios, for example, we want to tune the application thread pool size, or the memory consumption algorithm. We may want to use the pod name and the host name while logging, or while sending metrics to a centralized location. We may want to discover other pods in the same namespace with a specific label and join them into a clustered application, etc.

In kuberentes, all the cases above can be addressed by querying the APIs server from the pod itself. Pods use service accounts to authenticate against the APIs server. The authentication token used by the service account is passed to any pod running in kuberentes and mounted as secret.

For example, the following pod descriptor implements an API call to read the pod namespace and put it into a pod environment variable

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-web-app
  namespace:
  labels:
    app:nodejs
spec:
  containers:
  - name: nodejs
    image: kalise/nodejs-web-app:latest
    ports:
    - containerPort: 8080
    env:
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: MESSAGE
      value: "Hello $(POD_NAMESPACE)"
  serviceAccount: default
```

The pod above uses the default service account. Such service account is created by kubernetes with a limited set of permissions. In case we want our service account to have more permissions, we can give them such permissions or create a dedicated service account with the required permissions.

Some of the information Kuberentes allows you to pass to the container:

- The pod’s name
- The pod’s IP address
- The namespace the pod belongs to
- The name of the node the pod is running on
- The name of the service account the pod is running under
- The CPU and memory requests for each container
- The CPU and memory limits for each container
- The pod’s labels
- The pod’s annotations

### Life Cycle Conformance patterns
Microservices based applications require a more fine grained interactions and life cycle management capabilities for a better user experience. Some of these applications require a start up procedure while other need a gentle and clean shut down procedure. For these and other use cases, kubernetes provides a set of tools to help the management of the application life cycle.

#### Pod temination
Containers can be terminated at any time, due to an autoscaling policy, node failure, pod deletion or while rolling out an update. In most of such cases, we need a graceful shutdown of the processes running into the containers.

When a pod is deleted, a SIGTERM signal is sent to the main process (PID 1) in each container, and a grace period timer starts (defaults to 30 seconds). Upon the receival of the SIGTERM signal, each container starts a graceful shutdown of the running processes and exit. If a container does not terminate within the grace period, a SIGKILL signal is sent to the container for a forced termination.

The default grace period is 30 seconds. To change it, specify a new value in the pod descriptor file

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace:
  labels:
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
  terminationGracePeriodSeconds: 60
```

A common pitfall about the SIGTERM signal is how to handle the PID 1 process. A process identifier (PID) is a unique identifier that the Linux kernel gives to each process. PIDs are namespaced, meaning that a container has its own set of PIDs that are mapped to PIDs on the host system. The first process launched when starting a Linux kernel has the PID 1. For a normal operating system, this process is the init system. In a container, the first process gets PID 1. When the pod is deleted, the SIGTERM signal is sent to the process with PID 1. If such process is not the main application process, the application does not start its shutdown and a SIGKILL signal is required, leading the application in user-facing errors, interrupted i/o on devices, etc.

For example, is we start the main process of a container with a shell script, the shell will get the PID 1 and not the main process. When sending a SIGTERM to the shell, depending on the shell, such signal might be or not be passed to the shell's child process. To avoid this pitfall, make sure to start the main process of a container with PID 1.

#### Life cycle hooks
The pod manifest file permits to define two other additional life cycle hooks:

  * **Post Start Hook**: is executed after the container is created.
  * **Pre Stop Hook**: is executed immediately before a container is terminated.

The post start hook can be useful to perform some additional tasks when the application starts. This might be always done within the source code but, having an external tool, is useful to run additional commands without touching the source code.

For example, the following pod descriptor define a post start hook for a ``minio`` server

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: minio
  namespace:
  labels:
    app: minio
spec:
  containers:
  - name: minio
    image: minio/minio:latest
    args:
      - server
      - /storage
    env:
    - name: MINIO_ACCESS_KEY
      value: "minio"
    - name: MINIO_SECRET_KEY
      value: "minio123"
    ports:
    - containerPort: 9000
    volumeMounts:
    - name: data
      mountPath: /storage
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "mkdir -p /storage/bucket"]
  volumes:
  - name: data
    hostPath:
      path: "/mnt"
```

The minio server does not provide a default bucket when it starts. To create a default bucket, without changing the source code of minio, we can use a simple post start hook to create it.

While a post start hook is executed after the container's process started, a pre stop hook is executed immediately before a container's process is terminated. The pre stop hook can be used to run additional tasks in preparation of the process shutdown. 

For example, the following snippet define a pre stop hook for the previous minio server

```yaml
...
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "mkdir -p /storage/default"]
      preStop:
        exec:
          command: ["/bin/sh", "-c", "rm -rf /storage/default"]
...
```

The pre stop hook above disposes the default bucket before to terminate the container. 

A pre stop hook can be also used to initiate a graceful shutdown of the container's process, if - for some reasons - it does not shut down gracefully upon receipt of a SIGTERM signal. This usage of the pre stop hook avoids kubelet killing the process with a SIGKILL signal if it does not terminate gracefully. However, best practice is to make sure the application's process correctly handles the SIGTERM signal and initiate the grace shoutdown without waiting for the SIGKILL signal.

### Structure Patterns
Structural Patterns refer to how organize containers interaction:

* [Sidecar](#sidecar)
* [Initialiser](#initialiser)
* [Ambassador](#ambassador)
* [Adapter](#adapter)

### Sidecar
The sidecar pattern describes how to extend and enhance the functionality of a preexisting container without changing it. A good container, behaves like a single Unix process, solves one single problem and does it very well. A good container design requires it is created with the idea of replaceability and reuse. But having single purpose reusable containers, requires a way of extending the container functionality. The Sidecar pattern describes a technique where a container enhances the functionality of the main container.

The following is an example of sidecar pattern

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
  - name: sidecar
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

In the example above, the main container is an nginx webserver serving static web pages. It is supported by a sidecar container that dynamically create the content web page that the main container is going to serve. The two containers use a shared volume to pass data among them. 

### Initialiser
The Initialiser pattern describes how to initialise a container with data. In kuberentes, this patterns is implemented by mean of the init containers. An init container starts first before the main and the other supporting containers in the same pod.

For example, the following file describe a pod with one main container and an init container using a shared volume to comminicate each other

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubeo
  namespace:
  labels:
    run: kubeo
spec:
  initContainers:
    - name: git-clone
      image: alpine/git
      args: ["clone", "--", "https://github.com/kalise/kubeo-website.git", "/repo"]
      volumeMounts:
        - name: content-data
          mountPath: /repo
  containers:
    - name:  nginx
      image: nginx:latest
      volumeMounts:
        - name: content-data
          mountPath: /usr/share/nginx/html
  volumes:
    - name: content-data
      emptyDir: {}
```

The init container above initialises the main container by pulling data from a GitHub repository to a local shared volume. Once pulled the content, the init container exits leaving the main container initialised with pulled data. 

### Ambassador
The Ambassador pattern describes a special case of the Sidecar pattern where the sidecar container is responsible for hiding the complexity and providing a unified interface for accessing services outside of the pod. This pattern is often used to proxy a local connection to remote services by hiding the complexity of such services. For example, if the main application needs to access a SSL based service, we can create an ambassador container to proxy from HTTP to HTTPS.

      [provide a good example of ambassador]

### Adapter
The Adapter pattern is another variant of the Sidecar pattern. In contrast to the ambassador, which presents a simplified view of the outside world to the application, the adapter pattern present a simplified view of an application to the external world. A concrete example of the adapter pattern is an adapter container that implements a common metering interface to a remote monitoring system.

      [provide a good example of ambassador]

## Configuration Patterns
Configuration Patterns refer to how handle configurations in containers.

### Environment Variables
For small sets of configuration values, the easiest way to pass configuration data is by putting them into environment variables. The following descriptor sets some common configuration parameters to a MySQL pod, using well defined environment variables 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  namespace:
  labels:
    run: mysql
spec:
  containers:
  - name: mysql
    image: mysql:5.6
    env:
    - name: MYSQL_RANDOM_ROOT_PASSWORD
      # The generated root password will be printed to stdout
      # kubectl logs mysql | grep GENERATED
      value: "yes"
    - name: MYSQL_DATABASE
      value: "employee"
    - name: MYSQL_USER
      value: "admin"
    - name: MYSQL_PASSWORD
      value: "password"
    ports:
    - name: mysql
      protocol: TCP
      containerPort: 3306
```

### Config Maps
Kubernetes allows separating configuration options into a separate object called Config Map, which is a map containing key/value pairs with the values ranging from short literals to full config files. An application doesn’t need to read the Config Map directly or even know that it exists. The contents of the map are instead passed to containers as either environment variables or as files in a volume.

For example, the following snippet defines a ``nginx`` pod mounting a config map as configuration file

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace:
  labels:
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
      - name: config
        mountPath: /etc/nginx/conf.d
        readOnly: true
  volumes:
    - name: config
      configMap:
        name: nginx
```

Here an example of Config Map that can be created to mount a custom configuration file to the pod above.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx
  namespace:
data:
  nginx.conf: |
    server {
        listen   9080;
        server_name  www.example.com;
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
    }
```

Using a Config Map and exposing it through a volume brings the ability to update the configuration without having to recreate the pod or even restart the container. When updating a Config Map, the files in all the volumes referencing it are updated. It’s then up to the process to detect that they’ve been changed and reload them.

### Secrets
In addition to regular, not sensitive configuration data, sometimes we need to pass sensitive information, such as credentials, tokens and private encryption keys, which need to be kept secure. For these use cases, kubernetes provides Secrets. Secrets are kept safe by distributing them only to the nodes that run the pods that need access to the secrets. 

Also, on the nodes, secrets are always stored in memory and never written to the disk. On the master node, secrets are stored in encrypted form into the etcd database.

As for the Config Maps, secrets can be passed to containers by mounting a volume or via environment variables. For example, the following descriptor defines a secret to pass the user's credentials to a ``mysql`` pod.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql
  namespace:
data:
  MYSQL_RANDOM_ROOT_PASSWORD: eWVz
  MYSQL_DATABASE: ZW1wbG95ZWVz
  MYSQL_USER: YWRtaW4=
  MYSQL_PASSWORD: cGFzc3dvcmQ=
```

In a secret data are base64 encoded.

Data can be passed via environment variables to the MySQL server as defined in the following descriptor

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  namespace:
  labels:
    run: mysql
spec:
  containers:
  - name: mysql
    image: mysql:5.6
    envFrom:
    - secretRef:
        name: mysql
    ports:
    - name: mysql
      protocol: TCP
      containerPort: 3306
```


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
