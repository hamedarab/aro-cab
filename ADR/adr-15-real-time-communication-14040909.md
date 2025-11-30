**File Name:** `adr-15-real-time-communication-14040909.md`

# ADR 15: Real-Time Communication Protocol Strategy

## Status
Accepted

## Context
As defined in **[ADR-08](./adr-08-inter-city-ride-sharing-14040820.md)**, the platform requires sub-second responsiveness for critical features such as Driver Location Tracking, Ride Matching, and SOS alerts. Reliability is paramount given the target environment involves inter-city travel with potentially unstable mobile network coverage (2G/3G/4G switching).

We evaluated several protocols to handle bi-directional communication between Mobile Clients and the Backend, specifically addressing the trade-off between **Development Velocity**, **Operational Complexity**, and **Raw Performance**.

## Decision
We decided to adopt **ASP.NET Core SignalR** as the primary real-time communication framework, utilizing the **MessagePack Hub Protocol** for binary optimization and **Redis Backplane** for horizontal scaling.

### Core Configuration
1.  **Protocol Abstraction:** SignalR will manage the underlying transport, utilizing **WebSockets** as the default, with automatic fallback to **Server-Sent Events (SSE)** or **Long Polling** where networks or firewalls block WebSockets.
2.  **Serialization:** We will mandate the use of **MessagePack** (binary) instead of JSON to minimize payload size (bandwidth overhead) closer to Protobuf efficiency.
3.  **Horizontal Scaling:** A **Redis Backplane** will be deployed to handle Pub/Sub messaging between backend pods, ensuring clients connected to different server instances receive broadcast messages seamlessy.

## Justification (The Architectural "Why")

While tools like Raw WebSockets or gRPC offer lower level control, SignalR was selected for the following strategic reasons:

1.  **High-Level Abstraction vs. Plumbing:**
    *   Raw access requires reinventing complex logic: Connection Handshakes, Heartbeats (Ping/Pong), Auto-Reconnection, and Frame Parsing. SignalR handles this "plumbing" out-of-the-box, allowing the team to focus on business logic.
2.  **First-Class "Group" Management:**
    *   The business domain requires complex broadcasting (e.g., *"Notify all drivers in Mazandaran"*). SignalR provides native support for **Hub Groups**, making geospatial broadcasting significantly easier to implement than managing subscriber lists manually in Raw WebSockets or gRPC.
3.  **Resilience in Unstable Networks:**
    *   Iranian roads often have spotty coverage. SignalR’s client SDKs (for Flutter/Android) possess built-in, robust logic for handling intermittent connectivity and automatic reconnection, which is non-trivial to implement reliably in gRPC or custom socket solutions.
4.  **Operational Simplicity:**
    *   Unlike MQTT, which requires maintaining a separate Broker cluster (e.g., Mosquitto/EMQ X), SignalR runs in-process with the application application. This aligns with our ".NET First" strategy and reduces DevOps overhead.

## Alternatives Comparison

We conducted a comparative analysis of the leading alternatives against our specific constraints (.NET Team, Iran Network Context):

| Criteria | **SignalR (Core)** | **Raw WebSockets** | **gRPC Streaming** | **MQTT (Standalone)** |
| :--- | :--- | :--- | :--- | :--- |
| **Dev Velocity** | **Very High** (Abstraction handles plumbing) | **Low** (Requires manual state mgmt) | **Medium** (Contract-first/Typed) | **Low** (Requires Broker Integration) |
| **Network Overhead** | **Medium** (Handshake/Metadata) *[Mitigated by MessagePack]* | **Very Low** | **Very Low** (Protobuf) | **Very Low** (Optimized for IoT) |
| **Data Format** | Text (JSON) or **Binary (MessagePack)** | Developer Defined | **Binary (Protobuf)** | Binary |
| **Connection Stability** | **Automatic Reconnect Logic** | Manual Implementation (Complex) | Limited (Retry policies needed) | **Excellent** (QoS Levels) |
| **Client Support** | Strong Official SDKs (Flutter/Android) | Universal Standard | Good (Web implementation is tricky) | Excellent (IoT Ecosystem) |
| **Scalability** | **Redis Backplane** (Simple Setup) | Complex (Manual Pub/Sub impl) | Complex (Needs Service Mesh) | Scalable (Cluster management needed) |
| **Operational Cost** | **In-Process** (Part of App) | In-Process | In-Process | **High** (Dedicated Broker Cluster) |

## Industry Alignment & Analysis

To validate our decision, we analyzed architectures of global ride-hailing leaders. While we learn from them, we adapt to our scale and resources:

1.  **Uber:** Uses a custom protocol on top of **QUIC** (HTTP/3) and Edge services written in Go/Java.
    *   *Our Stance:* We lack the R&D resources to build custom Layer-4 protocols. SignalR provides 80% of the functionality at 10% of the cost.
2.  **Lyft:** Uses **gRPC** extensively with Envoy Proxy at the edge.
    *   *Our Stance:* We align on using gRPC for *internal* service-to-service calls (like Lyft), but prefer SignalR for client communication due to easier state management on mobile.
3.  **Grab:** Heavily relies on **Go (Golang)** for handling thousands of concurrent connections (Goroutines).
    *   *Our Stance:* We acknowledge this advantage. As noted in **ADR-10**, if SignalR consumes excessive memory for ingestion at scale, we reserve the option to offload the "Location Ingestion" component to a **lightweight Go sidecar**.
4.  **Generic IoT/Logistics:** Often uses **MQTT**.
    *   *Our Stance:* Valid for pure IoT, but adding an operational layer (MQTT Broker) increases complexity. SignalR abstracting Redis Pub/Sub provides a similar localized outcome with lower DevOps overhead.

## Consequences

### Positive
*   **Speed to Market:** leveraging the team's deep .NET expertise to deliver robust real-time features rapidly.
*   **Infrastructure Uniformity:** No need to provision and maintain separate MQTT clusters.
*   **Bandwidth Optimization:** Configuring **MessagePack** brings the payload size close to Protobuf, addressing mobile data usage concerns.

### Negative / Risks
*   **Memory Footprint:** SignalR keeps connection state in memory. At 100k+ concurrent users, memory consumption on K8s pods will increase.
    *   *Mitigation:* Horizontal scaling via HPA (ADR-10) and, if necessary, offloading to the Go Ingestion Service.
*   **Protocol Overhead:** Even with binary protocols, SignalR has slightly more header overhead than raw TCP/UDP.
    *   *Accepted Trade-off:* We accept this minor overhead in exchange for reliability and connection management features.