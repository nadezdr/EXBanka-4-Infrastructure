# EXBanka-4-Infrastructure

## What this repo is
A single `docker-compose.yml` that runs the entire EXBanka stack as Docker containers — all databases, RabbitMQ, all Go microservices, and the frontend. This is the deployment/integration environment. For local development of individual services, use `dev.sh` in the backend repo instead.

## How to use

```bash
# Start the full stack (builds all images on first run)
docker compose up

# Start in background
docker compose up -d

# Stop everything (containers only, volumes preserved)
docker compose down

# Rebuild a specific service after code changes
docker compose up --build <service-name>

# Nuke everything including all data volumes
docker compose down -v
```

## Relation to other repos
- **Backend**: Build contexts point to `../EXBanka-4-Backend`. The Dockerfiles live there, one per service at `services/<name>/Dockerfile`.
- **Frontend**: Build context points to `../EXBanka-4-Frontend`.
- Both repos must be cloned as siblings of this repo for the relative paths to resolve.

## .env file
Required variables (copy and fill in):
```
# email-service — SMTP
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASSWORD=
FROM_EMAIL=

# auth-service / api-gateway
JWT_SECRET=

# securities-service — external API keys
ALPACA_API_KEY=
ALPACA_API_SECRET_KEY=
ALPHAVANTAGE_API_KEY=
```

## Exposed host ports

| Service | Host Port | Notes |
|---|---|---|
| `api-gateway` | 8083 | Main HTTP entry point |
| `frontend` | 3000 | React app (nginx) |
| `rabbitmq` | 5672, 15672 | AMQP + management UI |

All other services (databases, Go microservices) communicate only over the internal `exbanka-net` Docker network and are not exposed to the host.

## Service overview

| Go service | gRPC port (internal) | DB |
|---|---|---|
| `employee-service` | 50051 | `employee-db` |
| `auth-service` | 50052 | `auth-db` |
| `email-service` | 50053 | RabbitMQ |
| `account-service` | 50054 | `account-db` |
| `payment-service` | 50055 | `payment-db` |
| `client-service` | 50056 | `client-db` |
| `exchange-service` | 50057 | `exchange-db` |
| `loan-service` | 50058 | `loan-db` |
| `card-service` | 50059 | `card-db` |
| `securities-service` | 50060 | `securities-db` |
| `order-service` | 50061 | `order-db` |
| `portfolio-service` | 50062 | `portfolio-db` |

## Adding a new service
Checklist when a new Go microservice is added to the backend:

1. **DB container** — add a `<name>-db` service (postgres:16-alpine), healthcheck, volume mount for `schema.sql`
2. **Service container** — add `<name>-service` with `build.context: ../EXBanka-4-Backend`, correct Dockerfile path, `PORTFOLIO_DB_URL`-style env var, `depends_on` the DB healthcheck
3. **Volume** — add `<name>-db-data:` to the top-level `volumes` block
4. **Dependents** — add `<SERVICE>_SERVICE_ADDR: <name>-service:<port>` env var and `depends_on` entry to any service that calls it
5. **api-gateway** — if the gateway routes to this service, add `<NAME>_SERVICE_ADDR` env var and `depends_on` entry there too
