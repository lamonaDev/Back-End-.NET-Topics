# Docker Image Building

## Introduction

Docker image building is the process of creating reusable, portable container images from application code and dependencies. Images serve as the template for containers and contain everything needed to run an application: code, runtime, system tools, libraries, and settings. This document covers the fundamentals of building custom Docker images, understanding Dockerfile instructions, and managing the image build process effectively.

---

## 1. Creating Custom Images Pipeline

Building custom Docker images involves a systematic pipeline that transforms your application into a deployable container image.

### Pipeline Overview

```
+------------------------------------------------------------------+
|                 IMAGE BUILDING PIPELINE                           |
+------------------------------------------------------------------+
|                                                                  |
|  +-------------+    +-------------+    +-------------+          |
|  | Application |    |  Dockerfile |    | Docker Image|          |
|  |   Code      |--> |   Script    |--> |   Build     |          |
|  +-------------+    +-------------+    +-------------+          |
|        |                  |                   |                   |
|        v                  v                   v                   |
|  +-------------+    +-------------+    +-------------+          |
|  | Dependencies|    | Build Steps |    |  Container  |          |
|  |   (.NET,    |    | (RUN, CMD)  |    |   Image     |          |
|  |   Node.js) |    |             |    |             |          |
|  +-------------+    +-------------+    +-------------+          |
|                                                                  |
+------------------------------------------------------------------+
```

### Key Components

| Component | Description |
|-----------|-------------|
| **Base Image** | The foundational image from which your container is built |
| **Dependencies** | External packages or libraries required by your application |
| **Dockerfile** | A script containing instructions that Docker uses to build the image |
| **Container Run Command** | The command used to start a container from the built image |

### Example: .NET Application Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0
WORKDIR /app
COPY . .
RUN dotnet restore
RUN dotnet build
CMD ["dotnet", "run"]
```

---

## 2. BuildKit in Docker

BuildKit is Docker's modern build engine that provides improved performance, caching, and build functionality.

### Verbose Output with --progress=plain

By default, Docker shows a compact progress output. To see detailed output during the build process:

```bash
docker build --progress=plain .
```

This displays each build step as it executes, making it easier to debug build issues.

### Building Without Cache

To force a complete rebuild without using cached layers:

```bash
docker build --no-cache .
```

This is useful when you want to ensure all steps are executed fresh, particularly when debugging Dockerfile issues.

---

## 3. Dockerfile Instructions

### FROM Instruction

The `FROM` instruction initializes a new build stage and sets the base image for subsequent instructions.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0
```

The base image is the starting point for your container, providing the operating system and runtime environment.

#### Alpine Linux

Alpine is a lightweight Linux distribution designed for security, simplicity, and resource efficiency. Alpine-based images are significantly smaller than full Linux distributions.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine
```

#### Image Size Comparison

```
+----------------------+----------------------+
| Standard Image       | Alpine Image         |
+----------------------+----------------------+
| Ubuntu: 22.04 ~ 77MB | Alpine: 3.18 ~ 5MB   |
| .NET SDK: 8.0 ~ 1GB  | .NET SDK: 8.0 ~ 200MB|
+----------------------+----------------------+
```

---

### RUN Instruction

The `RUN` instruction executes commands during the image build process. These commands add layers to the image and install dependencies.

```dockerfile
RUN dotnet restore
RUN dotnet build
```

Each `RUN` instruction creates a new layer in the image. Multiple commands can be combined to reduce layers:

```dockerfile
RUN dotnet restore && dotnet build
```

#### Layer Creation Process

```
+------------------------------------------+
|         LAYER CREATION                   |
+------------------------------------------+

Base Image
    |
    v
+------------------+
| RUN Command 1   |  --> Layer 1
+------------------+
    |
    v
+------------------+
| RUN Command 2   |  --> Layer 2
+------------------+
    |
    v
+------------------+
| Final Image     |  --> Stacked Layers
+------------------+

+------------------------------------------+
```

---

### CMD Instruction

The `CMD` instruction defines what command to run when a container starts. It provides the default execution command for the container.

```dockerfile
CMD ["dotnet", "run"]
```

Unlike `RUN`, which executes during build time, `CMD` executes when the container starts.

#### RUN vs CMD Comparison

```
+------------------------------------------+
|    RUN vs CMD COMPARISON                 |
+------------------------------------------+

RUN:
  - Executes during BUILD TIME
  - Creates new LAYERS in image
  - Results are COMMITTED to image
  - Used for: installing deps, building

CMD:
  - Executes at RUN TIME
  - Does NOT create layers
  - Sets DEFAULT command
  - Used for: starting application

+------------------------------------------+
```

#### Multiple CMD Instructions

If multiple CMD instructions exist in a Dockerfile, only the last one takes effect.

Example:

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl
CMD ["bash"]
```

In this example:
- `RUN` installs curl during build
- `CMD` sets bash as the default shell when the container starts

---

## 4. Image Building Process

### Relationship Between FROM and RUN

The build process follows a sequential flow:

```
+------------------------------------------+
|       BUILD PROCESS FLOW                 |
+------------------------------------------+

Step 1: FROM
    |
    v
Step 2: RUN (Layer 1)
    |
    v
Step 3: RUN (Layer 2)
    |
    v
Step 4: RUN (Layer 3)
    |
    v
Final Image (Stacked Layers)

+------------------------------------------+
```

1. `FROM` establishes the base image
2. `RUN` instructions execute commands on top of that base
3. Each instruction creates a new layer
4. The final image is a stacked combination of all layers

### Rebuilding with Cache

Docker caches layers from previous builds to speed up subsequent builds. When a layer hasn't changed, Docker reuses the cached version.

#### Cache Behavior

```
Build 1 (First Time):
+------------------+
| FROM (cached)    |  --> Use cache
+------------------+
| RUN (cached)     |  --> Use cache
+------------------+
| RUN (execute)    |  --> Execute
+------------------+

Build 2 (After Code Change):
+------------------+
| FROM (cached)    |  --> Use cache
+------------------+
| RUN (cached)     |  --> Use cache
+------------------+
| RUN (execute)    |  --> Execute (rebuild)
+------------------+
```

Example:

```bash
# First build - takes time
docker build -t myapp:v1 .

# Second build - faster due to caching
docker build -t myapp:v2 .
```

If you modify a line in the Dockerfile, only layers after that point are rebuilt.

---

## 5. Tagging Images

Tagging assigns a meaningful name and version to your Docker image, making it easier to identify and manage.

### Using -t Flag

```bash
docker build -t myapp:latest .
```

The `-t` flag tags the image with:
- `myapp`: Repository name
- `latest`: Tag (version)

### Tagging Pipeline

A typical image tagging pipeline:

```bash
# Build and tag with multiple tags
docker build -t myapp:1.0.0 -t myapp:latest .

# Tag an existing image with new tag
docker tag myapp:1.0.0 myapp:stable
```

### Tag Strategy

```
+------------------------------------------+
|        IMAGE TAG STRUCTURE               |
+------------------------------------------+

Registry/Repository:Tag

Example:
  docker.io/library/nginx:latest
  |    |      |       |
  |    |      |       +-- Tag (version)
  |    |      +---------- Image name
  |    +----------------- Repository
  +---------------------- Registry (optional)

+------------------------------------------+
```

---

## 6. Manual Image Generation with Docker Commit

Beyond Dockerfile-based builds, Docker allows creating images from running containers using `docker commit`.

### Process

```
+------------------------------------------+
|       DOCKER COMMIT PROCESS              |
+------------------------------------------+

1. Start Container
   docker run -it ubuntu:22.04 /bin/bash
           |
           v
2. Make Changes Inside Container
   (install packages, modify files)
   apt-get update
   apt-get install -y nginx
           |
           v
3. Exit Container (Ctrl+P Ctrl+Q)
           |
           v
4. Commit Changes
   docker commit <container_id> mywebserver:v1
           |
           v
5. New Image Created
   mywebserver:v1

+------------------------------------------+
```

### Example

```bash
# Run a container
docker run -it ubuntu:22.04 /bin/bash

# Inside container, install dependencies
apt-get update
apt-get install -y nginx

# Exit container (Ctrl+P Ctrl+Q to detach)

# Commit the changes to create new image
docker commit <container_id> mywebserver:v1
```

### Use Cases

| Use Case | Description |
|----------|-------------|
| Quick Prototypes | Creating fast test environments |
| Debugging | Saving container state for analysis |
| Reusable Images | Converting configured container to reusable image |

### Important Notes

- Manual commits create images without documentation in a Dockerfile
- Dockerfiles are preferred for reproducible builds
- Manual commits may include unnecessary artifacts

---

## Summary

Building Docker images is a fundamental skill that involves understanding:

| Concept | Description |
|---------|-------------|
| **Dockerfile Structure** | How to structure a Dockerfile with appropriate base images |
| **BuildKit Features** | Using BuildKit features for better build control |
| **RUN vs CMD** | Understanding the difference between RUN and CMD instructions |
| **Image Tagging** | Tagging images for proper version management |
| **Image Creation Methods** | Creating images both declaratively (Dockerfile) and imperatively (docker commit) |

Mastering these concepts enables you to create efficient, reproducible, and maintainable container images for your applications.