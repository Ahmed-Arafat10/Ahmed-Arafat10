# BIS Graduation Project Management System (GPMS) — Engineering Case Study
> [!NOTE]
> **Confidentiality Notice**
>
> This document is an engineering case study describing my architectural decisions, engineering challenges, technical trade-offs, and personal contributions while working on this project.
>
> To respect confidentiality obligations, proprietary implementation details, internal source code, business logic, infrastructure, workflows, and other confidential information have been intentionally omitted or generalized.
>
> The purpose of this document is to demonstrate my engineering approach and technical decision-making rather than disclose how the original product was implemented.

## 1. Engineering Context & Core Challenges

This project required architecting a centralized digital platform to manage the complex lifecycle of academic graduation projects for a large university. The system serves as the primary hub for thousands of students, academic supervisors, and administrators, replacing highly fragmented, error-prone legacy administrative processes (e.g., decentralized Excel sheets and paper forms).

**The Core Engineering Challenges I Addressed:**
*   **Data Migration & Normalization:** Ingesting, parsing, and normalizing massive, unstructured datasets from legacy university spreadsheets while ensuring data integrity and preventing duplicate records.
*   **Concurrent Resource Allocation:** Managing complex, highly constrained team formation and supervisor assignment algorithms, ensuring strict adherence to academic prerequisites and capacity limits without encountering race conditions.
*   **High-Volume Asynchronous Processing:** Designing a reliable background queue system to handle bulk operations, such as dispatching thousands of transactional emails and system notifications without degrading API response times.
*   **Endpoint Security & Abuse Prevention:** Architecting targeted rate limiters and strict multi-guard authentication to protect sensitive endpoints (like credential recovery via Social Security Numbers) from brute-force attacks and abuse.

## 2. Architectural Decisions & Trade-offs

### A. Service-Oriented Architecture (SOA)
*   **The Problem:** The standard MVC pattern often leads to "fat controllers" and tightly coupled logic, especially when an application must serve vastly different user roles (Students, Supervisors, Administrators) with distinct business rules.
*   **My Decision:** I bypassed the default monolithic MVC structure in favor of a Service-Oriented Architecture (SOA). I segregated the application logic into distinct domain modules, each containing its own controllers, services, and routing definitions.
*   **Trade-offs Considered:** SOA introduces a steeper learning curve for onboarding new developers and requires more upfront boilerplate code. However, the strict domain isolation prevents monolithic spaghetti code and allows multiple teams to scale feature development independently.
*   **Lessons Learned:** This architectural choice proved invaluable when adding the administrative compliance module later in the project, as it required zero modifications to the existing student or supervisor domains.

### B. Cache-Aside Strategy for Aggregate Data
*   **The Problem:** Dashboards required complex, aggregated reporting (e.g., real-time counts of unassigned students, open complaints, and pending team approvals) which caused severe database I/O bottlenecks during peak traffic periods (e.g., registration deadlines).
*   **My Decision:** I implemented a Cache-Aside strategy using Redis. Heavy queries are cached for configurable TTLs (10-20 minutes). The system fetches from the database only on cache misses, updating the cache transparently.
*   **Trade-offs Considered:** Caching introduces the risk of serving slightly stale data to administrators. To mitigate this, I implemented targeted cache invalidation hooks on critical write operations.
*   **Lessons Learned:** The massive reduction in database load during traffic spikes validated this approach. It highlighted the importance of measuring read-to-write ratios before prematurely optimizing SQL queries.

## 3. Major Engineering Problems Solved

### A. Legacy Data Migration & Automation
*   **The Challenge:** Every semester, administrators provided messy, unstructured data files containing student records. Attempting to parse this via synchronous HTTP uploads resulted in timeouts and silent failures.
*   **My Implementation:** I engineered a suite of robust CLI commands (custom Artisan tasks) specifically designed to run on the server. These commands bulk-import, sanitize, and validate the data against academic constraints, resolving conflicts before inserting records into the relational database.
*   **Engineering Reasoning:** Shifting heavy data migrations to the CLI rather than the web layer prevents web server timeouts, allows for long-running batch transactions, and provides granular progress logging for system administrators.

### B. Defensive Security & Targeted Rate Limiting
*   **The Challenge:** The platform handles sensitive student data (e.g., SSNs, CGPAs) and acts as the credential distributor. Exposing these endpoints publicly invites brute-force attacks and malicious spam.
*   **My Implementation:** Beyond standard API tokens (Sanctum) and strict CORS configurations, I built custom middleware rate limiters dynamically mapped to unique identifiers (like IP addresses and Student IDs). For example, automated password reset requests are hard-capped at two attempts per rolling 24-hour window.
*   **Engineering Reasoning:** Standard global rate limiting is insufficient against distributed attacks targeting specific user accounts. Granular, identifier-based throttling protects individual user integrity without punishing the entire network.

### C. Asynchronous Event-Driven Processing
*   **The Challenge:** During registration spikes, forming a team triggers a cascade of synchronous events: database writes, constraint validations, audit logging, and email notifications to all members. This caused unacceptable API latency (3-5 seconds per request).
*   **My Implementation:** I refactored the architecture to be event-driven. The HTTP request now only handles constraint validation and immediate database mutations. Actions like audit logging and email dispatch are pushed onto a Redis-backed queue to be processed by background daemon workers.
*   **Engineering Reasoning:** Decoupling non-critical I/O operations from the main thread guarantees sub-second API response times and ensures that temporary SMTP outages do not cause application-wide transaction rollbacks.

## 4. Engineering Retrospective & Growth

Reflecting on the system's architecture, there are key areas I would optimize today to handle even higher scale:

1.  **Test-Driven Development (TDD) for Migrations:** While the CLI data importers handle current edge cases well, relying on manual validation of dirty legacy data is fragile. I would now mandate strict, automated unit tests covering all data mutation edge cases, duplicate resolution strategies, and rollback scenarios.
2.  **GraphQL Integration:** The REST API strictly versions endpoints, but the varied dashboard reporting requirements often lead to over-fetching or under-fetching data. Implementing a GraphQL layer would allow frontend clients to dynamically specify their data requirements, further optimizing payload sizes and reducing network overhead.

This project was a rigorous exercise in translating complex organizational processes into a secure, performant digital infrastructure. It demonstrates my ability to design scalable APIs, protect sensitive endpoints, and architect resilient background processing systems.
