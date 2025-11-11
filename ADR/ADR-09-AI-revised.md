نسخه‌ی بازنویسی‌شده و بهینه‌شده‌ی ADR-008 را می‌بینی — با حفظ محتوای فنی و لحن حرفه‌ای، ولی با ساختار خواناتر، خلاصه‌تر و با بخش‌های استاندارد‌تر (مطابق الگوی Michael Nygard ADRs).
در این نسخه هدف این بوده که تصمیم‌ها در مرکز سند قرار بگیرند و جزئیات فنی در حد لازم باقی بمانند


ADR-008: Inter-City Ride-Sharing Platform

Status

Proposed

> Next: Review by architecture board → Approval pending




---

Context

We are defining the architecture for an inter-city ride-sharing platform in Iran, inspired by Snap/Uber, initially focusing on Tehran-origin trips to Mazandaran, Gilan, and Qazvin.

The system must address long-standing intercity transport problems:

Low passenger satisfaction

Poor trip safety

Lack of traceability and regulatory visibility


To achieve this, the platform aims to:

Organize and monitor intercity trips via a unified platform.

Improve safety, oversight, and regulatory compliance.

Deliver responsive, scalable, and observable operations.



---

Business Goals

Improve passenger satisfaction through transparency and reliability.

Enable safety oversight and real-time monitoring.

Support regulatory compliance with auditable trip and payment data.


Technical Goals

Responsiveness: Sub-200ms system reactions to key user actions.

Scalability: Handle 20x growth in traffic with zero degradation.

Reliability: 99.9% uptime with fault-tolerant design.

Auditability: Immutable, traceable records for every user action.

Includes observability (metrics, logs, traces).

Includes traceability (end-to-end request correlation).




---

Scale & Traffic Estimates

Metric	Initial	Scaled (20×)	Notes

Trips/hour	350	7,000	~3-hour avg trip
Requests/sec	1–2	20–40	Includes updates and events
Drivers (online)	500–1,000	Fixed pool	Northern routes
Passengers (concurrent)	200–1,000	4,000–20,000	2–3× peak on holidays



---

Decision

We will:

1. Adopt a microservices architecture with observability and traceability embedded as part of an overarching auditability goal.


2. Deploy on self-managed Rooz-e-Aval infrastructure, using Docker and Kubernetes for orchestration.


3. Select communication protocols by use case:

HTTP/2 for standard API calls.

WebSockets for real-time streams.

gRPC over HTTP/2 for internal microservice calls.

HTTP/3 (QUIC) as an optional upgrade for mobile optimization.



4. Implement responsive scaling and monitoring via Prometheus, OpenTelemetry, Grafana, and Horizontal Pod Autoscaler (HPA).


5. Ensure auditability through immutable logs and end-to-end tracing.




---

Architecture Overview

Network Layer

Purpose: Enable responsive, secure, and scalable communications.

Protocol	Use Case	Why It Fits	Notes

HTTP/2	RESTful APIs (booking, auth, payments)	Multiplexing, low latency	Default choice
HTTP/3 (QUIC)	Mobile web APIs	0-RTT resume, better on lossy networks	Future upgrade path
WebSockets	Real-time trip updates, SOS alerts	Persistent duplex channel	Sticky sessions only
gRPC	Service-to-service (pricing, audit, matching)	Binary efficiency, strong typing	Internal only


Security & Auditability

TLS 1.3 everywhere; mTLS for internal gRPC.

WAF (e.g., Coraza or ModSecurity) for OWASP Top 10 protection.

Centralized immutable logs via ELK or Loki.


Scalability & Responsiveness

Protocol-aware routing (e.g., /api/*, /ws/*).

Auto-scaling load balancers (NGINX/APISIX).

Regional edge nodes for Tehran/North latency optimization.



---

Infrastructure Layer

Purpose: Provide scalable, observable foundations for all services.

Compute: Rooz-e-Aval VMs (2–4 initial → 40–80 scaled).

Database: PostgreSQL or MSSQL (primary) + Redis + Cassandra (cache/distributed data).

Messaging: Kafka or RabbitMQ for asynchronous events.

Orchestration: Docker + Kubernetes with HPA and metrics-based triggers.

Observability Stack: Prometheus + Grafana + OpenTelemetry.

Backup & Audit: Automated snapshots, versioned logs, immutable storage.



---

Application Layer

Purpose: Implement domain logic, user interaction, and security.

Backend: Node.js or Go-based microservices (Trip, User, Audit, Payment).

Frontend:

Mobile apps: Flutter (cross-platform) or Kotlin/Swift (native).

Admin dashboards for operations and compliance.


Security: SOS triggers, trip anomaly detection, live GPS audit logs.

Auditability: Request IDs propagated end-to-end; versioned immutable edits.

Performance: Async processing, caching, sharding, retry mechanisms.



---

System Flow (Simplified)

flowchart LR
    Passenger -->|Trip Request| Backend
    Driver -->|Location Update| Backend
    Backend -->|Trace & Observe| DB[(Database + Cache)]
    Backend --> MQ[(Message Queue)]
    MQ --> Tracing[(Telemetry + Audit Logs)]
    Admin -->|Inspect| Backend
    Monitoring -->|Metrics/Alerts| Admin


---

Consequences

✅ Enhanced observability improves debugging and compliance.

⚙️ Increased DevOps overhead (self-managed infra).

🔒 Stronger data safety and traceability for regulatory needs.

📈 Horizontal scalability ensures growth without redesign.

⏱ Minor latency overhead (~5–10%) due to tracing instrumentation.



---

Next Steps

[ ] Review by architecture board.

[ ] Finalize tool stack selection (Prometheus, Loki, etc.).

[ ] Define SLA metrics and monitoring thresholds.

[ ] Link to “Detailed Network Design” and “Deployment Topology” docs.



---
