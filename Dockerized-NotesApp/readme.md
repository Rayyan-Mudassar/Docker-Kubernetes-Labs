# Project 5 — Dockerized Notes API

**Flask + PostgreSQL + Docker Compose**

A containerized REST API for managing notes. This project demonstrates real-world containerization — multi-container orchestration, data persistence, container networking, and security hardening.

> **Parallel build to Project 2** — same application logic as the Serverless Notes API (API Gateway + Lambda + DynamoDB), different infrastructure layer. Built to demonstrate understanding of both serverless and container-based architectures and the tradeoffs between them.

---

## The Problem It Solves

Running multi-container apps manually with individual `docker run` commands is error-prone. Managing networks, volumes, environment variables, and startup order by hand doesn't scale and isn't reproducible across environments.

Docker Compose orchestrates the entire stack in a single YAML file — one command spins up Flask + PostgreSQL with proper networking, persistent storage, and environment configuration. Fully reproducible on any machine with Docker installed.

---

## Tech Stack

| Component | Technology |
|---|---|
| Backend | Python 3.11 + Flask 3.0 |
| Database | PostgreSQL 15 (Alpine) |
| Orchestration | Docker Compose |
| Base Image | python:3.11-slim |
| DB Driver | psycopg2-binary |
| Security | Non-root container user, secrets via env vars |

---

## Architecture

```
browser / curl
      ↓
[ app container ]  Flask API — port 5000
      ↓  (hostname: db)
[ db container ]   PostgreSQL — port 5432
      ↓
[ named volume ]   postgres_data (persistent storage)
```

Containers communicate by **service name**, not IP address. Docker Compose creates an internal network automatically — the Flask app connects to PostgreSQL using the hostname `db`. Standard microservices pattern.

---

## Project Structure

```
docker-notes-app/
├── docker-compose.yml
└── app/
    ├── Dockerfile
    ├── app.py
    └── requirements.txt
```

---

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/notes` | Create a new note |
| `GET` | `/notes` | Get all notes |
| `GET` | `/notes/<id>` | Get a single note by ID |
| `DELETE` | `/notes/<id>` | Delete a note by ID |

---

## Key Implementation Decisions

### 1. Layer Caching Optimization
`requirements.txt` is copied and pip installed **before** copying application code. Dependencies change rarely, app code changes constantly. This maximizes Docker build cache hits — rebuilds only re-run `pip install` when dependencies actually change.

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .   # app code copied last
```

### 2. Non-Root Container User
The container runs as a dedicated `appuser` instead of root. Security hardening — principle of least privilege. If the container is compromised, the attacker has no root access.

```dockerfile
RUN useradd -m appuser
USER appuser
```

### 3. Named Volumes for Data Persistence
PostgreSQL data is stored in a Docker-managed named volume. Database contents survive container restarts and rebuilds. Without this, every `docker-compose down` wipes all data.

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

### 4. Secrets via Environment Variables
Database credentials are injected at runtime as environment variables — never hardcoded in the image. No secrets baked into Docker image layer history.

---

## Debugging: The Postgres Race Condition

**The problem:** On first run, Flask crashed immediately with `psycopg2.OperationalError — connection refused`.

**Root cause:** `depends_on` only waits for the PostgreSQL *container* to start — not for PostgreSQL to finish initializing and become ready to accept TCP connections. Flask called `init_db()` before the database was ready.

**The fix:** Retry loop with 5 attempts and 3-second delays between each.

```python
def init_db_with_retry():
    retries = 5
    while retries > 0:
        try:
            init_db()
            print("Database initialized successfully")
            return
        except Exception as e:
            retries -= 1
            print(f"DB not ready, retrying in 3s... ({retries} retries left)")
            time.sleep(3)
    raise Exception("Could not connect to database after multiple retries")
```

This is a **real production pattern** — the same race condition exists in Kubernetes and is solved identically.

---

## Project 2 vs Project 5 — Serverless vs Containerized

| Aspect | Project 2 (Serverless) | Project 5 (Containerized) |
|---|---|---|
| Compute | AWS Lambda | Flask in Docker |
| Database | DynamoDB (NoSQL) | PostgreSQL (SQL) |
| DB Driver | boto3 AWS SDK | psycopg2 |
| Auth | IAM roles | Env var credentials |
| Scaling | Auto (AWS managed) | Manual |
| Cost model | Pay per invocation | Pay per uptime |
| Local dev | Requires SAM/localstack | `docker-compose up` |
| Portability | AWS-specific | Runs anywhere |

---

## How to Run Locally

**Prerequisites:** Docker + Docker Compose installed

```bash
# 1. Clone the repo
git clone https://github.com/Rayyan-Mudassar/Cloud-Labs-portifolio
cd Cloud-Labs-portifolio/project-5-docker-notes

# 2. Start the stack
docker-compose up --build

# 3. Test the API
curl -X POST http://localhost:5000/notes \
  -H "Content-Type: application/json" \
  -d '{"content": "My first Docker note"}'

curl http://localhost:5000/notes

curl http://localhost:5000/notes/1

curl -X DELETE http://localhost:5000/notes/1

# 4. Tear down
docker-compose down
```

---

## Key Learnings

**Container networking** — Containers in the same Compose file talk to each other by service name. Docker handles DNS resolution internally. No IPs, no manual network config.

**Race conditions in containerized environments** — `depends_on` is not enough. Any app connecting to a database at startup needs a retry mechanism. This is a baseline production requirement, not a workaround.

**Layer caching matters** — Ordering Dockerfile instructions from least-to-most frequently changed cuts rebuild times significantly in CI/CD pipelines.

**Security by default** — Non-root user, slim base images, secrets via env vars. These aren't optional best practices — they're baseline requirements for production containers.

---

*Rayyan Mudassar — Cloud & Security Engineer* 
