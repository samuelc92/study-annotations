# Kubernetes: Up and Running

**Table of Contents**

- [Kubernetes: Up and Running](#kubernetes-up-and-running)
  - [Container Images](#container-images)
    - [System Container](#system-container)
    - [Application Container](#application-container)
  - [Dockerfile](#dockerfile)

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

## Dockerfile

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