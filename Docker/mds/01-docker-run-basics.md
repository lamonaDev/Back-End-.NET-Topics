# Docker Run & Basic Commands

## Introduction

Docker is an open platform for developing, shipping, and running applications using containerization technology. Containers are lightweight and include everything needed to run the software: code, runtime, system tools, libraries, and settings. This document covers the fundamental Docker commands for running containers and understanding their basic operations.

---

## Docker Run Command

The `docker run` command is one of the most frequently used commands in Docker. It creates and starts a container from a specified image.

### Command Syntax

```bash
docker run <image-name>
```

### What It Does

The `docker run` command performs three main operations in sequence:

```
+------------------------+     +-------------------+     +------------------+
|  1. PULL IMAGE        |---->|  2. CREATE        |---->|  3. EXECUTE      |
|  (from registry if    |     |  (new container   |     |  (run specified  |
|   not cached)         |     |   from image)     |     |   command)       |
+------------------------+     +-------------------+     +------------------+
```

### Common Examples

```bash
# Run Alpine Linux with ping command
docker run alpine ping google.com

# Run Ubuntu with echo command
docker run ubuntu echo hello world

# Run Nginx (starts web server)
docker run nginx
```

---

## Override Command

Docker images define a default command through `CMD` or `ENTRYPOINT` instructions in their Dockerfile. The override command allows you to replace this default with a different command.

### When to Use Override

| Scenario | Description |
|----------|-------------|
| Testing | Run specific tests without starting the main service |
| Debugging | Access shell inside container for troubleshooting |
| Custom Execution | Run a different process from the installed tools |
| One-time Tasks | Execute a single command without persistent service |

### Examples

```bash
# Override with ping command
docker run alpine ping google.com

# Override with echo command
docker run ubuntu echo hello world

# Override to get interactive shell
docker run -it ubuntu bash
```

---

## Docker PS Command

The `docker ps` command lists all running containers under the Docker kernel.

### Command Syntax

```bash
docker ps
```

### Output Columns

| Column | Description |
|--------|-------------|
| CONTAINER ID | Unique identifier for the container |
| IMAGE | The Docker image used |
| COMMAND | The command being executed |
| STATUS | Current status (Up, Exited, Paused) |
| PORTS | Port mappings |
| NAMES | Auto-generated or custom container name |

### Note

If no containers are running, the output will be empty with no rows displayed.

---

## Docker PS --all Command

The `docker ps --all` (or `docker ps -a`) command lists ALL containers regardless of their status.

### Command Syntax

```bash
docker ps --all
# or
docker ps -a
```

### Container States Displayed

```
+----------------------------------------------------------+
|                    CONTAINER STATES                       |
+----------------------------------------------------------+
|                                                           |
|  +----------------+    +----------------+                 |
|  |   RUNNING      |    |   STOPPED      |                 |
|  |   containers   |    |   (Exited)     |                 |
|  +----------------+    +----------------+                 |
|                                                           |
|  +----------------+    +----------------+                 |
|  |   PAUSED       |    |   CREATED      |                 |
|  |   containers   |    |   (never start)|                 |
|  +----------------+    +----------------+                 |
|                                                           |
+----------------------------------------------------------+
```

### Use Case

Useful for viewing container history, finding old containers to restart, or identifying containers that need cleanup.

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker run <image>` | Run a container from an image |
| `docker run -it <image> sh` | Interactive shell in container |
| `docker run --rm <image>` | Auto-delete container after exit |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers |