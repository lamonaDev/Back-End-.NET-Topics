# Container Logging & Stopping

## Introduction

When running containers in production or development environments, understanding how to retrieve logs and properly stop containers is essential for debugging, monitoring, and resource management. This document covers the commands for retrieving container outputs and the different methods for stopping containers, including the important distinction between graceful and forced termination.

---

## Retrieving Log Outputs

### Docker Logs Command

The `docker logs` command fetches the logs of a container, including both stdout and stderr streams.

### Command Syntax

```bash
docker logs <container-id>
```

### Useful Flags

| Flag | Description |
|------|-------------|
| `--tail N` | Show last N lines |
| `-f` | Follow log output in real-time |
| `-t` | Show timestamps |
| `--since` | Show logs since specific time |

### Examples

```bash
# View logs
docker logs 2dd34fbda5e7

# View last 100 lines
docker logs --tail 100 2dd34fbda5e7

# Follow logs in real-time
docker logs -f 2dd34fbda5e7

# Show timestamps
docker logs -t 2dd34fbda5e7

# Combine options
docker logs -ft --tail 50 2dd34fbda5e7
```

---

## Stopping Containers

### Docker Stop

The `docker stop` command gracefully stops a running container.

### Command Syntax

```bash
docker stop <container-id>
```

### What It Does

```
+------------------------------------------+
|         DOCKER STOP PROCESS             |
+------------------------------------------+

  Step 1: Send SIGTERM signal to process
              |
              v
  Step 2: Wait for graceful shutdown
     (default timeout: 10 seconds)
              |
              v
  Step 3: If timeout exceeded, send SIGKILL
              |
              v
  Step 4: Container stopped

+------------------------------------------+
```

### Custom Timeout

```bash
docker stop -t 30 2dd34fbda5e7
# Wait 30 seconds before SIGKILL
```

---

### Docker Kill

The `docker kill` command immediately stops a container by sending SIGKILL.

### Command Syntax

```bash
docker kill <container-id>
```

### What It Does

- Immediately sends SIGKILL signal
- Does NOT wait for graceful shutdown
- Container stops immediately without cleanup

---

## Difference: docker stop vs docker kill

```
+------------------------+------------------+------------------+
| Aspect                 | docker stop      | docker kill      |
+------------------------+------------------+------------------+
| Signal Sent            | SIGTERM          | SIGKILL          |
| Graceful Shutdown      | Yes              | No               |
| Cleanup Time           | Waits (10s def)  | Immediate        |
| Data Safety            | Higher           | Lower            |
| Use Case               | Normal shutdown  | Force stop       |
+------------------------+------------------+------------------+
```

### Signal Flow Diagram

```
docker stop:
  SIGTERM ----> Container --(waits)--> Clean Exit

docker kill:
  SIGKILL ----> Container --(immediate)--> Force Exit
```

---

## Signal Messages Explained

### SIGTERM (Signal 15)

| Property | Value |
|----------|-------|
| Signal Number | 15 |
| Purpose | Graceful termination request |
| Behavior | Process can catch, handle, and clean up before exiting |
| Example Use | Normal application shutdown, saving data, closing connections |

### SIGKILL (Signal 9)

| Property | Value |
|----------|-------|
| Signal Number | 9 |
| Purpose | Immediate termination |
| Behavior | Cannot be caught or ignored - process is killed instantly |
| Example Use | Force stopping a frozen/hanging container |

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker stop <container>` | Gracefully stop container |
| `docker kill <container>` | Force stop container |
| `docker logs <container>` | View container logs |
| `docker logs -f <container>` | Follow logs in real-time |