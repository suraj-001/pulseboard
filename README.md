# PulseBoard 🎯

> **A Team Weekly Status & Intelligence Platform**  
> Replaces manual Word-doc/email status reporting with a live, intelligent, auto-aggregated team dashboard.

---

## 💡 What is PulseBoard?

Most engineering teams waste hours every week on status reporting:
- Team members write updates in Word documents or emails
- Managers manually aggregate them into a summary
- Action items get lost, blockers go unnoticed, patterns never surface

**PulseBoard fixes this.**

Every team member spends **3 minutes** submitting their weekly update. The platform does the rest:
- Manager sees a **live dashboard** — who submitted, who hasn't, team completion rate
- **Auto-generated PDF report** every Friday — no manual work
- **Intelligence layer** spots recurring blockers before they become crises
- **Commitment accuracy scores** help teams estimate and plan better over time

---

## 🏗️ Architecture

PulseBoard is built as a **microservices system** — 10 independent Spring Boot services that communicate through an API Gateway and register with a Service Discovery server.
[ React Frontend ]
|
[ API Gateway — port 8080 ]
|
┌─────────────────────┼─────────────────────┐
| | |
[ auth-service ] [ team-service ] [ update-service ]
| | |
└─────────────────────┼─────────────────────┘
|
┌───────────────┼───────────────┐
| | |
[ analytics-service ] [ report-service ] [ notification-service ]
| | |
└───────────────┼───────────────┘
|
[ nlp-service — Python ]
|
┌──────────────┼──────────────┐
[Kafka] [Redis] [PostgreSQL]


### Services

| Service | Port | Responsibility |
|---|---|---|
| discovery-server | 8761 | Eureka — service registry, all services register here |
| config-server | 8888 | Centralized configuration for all services |
| api-gateway | 8080 | Single entry point, routing, auth filter, rate limiting |
| auth-service | 8081 | Registration, login, JWT tokens, roles |
| team-service | 8082 | Teams, members, invites, availability |
| update-service | 8083 | Weekly updates, targets, completion tracking |
| analytics-service | 8084 | Commitment scores, trends, team health |
| report-service | 8085 | PDF report generation, scheduled jobs |
| notification-service | 8086 | Email reminders, digest emails |
| nlp-service | 8087 | Python FastAPI — blocker clustering, NLP |

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 17 LTS |
| Framework | Spring Boot 3.2.5 |
| Build Tool | Gradle 8.8 (multi-module) |
| Service Discovery | Spring Cloud Netflix Eureka |
| API Gateway | Spring Cloud Gateway |
| Config Management | Spring Cloud Config Server |
| Database (local) | H2 in-memory |
| Database (production) | PostgreSQL on AWS RDS |
| Cache | Redis (AWS ElastiCache in prod) |
| Messaging | Apache Kafka (AWS MSK in prod) |
| Auth | Spring Security + JWT |
| Reports | Spring Batch + iText PDF |
| NLP | Python FastAPI + HuggingFace |
| Frontend | React + TypeScript + TailwindCSS |
| Container | Docker + Kubernetes (AWS EKS) |
| CI/CD | GitHub Actions |
| Monitoring | Prometheus + Grafana |
| Tracing | OpenTelemetry + Jaeger |

---

## 📁 Project Structure
pulseboard/
├── services/
│ ├── auth-service/ # JWT auth, user management
│ ├── team-service/ # Teams and members
│ ├── update-service/ # Weekly status updates
│ ├── analytics-service/ # Scores and trends
│ ├── report-service/ # PDF report generation
│ ├── notification-service/ # Emails and reminders
│ ├── api-gateway/ # Request routing
│ ├── config-server/ # Centralized config
│ └── discovery-server/ # Eureka service registry
├── nlp-service/ # Python NLP microservice
├── frontend/ # React TypeScript app
├── infrastructure/
│ ├── docker-compose.yml # Full stack (for deployment)
│ ├── k8s/ # Kubernetes manifests
│ └── helm/ # Helm charts
└── .github/
└── workflows/ # GitHub Actions CI/CD



---

## 🚀 Running Locally (Development)

### Prerequisites

| Tool | Version | Download |
|---|---|---|
| Java | 17 LTS | [adoptium.net](https://adoptium.net) |
| Git | Any | [git-scm.com](https://git-scm.com) |
| IntelliJ IDEA | Community | [jetbrains.com/idea](https://jetbrains.com/idea) |

> **Note:** No Docker needed for local development. All services use H2 in-memory database locally.

### Step 1 — Clone the repo

```bash
git clone https://github.com/suraj-001/pulseboard.git
cd pulseboard

Step 2 — Start the Discovery Server (always start this first)
./gradlew :services:discovery-server:bootRun

Open http://localhost:8761 — you should see the Eureka dashboard.
Step 3 — Start the Config Server
./gradlew :services:config-server:bootRun
Step 4 — Start the API Gateway
./gradlew :services:api-gateway:bootRun
Step 5 — Start individual services
# Auth Service
./gradlew :services:auth-service:bootRun

# Team Service
./gradlew :services:team-service:bootRun

# Update Service
./gradlew :services:update-service:bootRun
Step 6 — Access the platform
URL
WHAT YOU SEE
http://localhost:8761
Eureka dashboard — all registered services
http://localhost:8888
Config server
http://localhost:8080
API Gateway — main entry point
http://localhost:8080/swagger-ui.html
API documentation


📊 Development Progress
| Phase | Status | What gets built |
|---|---|---|
| Phase 0 — Foundation | 🔄 In Progress | Project structure, infra services, Gradle setup |
| Phase 1 — Auth & Teams | ⏳ Pending | Login, registration, team creation |
| Phase 2 — Core Updates | ⏳ Pending | Weekly update form, manager dashboard |
| Phase 3 — Reports | ⏳ Pending | Auto PDF report, commitment scores |
| Phase 4 — Intelligence | ⏳ Pending | NLP blockers, team health score |
| Phase 5 — Observability | ⏳ Pending | Prometheus, Grafana, tracing |
| Phase 6 — Deployment | ⏳ Pending | AWS EKS, CI/CD, live URL |

Currently Running ✅
discovery-server — Eureka dashboard at http://localhost:8761
Coming Next 🔄
config-server — centralized configuration
api-gateway — request routing
🗺️ Roadmap
MVP (Phase 1-3) — Team submits weekly update → Manager sees live dashboard → Auto PDF report every Friday

V2 (Phase 4) — AI clusters recurring blockers → Team health score → Manager insight digest email

V3 (Phase 6) — Multi-tenant SaaS → Jira/Slack integration → Mobile responsive

🤝 Contributing
This is currently a personal portfolio project. Feel free to fork and build on it.

📄 License
MIT License — free to use and modify.

Built with ❤️ by Suraj Agarwal

