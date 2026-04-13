# Kubernetes Manifests

This directory contains the Kubernetes manifests for the EventSLK stack:

- MySQL database
- Spring Boot backend API
- Client portal frontend
- Admin portal frontend

## What Gets Deployed

### MySQL

- Deployment name: `mysql`
- Service name: `mysql`
- Type: `ClusterIP`
- Port: `3306`
- Node selector: `node-role=storage`

The database pod is pinned to the storage node and mounts data from `/mnt/mysql-data` through `hostPath`.

### Backend API

- Deployment name: `backend`
- Service name: `backend`
- Type: `NodePort`
- Container port: `8080`
- NodePort: `30080`
- Node selector: `node-role=application`

The backend reads database and JWT configuration from environment variables and connects to the internal MySQL service.

### Client Portal

- Deployment name: `client-portal`
- Service name: `client-portal`
- Type: `NodePort`
- Container port: `80`
- NodePort: `30081`
- Node selector: `node-role=application`

### Admin Portal

- Deployment name: `admin-portal`
- Service name: `admin-portal`
- Type: `NodePort`
- Container port: `80`
- NodePort: `30082`
- Node selector: `node-role=application`

## Prerequisites

- A Kubernetes cluster with at least two labeled nodes
- A storage node labeled `node-role=storage`
- An application node labeled `node-role=application`
- `kubectl` configured against the target cluster

Label nodes if needed:

```bash
kubectl label node <storage-node> node-role=storage
kubectl label node <application-node> node-role=application
```

## Apply Order

Apply the manifests in this order:

1. `mysql-deployment.yaml`
2. `backend-deployment.yaml`
3. `client-portal-deployment.yaml`
4. `admin-portal-deployment.yaml`

The backend depends on the database service being available first.

## Deploy

```bash
kubectl apply -f mysql-deployment.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f client-portal-deployment.yaml
kubectl apply -f admin-portal-deployment.yaml
```

## Access URLs

If you are using a node IP, the exposed ports are:

- Backend API: `http://<node-ip>:30080`
- Client portal: `http://<node-ip>:30081`
- Admin portal: `http://<node-ip>:30082`

Inside the cluster, the services are reachable by name:

- `mysql:3306`
- `backend:8080`
- `client-portal:80`
- `admin-portal:80`

## Configuration Notes

- `backend-deployment.yaml` sets `SPRING_PROFILES_ACTIVE=dev`.
- `backend-deployment.yaml` supplies `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, and `SPRING_DATASOURCE_PASSWORD`.
- The frontend manifests set `API_BASE_URL` to `http://localhost:30080`.
- Readiness and liveness probes are enabled for all workloads.

## Images

The manifests currently point to prebuilt container images in the `kaveengayanga12` registry namespace. Update the image tags before applying them if you want to deploy a different build.

## Notes

- Do not commit real production secrets into these manifests.
- If the MySQL pod fails to start, confirm `/mnt/mysql-data` exists on the storage node and that the node selector is correct.
- If the backend cannot connect to the database, verify that the `mysql` service is running and the secret values match the application config.
