# Microservices & Spring Cloud — Interview Questions

> Phase added: Phase 0
> Real project reference: PulseBoard — 10 microservices with Eureka, Gateway, Config Server

---

## Q1: What are microservices and what problem do they solve?

Microservices is an architectural style where an application is built as a collection of small, 
independently deployable services, each responsible for a specific business capability.

**Problems they solve:**
- **Scaling** — scale only the services that need it, not the whole application
- **Independent deployment** — deploy auth-service without touching report-service
- **Technology flexibility** — PulseBoard's NLP service is Python, everything else is Java
- **Fault isolation** — if report-service crashes, users can still submit updates

**Trade-offs:**
- Increased operational complexity
- Distributed system problems — network failures, eventual consistency
- Need for service discovery, API gateway, distributed tracing
- More difficult to debug across service boundaries

In PulseBoard I have 10 services. The decision was conscious — I wanted to learn and 
demonstrate these patterns.

---

## Q2: What is Service Discovery and why is it needed?

In microservices, services need to call each other. Hardcoding IP addresses breaks when 
services scale, move, or restart. Service discovery solves this.

**How Eureka works in PulseBoard:**
1. Every service registers itself with Eureka on startup — name, IP, port, health URL
2. Eureka maintains a registry of all running instances
3. When API Gateway needs to route to auth-service, it asks Eureka for the current address
4. `lb://auth-service` in the gateway config means "load balance across all Eureka instances 
   of auth-service"

I verified this live — starting auth-service on port 8081 made it appear in Eureka dashboard 
at `http://localhost:8761` within 30 seconds showing status UP.

---

## Q3: What is an API Gateway pattern?

API Gateway is a single entry point for all client requests to a microservices system.

**What it does in PulseBoard:**
- **Routing** — `/api/auth/**` → auth-service, `/api/teams/**` → team-service
- **Load balancing** — distributes requests across multiple instances via Eureka
- **Cross-cutting concerns** — JWT validation (Phase 1), rate limiting, CORS — implemented 
  once in the gateway instead of in every service
- **Abstraction** — clients only know port 8080, not individual service ports

I use Spring Cloud Gateway which is reactive (non-blocking, built on Netty) as opposed to 
the older Zuul which was servlet-based.

---

## Q4: What is Spring Cloud Config Server?

Centralized configuration management for all microservices. Instead of maintaining separate 
`application.yml` files in 10 services, all services fetch their configuration from one place.

**In PulseBoard:**
- Config Server runs on port 8888
- All services call it on startup to get shared configuration
- The shared config includes H2 datasource settings, JPA config, actuator endpoints
- Verified working: `http://localhost:8888/actuator/health` returns full config being served

**In production:** Config Server would be backed by a Git repository — config changes are 
version controlled and services can refresh config without restarting.

---

## Q5: What is the difference between synchronous and asynchronous communication?

**Synchronous (REST/HTTP):**
- Caller waits for response
- Simple, easy to understand
- Tight coupling — if called service is slow/down, caller is affected
- Used in PulseBoard for: client → API Gateway → services

**Asynchronous (Kafka/messaging):**
- Caller publishes an event and continues — doesn't wait
- Loose coupling — services are independent
- Higher resilience — events persist in Kafka even if consumer is down
- Used in PulseBoard for: update-service publishes `UpdateSubmittedEvent` → 
  notification-service consumes it to send reminder emails

**When to use which:**
- Use sync when you need an immediate response (login, data fetch)
- Use async when the operation can happen independently (send email, generate report, analytics)

---

## Q6: What is Kafka and how does it work?

Apache Kafka is a distributed event streaming platform. Think of it as a high-throughput, 
durable message queue.

**Key concepts:**
- **Producer** — publishes events to a topic
- **Consumer** — reads events from a topic
- **Topic** — named channel for events (like a database table but for events)
- **Partition** — topics are split into partitions for parallelism
- **Consumer Group** — multiple consumers sharing the work of processing a topic

**In PulseBoard:**
- update-service is a producer — publishes `UpdateSubmittedEvent` when a team member submits
- notification-service is a consumer — listens to the topic and sends emails
- If notification-service is down when the event is published, Kafka retains the event and 
  notification-service processes it when it comes back up

---

## Q7: What is load balancing and how does Spring Cloud implement it?

Load balancing distributes incoming requests across multiple instances of a service to prevent 
any single instance from being overwhelmed.

Spring Cloud LoadBalancer (client-side load balancing) works with Eureka:
1. Service registers multiple instances with Eureka
2. When API Gateway routes to `lb://auth-service`, Spring Cloud LoadBalancer fetches all 
   instances of auth-service from Eureka
3. It selects one using a load balancing algorithm (round-robin by default)
4. Request is forwarded to the selected instance

This is client-side load balancing — the client (gateway) decides which instance to call, 
as opposed to server-side (like AWS ALB) where a separate load balancer sits in front.

---

## Q8: What is a Circuit Breaker pattern?

Circuit Breaker prevents cascading failures in a microservices system. If service A calls 
service B and service B is failing, without a circuit breaker service A will keep waiting for 
responses, exhausting its threads and eventually failing too.

**States:**
- **Closed** — requests flow normally
- **Open** — requests fail fast (no waiting for timeout) when failure threshold is exceeded
- **Half-open** — allows a few test requests to check if the downstream service recovered

In PulseBoard I implement this using Resilience4j in Phase 5. For example, if analytics-service 
is down, the API Gateway fails fast with a meaningful error instead of hanging for 30 seconds.

---

## Q9: How do microservices handle distributed transactions?

This is one of the hardest problems in microservices. Unlike a monolith where you can use a 
single database transaction, microservices have separate databases.

**Common patterns:**
- **Saga pattern** — break transaction into local transactions with compensating transactions 
  if something fails. Example in PulseBoard: creating a team (team-service) and sending a 
  welcome email (notification-service) are separate operations. If email fails, we log it 
  but don't roll back team creation.
- **Eventual consistency** — accept that data across services may be temporarily inconsistent 
  but will eventually converge
- **Outbox pattern** — write to local DB and an outbox table atomically, then publish events 
  from the outbox

In PulseBoard I use eventual consistency — the update submission and the notification are 
independent. Kafka ensures the notification eventually happens even if delayed.

---

## Q10: What is the startup order for PulseBoard services and why?
discovery-server (Eureka) — must start first, everything registers with it
config-server — services fetch config on startup, needs to be available
api-gateway — needs Eureka to be up to register and discover routes
auth-service, team-service, update-service — can start in any order

In production this is handled by Kubernetes health checks and readiness probes — a service 
won't receive traffic until it's healthy, and dependent services wait for their dependencies 
to be ready before starting.