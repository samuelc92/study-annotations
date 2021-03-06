# Kubernetes: Up and Running

**Table of Contents**

- [Kubernetes: Up and Running](#kubernetes-up-and-running)
  - [Container Images](#container-images)
    - [System Container](#system-container)
    - [Application Container](#application-container)
  - [Docker](#docker)
    - [Commands](#commands)
    - [Dockerfile](#dockerfile)
    - [Multistage Image Builds](#multistage-image-builds)
  - [Kubernetes](#kubernetes)
    - [Commands](#commands-1)
    - [Proxy](#proxy)
    - [DNS](#dns)
    - [Namespaces](#namespaces)
    - [API Objects](#api-objects)
    - [Cordon and drain](#cordon-and-drain)
    - [Pods in Kubernetes](#pods-in-kubernetes)
      - [Liveness Probe](#liveness-probe)
      - [Readiness Probe](#readiness-probe)
      - [Types of Health Checks](#types-of-health-checks)
      - [Resources Requests: Minimum Required Resources](#resources-requests-minimum-required-resources)
      - [Requested Limit Details](#requested-limit-details)
      - [Capping Resources Usage with Limits](#capping-resources-usage-with-limits)
    - [Labels and Annotations](#labels-and-annotations)
    - [Service Discovery](#service-discovery)
      - [Domain Name System (DNS)](#domain-name-system-dns)
    - [Kube-Proxy and Cluster IPs](#kube-proxy-and-cluster-ips)
    - [Ingress](#ingress)
    - [ReplicaSet](#replicaset)
    - [Deployments](#deployments)
    - [DaemonSet](#daemonset)
    - [Job](#job)
      - [Cron Jobs](#cron-jobs)
    - [ConfigMaps and Secrets](#configmaps-and-secrets)
    - [Role-Based Access Control for Kubernetes (RBAC)](#role-based-access-control-for-kubernetes-rbac)
      - [Identity in Kubernetes](#identity-in-kubernetes)
      - [Role and Role Bindings](#role-and-role-bindings)
    - [Service Mesh](#service-mesh)
    - [Integrating Storage Solutions and Kubernetes](#integrating-storage-solutions-and-kubernetes)
  - [Anti-pattern](#anti-pattern)
    - [Lack of Health Checks](#lack-of-health-checks)
    - [Not Using Blue/Green, or Canary Deployments Models](#not-using-bluegreen-or-canary-deployments-models)
    - [Not Using Circuit Breakers](#not-using-circuit-breakers)
    - [Not Collecting Enough Metrics](#not-collecting-enough-metrics)

## Container Images

- Container image is a binary package that encapsulates all of the files necessary to run a program inside of an OS container.
- The image is not a single file but rather a specification for a manifest file that points to other files.
- Container images are constructed with a series of file system layers, where each layer inherits and modifies the layers that came before it.

```
---- container A: a base operating system only, such as Debian
    ---- container B: built upon #A, by adding Ruby v2.1.10
        ---- container D: built upon #B, by adding Rails v4.2.6
        ---- container E: built upon #B, by adding Rails v3.2.x
    ---- container C: built upon #A, by adding Golang v1.6
```

- Containers fall into two main categories

### System Container

- System container seek to mimic virtual machine and often run a full boot process. It has come to be seen as poor practice hence application containers have gained favor.

### Application Container

- Application container differ from system container in that they commonly run a single program.

## Docker

- Docker is a set of platform as a service (PaaS) products that use OS-level virtualization to deliver software in packages called containers.
- Each container gets its own IP address, so listening on localhost inside the container doesn't cause you to listen on your machine. Without the port forwarding (-p 8080:8080)
  connections will be inaccessible to your machine.

### Commands

- `docker system prune -a` This will remove all stopped containers, all untagged imags, an all unused image layers cached as part of the build process.

### Dockerfile

- Dockerfile can be used to automate the creation of a Docker container image.

```dockerfile
# Start from a Node.js 16 (LTS) image - 1
FROM node:16

# Specify the directory inside the image in which all commands will run - 2
WORKDIR /usr/src/app

# Copy package files and install dependencies - 3
COPY package*.json ./
RUN npm install
RUN npm install express

# Copy all of the app files into the image - 4
COPY . .

# The default command to run when starting the container - 5
```

- 1: Every Dockerfile builds on other container images. This line specifies that we are starting from the 'node:16' image on the Docker Hub. This is a pre-configured image with
  Node.js 16
- 2: This line sets the work directory, in the container image, for all following commands.
- 3: These two lines initialize the dependencies for Node.js. First we copy the package files into the image. This will include package.json and package-lock.json. The 'RUN' command
  the runs the correct command in the container to install the necessary dependencies.
- 4: Now we copy the rest of the program files into the image. This will include everything expect files which are specified in the '.dockerignore' file.
- 5: Finally, we specify the command that should be run when the container is run.

- In general, we want to order the layers from least likely to change to most likely to change in order to optimize the image size for pushing and pulling. This is why we copy
  the 'package\*.json' files and install dependencies before copying the rest of the program files. A developer is going to update and change the program files much more often than
  the dependencies.

### Multistage Image Builds

- With multistage builds, rather than producing a single image a Dockerfile can actually produce multiple images, each image is considered a stage.

```dockerfile
# STAGE 1: Build
FROM golang:1.17-alpine AS build

# Install Node and NPM
RUN apk update && apk upgrade && apk add --no-cache git nodejs

# Get dependencies for GO part of build
RUN go get -u github.com/jte...
RUN go get github.com/...
WORKDIR /go/src/github.com/...

# Copy all sources in
COPY . .

# This is a set of variables that the build script expects
ENV VERBOSE=0
ENV VERSION=1.0

# Do the build script is part of incoming sources.
RUN build/build.sh

# STAGE 2: Deployment
FROM alpine

USER nobody:nobody
COPY --from=build /go/bin/kuard /kuard

CMD ["/kuard"]
```

- This Dockerfile produces two images, the first one contains the GO compiler, Reactjs toolchain, and source code for the program. The second contains the compiled binary.

## Kubernetes

- Kubernetes services provide load balancing, naming, and discovery to isolate one microservice from another.
- Namespaces provide isolation and access control, so that each microservice can control the degree to which other services interact with it.
- Ingress objects provide an easy-to-use frontend that can combine multiple microservices into a single externalized API surface area.
- Resources requested by a Pod are guaranteed to be present on the node, while a Pod's limit is the maximum amount of a given resource that a Pod can consume.
  A Pod's limit can be higher than its request, in which case the extra resource are supplied on a best-effort basis. They are not guaranteed to be present on the node

### Commands

- `kubectl get component statuses` Get a simple diagnostic for the cluster. It returns the components that make up(compose) the Kubernetes cluster.

  - The `controller -manager` is responsible for running various controllers that regulate behavior in the cluster, for example, ensuring that all of the replicas
    of a service are available and healthy.
  - The `scheduler`is responsible for placing different Pods into different nodes in the cluster.
  - Finally, the `etcd`server is the storage for the cluster where all of the API objects are stored

- `kubectl get nodes` List out all of the nodes in your cluster.

  - In Kubernets, nodes are separated into `control-plane`nodes that contain containers like the API server, scheduler, etc., which manage the cluster,
    and `worker` nodes where your containers will run.

- `kubectl port-forward kuard 8080:80` Creates a secure tunnel from your local machine, through the Kubernetes master, to the instance of the Pod running on one
  of the worker nodes;
  - opens up a connection that forwards traffic from the local machine on port 8080 to the remote container on port 80;

### Proxy

- It is responsible for routing network traffic to load-balanced services in the Kubernetes cluster;
- The proxy must be present on every node in the cluster;

### DNS

- It provides naming and discovery for the services that are defined in the cluster.
  - `kubectl get deployments --namespace=kube-system core-dns`
  - `kubectl get services --namespace=kube-system core-dns`

### Namespaces

- Kubernetes uses namespaces to organize objects in the cluster;
- Namespaces is similar to a folder that holds a set of objects;

### API Objects

- Everything contained in Kubernetes is represented by a RESTful resource. Each Kubernetes objects exist at a unique HTTP path, for example: https://your-k8s.com/api/v1/namespace/default/pods/my-pod, leads to the representation of a Pod in the default namespace named "my-pod";
- Objects in the Kubernetes API are represented as JSON or YAML files;
- You can use `kubectl` to create an object in Kubernetes by running:
  - `kubectl apply -f obj.yaml`

### Cordon and drain

- When you cordon a node you prevent future Pods from being scheduled onto that machine;
- When you drain a node, you remove any Pods that are currently running on that machine;
- A good example use case for these command would be removing a physical machine for repairs or upgrades;

### Pods in Kubernetes

- A Pod is a collectin of application containers and volumes running in the same execution environment;
- Pods (single atomic unit), or groups of containers, can group together container images developed by different teams into a single deployable unit;
- All of the containers in a Pod always land on the same machine;
- Applications running iin the same Pod share the same IP address and port space (network namespace), have the same hostname and can communicate using native interprocess communication channels ofr POSIX message queues;
- If the containers work correctly on differente machine multiple Pods is probably the correction solution. If the containers don't work correctly on differente machines a Pod is the correct grouping for the containers;

#### Liveness Probe

- Liveness health checks run application-specific logic, like loading a web page, to verify that the application is not just still running, but is functioning properly.
- It must be defined in the Pod manifest.
- Liveness probes are defined per container, which means each container inside a Pod is health-checked separately.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata: name:kuard
  spec:
    containers:
      - images: gcr.io/kuard-demo/kuard-amd64:blue
      name: kuard
      livenessProbe:
        httpGet:
          path: /healthy
          port: 8080
        initialDelaySeconds: 5
        timeoutSeconds: 1
        periodSeconds: 10
        failureThreshold: 3
      ports:
        - containerPort: 8080
        name: http
        protocol: TCP
  ```

- While the default response to a failed liveness check is to restart the Pod, the actual behavior is governed by the Pod's `restartPolicy`. There are three options for the restart policy `Always`(the default), `OnFailure`(restart only on liveness failure or nonzero process exit code), or `Never`.

#### Readiness Probe

- Describes when a container is ready to serve user request.
- Containers that fail readiness checks are removed from service load balancers.
- Readiness probe are configured similarly to liveness probe.
- Combining the readiness and liveness probes helps ensure only healthy containers are running within the cluster.

#### Types of Health Checks

- Kubernetes also support `tcpSocket` health checks that open a TCP socket, if the connection succeeds, the probe succedes.
- Kubernetes allows `exec` probes. These execute a script or program in the context of the container, if this script returns a zero exit code, the probe succeds, otherwise it fails.

#### Resources Requests: Minimum Required Resources

- When a Pod requests the resources required to run its containers, Kubernetes guarantees that these resources are available to the Pod.
- Resources are requested per container, not per Pod. The total resources requested by the Pod is the sum of all resources requested by all containers in the Pod because the different containers often have very different CPU requirements.

#### Requested Limit Details

- Requests are used when scheduling Pods to nodes. The Kubernetes scheduler will ensure that the sum of all requests of all Pods on a node does not exceed the capacity of the node. Therefore, a Pod is guaranteed to have at least the requested resources when running on the node.
- Memory requests are handled similarly, to CPU, byt there is an important difference. If a container is over its memory request, the OS can't just remove memory from the process, because it's been allocated. Consequently, when the system runs out of memory, the `kubelet` terminates containers whose memory usage is greater than their requested memory. These containers are automatically restarted, but with less available memory on the machine for the container to consume.

#### Capping Resources Usage with Limits

- In addition to setting resources required by a Pod, which establishes the minimum resources available to it, you can also set a maximum on it's resources usage via resources `limits`.
- When a limit is established on a container, the kernel is configured to ensure that consumption cannot exceed these limits. A container with a CPU limit of 0.5 cores will only ever get 0.5 cores, even if the CPU is otherwise idle. A container with a memory limit of 256MB will not be allowed additional memory, for example `malloc` will fail, if its memory usage exceeds 256MB.

### Labels and Annotations

- Labels are Key/Value pairs that can be attached to Kubernetes Objects such as Pods and ReplicaSets. They can be arbitrary, and are useful for attaching identifying information to Kubernetes Objects.
- Label provide the foundation for grouping objects.
- Annotations provide a storage mechanism that resembles labels. Annotations are Key/Value pairs designed to hold non-identifying information that tools and libraries can leverage.

### Service Discovery

- Service Discovery tools help solve the problem of finding which processes are listening at which addresses for which services.
- Real Service Discovery in Kubernetes starts with a Service Object.
- Use the command `kubectl expose` to create a service.

#### Domain Name System (DNS)

- The DNS is the traditional system of Service Discovery on the internet.
- DNS is designed for relatively stable name resolution with wide and efficient caching.

### Kube-Proxy and Cluster IPs

- Cluster IPs are stable virtual IPs that load-balance traffic across all of the endpoints in a service. This magic is performed by a component running in every node in the cluster called the `kube-proxy`.

  ![kube-proxy](./img/kube-proxy.png)

- The `kube-proxy` watches for new services in the cluster via the API server. It then programs a set of `iptables` rules in the Kernel of that host to rewrite the destinations of packets so they are directed at one of the endpoints for that service.
- Service object operates at Layer 4, this means that it only forwards TCP and UDP connections and doesn't look inside of those connections.

### Ingress

- _virtual hosting_ is a mechanism to host many HTTP sites on a single IP address. Typically, the user uses a load balancer or reverse proxy to accept incoming connections on HTTP(80) and HTTP(443) ports. That program then parse the HTTP connection and, based on the _host header_ and the URL path is requested, proxies the HTTP call to some other program. In this way, that load balancer or reverse proxy plays _traffic cop_ decoding and directing incoming connections to the right _upstream_ server.
- Kubernetes calls its HTTP-based load-balacing System Ingress
- Ingress is a Kubernetes-native way to implement the _virtual hosting_ pattern.

  ![ingress](./img/ingress.png)

  - In the image the Ingress controller is a software system exposed outside the cluster using a service of `type:LoadBalancer. It then proxies requests to _upstream_ servers.

### ReplicaSet

- It acts as cluter-wide Pod manager, ensuring that the right types and numbers of Pods are running at all times.
- Pods managed by ReplicaSets are automatically rescheduled under certain failure conditions, such as node failures and network partitions.
- ReplicaSets use label queries to identify the set of Pods they should be managing.
- The Selector in the ReplicaSet `spec` should be a proper subset of the labels in the Pod template.

### Deployments

- The Deployment object exists to manage the release of new versions. It represents deployed application in a way that transcends any particular version. Additionally, Deployments enable you to easily move from one version of your code to the next.
- Using Deployments, it is simply an reliably rollout new software versions without downtime or errors.
- Deployments manage ReplicaSets. As with all relationships in Kubernetes, this relationship is defined by labels and a label Selector.
- Recreate Strategy updates the ReplicaSet it manages to use the new image and terminates all of the Pods associated with the Deployment. The ReplicaSet notices that it no longer has any replicas and recreates all Pods using the new image. It should only be usd for test Deployment where service downtime is acceptable.
- RollingUpdate Strategy updates a few Pods at a time, moving incrementally until all of the Pods are running the new version of the software.
- Setting `maxSurge: 100%` is equivalent to _blue/green_ deployment.

### DaemonSet

- A DaemonSet ensures that a copy of a Pod is running accross a set of nodes in a Kubernetes cluster. It is used to deploy system daemons such as log collectors and monitoring agents, which typically must run on every node.

- ReplicaSets should be used when your application is completely decoupled from the node and you can run multiple copies on a given node without special consideration. DaemonSets should be used when a single copy of you application must run on all or a subset of the nodes in the cluster.

- Example of a DaemonSets that creates a `fluentd` logging agent on every node in the target cluster.

```yaml
apiVersion: apps/v1
kind: DaemonSets
metadata:
  name: fluentd
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
        - name: fluentd
          image: fluent/fluentd:v0.14.10
          resources:
            limits:
              memory: 200M
            requests:
              cpu: 100M
              memory: 200M
          volumeMounts:
            - name: varLog
              mountPath: /var/log
            - name: varlibdockercontainer
              mountPath: /var/lib/docker/container
              readOnly: true
    terminationGracePeriodSeconds: 30
    volumes:
      - name: varLog
        hostPath:
          path: /var/log
      - name: varlibdockercontainer
        hostPath:
          path: /var/lib/docker/container
```

### Job

- The Job Object is responsible for creating and managing Pods in a template in the job specification.
- Te Job Object coordinates running a number of Pods in parallel. Job has the following patterns:

| Type                       | Use Case                                               | Behaviour                                                                          | Completions | Parallelism |
| -------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------- | ----------- | ----------- |
| One Shot                   | Database Migration                                     | A single Pod running once until successful termination                             | 1           | 1           |
| Parallel fixed completions | Multiple Pods processing a set of work in parallel     | One or more Pods running one or more times until reaching a fixed completion count | 1+          | 1+          |
| Work queue: parallel jobs  | Multiple Pods processing from a centralized work queue | One or more Pods running once until sucessful termination                          | 1           | 2+          |

#### Cron Jobs

- It is responsible for creating a new Job Object at a particular interval.

### ConfigMaps and Secrets

- ConfigMaps are used to provide configuration information for workloads.
- Secrets are similar to ConfigMaps but focus on making sensitive information available to the workload. They can be used for things like credentials or TLS certificates.
- There are three main ways to use a ConfigMap:
  - Filesystem: you can mount a ConfigMap into a Pod. A file is created for each entry based on the key name. The contents of that file are set to the value.
  - Environment variables: a ConfigMap can be used to dynamically set the value of an environment variable.
  - Command-line argument: Kubernetes supports dynamically creating the command line for a container based on ConfigMap values.

### Role-Based Access Control for Kubernetes (RBAC)

- RBAC provides a mechanism for restricting both access to and actions on Kubernetes APIs to ensure that only appropriate users have access.
- Every request to Kubernetes is first authenticated. Authentication provides the identity of the caller issuing the request. Kubernetes does not have a built-in identity store, foccusing instead on integrating other identity sources within itself.
- Once users have been authenticated, the authorization phase determines whether they are authorized to perform the request.

#### Identity in Kubernetes

- Every request to Kubernetes is associated with some identity. Even a request with no identity is associated with the `systemunauthenticated` group.
- Service accounts are created and managed by Kubernetes itself and are generally associated with components running inside the cluster.
- User accounts are all other accounts associated with actual users of the cluster, and often include automation like continuous delivery services that run outside the cluster.

#### Role and Role Bindings

- A role is a set of abstract capabilities. For example, the `appdev` role might represent the ability to create Pods and Services.
- A *role binding* is an assignment of a role to one or more identities.

### Service Mesh

- There are three general capabilities provided by most Service Mesh implementations: Network encryption and authorization, traffic shaping and observability.
- Installing a Service Mesh on you Kubernetes cluster automatically provides encyption to network traffic between every Pod in the cluster. The Service Mesh adds a sidecar container to every Pod that transparently intercepts all network communication. In addition to securig the communication, mutual TLS adds identity to the encryption using client certificates so the application securely knows the idenity of every network client.
- *Traffic Shaping*: routing of requests to different service implementations based on the characteristics of the requests. Example: Route all traffic from your companies internal users went to service "y" while all traffic from the rest of the world still went to service "x".
- *Introspection*: The Service Mesh knows all communication between Pods, so it tracks the request giving to the developer the option to see a single aggregate request that defines the user experience of their complete journey.
- The Service Mesh is deeply integrated into the core communication of your service thus if the Service Mesh fails, you entire application stops working.

### Integrating Storage Solutions and Kubernetes

- Importing External Services
  - Import external services (either cloud services or running on VMs) into Kubernetes gives the advantage of all of the built-in naming and Service Discovery primitives provided by Kubernetes.
  - Represent both databases as Kubernetes services enables you to maintain identical conofigurations in both testing and production.
  - To import an external database service into Kubernetes, we start by creating a service without a Pod selector that references the DNS name of the database server.
 ```yaml
 kind: Service
 apiVersion: v1
 metadata:
  name: external-database
 spec:
  type: ExternalName
  externalName: database.company.com
 ```
- Kubernetes-Native Storage with StatefulSets
  - StatefulSets are replicated groups of Pods, similar to ReplicaSet. But unlike a ReplicaSet, they have certain unique properties:
    - Each replica gets a persistent hostname with a unique index (e.g: `database-0`, `database-1`, etc).
    - Each replica is created in order from lowest to highest index, and creation will block until the Pod at the previous index is healthy and available. This also applies to scaling up.
    - When a StatefulSet is deleted, each of the managed replica Pod is also deleted in order form highest to lowest. This also applies to scaling down the number of replicas.

### Securing Pods

- Defense in depth is a concept where you utilize multiple layers of security controls across your computing systems which include Kubernetes.

## Anti-pattern

### Lack of Health Checks

A health check allows the status of a service been validated. It helps assess key information like service availability, system metrics, or available database connections. Kubernetes supports Container probes (`livenessProbe`, `readinessProbe`, `startupProbe`) that allow monitoring services and take actions when the probe is successful.

### Not Using Blue/Green, or Canary Deployments Models

Kubernetes allows you to define `Recreate` and `RollingUpdate` deployment strategies. `Recreate` will kill all the Pods before creating new ones, while `RollingUpdate` will update Pods in a rolling fashion and permit configuring `maxUnavailable` and `maxSurge` to control the process.
While these deployment strategies can serve many use cases, they also have limitations. For example, `Recreate` causes downtime while `RollingUpdate` can make rollback harder. None of these methods allows rapid experimentation and feedback on new versions of your services.

A blue/green deployment is a deployment model that creates copies of your service (the old version being blue and the new version being green) with both services running in parallel.

Canary deployment is a technique that only routes the traffic to the new service for a subset of users.

### Not Using Circuit Breakers

Services running on a Kubernetes cluster talk to one another by making remote calls. The goal of the circuit breaker is to prevent failure after identifying a fault, it prevents further calls to the service that is failuring. This allows systems to deal with the failure and route requests to healthy instances of the same service.

### Not Collecting Enough Metrics

Observability is key to understanding system behavior, and effective observability depends on the proper collection of metrics. A lack of key metrics will severely limit your ability to understand how your services are performing and if they're performing at the desired level.
