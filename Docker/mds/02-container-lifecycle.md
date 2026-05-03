# Container Lifecycle

## Introduction

Understanding the container lifecycle is fundamental to working effectively with Docker. Containers move through various states from creation to deletion, and knowing how to manage these transitions is essential for container orchestration and troubleshooting. This document explains the complete container lifecycle and the commands to manage each state.

---

## Container States Overview

A Docker container can exist in several states throughout its lifetime:

```
+------------------+     +------------------+     +------------------+
|    CREATED       |---->|    RUNNING       |---->|    STOPPED       |
|                  |     |                  |     |    (Exited)      |
+------------------+     +------------------+     +------------------+
       ^                       |                        |
       |                       |                        |
       +-----------------------+------------------------+
                    Container States
```

## States Explained

| State | Description |
|-------|-------------|
| **Created** | Container exists but has not been started |
| **Running** | Container is currently executing |
| **Paused** | Container processes are suspended |
| **Stopped (Exited)** | Container ran and completed or was stopped |
| **Dead** | Container exists but cannot be started |

---

## Docker Create

The `docker create` command creates a container from an image WITHOUT starting it.

### Command Syntax

```bash
docker create <image-name>
```

### What It Does

- Creates a new container from the specified image
- Returns the container ID
- Allocates filesystem space for the container
- Does NOT start the container

### Example

```bash
# Create container without starting it
docker create nginx

# Output: a1b2c3d4e5f6...
```

### Difference from docker run

```
+------------------------------------------+
|           OPERATION COMPARISON           |
+------------------------------------------+

docker run     = docker create + docker start
docker create = only creates the container
docker start  = starts an existing container

+------------------------------------------+
|  docker run nginx                        |
|  (single command does both)             |
+------------------------------------------+
```

---

## Docker Start

The `docker start` command starts a previously created or stopped container.

### Command Syntax

```bash
docker start <container-id>
```

### What It Does

- Starts a previously created container
- Continues execution from where it stopped
- Can attach output to terminal

### With Attached Output

```bash
docker start -a <container-id>
```

The `-a` flag attaches the container's output to your terminal.

---

## Restarting Exited Containers

To restart a container that has exited:

### Step-by-step Process

```bash
# Step 1: List all containers (including stopped)
docker ps -a

# Step 2: Find the exited container ID
# Let's say: 2dd34fbda5e7

# Step 3: Start the container
docker start 2dd34fbda5e7

# Step 4: View running containers
docker ps
```

---

## Removing Stopped Containers

### Command Syntax

```bash
docker rm <container-id>
```

### Examples

```bash
# Remove a specific stopped container
docker rm 2dd34fbda5e7

# Remove multiple containers
docker rm abc123 def456

# Remove all stopped containers
docker ps -a | grep Exited | awk '{print $1}' | xargs docker rm
```

---

## Docker System Prune

The `docker system prune` command cleans up unused Docker resources.

### Command Syntax

```bash
docker system prune
```

### What It Removes

- All stopped containers
- All dangling images (untagged)
- All build cache
- All networks not used by at least one container

### Variations

```bash
# Basic prune (prompts for confirmation)
docker system prune

# Prune without confirmation
docker system prune -f

# Remove all unused images (not just dangling)
docker system prune -a

# Remove all volumes (WARNING: deletes data)
docker system prune --volumes

# Show what would be removed without actually removing
docker system prune --dry-run
```

### Output Example

```
WARNING! This will remove:
  - all stopped containers
  - all networks not used by at least one container
  - all dangling images
  - all build caches
Are you sure you want to continue? [y/N] y
Deleted Containers...
Deleted Images...
Total reclaimed space: ...
```

---

## Container Lifecycle Flow

```
1. CREATE   : docker create nginx
2. START    : docker start nginx
3. RUN      : docker run nginx (create + start)
4. STOP     : docker stop nginx
5. START    : docker start nginx (again)
6. REMOVE   : docker rm nginx

+------------------------------------------+
|         COMPLETE LIFECYCLE               |
+------------------------------------------+
|                                          |
|  [Image] ---> [Create] ---> [Running]   |
|                           |              |
|                           v              |
|                      [Stopped]           |
|                           |              |
|                           v              |
|                      [Removed]           |
|                                          |
+------------------------------------------+
```