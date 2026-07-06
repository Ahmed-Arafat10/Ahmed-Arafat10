# Graduation Section Registration System (SRS) — Engineering Case Study
> [!NOTE]
> **Confidentiality Notice**
>
> This document is an engineering case study describing my architectural decisions, engineering challenges, technical trade-offs, and personal contributions while working on this project.
>
> To respect confidentiality obligations, proprietary implementation details, internal source code, business logic, infrastructure, workflows, and other confidential information have been intentionally omitted or generalized.
>
> The purpose of this document is to demonstrate my engineering approach and technical decision-making rather than disclose how the original product was implemented.

## 1. Engineering Context & Core Challenges

This project involved designing and developing a highly specialized scheduling and resource registration platform. The system handles extreme traffic spikes during defined registration windows, where thousands of users simultaneously compete for limited, highly contested resources (e.g., schedule slots and team allocations).

**The Core Engineering Challenges I Addressed:**
*   **High-Concurrency Race Conditions:** Preventing data corruption and over-allocation when thousands of users attempt to reserve the same limited resource at the exact same millisecond.
*   **Defensive Security & Sandboxing:** Architecting a secure processing pipeline to accept, unpack, and statically analyze thousands of user-uploaded code archives (ZIP files), mitigating severe Remote Code Execution (RCE) risks.
*   **Complex Domain Constraints:** Designing robust validation engines to enforce deeply nested, multi-layered prerequisites (e.g., team size limits, mutual exclusivity rules, and eligibility whitelists) within a single transactional boundary.
*   **Throughput Bottlenecks in Third-Party APIs:** Engineering a resilient, load-balanced background job system to dispatch tens of thousands of transactional notifications without triggering strict SMTP provider rate limits.

## 2. Architectural Decisions & Trade-offs

### A. Vertical Slice Architecture vs. Standard MVC
*   **The Problem:** The standard MVC (Model-View-Controller) structure groups files by technical type (Controllers together, Services together). As the domain complexity grew, tracking the flow of a single feature across disparate directories severely hindered velocity and increased cognitive load.
*   **My Decision:** I transitioned the core application to a Vertical Slice Architecture. Each major domain feature (e.g., `RegisterSection`, `UploadDeliverable`) acts as a self-contained module encompassing its own Controller, Request Validator, and Domain Service.
*   **Trade-offs Considered:** This deviates from framework conventions and requires custom routing configurations. However, the immense gain in feature isolation and high cohesion made refactoring and debugging significantly faster.

### B. Pessimistic Database Locking
*   **The Problem:** During registration windows, concurrent requests would read the same "available seats" value and successfully commit, leading to over-booked capacity.
*   **My Decision:** I implemented strict database-level transactions leveraging pessimistic row locking. When a request initiates a registration, it locks the specific resource row, forcing concurrent requests for the same resource to wait until the transaction completes.
*   **Trade-offs Considered:** Pessimistic locking can introduce query bottlenecks or deadlocks under extreme load. I mitigated this by keeping the transaction scope exceptionally narrow—validating all preconditions *before* acquiring the lock, and only wrapping the final capacity check and insertion within the transaction.

### C. In-Memory ZIP Scanning (Security)
*   **The Problem:** The system required users to upload zipped code archives. Writing these unverified archives to disk before processing posed a massive security risk (e.g., directory traversal, reverse shells).
*   **My Decision:** I engineered a custom sandboxing service that streams the uploaded archive directly into memory. It recursively inspects file headers and statically analyzes text/code files against strict regex signatures (blocking execution patterns like `eval`, `shell_exec`) *before* any payload is permitted to persist on the storage drive.
*   **Trade-offs Considered:** In-memory decompression significantly increases CPU and RAM overhead per request. To prevent Denial of Service (DoS) attacks via "zip bombs," I implemented strict byte-size limits and maximum file count thresholds within the extraction loop.

## 3. Major Engineering Problems Solved

### A. High-Throughput Notification Load Balancing
*   **The Challenge:** Upon successful registration, the system guarantees an instant digital receipt. Dispatching thousands of synchronous emails overwhelmed the SMTP provider, resulting in blocked IPs and dropped connections.
*   **My Implementation:** I completely decoupled the notification layer. I built a custom "Smart Mail Sender" service running on background queue workers. The service acts as a round-robin load balancer across multiple configured SMTP credentials. It tracks daily quotas in real-time and automatically rotates the sending account to stay beneath provider limits.
*   **Engineering Reasoning:** Rather than relying on expensive enterprise transactional email services, solving the throughput constraint at the application level demonstrated resourcefulness and reduced infrastructure costs dramatically.

### B. Deeply Nested Array Validation via Form Requests
*   **The Challenge:** Validating complex, deeply nested JSON payloads for team creation (e.g., verifying that arrays of user IDs meet specific criteria across multiple relational tables) traditionally required massive, inefficient N+1 database queries scattered throughout controller logic.
*   **My Implementation:** I leveraged the framework's Form Request lifecycle to build highly complex, custom validation rules. These rules aggregate all necessary IDs from the payload and perform a single, efficient `WHERE IN` database query to validate the entire nested structure simultaneously.
*   **Engineering Reasoning:** Pushing complex domain validation strictly into the request lifecycle ensures that controllers and services *only* ever receive pristine, business-compliant data, eliminating defensive programming clutter downstream.

## 4. Engineering Retrospective & Growth

Reflecting on the system's architecture, there are key areas I would optimize today to handle even higher scale:

1.  **Read-Heavy Caching Layer:** While pessimistic locking solves write-concurrency, the system still reads capacity data directly from the database to render the UI. I would implement an in-memory datastore (e.g., Redis) to cache "available seats," significantly reducing database read I/O during peak traffic.
2.  **Automated Testing for Concurrency:** The complexity of the row-locking mechanisms was primarily validated through rigorous manual stress testing. I would now mandate automated integration tests simulating concurrent race conditions to ensure regression safety when modifying the transaction boundaries.

This project was a rigorous exercise in system reliability and defensive engineering. It demonstrates my ability to identify concurrency vulnerabilities, implement robust architectural patterns, and design security-first data processing pipelines.
