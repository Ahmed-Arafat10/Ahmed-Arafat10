# Buy2 — Engineering Case Study
> [!NOTE]
> **Confidentiality Notice**
>
> This document is an engineering case study describing my architectural decisions, engineering challenges, technical trade-offs, and personal contributions while working on this project.
>
> To respect confidentiality obligations, proprietary implementation details, internal source code, business logic, infrastructure, workflows, and other confidential information have been intentionally omitted or generalized.
>
> The purpose of this document is to demonstrate my engineering approach and technical decision-making rather than disclose how the original product was implemented.

## 1. Engineering Context & Core Challenges

This project required the architecture and implementation of an enterprise-grade backend system designed to unify workforce scheduling, high-fidelity time tracking, and performance evaluation. The system needed to operate as the single source of truth for dynamic resource allocation and temporal constraint validation.

**The Core Engineering Challenges I Addressed:**
*   **Recursive Scheduling Conflict Engine:** Designing a deterministic scheduling pipeline that replicates matrix-based schedules across future dates while evaluating thousands of potential overlaps, availability conflicts, and contractual constraints in real time.
*   **Zero-Hardware Geofencing:** Implementing a reliable, low-latency physical presence verification system without relying on easily spoofed GPS data or expensive biometric hardware.
*   **Time-Series Overtime Accumulation:** Architecting a multi-week projection engine capable of calculating cumulative overtime exposure by evaluating both historical time logs and future planned shift matrices against strict contractual thresholds.
*   **Event-Driven Asynchronous Scoring:** Building a robust cron-based evaluation engine to parse large datasets of employee actions and distribute performance scores and loyalty points asynchronously.

## 2. Architectural Decisions & Trade-offs

### A. Modular Monolith & Domain-Driven Design (DDD)
*   **The Problem:** Traditional MVC folder structures in large frameworks rapidly deteriorate into tight coupling and sprawling "god classes" when dealing with hundreds of models and enterprise workflows.
*   **My Decision:** I architected the application as a strict Modular Monolith, mapping directories directly to bounded business contexts (e.g., Scheduling, Identity, Loyalty, Time Tracking). I enforced a strict Service Layer pattern, wrapping complex logic in Facades.
*   **Trade-offs Considered:** While a Microservices architecture was initially considered, the overhead of distributed data synchronization and cross-service transactions was deemed too high for the initial scale. The Modular Monolith provided the maintainability of DDD while preserving the transactional simplicity of a single database.
*   **Lessons Learned:** This architectural boundary enforcement allowed multiple developers to work independently without merge conflicts. Should the system need to scale into separate microservices in the future, the bounded contexts are already cleanly segregated.

### B. Request-Layer Domain Validation
*   **The Problem:** Placing heavy business rule validation inside service classes clutters the core logic and makes the services harder to unit test.
*   **My Decision:** I shifted all constraint evaluations (e.g., "Is this employee already booked?", "Is this site active?") into dedicated, custom Request Validator classes utilizing parameterized rule sets.
*   **Trade-offs Considered:** This approach creates slightly "thicker" request validation pipelines, which can result in multiple pre-flight database queries. To optimize this, I implemented query caching and bulk data loading within the validation rules.
*   **Lessons Learned:** Ensuring that the Service Layer only receives 100% sanitized and constraint-verified data drastically reduced runtime exceptions and allowed the services to remain pure and strictly focused on database mutations.

## 3. Major Engineering Problems Solved

### A. Shift Replication & Conflict Resolution Engine
*   **The Challenge:** Administrators needed the ability to dynamically replicate successful operational schedules across subsequent weeks or months. This operation had to recursively evaluate future dates against a changing landscape of employee leave requests, altered site hours, and strict qualification requirements.
*   **My Implementation:** I developed a multi-step matrix processing pipeline. During a copy operation, the engine pre-fetches all target date constraints in a single bulk query to prevent N+1 bottlenecks. It then runs a conflict evaluation matrix in memory.
*   **Engineering Reasoning:** Instead of outright failing the entire batch upon encountering a single conflict, the engine compiles a structured array of warnings and successful mappings. It then uses database transactions to tentatively apply the valid schedules, returning the conflict payload to the client for manual resolution.

### B. Zero-Hardware Location Geofencing
*   **The Challenge:** Time theft ("buddy punching" or remote clock-ins) is a massive liability. GPS data is notoriously unreliable indoors and easily spoofed via mobile software. 
*   **My Implementation:** I implemented a network-layer validation strategy. The backend extracts the client device's connected Wi-Fi Router MAC address directly from encrypted HTTP headers during the clock-in request. It then cross-references this MAC address against an indexed table of authorized hardware registered to that specific operational site.
*   **Engineering Reasoning:** By relying on immutable network hardware metadata rather than client-reported GPS coordinates, the system achieved a highly secure, zero-cost physical geofence that operates flawlessly deep inside large concrete structures.

### C. Time-Series Overtime (OT) Accumulation Engine
*   **The Challenge:** Approving a future shift requires the system to project if that shift will push an employee past their contractual weekly hour limit, necessitating overtime pay.
*   **My Implementation:** I built an evaluation algorithm that constructs custom date-range queries dynamically based on the employee's specific contractual work week (e.g., Sunday-Thursday vs. Monday-Friday). It aggregates previously committed time logs and sums them against the proposed future shift duration.
*   **Engineering Reasoning:** Operating strictly on UTC timestamps at the database level and applying timezone offsets only during the application-layer evaluation ensured that boundary edge cases (shifts spanning midnight across timezone changes) were calculated perfectly.

## 4. Engineering Retrospective & Growth

Reflecting on the system's architecture, there are key areas I would optimize today based on subsequent experience:

1.  **Test-Driven Development (TDD) for Scheduling Logic:** While the domain boundaries are exceptionally clean, relying on manual QA for complex, multi-variable shift conflict resolution is risky. I would now mandate strict, automated unit and integration tests covering all permutation edge cases of the conflict engine to guarantee regression safety.
2.  **RESTful Paradigm Strictness:** In the early iterations, high-mutation endpoints (like clocking in or out) were configured as `GET` requests for simplicity in mobile integrations. I would now enforce strict RESTful compliance, converting all state-mutating operations to idempotent `POST` or `PUT` requests to ensure proper caching behavior and semantic correctness.
3.  **Event-Driven Architecture (EDA) for Notifications:** Currently, notification dispatches (emails and push notifications) are tightly coupled to the end of successful transaction blocks. I would refactor these into an asynchronous Event Bus, decoupling the notification logic from the core HTTP request lifecycle to improve API response latency. 

This project was a rigorous exercise in designing robust, constraint-heavy enterprise systems. It demonstrates my ability to navigate the complexities of dynamic scheduling algorithms, strict data validation, and scalable modular architecture.
