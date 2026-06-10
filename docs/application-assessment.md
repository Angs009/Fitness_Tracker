# Fitness Tracker — Application Assessment

**Repository:** [https://github.com/Angs009/Fitness_Tracker.git](https://github.com/Angs009/Fitness_Tracker.git)  
**Assessment Date:** 2026-06-10  
**Phase:** 1 — Repository Assessment  
**Assessor:** DevOps Platform Engineering

---

## Executive Summary

**FitTrack Pro** (package name: `fitness-tracker`) is a monolithic full-stack fitness tracking web application. It combines a **Node.js / Express** API with a **vanilla HTML/CSS/JavaScript** frontend served as static assets from the same process. Data is persisted in **MongoDB** via **Mongoose**.

The repository already contains foundational DevOps artifacts (Docker, Docker Compose, basic Kubernetes manifests, a partial Helm chart, Jenkins pipeline, Ansible playbook, and Datadog APM integration). However, these artifacts are incomplete, inconsistent, or not aligned with enterprise GitOps practices. This assessment establishes the baseline for the phased DevOps platform implementation.

| Dimension | Current State | Target State (Phases 2–14) |
|-----------|---------------|----------------------------|
| CI/CD | Jenkins (`Jenkinsfile`) | GitHub Actions with SonarQube Quality Gate |
| Container | Basic Dockerfile (root user) | Multi-stage, non-root, hardened image |
| Orchestration | Flat K8s YAML (2 replicas) | Helm chart + ArgoCD GitOps on Amazon EKS |
| Ingress | `LoadBalancer` Service | Kubernetes Gateway API |
| Observability | App-level `dd-trace` + compose agent | Full Datadog platform (agent, dashboards, APM) |
| Testing | Not implemented | Unit tests + coverage in CI |
| Git strategy | Undocumented | Enterprise GitFlow |

---

## 1. Application Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Client Browser                               │
│  (HTML pages, CSS, Vanilla JS, Chart.js, localStorage auth)     │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTP (port 5000)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Node.js / Express Application                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Static      │  │  REST API    │  │  Datadog APM         │  │
│  │  File Server │  │  /api/auth/* │  │  (dd-trace + logger) │  │
│  │  (public/)   │  │              │  │                      │  │
│  └──────────────┘  └──────┬───────┘  └──────────────────────┘  │
└───────────────────────────┼─────────────────────────────────────┘
                            │ Mongoose ODM
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MongoDB 7.x                                    │
│  Collections: users, workouts, metrics, plans                    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Architectural Pattern

| Aspect | Detail |
|--------|--------|
| **Pattern** | Monolith — single deployable unit |
| **Frontend** | Server-rendered static pages (not a SPA framework) |
| **Backend** | REST API under `/api/auth` |
| **Authentication** | Client-side `localStorage` session; plaintext password comparison in DB |
| **State** | Stateless application tier; state in MongoDB |
| **Port** | `5000` (configurable via `PORT`) |
| **Bind address** | `0.0.0.0` in container/K8s environments |

### 1.3 User Roles and Features

| Role | Capabilities |
|------|-------------|
| **Client** | Register, login, log workouts, track health metrics (weight, BMI, body fat), view progress charts |
| **Trainer** | Register, login, assign/edit/delete workout plans for clients, trainer panel |

### 1.4 API Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/api/auth/register` | User registration (client or trainer) |
| `POST` | `/api/auth/login` | User authentication |
| `POST` | `/api/auth/logout` | Client-side logout acknowledgment |
| `POST` | `/api/auth/log-workout` | Record workout session |
| `GET` | `/api/auth/workouts/:email` | Retrieve user workouts |
| `POST` | `/api/auth/metrics` | Store health metrics |
| `GET` | `/api/auth/metrics/:email` | Retrieve user metrics |
| `POST` | `/api/auth/plans` | Trainer creates workout plan |
| `GET` | `/api/auth/plans/:trainer` | List trainer plans |
| `PUT` | `/api/auth/plans/:id` | Update plan |
| `DELETE` | `/api/auth/plans/:id` | Delete plan |
| `POST` | `/api/auth/test` | Connectivity health check |

### 1.5 Data Model (Mongoose Schemas)

| Collection | Key Fields |
|------------|------------|
| `users` | `fullname`, `email` (unique), `password`, `role` (`client` \| `trainer`), role-specific fields |
| `workouts` | `email`, `type`, `duration`, `calories`, `date`, `notes` |
| `metrics` | `email`, `date`, `weight`, `bmi`, `fat` |
| `plans` | `trainer`, `client`, `plan` |

MongoDB initialization script (`mongodb/init/01-init-fitness-tracker.js`) defines JSON schema validators, indexes, and sample seed data.

---

## 2. Technology Stack

### 2.1 Detected Technologies

| Layer | Technology | Version (detected) |
|-------|------------|-------------------|
| **Runtime** | Node.js | `>=14.0.0` (engines); Docker uses `18-alpine` |
| **Web Framework** | Express.js | `^5.1.0` |
| **Database** | MongoDB | `7.0` / `7-jammy` |
| **ODM** | Mongoose | `^8.17.0` |
| **Config** | dotenv | `^16.3.1` |
| **APM / Logging** | dd-trace (Datadog) | `^5.67.0` |
| **Frontend** | HTML5, CSS3, Vanilla ES6+ JS | — |
| **Charts** | Chart.js (CDN) | — |
| **Icons** | Font Awesome (CDN) | — |
| **Dev Server** | nodemon | `^3.0.1` |

### 2.2 What This Application Is NOT

- Not React, Vue, Angular, or Next.js
- Not a compiled/bundled frontend (no Webpack, Vite, or esbuild)
- Not Java, Python, or .NET
- Not a microservices architecture

### 2.3 Dependency Structure

The project uses a **two-tier npm layout**:

```
fitness-tracker/          ← Root package.json (scripts, nodemon)
└── server/               ← Server package.json (runtime dependencies)
    ├── app.js
    ├── datadog.js
    └── routes/auth.js
```

**Root `package.json` dependencies:** Primarily nodemon and its transitive tree. Runtime server dependencies live under `server/`.

**Server `package.json` dependencies:**

| Package | Purpose |
|---------|---------|
| `express` | HTTP server and routing |
| `mongoose` | MongoDB connection and schemas |
| `dotenv` | Environment variable loading |
| `dd-trace` | Datadog distributed tracing and structured logging |

---

## 3. Project Structure

```
Fitness_Tracker/
├── public/                      # Frontend static assets
│   ├── assets/                  # Images (logo, etc.)
│   ├── css/                     # main.css, components.css, dashboard.css, style.css
│   ├── js/                      # app.js (main logic), main.js
│   └── pages/                   # HTML pages (index, login, log-workout, etc.)
├── server/
│   ├── app.js                   # Express entry point
│   ├── datadog.js               # Datadog tracer + structured logger
│   ├── routes/auth.js           # All API routes and Mongoose models
│   └── data/users.json          # Legacy/static data (superseded by MongoDB)
├── mongodb/init/
│   └── 01-init-fitness-tracker.js
├── Fitness_Chart/               # Partial Helm chart (legacy)
│   ├── Chart.yaml
│   └── templates/
│       ├── app-deployment.yaml
│       └── mongodb-deployment.yaml
├── app-deployment.yaml          # Flat K8s app manifest
├── mongodb-deployment.yaml      # Flat K8s MongoDB manifest
├── configmap.yml                # ConfigMap (not wired into deployment)
├── Dockerfile
├── .dockerignore
├── docker-compose.yml
├── docker-compose.dev.yml
├── Jenkinsfile                  # Jenkins CI/CD (to be replaced)
├── ansible.yaml                 # VM-based deployment playbook
├── .env.example
├── package.json
└── README.md
```

---

## 4. Build Process

### 4.1 Local Development

```bash
# Install dependencies
npm install          # Root (nodemon)
npm run setup        # cd server && npm install

# Run
npm run dev          # nodemon server/app.js (hot reload)
npm start            # node server/app.js (production)
```

### 4.2 Build Characteristics

| Step | Present | Notes |
|------|---------|-------|
| Frontend compilation | No | Static files served directly |
| TypeScript compilation | No | Plain JavaScript |
| Asset bundling/minification | No | — |
| Backend transpilation | No | Direct Node.js execution |
| Test execution | No | Placeholder exits with code 1 |
| Linting | No | Referenced in README but no ESLint config found |
| Security audit | Manual | `npm audit` possible but not automated |

### 4.3 Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `5000` | HTTP listen port |
| `HOST` | `0.0.0.0` | Bind address |
| `NODE_ENV` | `development` | Runtime environment |
| `MONGODB_URI` | `mongodb://localhost:27017/fitness-tracker` | Database connection |
| `DD_SERVICE` | `fitness-tracker` | Datadog service name |
| `DD_ENV` | `development` | Datadog environment tag |
| `DD_VERSION` | `1.0.0` | Datadog version tag |
| `DD_AGENT_HOST` | — | Datadog agent hostname |
| `DD_TRACE_AGENT_PORT` | `8126` | Datadog APM port |

---

## 5. Existing Docker Support

### 5.1 Dockerfile (Current)

**Location:** `Dockerfile`

| Attribute | Current State | Gap vs. Production Target |
|-----------|---------------|----------------------------|
| Base image | `node:18-alpine` | Acceptable |
| Multi-stage build | No | Required in Phase 5 |
| Non-root user | No (runs as root) | Required in Phase 5 |
| Health check | No | Should add `HEALTHCHECK` |
| Layer optimization | Partial | Can improve with `npm ci` |
| Security scanning | None | Phase 3 CI will add Trivy/Grype |

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
COPY server/package*.json ./server/
RUN npm install --omit=dev && cd server && npm install --omit=dev
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

> **Note:** The README documents a `USER nodejs` directive that is **not present** in the actual Dockerfile.

### 5.2 Docker Compose

**`docker-compose.yml`** orchestrates three services:

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `fitness-app` | Built from `Dockerfile` | `5000` | Application |
| `mongodb` | `mongo:7.0-jammy` | `27017` | Database (persistent volume) |
| `datadog-agent` | `gcr.io/datadoghq/agent:7` | — | Observability |

**`docker-compose.dev.yml`** overrides for development:
- `NODE_ENV=development`
- Volume mounts for hot reload (`./server`, `./public`)
- Command: `npm run dev`

### 5.3 Docker Scripts (package.json)

| Script | Command |
|--------|---------|
| `docker:build` | `docker build -t fitness-tracker .` |
| `compose:up` | `docker-compose up -d` |
| `compose:down` | `docker-compose down` |
| `compose:build` | `docker-compose build` |
| `docker:dev` | `docker-compose -f docker-compose.yml -f docker-compose.dev.yml up` |

### 5.4 Docker Gaps

- No image tagging strategy (dev/stage/prod)
- No image promotion workflow
- Hardcoded Datadog API key in `docker-compose.yml` (security risk)
- No container image vulnerability scanning
- Dockerfile runs as root

---

## 6. Existing Kubernetes Support

### 6.1 Current Manifests

| File | Resources | Notes |
|------|-----------|-------|
| `app-deployment.yaml` | Deployment (2 replicas) + LoadBalancer Service | Hardcoded image `pulkit197/fitness_tracker-master-copy3-fitness-app:latest` |
| `mongodb-deployment.yaml` | Deployment (1 replica) + ClusterIP Service | Uses `emptyDir` — **data not persistent** |
| `configmap.yml` | ConfigMap | **Not referenced** by deployment (env vars inlined) |

### 6.2 Kubernetes Feature Coverage

| Feature | Present | Notes |
|---------|---------|-------|
| Deployment | Yes | 2 replicas (target: 3 in Phase 7) |
| Service | Yes | LoadBalancer type |
| ConfigMap | Yes | Unused |
| Secret | No | — |
| Namespace | No | — |
| PVC | No | MongoDB uses emptyDir |
| HPA | No | — |
| NetworkPolicy | No | — |
| Probes (readiness/liveness) | No | — |
| Security context | No | — |
| Resource requests/limits | No | — |
| Gateway API | No | — |
| Ingress | No | Uses LoadBalancer directly |

### 6.3 Partial Helm Chart (`Fitness_Chart/`)

| Attribute | Value |
|-----------|-------|
| Chart name | `fintess-tracker` (typo) |
| Version | `1.0.0` |
| Templates | `app-deployment.yaml`, `mongodb-deployment.yaml` only |
| Values files | None |
| Templating | Minimal — mostly static YAML copies |

This chart will be superseded by the enterprise `helm/fitness-tracker/` chart in Phase 9.

### 6.4 Existing CI/CD to Kubernetes

The `Jenkinsfile` deploys to **Amazon EKS** (`pulkit-cluster`, `us-east-1`):

1. Build Docker image
2. Docker Compose integration test
3. Push to Docker Hub (`pulkit197/fitness_tracker-master-copy3-fitness-app`)
4. `kubectl apply` flat manifests
5. `kubectl set image` with build number tag
6. Wait for rollout + print LoadBalancer URL

This will be replaced by GitHub Actions + ArgoCD GitOps in Phases 3 and 10.

---

## 7. Existing Observability

### 7.1 Application Instrumentation

**File:** `server/datadog.js`

- Initializes `dd-trace` with service, env, version tags
- Structured JSON logging via custom `logger` (info, warn, error)
- Console output overridden to emit JSON logs
- Log injection enabled for trace correlation

**File:** `server/app.js`

- HTTP request duration logging middleware
- MongoDB connection logging with credential redaction
- Server startup metadata logging

### 7.2 Infrastructure Observability

| Component | Status |
|-----------|--------|
| Datadog Agent (Docker Compose) | Configured |
| Datadog APM env vars (K8s) | Configured via `status.hostIP` |
| Log collection annotations (K8s) | Present on pod template |
| Dashboards | Not present |
| Monitors/alerts | Not present |
| SLOs | Not present |

---

## 8. Security Assessment (Baseline)

| Finding | Severity | Recommendation |
|---------|----------|----------------|
| Passwords stored in plaintext | High | Hash with bcrypt (application change — out of DevOps scope but noted) |
| Hardcoded `DD_API_KEY` in compose and `.env.example` | High | Move to secrets management (K8s Secret, GitHub Secrets) |
| No TLS/HTTPS termination | Medium | Gateway API + cert-manager in Phase 8 |
| Container runs as root | Medium | Non-root user in Phase 5 Dockerfile |
| No NetworkPolicy | Medium | Phase 7 |
| No image vulnerability scanning | Medium | Phase 3 CI |
| Auth via localStorage only | Medium | Application-level concern |
| MongoDB `emptyDir` in K8s | High | PVC in Phase 7 |
| Sensitive data in logs (`req.body`, `req.headers` logged on auth routes) | Medium | Review log redaction |

---

## 9. Testing and Quality Baseline

| Area | Status |
|------|--------|
| Unit tests | **Not implemented** — `npm test` exits with error |
| Integration tests | Jenkins runs Docker Compose smoke test only |
| E2E tests | None |
| Linting (ESLint) | Mentioned in README, no config file |
| Code coverage | None |
| SonarQube | Not configured |
| Dependency scanning | Manual `npm audit` only |

> **CI Impact:** Phase 3 will need a test framework (Jest or Mocha) added to enable meaningful unit test and coverage stages. The placeholder `npm test` script must be replaced.

---

## 10. Gap Analysis — DevOps Platform Readiness

### 10.1 Artifacts to Create (Phases 2–14)

| Phase | Artifact | Status |
|-------|----------|--------|
| 1 | `docs/application-assessment.md` | This document |
| 2 | `docs/git-strategy.md` | Not started |
| 3 | `.github/workflows/ci.yml` | Not started |
| 4 | `sonar-project.properties`, `docs/sonarqube.md` | Not started |
| 5 | Hardened `Dockerfile`, `.dockerignore` | Partial (exists, needs hardening) |
| 6 | `scripts/promote_image.py`, `docs/image-promotion.md` | Not started |
| 7 | `k8s/` manifests (8 files) | Partial (flat YAML exists) |
| 8 | `gateway/` Gateway API resources | Not started |
| 9 | `helm/fitness-tracker/` chart | Partial (`Fitness_Chart/` legacy) |
| 10 | `argocd/` manifests + docs | Not started |
| 11 | `monitoring/datadog/` + `docs/datadog.md` | Partial (app instrumentation only) |
| 12 | `ai-agent/` Python troubleshooting agent | Not started |
| 13 | `architecture/architecture.md` | Not started |
| 14 | Enterprise `README.md` | Partial (app README exists) |

### 10.2 Artifacts to Retain (Do Not Remove)

- All application source code (`server/`, `public/`)
- MongoDB init script
- Existing Docker Compose files (enhance, do not delete)
- Datadog application instrumentation (`server/datadog.js`)
- `.env.example` (sanitize secrets in later phases)

### 10.3 Artifacts to Supersede (Keep Until Migration Complete)

| Legacy Artifact | Replaced By |
|-----------------|-------------|
| `Jenkinsfile` | `.github/workflows/ci.yml` |
| `Fitness_Chart/` | `helm/fitness-tracker/` |
| Flat `app-deployment.yaml`, `mongodb-deployment.yaml` | `k8s/` + Helm templates |
| Root-level `configmap.yml` | Helm-managed ConfigMap |

---

## 11. Recommended Image and Registry Strategy

Based on existing conventions in the repository:

| Attribute | Recommended Value |
|-----------|-------------------|
| Registry | Docker Hub |
| Image name | `<DOCKER_USERNAME>/fitness-tracker` |
| Tag strategy | `{git-sha}`, `dev`, `stage`, `prod` |
| Base image | `node:18-alpine` (multi-stage) |
| EKS cluster | Amazon EKS (existing: `pulkit-cluster`, `us-east-1`) |

---

## 12. Prerequisites for Downstream Phases

Before proceeding to Phase 2, confirm the following:

- [ ] GitHub repository access for `Angs009/Fitness_Tracker`
- [ ] Docker Hub account and credentials
- [ ] SonarQube server (cloud or self-hosted) with project provisioned
- [ ] Amazon EKS cluster provisioned
- [ ] ArgoCD installed on EKS (or will be installed in Phase 10)
- [ ] Datadog account and API key (replace hardcoded key)
- [ ] Gateway API controller installed on EKS (for Phase 8)

---

## 13. Conclusion

The Fitness Tracker application is a **Node.js monolith** with a **vanilla JavaScript frontend** and **MongoDB** backend. It is containerization-ready with existing Docker and basic Kubernetes support, plus partial Datadog APM integration. The codebase lacks automated testing, linting configuration, production-hardened containers, GitOps workflows, and enterprise Kubernetes manifests.

The phased DevOps implementation will build on existing artifacts without removing application functionality, migrating from Jenkins to GitHub Actions, flat YAML to Helm + ArgoCD, and LoadBalancer services to Gateway API — while adding SonarQube quality gates, image promotion automation, full observability, and an AI operations agent for Kubernetes troubleshooting.

---

*Document generated as part of Phase 1 — Repository Assessment. Proceed to Phase 2 (Git Strategy) upon approval.*
