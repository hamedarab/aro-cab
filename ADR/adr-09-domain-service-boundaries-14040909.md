
**File Name:** `adr-09-domain-service-boundaries-14040820.md`

# ADR 09: Domain Decomposition and Service Boundaries

## Status
Accepted

## Context
As defined in [ADR-08](./adr-08-inter-city-ride-sharing-14040820.md), we are adopting a transition strategy from a Modular Monolith to Microservices. To prevent a "Distributed Monolith" and ensure independent scalability of high-load components, we must clearly define Bounded Contexts based on Domain-Driven Design (DDD) principles.

The system requires distinct handling of operational concerns (Booking, Utility) versus support concerns (Notifications, Audit) to meet the business goals of organized inter-city transport and regulatory compliance.

For the mapping of these contexts to specific technologies (Data Stores, Protocols, Infrastructure), please refer to **[ADR-10: Backend Technology Stack](./adr-10-technology-stack-and-components-14040909.md)**.

## Decision
We decided to decompose the backend architecture into the following Logical Concept Services (Bounded Contexts). In **Phase 1**, these will exist as separate modules/libraries within a unified solution. In **Phase 2**, critical domains (marked with *) will be extracted into independent Microservices.

> **Note:** The specific component selection and storage strategy for each domain below is detailed in the **[Component-to-Domain Mapping](./adr-10-technology-stack-and-components-14040909.md#component-to-domain-mapping)** section of ADR-10.

### 1. Identity & Access Management (IAM)
*   **Responsibilities:** User registration, Authentication (SSO/JWT), Role Management (Passenger/Driver/Operator), and Driver KYC (Know Your Customer) verification flows.
*   **Key Data:** Users, Roles, Permissions, KYC Documents.

### 2. User Profile
*   **Responsibilities:** Management of extended user data, Driver documentation storage, Verification status tracking, and User Scoring/Rating calculation.
*   **Key Data:** Profiles, Ratings, License Info.

### 3. Trip Management
*   **Responsibilities:** The core lifecycle machine of a trip (Created -> Matched -> Started -> Ended), trip history, and status management.
*   **Key Data:** Trip Aggregate, Trip State Log.

### 4. Matching Service
*   **Responsibilities:** Algorithms to pair passengers with drivers, calculating capacity, handling cancellations, and managing supply/demand queues.
*   **Key Data:** Active Drivers Queue, Passenger Request Queue.

### 5. Pricing & Fare
*   **Responsibilities:** Calculation of trip costs based on distance/time, Dynamic Pricing (surge pricing) logic, Discount codes, and Commission calculation.
*   **Key Data:** Tariff Rules, Discount Coupons, Surge Multipliers.

### 6. Payment & Wallet 
*   **Responsibilities:** Online payment gateway integration, Wallet management (Credit/Debit), and Driver Settlement (payouts).
*   **Key Data:** Wallet Transactions, Ledgers, Gateway Logs.

### 7. Real-time Tracking 
*   **Responsibilities:** High-frequency ingestion of Driver/Passenger locations, WebSocket connections, Breadcrumbs storage, and Geofencing triggers.
*   **Key Data:** Geo-spatial Time-series data, Active Sessions.

### 8. Safety & SOS
*   **Responsibilities:** Emergency button logic, Real-time behavior monitoring (speeding/anomaly detection), and Alert dispatching to the Operation Center.
*   **Key Data:** Safety Alerts, Incident Reports.

### 9. Notification Service
*   **Responsibilities:** A centralized channel for sending outbound messages via SMS, Push Notifications, Email, and In-app messages.
*   **Key Data:** Notification Templates, Delivery Logs.

### 10. Admin & Ops
*   **Responsibilities:** Back-office dashboard for monitoring active trips, applying restrictions (bans), inspection workflows, and operational reporting.
*   **Key Data:** Admin Audit Logs, tickets.

### 11. Audit & Logging Service 
*   **Responsibilities:** As a first-class citizen for compliance; collecting sensitive business events, immutable logging, and query capabilities for regulatory oversight.
*   **Key Data:** Immutable Event Store, Trace Logs.

## Consequences
*   **Establishment of Boundaries:** Developers must respect these boundaries across the codebase immediately, preventing spaghetti code dependencies.
*   **Scalability Roadmap:** High-throughput domains (Matching, Tracking, Audit) are identified early for isolated scaling.
*   **Complexity:** Managing inter-module communication (even in-process) requires strict contract definitions.