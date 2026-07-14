# Kubernetes Microservices & PostgreSQL Migration Log

**Date:** 2026-05-17
**Topic:** Monolith to Microservices K8s Manifest Migration with PostgreSQL Integration

## Database Engine Change

The backend persistence layer has moved from **MySQL 8.0** to **PostgreSQL 16**.

- **Image:** `mysql:8.0` → `postgres:16`
- **Port:** `3306` → `5432`
- **Default database:** `event_reg_db` (unchanged)
- **Admin user:** `event_user` → `root1`
- **Secret name:** `mysql-secret` → `postgres-secret`
- **Secret keys:** `MYSQL_USER` / `MYSQL_PASSWORD` / `MYSQL_DATABASE` → `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB`
- **Service / Deployment name:** `mysql` → `postgresdb`
- **hostPath data directory:** `/mnt/mysql-data` → `/mnt/postgres-data` (mounted at `/var/lib/postgresql/data`, with `PGDATA=/var/lib/postgresql/data/pgdata` to keep ownership clean)
- **JDBC URL:** `jdbc:mysql://mysql:3306/event_reg_db` → `jdbc:postgresql://postgresdb:5432/event_reg_db`
- **Hibernate dialect:** `org.hibernate.dialect.PostgreSQLDialect` (explicitly set on the event-registration API)
- **Readiness probe:** TCP socket → `pg_isready` exec probe for a real readiness signal

The Spring Cloud microservices read all PostgreSQL credentials from the new `postgres-secret`. The existing `backend-secret` (containing `JWT_SECRET`) is preserved and reused by `event-registration-api`.

## Architectural Change: Monolith → Microservices

The single `backend` Spring Boot service has been decomposed into four cooperating services behind an API gateway:

| Old | New |
|---|---|
| `backend` (NodePort `30080`) | `eventslk-api-gateway` (NodePort `30080`) — single edge entry point |
| — | `eventslk-service-discovery` — Eureka-style registry on `8761` |
| `backend` business logic | `event-registration-api` on `8081` |
| `backend` async work | `eventslk-notification-service` on `8082`, consuming Kafka at `kafka:29092` |

The external `30080` NodePort is retained on the gateway so the existing client/admin portals keep working without a configuration change.

## Changes Made

**Created**

- `k8s-manifests/postgres-deployment.yaml` — Secret (`postgres-secret`), Deployment (`postgresdb`), ClusterIP Service on `5432`, hostPath `/mnt/postgres-data`, `node-role=storage`.
- `k8s-manifests/gateway-deployment.yaml` — Deployment (`eventslk-api-gateway`) with `imagePullPolicy: Always`, NodePort Service mapping `8080 → 30080`, `/actuator/health` probes, `node-role=application`.
- `k8s-manifests/discovery-deployment.yaml` — Deployment (`eventslk-service-discovery`), ClusterIP Service on `8761`, `node-role=application`.
- `k8s-manifests/event-api-deployment.yaml` — Deployment (`event-registration-api`), ClusterIP Service on `8081`, PostgreSQL env from `postgres-secret`, `JWT_SECRET` from `backend-secret`, `/auth` probes, `node-role=application`.
- `k8s-manifests/notification-deployment.yaml` — Deployment (`eventslk-notification-service`), ClusterIP Service on `8082`, Kafka env (`KAFKA_BOOTSTRAP_SERVERS=kafka:29092`), `/actuator/health` probes, `node-role=application`.
- `k8s-manifests/K8S_MICROSERVICES_MIGRATION.md` — this log.

**Modified**

- `k8s-manifests/README.md` — rewritten to reflect the new microservice topology, PostgreSQL service, updated apply order, and access URLs.

**Deleted**

- `k8s-manifests/backend-deployment.yaml` — replaced by the gateway + discovery + event-api + notification stack.
- `k8s-manifests/mysql-deployment.yaml` — replaced by `postgres-deployment.yaml`.

**Unchanged**

- `k8s-manifests/client-portal-deployment.yaml`
- `k8s-manifests/admin-portal-deployment.yaml`

## Port and Routing Map

| Service Name | K8s Service Type | Container Port | External NodePort |
|---|---|---|---|
| `postgresdb` | ClusterIP | 5432 | — |
| `eventslk-service-discovery` | ClusterIP | 8761 | — |
| `eventslk-api-gateway` | NodePort | 8080 | **30080** |
| `event-registration-api` | ClusterIP | 8081 | — |
| `eventslk-notification-service` | ClusterIP | 8082 | — |
| `client-portal` | NodePort | 80 | 30081 |
| `admin-portal` | NodePort | 80 | 30082 |

All non-edge services are deliberately `ClusterIP` — external traffic must enter through the gateway on `30080`.

## Node Placement

| Workload | `nodeSelector` |
|---|---|
| `postgresdb` | `node-role: storage` |
| `eventslk-api-gateway`, `eventslk-service-discovery`, `event-registration-api`, `eventslk-notification-service`, `client-portal`, `admin-portal` | `node-role: application` |

Label nodes before deploying:

```bash
kubectl label node <storage-node> node-role=storage
kubectl label node <application-node> node-role=application
```

## Deployment Order Guide

Apply in the order below so each dependency is reachable before its consumers boot:

1. **Prerequisites**
   - Verify the storage node has `/mnt/postgres-data` (will be created by `DirectoryOrCreate` if missing, but the path must be writable by the kubelet).
   - Verify a Kafka broker is reachable at `kafka:29092` inside the cluster.
   - Create `backend-secret` if it does not already exist:
     ```bash
     kubectl create secret generic backend-secret \
       --from-literal=JWT_SECRET=<your-jwt-secret>
     ```

2. **Database** — must be ready before any API pod can become ready.
   ```bash
   kubectl apply -f postgres-deployment.yaml
   kubectl rollout status deployment/postgresdb
   ```

3. **Service discovery** — must be up so other Spring Cloud services can register.
   ```bash
   kubectl apply -f discovery-deployment.yaml
   kubectl rollout status deployment/eventslk-service-discovery
   ```

4. **API gateway** — registers with discovery and exposes the edge `NodePort`.
   ```bash
   kubectl apply -f gateway-deployment.yaml
   kubectl rollout status deployment/eventslk-api-gateway
   ```

5. **Event registration API** — depends on PostgreSQL, discovery, and `backend-secret`.
   ```bash
   kubectl apply -f event-api-deployment.yaml
   kubectl rollout status deployment/event-registration-api
   ```

6. **Notification service** — depends on Kafka and discovery.
   ```bash
   kubectl apply -f notification-deployment.yaml
   kubectl rollout status deployment/eventslk-notification-service
   ```

7. **Frontends**
   ```bash
   kubectl apply -f client-portal-deployment.yaml
   kubectl apply -f admin-portal-deployment.yaml
   ```

8. **Smoke test**
   ```bash
   kubectl get pods -o wide
   kubectl get svc
   curl http://<node-ip>:30080/actuator/health
   ```

## Rollback

If the migration needs to be undone, remove the new resources in reverse order:

```bash
kubectl delete -f notification-deployment.yaml
kubectl delete -f event-api-deployment.yaml
kubectl delete -f gateway-deployment.yaml
kubectl delete -f discovery-deployment.yaml
kubectl delete -f postgres-deployment.yaml
```

The deleted `mysql-deployment.yaml` and `backend-deployment.yaml` files are available in git history if a legacy rollback is required.

---

# Stateful Infrastructure & HA Datastores

**Date:** 2026-06-08
**Topic:** In-cluster Kafka message bus + high-availability PostgreSQL and Redis for the production profile.

## Why

The microservice manifests referenced infrastructure that did not exist as manifests:

- `notification-deployment.yaml` consumed Kafka at `kafka:29092`, but there was no Kafka manifest — the broker was assumed to be supplied externally.
- `application-prod.yml` describes a primary/replica PostgreSQL split (`DB_PRIMARY_HOST` / `DB_REPLICA_HOSTS`) and a Redis Sentinel topology (`REDIS_SENTINEL_*`), neither of which had manifests.

These three stateful systems are now defined in-cluster.

## Created

- `kafka-deployment.yaml` — KRaft (no ZooKeeper) Kafka, 3-broker `StatefulSet` with combined broker+controller roles, `kafka-headless` for per-broker DNS and the controller quorum, `kafka` ClusterIP as the `29092` bootstrap endpoint, and a `kafka-topic-init` Job that creates all five notification topics with `replication-factor 3` / `min.insync.replicas 2`. PodAntiAffinity spreads brokers across nodes. `node-role=application`.
- `postgres-ha-deployment.yaml` — `postgres-primary` StatefulSet (writes, WAL streaming source) + `postgres-replica` StatefulSet (2 hot standbys). Standbys clone the primary with `pg_basebackup -R -C --slot` in an initContainer and stream WAL through per-replica slots. `postgres-primary` ClusterIP is the write endpoint; `postgres-replica` is headless for per-standby DNS (for JDBC read load balancing). Credentials in `postgres-ha-secret`. Uses `volumeClaimTemplates` (StorageClass) rather than hostPath. `node-role=storage`.
- `redis-sentinel-deployment.yaml` — `redis` StatefulSet of 3 nodes, each running a `redis` data container and a `sentinel` sidecar (master `evtreg-master`, quorum 2). Nodes self-discover the master via the sentinels on boot; `redis-0` seeds the master on a cold cluster. `redis-sentinel` ClusterIP + `redis-headless` for per-pod DNS. Credentials in `redis-ha-secret`. `node-role=application`.

## Modified

- `event-api-deployment.yaml` — added `KAFKA_BOOTSTRAP_SERVERS` / `SPRING_KAFKA_BOOTSTRAP_SERVERS = kafka:29092` so the producer side of the notification pipeline reaches the in-cluster broker (previously defaulted to `localhost:9092`).
- `README.md` — documented the three new components, prerequisites (StorageClass), and dev-vs-prod apply orders.

## Port / Service Map (additions)

| Service | Type | Port | Purpose |
|---|---|---|---|
| `kafka` | ClusterIP | 29092 | Kafka bootstrap (producer + consumer) |
| `kafka-headless` | Headless | 29092 / 9093 | Per-broker DNS + KRaft controller quorum |
| `postgres-primary` | ClusterIP | 5432 | Write endpoint (`DB_PRIMARY_HOST`) |
| `postgres-replica` | Headless | 5432 | Read replicas (`DB_REPLICA_HOSTS`) |
| `redis-sentinel` | ClusterIP | 26379 | Sentinel discovery (`REDIS_SENTINEL_NODES`) |
| `redis-headless` | Headless | 6379 / 26379 | Per-pod redis + sentinel DNS |

## Production wiring (when `SPRING_PROFILES_ACTIVE=prod`)

The HA datastores match the env contract in `application-prod.yml`:

```
DB_PRIMARY_HOST=postgres-primary
DB_PRIMARY_PORT=5432
DB_NAME=event_reg_db
DB_REPLICA_HOSTS=postgres-replica-0.postgres-replica.default.svc.cluster.local:5432,postgres-replica-1.postgres-replica.default.svc.cluster.local:5432
REDIS_SENTINEL_MASTER=evtreg-master
REDIS_SENTINEL_NODES=redis-0.redis-headless.default.svc.cluster.local:26379,redis-1.redis-headless.default.svc.cluster.local:26379,redis-2.redis-headless.default.svc.cluster.local:26379
KAFKA_BOOTSTRAP_SERVERS=kafka:29092
```

## Security hardening (left as follow-up)

To keep the manifests consistent with the existing `dev`-profile stack, the in-cluster traffic is PLAINTEXT. For real production, layer on what `application-prod.yml` already anticipates: Kafka `SASL_SSL` + `SCRAM-SHA-512`, Postgres `sslmode=require` with server certs, Redis TLS, and real secret values delivered through a secret manager (the `*-secret` objects ship with `CHANGE_ME` placeholders).
