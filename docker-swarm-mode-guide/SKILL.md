---
name: docker-swarm-mode-guide
description: Guide for initializing, configuring, and managing Docker Swarm mode clusters. Use when setting up container orchestration, deploying services across multiple nodes, or managing swarm networking and scaling.
---

# Docker Swarm Mode Guide

A comprehensive skill for setting up and managing Docker Swarm mode clusters. Covers initialization, node management, service deployment, networking, scaling, secrets management, and production best practices.

## When to Use This Skill

- Setting up a new Docker Swarm cluster from scratch
- Deploying and managing multi-container services across nodes
- Configuring overlay networks for cross-node communication
- Scaling services up or down in a swarm
- Managing secrets and configs in a swarm environment
- Troubleshooting swarm node or service issues
- Planning a production-ready swarm deployment

## What This Skill Does

1. **Cluster Initialization**: Walks through initializing a swarm, joining manager and worker nodes, and verifying cluster health.
2. **Service Deployment**: Guides creating, updating, and rolling back services with replicas, constraints, and resource limits.
3. **Networking**: Configures overlay networks, ingress routing, and service discovery across the swarm.
4. **Scaling & Updates**: Manages replica scaling, rolling updates, and rollback strategies.
5. **Secrets & Configs**: Securely manages sensitive data and configuration files within the swarm.
6. **Monitoring & Troubleshooting**: Inspects services, tasks, and nodes to diagnose and resolve issues.

## How to Use

### Starting Swarm Mode

Initialize a swarm on the first manager node:

```bash
docker swarm init --advertise-addr <MANAGER-IP>
```

This outputs a join token. To retrieve tokens later:

```bash
# Get worker join token
docker swarm join-token worker

# Get manager join token
docker swarm join-token manager
```

### Joining Nodes

On each worker node, run the join command provided during init:

```bash
docker swarm join --token <WORKER-TOKEN> <MANAGER-IP>:2377
```

For additional manager nodes (recommended: 3 or 5 for HA):

```bash
docker swarm join --token <MANAGER-TOKEN> <MANAGER-IP>:2377
```

Verify the cluster:

```bash
docker node ls
```

### Deploying a Service

```bash
# Create a simple service with 3 replicas
docker service create \
  --name web \
  --replicas 3 \
  --publish published=80,target=80 \
  web-app:latest

# List running services
docker service ls

# View tasks (containers) for a service
docker service ps web
```

### Advanced Usage

Deploy a full stack using a Compose file:

```bash
docker stack deploy -c docker-compose.yml myapp
```

Example `docker-compose.yml` for swarm:

```yaml
version: "3.8"

services:
  web:
    image: nginx:alpine
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
    ports:
      - "80:80"
    networks:
      - frontend

  api:
    image: myapi:latest
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.role == worker
    secrets:
      - db_password
    networks:
      - frontend
      - backend

  db:
    image: postgres:16
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.storage == ssd
    volumes:
      - db-data:/var/lib/postgresql/data
    secrets:
      - db_password
    networks:
      - backend

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    internal: true

volumes:
  db-data:

secrets:
  db_password:
    external: true
```

## Example

**User**: "Set up a Docker Swarm cluster with 3 nodes and deploy an nginx service"

**Output**:

```bash
# Step 1: Initialize swarm on manager node
docker swarm init --advertise-addr 192.168.1.10

# Step 2: Join worker nodes (run on each worker)
docker swarm join --token SWMTKN-1-xxxxx 192.168.1.10:2377

# Step 3: Verify all nodes are ready
docker node ls
# ID          HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
# abc123 *   manager1   Ready   Active        Leader
# def456     worker1    Ready   Active
# ghi789     worker2    Ready   Active

# Step 4: Create an overlay network
docker network create --driver overlay app-net

# Step 5: Deploy nginx with 3 replicas
docker service create \
  --name nginx \
  --replicas 3 \
  --network app-net \
  --publish published=80,target=80 \
  nginx:alpine

# Step 6: Verify the service is running
docker service ls
docker service ps nginx
```

## Common Operations

### Scaling Services

```bash
# Scale up
docker service scale web=5

# Scale multiple services at once
docker service scale web=5 api=3
```

### Rolling Updates

```bash
# Update the image with a rolling strategy
docker service update \
  --image web-app:v2 \
  --update-parallelism 1 \
  --update-delay 10s \
  web

# Roll back if something goes wrong
docker service rollback web
```

### Managing Secrets

```bash
# Create a secret
echo "s3cur3p@ss" | docker secret create db_password -

# Use in a service
docker service create \
  --name api \
  --secret db_password \
  myapi:latest

# Inside the container, the secret is at /run/secrets/db_password
```

### Draining a Node for Maintenance

```bash
# Drain a node (reschedules its tasks)
docker node update --availability drain worker1

# Bring it back
docker node update --availability active worker1
```

### Inspecting and Troubleshooting

```bash
# Inspect a service
docker service inspect --pretty web

# View service logs
docker service logs web

# Check why a task failed
docker service ps --no-trunc web

# Inspect a specific node
docker node inspect --pretty worker1
```

## Tips

- Use an odd number of manager nodes (3 or 5) for fault-tolerant Raft consensus. Never use more than 7.
- Keep manager nodes dedicated to management in production; drain them from running workloads with `docker node update --availability drain <manager>`.
- Use `--update-delay` and `--update-failure-action rollback` for safe rolling deployments.
- Use overlay networks with `internal: true` for backend services that should not be publicly accessible.
- Store sensitive data in Docker secrets, not environment variables.
- Label nodes (`docker node update --label-add storage=ssd worker1`) and use placement constraints to control where services run.
- Use `docker stack deploy` with Compose files for reproducible multi-service deployments.
- Monitor swarm health with `docker node ls` and `docker service ls` regularly.
- Back up the swarm state by backing up `/var/lib/docker/swarm/` on manager nodes.
- Open required ports between nodes: TCP 2377 (cluster management), TCP/UDP 7946 (node communication), UDP 4789 (overlay network traffic).

## Common Use Cases

- Setting up a high-availability web application cluster
- Blue-green and rolling deployments with zero downtime
- Running microservices with internal service discovery
- Scheduling GPU or SSD-constrained workloads on specific nodes
- Managing database credentials and API keys with Docker secrets
- Performing zero-downtime node maintenance with drain/active toggling
