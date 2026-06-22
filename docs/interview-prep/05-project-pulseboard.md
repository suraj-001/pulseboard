# PulseBoard — Project Specific Interview Questions

> These questions are about the project itself — design decisions, architecture choices,
> and what you would do differently.

---

## Q1: Tell me about PulseBoard. What does it do and why did you build it?

PulseBoard is a Team Weekly Status and Intelligence Platform. I built it to solve a real 
problem my team at FICO faces — we write weekly status updates in Word documents, managers 
manually aggregate them, action items get lost, and recurring blockers are never spotted until 
they become crises.

The platform has three core flows:
1. **Team members** spend 3 minutes submitting a structured weekly update — targets set, 
   completion %, blockers, next week plan
2. **Managers** get a live dashboard showing who submitted, who hasn't, team completion rate, 
   and team availability for the week
3. **Intelligence layer** auto-generates the weekly PDF report every Friday, tracks commitment 
   accuracy per person over time, and uses NLP to cluster recurring blockers

I chose this project because it solves a problem I personally experience, it's complex enough 
to demonstrate serious engineering skills, and it's relevant to FICO's core business of 
analytics and intelligent decision-making.

---

## Q2: Walk me through the architecture of PulseBoard.

PulseBoard is a microservices system with 10 Spring Boot services:

**Infrastructure services (always start first):**
- `discovery-server` (port 8761) — Eureka service registry
- `config-server` (port 8888) — centralized configuration
- `api-gateway` (port 8080) — single entry point, routing, auth filter

**Business services:**
- `auth-service` (8081) — JWT auth, user management, roles
- `team-service` (8082) — teams, members, invites, availability
- `update-service` (8083) — weekly update submission and tracking
- `analytics-service` (8084) — commitment scores, trends, team health
- `report-service` (8085) — PDF generation, Spring Batch jobs
- `notification-service` (8086) — email reminders, digest emails
- `nlp-service` (8087) — Python FastAPI, blocker clustering

All services register with Eureka on startup. The API Gateway routes external requests using 
Eureka load balancing. Services communicate synchronously via REST for data queries and 
asynchronously via Kafka for events like update submissions and report generation triggers.

---

## Q3: Why did you choose Spring Cloud over alternatives like Consul or Kubernetes-native service discovery?

Spring Cloud integrates natively with Spring Boot — Eureka client is a single dependency and 
a few YAML properties. For a project being developed and demonstrated locally without Docker, 
Spring Cloud gave me service discovery, configuration management, and API gateway without any 
external infrastructure. Consul requires a running Consul agent. Kubernetes-native service 
discovery (DNS-based) only works inside a Kubernetes cluster. Spring Cloud works on a single 
laptop with no external dependencies — perfect for development and portfolio demonstration.

---

## Q4: How do you handle configuration across environments in PulseBoard?

I use Spring Cloud Config Server with Spring Profiles:

- **Local development** — Config Server serves from classpath (`native` profile), H2 in-memory 
  database, no Redis, no Kafka
- **Production (AWS)** — Config Server backed by Git repository, PostgreSQL on RDS, Redis on 
  ElastiCache, Kafka on MSK

The service code doesn't change between environments — only the config does. Spring Boot's 
`@Value` and `@ConfigurationProperties` inject values from wherever the Config Server says 
they come from. This is the 12-factor app methodology — config separated from code.

---

## Q5: How does the weekly update flow work end to end?

1. Team member opens the frontend React app and submits their weekly update via REST API 
   to `http://localhost:8080/api/updates/submit` (API Gateway)
2. Gateway routes the request to `update-service` on port 8083
3. `update-service` validates, saves to H2/PostgreSQL, and publishes `UpdateSubmittedEvent` 
   to Kafka topic `weekly-updates`
4. `notification-service` consumes the event — if it's a manager's aggregation trigger, it 
   assembles the team dashboard data
5. On Thursday, a scheduled job in `notification-service` checks who hasn't submitted and 
   sends reminder emails
6. On Friday 5pm, `report-service` Spring Batch job aggregates all updates, generates a PDF, 
   and emails the manager

---

## Q6: What is the commitment accuracy score and how is it calculated?

Every time a team member sets targets at the start of the week and marks completion % at the 
end, the analytics service computes their commitment accuracy:
accuracy = (targets_completed / targets_set) * 100

This is stored weekly. Over 12 weeks, a rolling average gives each person a commitment 
accuracy score. The insight is not punitive — it helps managers understand who estimates well 
and who consistently over-commits, so they can have better planning conversations.

---

## Q7: What would you do differently if you were starting PulseBoard again?

A few things I'd reconsider:

1. **Start with a modular monolith** — for a new product with uncertain requirements, a 
   modular monolith is easier to iterate on. Extract to microservices once boundaries are clear.
2. **API-first design** — define OpenAPI specs for all services before writing code, not after
3. **Event sourcing for updates** — instead of storing the latest state of a weekly update, 
   store every change as an event. This gives a full audit trail and enables replaying state.
4. **Earlier user testing** — get the MVP to real team members after Phase 3 (Week 11) and 
   iterate based on feedback before building intelligence features

---

## Q8: How does PulseBoard handle multi-tenancy?

In the current design, multi-tenancy is handled at the data layer — each team is an isolated 
unit with its own members, updates, and reports. In a future SaaS version, organizations would 
be the top-level tenant, with teams nested under them.

For database isolation I'd use schema-per-tenant approach in PostgreSQL — each organization 
gets its own schema, preventing data leakage between tenants while keeping infrastructure 
manageable.