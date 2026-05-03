# Multi-Command Containers & Execution

## Introduction

Containerized applications often require interactive execution, command overrides, and the ability to run additional processes inside running containers. This document covers how to execute commands in running containers, the concept of command override, and how to work with interactive terminal sessions.

---

## Multi-Command Containers

### Using Redis as Example

Redis is a popular in-memory data structure store used as a database, cache, and message broker. It demonstrates the concept of multi-command containers well.

### What is Redis?

- In-memory data structure store
- Used as database, cache, and message broker
- Single-threaded event loop

### Running Redis Container

```bash
# Basic Redis container (starts redis-server)
docker run redis

# Run Redis with custom command (override)
docker run redis redis-cli

# Run Redis and ping it
docker run redis redis-cli ping
```

### Command Override vs Default CMD

```
+-------------------+     +-------------------+
|    Default CMD    |     |  Override CMD     |
+-------------------+     +-------------------+
| Starts redis-server|    | Runs redis-cli    |
| and waits for    |     | to interact with  |
| connections       |     | the server        |
+-------------------+     +-------------------+
```

---

## Executing Commands in Running Containers

### Docker Exec

The `docker exec` command executes a command in a RUNNING container.

### Command Syntax

```bash
docker exec <container-id> <command>
```

### What It Does

- Executes a command in a running container
- Useful for debugging, troubleshooting, and administration
- Does not start a new container

### Examples with Redis

```bash
# Run ping command inside Redis container
docker exec 2dd34fbda5e7 redis-cli ping

# Set a key
docker exec 2dd34fbda5e7 redis-cli SET mykey "Hello"

# Get a key
docker exec 2dd34fbda5e7 redis-cli GET mykey

# Get all keys
docker exec 2dd34fbda5e7 redis-cli KEYS "*"
```

---

## The -it Flag Explained

### What is -it?

The `-it` flag is actually two flags combined:

| Flag | Name | Description |
|------|------|-------------|
| `-i` | Interactive | Keeps STDIN open even if not attached |
| `-t` | Pseudo-TTY | Allocates a pseudo-terminal (gives you a terminal) |

### Comparison

```
Without -it:          With -it:
+-------------+       +-------------+
| No terminal |       | Real terminal|
| Can't type  |       | Can type     |
| No input    |       | Full input  |
+-------------+       +-------------+
```

### Examples

```bash
# Without -it (non-interactive)
docker exec 2dd34fbda5e7 redis-cli ping

# With -it (interactive shell)
docker exec -it 2dd34fbda5e7 sh
docker exec -it 2dd34fbda5e7 bash

# Run a command interactively
docker exec -it 2dd34fbda5e7 redis-cli
```

---

## STDIN, STDOUT, STDERR

### Overview

Linux processes communicate through three standard streams:

```
+----------------------------------------------------------+
|                      HOST TERMINAL                        |
+----------------------------------------------------------+
|                                                          |
|  STDIN  <---------- Input ----------+                    |
|                                    |                    |
|  STDOUT -------- Output ----------+---> Container       |
|                                    |                    |
|  STDERR ------- Error ------------+                    |
|                                                          |
+----------------------------------------------------------+
```

### Stream Definitions

| Stream | Number | Description | Use Case |
|--------|--------|-------------|----------|
| **STDIN** | 0 | Input stream | Keyboard input, piped data |
| **STDOUT** | 1 | Normal output | Regular program output |
| **STDERR** | 2 | Error output | Error messages |

### Docker and Streams

```bash
# -i keeps STDIN open
docker run -i alpine  # Can type input but no prompt

# -t gives pseudo-terminal
docker run -t alpine  # Sees prompt but cannot type

# -it both
docker run -it alpine # Full interactive terminal
```

---

## Opening Shell Inside Container

### Using Docker Exec

```bash
# Open shell in running container
docker exec -it <container-id> sh
docker exec -it <container-id> bash
```

### Examples

```bash
# Shell in Redis container
docker exec -it 2dd34fbda5e7 sh

# Shell in Alpine container
docker exec -it abc123 sh

# Shell in Ubuntu container
docker exec -it def456 bash
```

### Using Docker Run (for temporary shell)

```bash
# One-time shell session
docker run -it alpine sh
docker run -it ubuntu bash
```

---

## Getting CMD in Container

### Method 1: Inspect Command

```bash
# View running container command
docker ps --no-trunc

# Or use inspect
docker inspect <container-id> --format '{{.Config.Cmd}}'
```

### Method 2: Inspect Image

```bash
# View default image command
docker inspect <image-name> --format '{{.Config.Cmd}}'
```

### Example with Redis

```bash
# View what command Redis runs by default
docker inspect redis --format '{{.Config.Cmd}}'
# Output: [redis-server]
```

---

## Container Isolation

### Overview

Docker containers are isolated from each other and from the host system.

```
+-----------------------------------------------------------+
|                      HOST SYSTEM                          |
+-----------------------------------------------------------+
|                                                           |
|  +---------------+    +---------------+    +------------+ |
|  | Container 1  |    | Container 2   |    | Container3 | |
|  | (Redis)      |    | (Nginx)      |    | (MySQL)    | |
|  | Port: 6379   |    | Port: 80      |    | Port: 3306 | |
|  +---------------+    +---------------+    +------------+ |
|                                                           |
|  +---------------------------------------------------+   |
|  |              Docker Engine (Kernel)               |   |
|  +---------------------------------------------------+   |
|                                                           |
+-----------------------------------------------------------+
```

### Isolation Levels

| Level | What is Isolated |
|-------|-------------------|
| **Process** | Each container has its own process namespace |
| **Network** | Each container has its own network stack |
| **Filesystem** | Each container has its own filesystem |
| **User** | Each container can have its own users/UIDs |
| **UTS** | Hostname is independent per container |
| **IPC** | Inter-process communication is separate |

### Key Points

1. **No direct access:** Containers cannot see each other's processes
2. **Separate filesystems:** Changes in one container don't affect others
3. **Network isolation:** Each container has its own IP address
4. **Resource limits:** CPU, memory, I/O can be limited per container

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker exec <container> <cmd>` | Execute command in container |
| `docker exec -it <container> sh` | Open interactive shell |
| `docker exec -it <container> redis-cli` | Run Redis CLI |