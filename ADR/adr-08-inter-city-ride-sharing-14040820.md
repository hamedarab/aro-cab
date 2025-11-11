# ADR-008: Inter-City Ride-Sharing Platform

## Status
**Proposed**

## Context
This ADR refines the architecture for an inter-city ride-sharing platform in Iran, inspired by Snap/Uber, with a focus on Tehran-origin trips to Mazandaran, Gilan, and Qazvin. 

We align observability and traceability under the broader auditability goal to ensure comprehensive monitoring, logging, and compliance. Additionally. 

We also introduces responsiveness (quick system reactions to user inputs/events) and scalability (ability to handle growth without degradation) as explicit goals. These enhancements address traditional inter-city transport issues: **low passenger satisfaction, and poor safety.** 

The platform aims to organize trips, improving oversight, satisfaction, and safety while enabling regulatory alignment.

_**A core justification for this project is to organize and regulate inter-city trips via the platform, reducing reliance on traditional systems while improving oversight, satisfaction, and safety.**_

### Key Business and Traffic Estimates
- **Initial Scale**: 350 trips/hour (~0.1 trips/second; 1-2 requests/second).
- **Scaled Scenario (20x Growth)**: 7,000 trips/hour (~2 trips/second; 20-40 requests/second).
- **Geographic Constraints**: Fixed driver pool; efficiency optimizations handle growth.

### Expanded Goals
- **Performance**: Latency <200ms for responses.
- **Reliability**: 99.9% uptime with redundancy.
- **Security**: In-trip safety (e.g., real-time monitoring, SOS features).
- **Auditability**: Immutable records with controlled edits; includes:
  - **Observability**: Metrics, logs, and traces for system health and performance insights.
  - **Traceability**: End-to-end request tracking to diagnose issues and ensure compliance.
- **Responsiveness**: Sub-second reactions to events (e.g., location updates, alerts).
- **Scalability**: Elastic handling of load spikes (e.g., auto-scaling for peaks).
- **Operational Focus**: Dashboards for monitoring/inspection, tracking for visibility.

---
### Business Goals
- Organize inter-city transport and ensure passenger safety.
- Increase satisfaction through real-time transparency.
- Enable regulatory oversight and compliance.

### Technical Goals
- Sub-200ms response times.
- 99.9% uptime.
- Full observability and traceability under auditability.
- Responsive scaling for peak travel seasons.
---

### Concurrent User Projections
- **Drivers (Fixed Pool)**: 500-1,000 online for both scales (optimized for northern routes; ~3-hour trips).
- **Passengers**:
  - Initial: 200-1,000 concurrent.
  - Scaled: 4,000-20,000 concurrent.
- **Peak Loads**: 2-3x higher (e.g., holidays).

### Infrastructure Constraints
- **Localization**: Rooz-e-Aval infrastructure or on-premise; self-managed tools (e.g., Kubernetes, PostgreSQL/SQL Server, kafka, etc.).
- **System Components**: Mobile apps, backend, admin/operational dashboards.
- **Challenges**: Real-time tracking/security; audit trails; responsive scaling.

**High-Level System Flow with Enhanced Goals** (Incorporating observability and scalability):
```mermaid
flowchart LR
    Passenger[Passenger App] -->|Request/Track| Backend[Backend Services]
    Driver[Driver App] -->|Update Location| Backend
    Backend -->|Trace & Observe| DB[Database/Cache]
    Backend -->|Async Scaling| MQ[Message Queue]
    Admin[Admin Dashboard] -->|Inspect/Audit| Backend
    Monitoring[Observability Tools] -->|Metrics/Alerts| Admin
    Security[Security Features] -->|Responsive Triggers| Backend
    Scaler[Auto-Scaler] -->|Handle Load| Backend
```

## Decision

We decided to:
1. Adopt a microservices architecture with embedded observability and traceability as first-class auditability features.
2. Host all core services on self-managed Rooz-e-Aval infrastructure.
3. Implement responsiveness and scalability through:
   - HTTP/2 + WebSockets + gRPC protocols.
   - Kubernetes HPA for horizontal scaling.
   - Prometheus + OpenTelemetry for metrics and traces.

_Illustrative; final stack in separate document.[TODO link to ADR]_

### Network Layer  
**Supports responsive comms, scalable distribution, and protocol-specific handling for real-time vs. web service calls.**

- **Load Balancer**: e.g., NGINX/HAProxy/APISIX with health checks and protocol-aware routing.  
  - **Initial**: 1,000–5,000 concurrent connections.  
  - **Scaled**: 20,000–100,000 via auto-provisioning and horizontal scaling.  
  - **Protocol Routing**:  
    - **HTTP/2 or HTTP/3 (QUIC)** → REST/gRPC APIs (web service calls).  
    - **WebSockets (wss://)** → Real-time bidirectional streams.  
    - **gRPC over HTTP/2** → Internal microservice calls (high-performance, typed contracts).

- **Protocols – Detailed Use Cases**:

  | Protocol       | Primary Use Case                     | Why It Fits                                                                 | Performance Notes                                                                 |
  |----------------|--------------------------------------|-----------------------------------------------------------------------------|-----------------------------------------------------------------------------------|
  | **HTTP/2**     | RESTful web service calls (e.g., trip booking, user auth, payment init) | Multiplexing, header compression, server push; reduces latency for API calls | Ideal for bursty, request-response traffic; supports 100s of concurrent streams per connection |
  | **HTTP/3 (QUIC)** | Future-proof web service calls (optional upgrade path) | 0-RTT connection resumption, better mobile performance on lossy networks | Recommended for mobile clients; self-managed via Caddy or custom QUIC-aware LB |
  | **WebSockets** | Real-time updates (location tracking, trip status, chat, SOS alerts) | Full-duplex, persistent, low overhead after handshake | Essential for driver/passenger live sync; sticky sessions required |
  | **gRPC**       | Inter-service communication (matching, pricing, audit events) | Efficient binary serialization (Protobuf), streaming support | Use internally; terminate at API gateway for external clients |

- **Security & Auditability**:  
  - WAF: e.g., ModSecurity or Coraza (open-source) with rules for OWASP Top 10.  
  - TLS 1.3 everywhere; mTLS for internal gRPC.  
  - Immutable network logs: Forward access/error logs to centralized observability (e.g., Loki/ELK) with tamper-proof retention.

- **Scalability & Responsiveness**:  
  - **Dynamic Routing**: Path-based routing (e.g., `/api/*` → HTTP/2, `/ws/*` → WebSocket).  
  - **Low-Latency Edges**: Deploy edge terminators in Rooz-e-Aval regions closest to Tehran/northern provinces.  
  - **Connection Management**:  
    - Reuse TCP connections via keep-alive.  
    - WebSocket ping/pong for heartbeat and fast failure detection.  
  - **Rate Limiting & Backpressure**: Per-IP and per-user throttling at LB to protect real-time streams.

**Network Goals Alignment Table**:

| Goal                | Protocol/Mechanism                          | Initial Impact               | Scaled Impact                     |
|---------------------|---------------------------------------------|------------------------------|-----------------------------------|
| **Responsiveness**  | WebSockets + HTTP/2 multiplexing            | <100ms update delivery       | Cached + edge-terminated streams  |
| **Scalability**     | Auto-scaling LB + protocol-aware clustering | Handles 5k conn/node         | 100k+ via sharded LB instances     |
| **Auditability**    | Immutable logs + request tracing headers    | Basic access logs            | Full correlation IDs across spans |
| **Security**        | TLS 1.3, WAF, mTLS for gRPC                 | Encrypted at rest/in-flight  | Zero-trust enforced at edge       |

**Recommendation Summary**:  
- Use **HTTP/2** as default for all API calls.  
- Use **WebSockets** exclusively for real-time features.  
- Introduce **gRPC** internally for microservices (via service mesh if needed later).  
- Plan for **HTTP/3** as a non-blocking upgrade for mobile resilience.  
- Enforce **sticky sessions** only for WebSockets; stateless for HTTP/gRPC.

### Infrastructure Layer
Enables scalable resources with observable/traceable operations.

- **Provider**: Rooz-e-Aval for VMs/scaling.
  - Initial: 2-4 instances.
  - Scaled: Auto-scale to 40-80 (triggers: CPU >70%, requests >400/s).
- **Database**: RDBMS (e.g., PostgreSQL, MSSQL) for structured data with replicas, and NoSQL (e.g., Cassandra for distributed storage, Redis for caching/pub-sub).
  - Auditability: Immutable logs with versioned edits; observability metrics on queries.
- **Message Queue**: e.g., RabbitMQ/Kafka for async tasks.
  - Traceability: Distributed tracing (e.g., Jaeger integration).
- **Deployment**: Docker + Kubernetes for responsive orchestration.
- **New Additions for Goals**:
  - Observability: Tools like Prometheus (metrics), Grafana (dashboards), ELK (logs).
  - Traceability: OpenTelemetry for end-to-end spans.
  - Scalability: Horizontal Pod Autoscaler; resource quotas.
  - Responsiveness: Circuit breakers (e.g., Hystrix-style) for fault tolerance.
  - Backup: Automated snapshots for audit integrity.

**Scaling and Observability Flow**:
```mermaid
flowchart TD
    Load[Monitor Load/Metrics] -->|Spike Detected| ScaleUp[Auto-Scale Resources]
    ScaleUp -->|Trace Events| Tracing[Distributed Tracing]
    Tracing -->|Observe| Dashboard[Grafana Dashboards]
    Dashboard -->|Alert| Admin[Responsive Alerts]
```

### Application Layer
Implements business logic with built-in goals.

- **Backend**: e.g., Node.js/Go for concurrency.
  - Microservices: Add "Audit Service" for observability/traceability.
- **Mobile Apps**:
  - Cross-Platform: Flutter (iOS/Android).
  - Native: Kotlin (Android), Swift (iOS).
  - Features: Responsive UI (e.g., real-time maps), SOS for security.
- **Operational Dashboards**:
  - Real-time views (active trips, metrics); inspection for compliance.
- **Trip Security/Tracking**:
  - Anomaly detection; GPS updates (traceable logs).
- **Auditability Enhancements**:
  - Observability: Exposed metrics/endpoints for monitoring.
  - Traceability: Request IDs propagated across services.
  - Edits: Admin-controlled with immutable audit trails (e.g., blockchain-like logs for key events).
- **Responsiveness/Scalability**:
  - Async processing for non-blocking responses.
  - Caching/sharding for scale; auto-retry mechanisms.
- **Best Practices**:
  - Rate limiting, geo-caching for northern routes.
  - Compliance: Auditable records for government oversight.

**Goals Integration Diagram** (With auditability focus):
```mermaid
graph TD
    subgraph "Auditability (incl. Observability & Traceability)"
        Action[User Action] -->|Log & Trace| ImmutableLog[Immutable Storage]
        ImmutableLog -->|Metrics| Observe[Observability Tools]
        Observe -->|Spans| Trace[Traceability System]
        Trace -->|Edit w/ History| Dashboard[Operational Dashboard]
    end

    subgraph "Responsiveness & Scalability"
        Event[User Event] -->|Quick Process| Responsive[Async Handlers]
        Responsive -->|Load Balance| Scale[Auto-Scalers]
        Scale -->|Monitor| Observe
    end

    subgraph "Security & Tracking"
        Trip[Trip] -->|Real-Time| Trace
        Trip -->|Anomalies| Alert[Alerts]
    end

    Action --> Event
    Alert --> Dashboard
```

## Consequences
- Increased observability adds minor overhead (~5-10%) in latency.
- Rooz-e-Aval self-management implies higher DevOps responsibility.
- Clear separation of protocols simplifies scaling but adds gateway complexity.

## Risk factors
...?!

