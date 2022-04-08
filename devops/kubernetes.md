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

- ```docker system prune -a``` This will remove all stopped containers, all untagged imags, an all unused image layers cached as part of the build process.

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
the 'package*.json' files and install dependencies before copying the rest of the program files. A developer is going to update and change the program files much more often than
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

 - Pods, or groups of containers, can group together container images developed by different teams into a single deployable unit.
 - Kubernetes services provide load balancing, naming, and discovery to isolate one microservice from another.
 - Namespaces provide isolation and access control, so that each microservice can control the degree to which other services interact with it.
 - Ingress objects provide an easy-to-use frontend that can combine multiple microservices into a single externalized API surface area.
 - Resources requested by a Pod are guaranteed to be present on the node, while a Pod's limit is the maximum amount of a given resource that a Pod can consume.
  A Pod's limit can be higher than its request, in which case the extra resource are supplied on a best-effort basis. They are not guaranteed to be present on the node

 ### Commands

 - ```kubectl get component statuses``` Get a simple diagnostic for the cluster. It returns the components that make up(compose) the Kubernetes cluster.
   - The ```controller -manager``` is responsible for running various controllers that regulate behavior in the cluster, for example, ensuring that all of the replicas
 of a service are available and healthy.
   - The ```scheduler```is responsible for placing different Pods into different nodes in the cluster.
   - Finally, the ```etcd```server is the storage for the cluster where all of the API objects are stored  

- ```kubectl get nodes```List out all of the nodes in your cluster.
  - In Kubernets, nodes are separated into ```control-plane```nodes that contain containers like the API server, scheduler, etc., which manage the cluster, 
  and ```worker``` nodes where your containers will run.