# Kubernetes Manifests

Kubernetes manifests for the EventsLK microservices stack, production profile:

- **Kong API gateway** (DB-less, declarative) — the single public API entry point
- **Event registration API** (Spring Boot, `prod` profile, Flyway migrations)
- **Notification service** (Go, Kafka consumer, Brevo email)
- **Kafka** (3-broker KRaft StatefulSet)
- **HA PostgreSQL** (primary + 2 streaming replicas)
- **HA Redis** (3 nodes + Sentinel sidecars, cache backend)
- **Zipkin** (tracing collector/UI — see prerequisite note in the manifest)
- **Eureka service discovery**, client portal, admin portal

> The Spring Cloud API gateway (`gateway-deployment.yaml`) has been **removed** —
> Kong replaced it. Kong proxies the raw backend paths (`/auth`, `/event`,
> `/book`, `/user`) with `strip_path: false`; there is no `/api/v1` prefix.

## Public surface

Only three NodePorts are exposed (the Terraform security group opens
30000-32767):

| Port | What | Notes |
|---|---|---|
| `30080` | Kong proxy | All API traffic. Rate-limited 60/min, strips `X-User-*` headers |
| `30081` | Client portal | nginx static |
| `30082` | Admin portal | nginx static |

Everything else — Kong admin (8001), actuator management port (9081), Zipkin
(9411), Kafka, Postgres, Redis, Eureka — is ClusterIP/headless only. Keep it
that way: the SG opens the whole NodePort range to the internet, so **adding a
NodePort publishes the service**.

An ALB Ingress is *not* used: this is a kubeadm + Calico cluster on EC2, where
the AWS Load Balancer Controller's `target-type: ip` cannot reach pod IPs. If
you later want an ALB, create it in Terraform with an instance target group on
port 30080.

## Prerequisites

1. **Cluster** built by `deploy/infrastructure` (Terraform → EC2, Ansible →
   kubeadm + Calico + ArgoCD). One worker labeled per role:

   ```bash
   kubectl label node <storage-node>     node-role=storage
   kubectl label node <application-node> node-role=application
   ```

2. **A default StorageClass.** kubeadm clusters ship with none, so the
   Kafka / HA-Postgres / Redis `volumeClaimTemplates` would stay `Pending`
   forever. Install the local-path provisioner and make it the default:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml
   kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
   ```

   (Local-path volumes live on the node's disk — data survives pod restarts
   but not node loss. For real durability move to EBS-backed storage later.)

3. **Capacity.** With 2× `m7i-flex.large` workers (8 GiB each) the full HA
   stack is *tight*: Kafka×3 + Redis×3 + Postgres×3 + apps. Either raise
   `worker_count` in `terraform.tfvars` (label ≥2 as `application`), or trim
   replica counts (Kafka 3→1 also means changing its replication-factor env
   and the topic-init job). Anti-affinity in the HA manifests is *preferred*,
   not required, so pods will co-locate on a single node — that runs, but a
   node loss takes out every "replica" at once.

4. **Secrets** (all in `default` namespace; never commit real values):

   ```bash
   # Postgres HA + Redis HA: edit the CHANGE_ME values inside
   # postgres-ha-deployment.yaml / redis-sentinel-deployment.yaml before
   # applying, or better, delete those inline Secrets and create them here:

   kubectl create secret generic postgres-ha-secret \
     --from-literal=POSTGRES_DB=event_reg_db \
     --from-literal=POSTGRES_USER=evtreg \
     --from-literal=POSTGRES_PASSWORD='<strong-password>' \
     --from-literal=REPLICATION_USER=replicator \
     --from-literal=REPLICATION_PASSWORD='<strong-password>'

   kubectl create secret generic redis-ha-secret \
     --from-literal=REDIS_PASSWORD='<strong-password>' \
     --from-literal=REDIS_SENTINEL_PASSWORD='<strong-password>'

   # Backend: RS256 keypair (PEM with literal \n escapes, same format as the
   # local .env) + seed admin credentials.
   kubectl create secret generic backend-secret \
     --from-literal=JWT_PRIVATE_KEY="$JWT_PRIVATE_KEY" \
     --from-literal=JWT_PUBLIC_KEY="$JWT_PUBLIC_KEY" \
     --from-literal=APP_ADMIN_EMAIL='admin@yourdomain' \
     --from-literal=APP_ADMIN_PASSWORD='<strong-password>'

   # Notification service: Brevo transactional-email API key.
   kubectl create secret generic notification-secret \
     --from-literal=BREVO_API_KEY='<brevo-key>'

   # Optional — only if not using the EC2 instance role for S3:
   kubectl create secret generic aws-s3-secret \
     --from-literal=AWS_S3_ACCESS_KEY='...' \
     --from-literal=AWS_S3_SECRET_KEY='...'
   ```

5. **Replace every `REPLACE_PUBLIC_HOST`** in `event-api-deployment.yaml`,
   `notification-deployment.yaml`, and the two portal manifests with your
   public domain or the master's elastic IP. These values end up in browsers
   (portal `API_BASE_URL`), CORS allowlists, and email links.

6. **S3 image uploads** (`event-api-deployment.yaml`): set `AWS_S3_BUCKET` to
   enable real uploads (empty = mock mode with placeholder images). Prefer the
   worker EC2 instance role over static keys — Terraform already sets the IMDS
   hop limit to 2 so pods can use it; attach an `s3:PutObject`/`GetObject`
   policy for the bucket. The bucket also needs a CORS rule allowing `PUT`
   with `Content-Type` + `Cache-Control` headers from the admin-portal origin,
   and public (or CloudFront-fronted) read for `events/*`.

## Apply order

```bash
# datastores first
kubectl apply -f postgres-ha-deployment.yaml     # or postgres-deployment.yaml (dev only)
kubectl apply -f redis-sentinel-deployment.yaml
kubectl apply -f kafka-deployment.yaml

# platform
kubectl apply -f discovery-deployment.yaml
kubectl apply -f zipkin-deployment.yaml          # optional (see note in file)
kubectl apply -f kong-gateway-deployment.yaml

# apps (wait for datastores to be Ready first: kubectl get pods -w)
kubectl apply -f event-api-deployment.yaml
kubectl apply -f notification-deployment.yaml
kubectl apply -f client-portal-deployment.yaml
kubectl apply -f admin-portal-deployment.yaml
```

The event API runs Flyway on boot (prod profile, `ddl-auto: validate`), so a
fresh database is migrated automatically; no manual schema step.

## Smoke test

```bash
PUB=<public-host>
curl http://$PUB:30080/actuator/health          # via Kong -> mgmt port 9081
curl http://$PUB:30080/event                    # public event list
# portals
open http://$PUB:30081                          # client
open http://$PUB:30082/login.html               # admin (seed admin creds)
```

## In-cluster service names

| Service | Address |
|---|---|
| Kong proxy / admin | `kong-proxy-svc:8000` / `kong-admin-svc:8001` |
| Event API (http / mgmt) | `event-registration-api:8081` / `:9081` |
| Notification | `eventslk-notification-service:8082` |
| Kafka bootstrap | `kafka:29092` |
| Postgres write endpoint | `postgres-primary:5432` |
| Postgres replicas (per-pod) | `postgres-replica-0.postgres-replica:5432`, `-1…` |
| Redis sentinels (per-pod) | `redis-{0,1,2}.redis-headless:26379` |
| Zipkin | `zipkin:9411` (`kubectl port-forward svc/zipkin 9411:9411`) |
| Eureka | `eventslk-service-discovery:8761` |

## Configuration notes

- The event API's prod profile has **no RoutingDataSource implementation yet**
  — `spring.datasource.primary/replica` in `application-prod.yml` is inert and
  the manifest wires `SPRING_DATASOURCE_URL` to `postgres-primary`. Reads do
  not hit the replicas until that bean exists; the replicas still give you a
  warm standby.
- In-cluster Kafka is PLAINTEXT, so the manifest overrides the prod SASL/SSL
  defaults; `KAFKA_SASL_JAAS_CONFIG` must exist (even empty) because Spring
  binds it. Same idea for Redis: `REDIS_SSL_ENABLED=false`.
- Actuator runs on management port **9081** with k8s probe groups
  (`/actuator/health/readiness` includes db + redis). Kong exposes only
  `/actuator/health` publicly, via a dedicated upstream to 9081.
- Zipkin receives nothing until the tracing dependencies are added to the
  Spring API — the exact pom/yml snippet is in `zipkin-deployment.yaml`.
- Images point at the `kaveengayanga12` registry; update tags to deploy a
  different build.

See [`K8S_MICROSERVICES_MIGRATION.md`](./K8S_MICROSERVICES_MIGRATION.md) for
the original monolith-to-microservices migration log.
