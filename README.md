# Kubernetes Manifests

This directory contains the Kubernetes manifests for the EventSLK microservices stack:

- PostgreSQL 16 database
- Spring Cloud API gateway
- Eureka-style service discovery
- Event registration microservice
- Notification microservice (Kafka consumer)
- Client portal frontend
- Admin portal frontend

## What Gets Deployed

### PostgreSQL Database

- Deployment name: `postgresdb`
- Service name: `postgresdb`
- Type: `ClusterIP`
- Port: `5432`
- Node selector: `node-role=storage`

The database pod is pinned to the storage node and mounts data from `/mnt/postgres-data` through `hostPath`. Credentials live in the `postgres-secret` Secret (`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`).

### API Gateway

- Deployment name: `eventslk-api-gateway`
- Service name: `eventslk-api-gateway`
- Type: `NodePort`
- Container port: `8080`
- NodePort: `30080`
- Node selector: `node-role=application`

This is the single external entry point. The legacy `30080` NodePort is preserved so existing frontend configurations keep working.

### Service Discovery

- Deployment name: `eventslk-service-discovery`
- Service name: `eventslk-service-discovery`
- Type: `ClusterIP`
- Container port: `8761`
- Node selector: `node-role=application`

### Event Registration API

- Deployment name: `event-registration-api`
- Service name: `event-registration-api`
- Type: `ClusterIP`
- Container port: `8081`
- Node selector: `node-role=application`

Reads PostgreSQL credentials from `postgres-secret` and the JWT signing key from `backend-secret`.

### Notification Service

- Deployment name: `eventslk-notification-service`
- Service name: `eventslk-notification-service`
- Type: `ClusterIP`
- Container port: `8082`
- Node selector: `node-role=application`

Consumes events from Kafka at `kafka:29092`.

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
- A Kafka broker reachable at `kafka:29092` inside the cluster
- `kubectl` configured against the target cluster

Label nodes if needed:

```bash
kubectl label node <storage-node> node-role=storage
kubectl label node <application-node> node-role=application
```

The `backend-secret` Secret (containing `JWT_SECRET`) must exist before applying the event registration manifest. It is created automatically when `event-api-deployment.yaml` is applied for the first time only if you add it; otherwise create it ahead of time:

```bash
kubectl create secret generic backend-secret \
  --from-literal=JWT_SECRET=<your-jwt-secret>
```

## Apply Order

Apply the manifests in this order so service dependencies resolve cleanly:

1. `postgres-deployment.yaml` — database must be reachable before any API pod becomes ready
2. `discovery-deployment.yaml` — service registry must be up before other services register
3. `gateway-deployment.yaml` — gateway depends on discovery for routing
4. `event-api-deployment.yaml` — depends on PostgreSQL, discovery, and `backend-secret`
5. `notification-deployment.yaml` — depends on Kafka and discovery
6. `client-portal-deployment.yaml`
7. `admin-portal-deployment.yaml`

## Deploy

```bash
kubectl apply -f postgres-deployment.yaml
kubectl apply -f discovery-deployment.yaml
kubectl apply -f gateway-deployment.yaml
kubectl apply -f event-api-deployment.yaml
kubectl apply -f notification-deployment.yaml
kubectl apply -f client-portal-deployment.yaml
kubectl apply -f admin-portal-deployment.yaml
```

## Access URLs

If you are using a node IP, the exposed ports are:

- API gateway: `http://<node-ip>:30080`
- Client portal: `http://<node-ip>:30081`
- Admin portal: `http://<node-ip>:30082`

Inside the cluster, the services are reachable by name:

- `postgresdb:5432`
- `eventslk-service-discovery:8761`
- `eventslk-api-gateway:8080`
- `event-registration-api:8081`
- `eventslk-notification-service:8082`
- `client-portal:80`
- `admin-portal:80`

## Configuration Notes

- All Spring Boot services run with `SPRING_PROFILES_ACTIVE=dev`.
- `event-api-deployment.yaml` supplies `SPRING_DATASOURCE_URL=jdbc:postgresql://postgresdb:5432/event_reg_db` and pulls username/password from `postgres-secret`.
- `notification-deployment.yaml` points at the in-cluster Kafka listener `kafka:29092`.
- The frontend manifests set `API_BASE_URL` to `http://localhost:30080`, which now routes through the API gateway.
- Readiness and liveness probes are enabled for every workload.

## Images

The manifests currently point to prebuilt container images in the `kaveengayanga12` registry namespace. Update the image tags before applying them if you want to deploy a different build.

## Notes

- Do not commit real production secrets into these manifests.
- If the PostgreSQL pod fails to start, confirm `/mnt/postgres-data` exists on the storage node and that the node selector is correct.
- If the event registration API cannot connect to the database, verify that the `postgresdb` service is running and the secret values match the application config.
- See [`K8S_MICROSERVICES_MIGRATION.md`](./K8S_MICROSERVICES_MIGRATION.md) for the full monolith-to-microservices migration log.
