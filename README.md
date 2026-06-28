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

### Kafka (message bus)

- StatefulSet name: `kafka` (3 brokers, KRaft mode — no ZooKeeper)
- Bootstrap service: `kafka` (ClusterIP) on `29092`
- Headless service: `kafka-headless` for per-broker DNS + the KRaft controller quorum
- Topic bootstrap: `kafka-topic-init` Job creates all topics with replication-factor 3, `min.insync.replicas=2`
- Node selector: `node-role=application`

Carries events from the `event-registration-api` producer to the
`eventslk-notification-service` consumer. Topics: `eventslk.user.signup`,
`eventslk.user.otp`, `eventslk.booking.confirmed`, `eventslk.booking.cancelled`,
`eventslk.promotion`. Defined in [`kafka-deployment.yaml`](./kafka-deployment.yaml).

### High-Availability PostgreSQL (production profile)

- Primary StatefulSet/Service: `postgres-primary` (writes) on `5432`
- Replica StatefulSet: `postgres-replica` (2 hot standbys, streaming replication)
- Replica headless Service: `postgres-replica` for per-standby DNS
- Secret: `postgres-ha-secret`
- Node selector: `node-role=storage`

Implements the primary/replica split that `application-prod.yml` expects
(`DB_PRIMARY_HOST` / `DB_REPLICA_HOSTS`). Standbys clone the primary via
`pg_basebackup` and stream WAL through per-replica replication slots. Defined in
[`postgres-ha-deployment.yaml`](./postgres-ha-deployment.yaml). This is the
production alternative to the single-instance `postgres-deployment.yaml` — apply
only one of the two per database.

### High-Availability Redis (production profile)

- StatefulSet: `redis` (3 nodes, each with a sentinel sidecar)
- Sentinel service: `redis-sentinel` (ClusterIP) on `26379`
- Headless service: `redis-headless` for per-pod DNS
- Secret: `redis-ha-secret`
- Master name: `evtreg-master`, quorum `2`
- Node selector: `node-role=application`

Implements the Redis Sentinel topology `application-prod.yml` expects
(`REDIS_SENTINEL_MASTER` / `REDIS_SENTINEL_NODES`). Defined in
[`redis-sentinel-deployment.yaml`](./redis-sentinel-deployment.yaml).

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
- A default `StorageClass` (or set `storageClassName` in the StatefulSet
  `volumeClaimTemplates`) for the Kafka / HA-Postgres / Redis persistent volumes
- `kubectl` configured against the target cluster

Kafka is now provided in-cluster by [`kafka-deployment.yaml`](./kafka-deployment.yaml)
(`kafka:29092`); you no longer need an external broker.

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

1. `postgres-deployment.yaml` (dev) **or** `postgres-ha-deployment.yaml` (prod) — database must be reachable before any API pod becomes ready
2. `redis-sentinel-deployment.yaml` — (prod profile) cache must be up; the prod readiness probe includes Redis
3. `kafka-deployment.yaml` — message bus must be up before the producer/consumer
4. `discovery-deployment.yaml` — service registry must be up before other services register
5. `gateway-deployment.yaml` — gateway depends on discovery for routing
6. `event-api-deployment.yaml` — depends on PostgreSQL, Kafka, discovery, and `backend-secret`
7. `notification-deployment.yaml` — depends on Kafka and discovery
8. `client-portal-deployment.yaml`
9. `admin-portal-deployment.yaml`

## Deploy

### Dev profile (single-instance datastores)

```bash
kubectl apply -f postgres-deployment.yaml
kubectl apply -f kafka-deployment.yaml
kubectl apply -f discovery-deployment.yaml
kubectl apply -f gateway-deployment.yaml
kubectl apply -f event-api-deployment.yaml
kubectl apply -f notification-deployment.yaml
kubectl apply -f client-portal-deployment.yaml
kubectl apply -f admin-portal-deployment.yaml
```

### Production profile (HA datastores)

Use the HA Postgres + Redis Sentinel manifests instead of the single-instance
Postgres, and switch the app deployments to `SPRING_PROFILES_ACTIVE=prod` with the
matching `DB_PRIMARY_HOST` / `DB_REPLICA_HOSTS` / `REDIS_SENTINEL_NODES` /
`KAFKA_BOOTSTRAP_SERVERS` env (see Configuration Notes below).

```bash
kubectl apply -f postgres-ha-deployment.yaml
kubectl apply -f redis-sentinel-deployment.yaml
kubectl apply -f kafka-deployment.yaml
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
