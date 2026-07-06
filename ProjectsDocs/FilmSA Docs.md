# FilmSA — Engineering Case Study
> [!NOTE]
> **Confidentiality Notice**
>
> This document is an engineering case study describing my architectural decisions, engineering challenges, technical trade-offs, and personal contributions while working on this project.
>
> To respect confidentiality obligations, proprietary implementation details, internal source code, business logic, infrastructure, workflows, and other confidential information have been intentionally omitted or generalized.
>
> The purpose of this document is to demonstrate my engineering approach and technical decision-making rather than disclose how the original product was implemented.

## 1. Engineering Context & Problem Statement

I architected and led the backend development of an enterprise-grade digital ecosystem designed to automate and audit complex financial incentive programs. The core engineering challenge was orchestrating massive financial data ingests, running automated regulatory compliance checks, and managing parallel, multi-stage approval processes.

Instead of building a standard CRUD application, I was tasked with designing a robust system that could handle:
*   **Complex State Management**: Orchestrating a multi-step state machine with granular section-level locking, rollback capabilities, and an immutable audit history.
*   **High-Volume Data Processing**: Ingesting, parsing, and validating large, unstructured financial spreadsheets against relational database schemas in real-time.
*   **Secure File Orchestration**: Managing multi-gigabyte document pipelines with zero risk of orphaned files or corrupted states.

**My Role & Contribution**: As the lead backend architect, I designed the core system components, established the architectural patterns, and implemented the most complex data-parsing and state-management engines. I owned the technical decisions surrounding data integrity, system modularity, and the application's overall security posture.

## 2. System Architecture & Design Decisions

### Modular Domain-Driven Architecture
**The Engineering Challenge**: Standard MVC framework structures frequently degrade into tightly coupled, unmaintainable monolithic architectures when forced to handle multiple distinct, highly complex business domains. 
**My Solution**: I engineered a decoupled, domain-driven variant of the MVC architecture. Rather than grouping all controllers, models, and services together globally, I strictly segregated the codebase into distinct business domains.
**Why I Chose This**: This approach enforced separation of concerns, preventing cross-domain side effects. Each domain fully encapsulated its own HTTP routing, business services, database models, and internal validation logic.
**Trade-offs Considered**: I thoroughly evaluated migrating to a full microservices architecture. However, I determined that a modular monolith provided the optimal balance of deployment simplicity and domain isolation for our engineering team size, mitigating the operational overhead of microservices while retaining high maintainability.
**Lessons Learned**: This structural decision proved invaluable. It enabled multiple engineers to work on disparate domains simultaneously without merge conflicts, minimized the risk of regression bugs, and made the codebase highly navigable for onboarding new engineers.

### Two-Phase Transactional File Upload Pipeline
**The Engineering Challenge**: The system required clients to upload multi-gigabyte document payloads. Processing these uploads synchronously during form submission led to network timeouts. Furthermore, partial database failures resulted in orphaned files lingering on the filesystem.
**My Solution**: I designed and implemented a Two-Phase Staging File Upload pipeline. Files are uploaded asynchronously to a temporary staging path and registered in a staging database table. Only when the final workflow step is successfully submitted does the backend assert file presence, migrate the files to permanent storage, and commit the overarching database transaction.
**Why I Chose This**: I chose this architecture to decouple file I/O from database transactions. It ensures immediate validation feedback to the client and mathematically guarantees a zero-orphan filesystem footprint.
**Lessons Learned**: Managing state between the filesystem and the database is inherently error-prone. Making file processing purely transactional ensures system integrity even under extreme load, network instability, or catastrophic application crashes.

### Dynamic Multi-Step State Machine
**The Engineering Challenge**: The platform required managing intricate workflows where different user roles (applicants and auditors) interacted with the exact same application simultaneously. If an auditor rejected a specific sub-section, the system needed to unlock exclusively that precise section for the applicant while strictly enforcing immutability on all previously approved sections.
**My Solution**: I engineered a dynamic state machine via custom traits and service layers. Every form section mapped to strict state boundaries. Before any write operation, the system dynamically evaluated the current state, preventing user inputs from overriding audited records.
**Engineering Reasoning**: I chose to build a declarative state machine over relying on scattered conditional checks to centralize the rules for read/write access. This ensured consistency across hundreds of API endpoints.
**Lessons Learned**: Extracting authorization and state logic completely out of the controller layer and into centralized, reusable state engines drastically reduces edge-case bugs in complex workflow applications. This abstraction also rendered unit testing the complex state transitions trivial.

## 3. Tackling Complex Technical Problems

### The Unstructured Data Parsing Engine
**The Problem**: The system needed to ingest massive, unstructured financial spreadsheets containing thousands of line items, validate them mathematically, and map them to relational database models. Relying on simple CSV imports resulted in corrupted data, unhandled exceptions, and severe database performance bottlenecks.
**My Approach**: I engineered a custom, high-performance spreadsheet compiler. Instead of naively reading rows, this engine acts as a highly opinionated structural schema validator:
*   **Coordinate & Cell-Level Validation**: It strictly enforces cell coordinates and recursively validates data types (e.g., date serials, decimal precision).
*   **On-the-Fly Database Mapping**: It evaluates foreign key mappings dynamically, verifying formats and metadata against active database records residing in memory.
*   **Error Reporting**: It returns a structured array of row-level errors for user correction, explicitly highlighting failing cell coordinates rather than failing silently.
**Trade-offs**: I initially chose synchronous processing with tight memory constraints over asynchronous background processing to provide an immediate feedback loop to the user. I aggressively utilized batch database insertions and transaction-level locks to optimize I/O.
**What I Would Improve Today**: Under massive concurrent loads, synchronous parsing presents a memory exhaustion risk. To optimize this further, I would offload this specific compiler engine to a queue-based background worker, implementing real-time WebSocket progress updates to maintain the immediate feedback loop while freeing up HTTP worker threads.

### Stateless Security & Real-Time Collaboration
**The Problem**: Combining high-security financial auditing with the requirement for instant collaboration (live chat, immediate system notifications) without introducing the heavy overhead of stateful session management.
**My Approach**: 
*   **Stateless Gateway**: I implemented a strictly stateless API gateway using JWT, heavily separating user and administrative guards at the foundational routing level. This eliminated cross-contamination risks between vastly different authorization contexts.
*   **Event-Driven WebSockets**: To facilitate real-time interactions without excessive HTTP polling, I integrated a high-performance WebSocket server. I built an event loop that broadcasted secure, encrypted events whenever state mutations occurred in the backend.
**Why I Chose This**: Stateless APIs provide massive horizontal scalability and vastly simplify caching layers. WebSockets provide the necessary real-time feedback loop for time-sensitive operations without consuming critical HTTP request threads.

## 4. Performance, Security, and Scalability

*   **Database Design**: I designed a highly normalized relational database schema (comprising over 200 distinct migrations) to carefully handle intricate relational constraints. Data integrity was ensured via strict foreign keys, comprehensive indexing strategies, and transactional safety across all complex data mutations.
*   **Security Posture**: I engineered a multi-guard state validation system, stateless token revocation, and an exponential-lockout OTP verification mechanism to thwart brute-force attacks. File uploads were strictly sandboxed, cryptographically hashed, and completely detached from user-provided input names.
*   **Scalability**: By maintaining the core API layer as purely stateless and offloading all real-time broadcasting to dedicated asynchronous nodes, the application can scale horizontally under severe load without introducing session management bottlenecks.

## 5. Retrospective & Key Takeaways

Through architecting and building this enterprise platform, I significantly refined my approach to designing large-scale backend systems:
*   **Data Immutability is Critical**: In auditing and financial applications, strict state lockouts and immutable histories are non-negotiable. Designing a robust state machine early in the software development lifecycle saves countless hours of debugging data inconsistencies later.
*   **Assume Bad Input**: Whether interfacing with external APIs or handling user-generated spreadsheets, defensive programming—especially the custom data compiler I built—proved absolutely essential for maintaining strict database integrity.
*   **Separation of Concerns Equals Velocity**: The deliberate architectural decision to enforce domain-driven modularization directly correlated with the engineering team's ability to ship new features quickly, safely, and predictably.
