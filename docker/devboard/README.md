# DevBoard — 3-Tier Multi-Container Architecture Deployment

A production-style containerized deployment of the DevBoard task management application (React frontend, Go REST API, and PostgreSQL database) orchestrated using Docker and Docker Compose on an AWS EC2 (Ubuntu 24.04) host.

---

## 📐 System Architecture

The application runs as three decoupled services communicating across an isolated user-defined Docker bridge network:
[ Client Browser ]
│
(Port 8082)
▼
┌────────────────────────────────────────────────────────┐
│  frontend (Vite / React)                               │
│  - Container Port: 4173 -> Host Port: 8082             │
│  - Internal Reverse Proxy: /api/* ──┐                  │
└─────────────────────────────────────┼──────────────────┘
│ (Docker Bridge Network)
▼
┌────────────────────────────────────────────────────────┐
│  backend (Go / Gin REST API)                           │
│  - Container Port: 8080 -> Host Port: 8080             │
│  - Endpoints: /projects, /tasks, /search               │
└─────────────────────────────────────┬──────────────────┘
│
▼
┌────────────────────────────────────────────────────────┐
│  postgres-db (PostgreSQL 16 Alpine)                    │
│  - Port 5432                                           │
│  - Health-checked via pg_isready                       │
│  - Persistent data volume: pgdata                      │
└────────────────────────────────────────────────────────┘
## Tech Stack & Tooling

* **Cloud Infrastructure:** AWS EC2 (Ubuntu 24.04 LTS)
* **Containerization:** Docker Engine, Multi-Stage Builds, Docker Compose
* **Frontend:** React, Vite (configured as internal reverse proxy), Node.js runtime
* **Backend:** Go (Gin REST API framework)
* **Database:** PostgreSQL 16 Alpine with custom SQL schema/seed initialization
* **Registry:** Docker Hub (`krsubham84/devboard-backend:v1`, `krsubham84/devboard-frontend:v1`)
-----------
## Phase 1: Manual Dockerfile Creation & Standalone Testing

### 1. Backend Multi-Stage Dockerfile (backend/Dockerfile)
### 2. Frontend Multi-Stage Dockerfile (frontend/Dockerfile)
### 3. Manual Build and Run Commands Used
Create shared bridge network
docker network create devboard_default

Start PostgreSQL container with seed data mounted

docker run -d \
  --name postgres-db \
  --network devboard_default \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=devboard \
  -v pgdata:/var/lib/postgresql/data \
  -v $(pwd)/init/postgres:/docker-entrypoint-initdb.d:ro \
  -p 5432:5432 \
  postgres:16-alpine

Build and run backend container

docker build -t devboard-backend:latest ./backend
docker run -d \
  --name backend \
  --network devboard_default \
  -e PORT=8080 \
  -e POSTGRES_URL="postgres://postgres:postgres@postgres-db:5432/devboard?sslmode=disable" \
  -p 8080:8080 \
  devboard-backend:latest

Build and run frontend container

docker build --no-cache -t devboard-frontend:latest ./frontend
docker run -d \
  --name frontend \
  --network devboard_default \
  -p 8082:4173 \
  devboard-frontend:latest

## Phase 2: Diagnostics & Troubleshooting The Client Proxy
The Problem:
The containers were running, but when browsing to http://<EC2-PUBLIC-IP>:8082, the dashboard displayed no projects.  

Debugging Commands Used:# Check running container statuses and port mappings
docker ps

1.Verify database initialized and seeded properly
docker logs postgres-db

2. Check Go Gin API requests and route registration
docker logs backend

3. Test internal container-to-container DNS resolution and API response
docker exec -it frontend wget -qO- http://backend:8080/projects

4. Test the external host proxy endpoint directly
curl -i http://localhost:8082/api/projects

### The Root Cause & Solution
Root Cause 1: Client-side JavaScript running in the browser cannot resolve Docker container hostnames (http://backend:8080). All requests needed to route through the frontend's /api proxy.

Root Cause 2: In the initial multi-stage Dockerfile, vite.config.js and node_modules were omitted in the runner stage. As a result, vite preview was running without its proxy configuration, returning 404 for /api/projects.

Fix: Added COPY vite.config.js . and COPY --from=builder /app/node_modules ./node_modules into Stage 2 so Vite could resolve its dependencies and route /api/* requests internally to http://backend:8080/*.

## Phase 3: Docker Hub Tagging & Publishing
Tagged the existing local images without rebuilding and pushed them to Docker Hub:
1. Authenticate with Docker Hub
docker login

2.Tag images with Docker Hub repository format
docker tag devboard-backend:latest krsubham84/devboard-backend:v1
docker tag devboard-frontend:latest krsubham84/devboard-frontend:v1

3. Push images to Docker Hub
docker push krsubham84/devboard-backend:v1
docker push krsubham84/devboard-frontend:v1

4. Cleanup dangling and builder cache layers
docker image prune -f
docker builder prune -f

## Phase 4: Full-Stack Orchestration via Docker Compose
1. wrote complete docker-compose.yml
2. Environment Configuration (.env)
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=devboard
BACKEND_PORT=8080

## How to Run This Project
Anyone can pull and run this entire architecture with zero local build steps:

1. Clone this repository
git clone [https://github.com/](https://github.com/)<your-username>/devops-learning-lab.git
cd devops-learning-lab/docker/devboard

2. Start all services in detached mode
docker compose up -d

3. Check container status
docker compose ps

4. View logs across all services
docker compose logs -f
Access the UI at:
Local: http://localhost:8082
AWS EC2: http://<EC2-PUBLIC-IP>:8082

To shut down the stack while preserving database records:
docker compose down
