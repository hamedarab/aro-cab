**File Name:** `adr-10-technology-stack-and-components-14040909.md`

# ADR 10: Backend Technology Stack and Component Selection

## Status
Proposed

## Context
Following the requirements in **[ADR-08](./adr-08-inter-city-ride-sharing-14040820.md)** (Sub-200ms latency, 99.9% uptime, Auditability) and the domain boundaries defined in **[ADR-09](./adr-09-domain-service-boundaries-14040820.md)**, we must select a technology stack that supports a **Modular Monolith to Microservices** transition.

The selection criteria are strict:
1.  **Operational Feasibility:** Must run efficiently on self-managed "Rooz-e-Aval" infrastructure (Bare metal/VMs with Kubernetes).
2.  **Auditability & Observability:** The stack must support OpenTelemetry and structured logging natively.
3.  **Talent Availability in Iran:** We need a stack where senior talent is accessible to mitigate hiring risks.
4.  **Performance:** Must handle high concurrency (WebSocket connections for tracking) and high throughput (Kafka ingestion).

## Decision

We decided to standardize on the **Microsoft .NET 8 (LTS)** ecosystem as the primary development stack, supplemented by specific Cloud-Native technologies for data and orchestration.

حتما، این نسخه بازنویسی و اصلاح شده‌ی بخش اول است. در این نسخه **.NET 10** و **C# 14** به عنوان استاندارد پایدار لحاظ شده‌اند، **Go** به عنوان گزینه لبه حفظ شده، و **Autofac** و **FluentValidation** نیز طبق نظر شما اضافه شده‌اند.

***

### 1. Core Development Stack
| Component | Technology | Justification |
| :--- | :--- | :--- |
| **Backend Framework** | **.NET 10** | Selected for its Native AOT capabilities (crucial for faster K8s startup), industry-leading Kestrel performance, and memory efficiency. It leverages the vast and mature .NET talent pool in Iran while offering strict modularity support for our "Modular Monolith" strategy. |
| **Language** | **C# 14** | Utilizing the latest non-preview features (e.g., enhanced Records for immutable DTOs, advanced Pattern Matching) to ensure type safety and cleaner business logic. |
| **Edge/High-Perf Service** | **Go (Golang)** | *Optional:* Reserved strictly for the **Real-time Tracking** or **Ingestion** layer if .NET footprint becomes too heavy for massive concurrent WebSocket connections (Sidecar pattern). |
| **Real-Time Proto** | **SignalR (Core)** | Native support for WebSockets with automatic fallback transports. Handles connection management and broadcasting groups (e.g., "Drivers in Mazandaran") efficiently. |
| **Inter-Service Comm** | **gRPC / Protobuf** | The standard for internal synchronous communication. Provides strongly-typed contracts, high-performance binary serialization (smaller payload than JSON), and supports streaming for bulk data transfer. |
| **Dependency Injection** | **Autofac** | Chosen over the built-in DI container for its advanced capabilities: Assembly scanning (vital for modular architecture), property injection, and better lifetime scope management for complex transaction boundaries. |
| **Validation** | **FluentValidation** | Ensures clear separation of validation logic from business entities. Allows building complex, conditional validation rules that are easily testable. |


### 2. Data Persistence & State
| Component | Technology Candidates | Evaluation Context (Deferred to [ADR-11]) |
| :--- | :--- | :--- |
| **Relational & Geo DB** | **SQL Server 2025**<br>_vs_<br>**PostgreSQL 18** | **Decision Pending ([See ADR-11](./adr-11-database-selection-strategy.md))**.<br><br>**SQL Server 2025 Strengths:** Deep integration with .NET ecosystem, and specifically **In-Memory OLTP (Hekaton)** which is critical for lock-free processing of high-concurrency tables like *ActiveTrips*. Matches the team's deep performance tuning expertise.<br><br>**PostgreSQL 18 Strengths:** Offers **PostGIS**, the industry standard for complex geospatial queries required for the *Matching Service*. Being open-source, it eliminates licensing costs as we scale horizontally on Linux/K8s nodes. |
| **Time-Series Data** | **TimescaleDB**<br>_vs_<br>**InfluxDB**<br>_vs_<br>**SQL Columnstore** | **Decision Pending ([See ADR-11](./adr-11-database-selection-strategy.md))**.<br>Required for storing massive volumes of driver location history (breadcrumbs) for Audit and Traceability.<br>We need to benchmark ingestion rates vs. query performance for "History of a Trip" scenarios between a specialized TSDB and RDBMS Columnstore features. |
| **Distributed Cache** | **Redis Stack** | **Selected.** Chosen for its versatility. It will handle:<br>1. **Session Store:** Fast access to user contexts.<br>2. **Geospatial Indexing:** `GEOADD`/`GEORADIUS` for real-time "Drivers Nearby" queries (if RDBMS latency is too high).<br>3. **Distributed Locks:** Ensuring transactional consistency in the `Matching` service. |
| **Event Store** | **Apache Kafka** | **Selected.** Functions as the immutable "System of Record" for events. Essential for decoupling the *Ordering* domain from *Billing* and *Audit* domains. |
| **Object Storage** | **MinIO** | **Selected.** Provides High-Performance, S3-compatible object storage for storing Driver Documents (KYC) and Trip Validations on-premise within the Rooz-e-Aval infrastructure. |

### 3. Traffic Management & Gateway
| Component | Technology | Justification |
| :--- | :--- | :--- |
| **API Gateway (Primary)** | **Apache APISIX** | **Selected.** Chosen for its fully open-source nature (Apache Foundation) which includes critical features like *Global Rate Limiting*, *Authentication*, and *Observability* out-of-the-box without licensing costs. Its architecture allows hot-reloading of plugins and supports highly dynamic routing for K8s ingress. |
| **API Gateway (Fallback)** | *Kong Gateway (OSS)* | Kept as a fallback option due to its industry maturity. However, it is a secondary choice because key advanced plugins (e.g., advanced security, monitoring, GUI) are locked behind the **Enterprise License**, which creates procurement and cost challenges in our local context. |
| **BFF / Aggregator** | *YARP (Microsoft)* | Considered strictly for the **MVP phase** to accelerate development if gateway configuration becomes a bottleneck. Ideally, we aim to use APISIX from Day 1 to avoid technical debt, but YARP remains a contingency for rapid C#-based routing logic. |
| **Service Mesh** | *Linkerd (Sidecar-less)* | **Deferred to Phase 2.** Currently, K8s native service discovery combined with gRPC client-side load balancing is sufficient. We will introduce a Service Mesh later when advanced traffic splitting (Canary releases) and mTLS observability become critical at scale. |

### 4. Infrastructure, Orchestration & Observability
| Component | Technology | Justification |
| :--- | :--- | :--- |
| **Containerization** | **Docker / Containerd** | Standard industry practice for packaging applications. |
| **Orchestration** | **Kubernetes 1.32+** | Creates the foundation for our "Self-Managed" cloud. Enables **Horizontal Pod Autoscaling (HPA)** for scalability and manages the lifecycle of our diverse workload (Stateful DBs, Stateless APIs, Sidecars). Managed via Helm Charts. |
| **Tracing & Standards** | **OpenTelemetry (OTEL)** | The "One Standard to Rule Them All". Used for manual and automatic instrumentation across .NET 10, APISIX, and Go services. Ensures no vendor lock-in for reliable Trace/Metric/Log collection. |
| **Metrics Store** | **VictoriaMetrics**<br>_or_<br>**Prometheus** | **VictoriaMetrics** is proposed for its superior long-term storage compression and drop-in compatibility. However, **Prometheus** remains a strong alternative candidate due to its ubiquity as the industry standard and the widespread availability of skilled DevOps engineers in the local market. |
| **Log Aggregation** | **Grafana Loki**<br>_or_<br>**ELK Stack** | **Loki** is preferred for its lightweight footprint, cost-efficiency, and seamless correlation via metadata labels. However, the **ELK Stack (Elasticsearch)** is retained as a valid option if complex, Google-like full-text search capabilities are required for deep auditing investigations. |
| **Visualization** | **Grafana** | The single pane of glass for Operational Dashboards (system health) and Business Dashboards (Ride metrics). Works seamlessly with both stack options (Loki/Prometheus or ELK). |

---

## Component-to-Domain Mapping

Here is how the selected stack maps to the Bounded Contexts defined in **ADR-09**:

| ADR-09 Domain | Tech Implementation | Critical Component & Storage Strategy |
| :--- | :--- | :--- |
| **1. Identity & Access** | Keycloak (Containerized) | **Keycloak** for OIDC/OAuth2 flow, backed by SQL Server for identity persistence. |
| **2. User Profile** | .NET 10 Web API | **SQL Server 2025** leveraging **JSON Columns** for flexible profile attributes (no need for NoSQL). |
| **3. Trip Management** | .NET 10 Web API | **SQL Server In-Memory OLTP (Hekaton)** for the *ActiveTrips* table to handle concurrent state changes without locking. |
| **4. Matching Service** | .NET 10 Worker Service | **In-Memory Logic** (Priority Queues) + **Redis** (Distributed Locks) to prevent double-booking. |
| **5. Pricing & Fare** | .NET 10 Web API | **Redis Stack** for high-speed caching of Tariff rules + **SQL Server** for rule persistence. |
| **6. Payment & Wallet** | .NET 10 Web API | **SQL Server** with strict ACID transactions and Constraint Checks for non-negative wallet balances. |
| **7. Real-time Tracking** | .NET 10 (SignalR) / Go | **Redis Geo** for real-time spatial queries ("Drivers Nearby") + **Kafka** for buffering massive location ingress. |
| **8. Safety & SOS** | .NET 10 gRPC Service | Direct high-priority pipe (WebSocket) to Admin Dashboard bypassing standard queues for sub-second alerts. |
| **9. Notification** | .NET 10 Consumer | **Kafka Consumer** listening to events -> Sending SMS/Push via 3rd party providers. |
| **10. Admin & Ops** | **Blazor (Server/WASM)** | Leveraging the team's C# skills for UI. Reads strictly from **DB Read Replicas** to avoid impacting operational transactional performance. |
| **11. Audit & Logging** | .NET 10 Consumer | **Kafka Consumer Group** -> Offloading immutable logs to the selected Audit Store (Loki / ELK / SQL Columnstore). |


## Consequences

### Positive (Strategic Advantages)
*   **Unified Technical Ecosystem:** By standardizing the language and framework across Backend, Worker Services, and Operational Dashboards, we significantly reduce cognitive load. This facilitates internal mobility—engineers can seamlessly switch between "Core Business Logic" and "Internal Tools," accelerating onboarding and **Time-to-Market**.
*   **Performance via Mature Engineering:** We are leveraging enterprise-grade, compiled runtimes and recognized database engines optimized for high concurrency. This allows us to achieve high-frequency processing and low latency on commodity hardware without resorting to niche or experimental technologies.
*   **Strategic Talent Alignment:** The chosen stack corresponds to the deepest and most mature talent pool available in the local market. This minimizes hiring risks and ensures we can scale the engineering team rapidly with senior resources who require minimal upskilling.
*   **Compliance by Design:** "Auditability" is not treated as a post-deployment feature but is architected into the system's core via distributed tracing standards and immutable event logs. This ensures regulatory alignment from Day 1.

### Negative (Accepted Complexities)
*   **Operational Severity:** Owning a "Self-Managed" private cloud infrastructure gives us data sovereignty but places the entire burden of reliability (HA, DR, Patching) on our internal DevOps capabilities. We do not have the safety net of Managed Cloud PaaS providers.
*   **Architectural Rigor Required:** Transitioning from a traditional layered architecture to an Event-Driven, Modular design introduces significant complexity regarding **Data Consistency**. Developers must adopt a disciplined mindset regarding *Idempotency* and *Eventual Consistency*, which requires strict governance and code reviews.
*   **Heavier Initial Development:** Implementing robust isolation, resilience patterns (Circuit Breakers), and observability piping upfront increases the initial development effort compared to a rapid "Script-based" approach, but this investment pays off in stability at scale.

## Risk Factors (Mitigation Strategies)

### 1. Complexity of Stateful Workloads in Orchestration
**Risk:** Running mission-critical, stateful components (Databases, Event Streams) inside container orchestrators requires advanced operational maturity. Misconfiguration can lead to data loss or performance degradation.
*   **Mitigation:**
    *   Utilize **Kubernetes Operators** to automate complex lifecycle management (backups, upgrades, failovers).
    *   Invest heavily in **Automated Disaster Recovery (DR)** drills.
    *   Keep critical stateful storage on dedicated nodes or outside the orchestration layer if stability benchmarks are not met during load testing.

### 2. Eventual Consistency & Integrity Gaps
**Risk:** In distributed asynchronous flows, message duplication or out-of-order delivery can act silently to corrupt sensitive business data (e.g., wallet balances), largely due to network unreliability.
*   **Mitigation:**
    *   Enforce **Idempotency** at the API and Consumer level using unique request/event IDs.
    *   Implement the **Outbox Pattern** to guarantee valid transaction boundaries between local database commits and event publishing.

### 3. Distributed "Split-Brain" Scenarios
**Risk:** Given the reliance on distributed caching and clustering, network partitioning within the data center could cause services to operate on stale or conflicting data.
*   **Mitigation:**
    *   Configure strict **Quorum** requirements for cluster leaders.
    *   Implement widespread use of **Resilience Libraries** (Retries with Exponential Backoff, Bulkheads) to handle transient failures gracefully.