# Uniters Academy — Engineering Case Study
> [!NOTE]
> **Confidentiality Notice**
>
> This document is an engineering case study describing my architectural decisions, engineering challenges, technical trade-offs, and personal contributions while working on this project.
>
> To respect confidentiality obligations, proprietary implementation details, internal source code, business logic, infrastructure, workflows, and other confidential information have been intentionally omitted or generalized.
>
> The purpose of this document is to demonstrate my engineering approach and technical decision-making rather than disclose how the original product was implemented.

## 1. Engineering Context & Core Challenges

This project required the architecture and implementation of a highly scalable, transaction-heavy financial back-office and hierarchical user management system. The platform operates at the intersection of a distributed educational service and a complex, performance-sensitive compensation and referral network. 

**The Core Engineering Challenges I Addressed:**
*   **High-Throughput Financial Reconciliation:** Designing an automated batch settlement engine capable of computing multi-tiered cyclic balances and payout caps across thousands of interconnected nodes without computational bottlenecks.
*   **Hierarchical Graph Traversal:** Managing a massive, deeply nested binary tree network, necessitating optimized graph traversal algorithms for real-time node placement and volume propagation.
*   **Strict Transactional Integrity:** Architecting a multi-wallet, double-entry financial ledger that rigorously prevents race conditions, double-spending, and data corruption in a highly concurrent environment.
*   **State-Machine Provisioning:** Orchestrating a complex, multi-stage user onboarding workflow that safely coordinates identity verification, third-party payment gateways, and asynchronous webhook callbacks.

## 2. Architectural Decisions & Trade-offs

### A. Modular Domain-Driven Design (DDD)
*   **The Problem:** The standard MVC architecture in modern PHP frameworks often degrades into a "Big Ball of Mud" when handling complex, multi-domain enterprise applications, leading to bloated controllers and tightly coupled logic.
*   **My Decision:** I bypassed the default monolithic structure and implemented a Modular Domain-Driven Design layout. I strictly segregated the application into distinct bounded contexts (e.g., Ledger, Tree Management, Identity Verification). Each module encapsulates its own request validation, business service layer, and data presentation.
*   **Trade-offs Considered:** Introducing DDD adds upfront scaffolding and abstraction overhead, making simple CRUD operations slightly more verbose. However, it was strictly necessary to maintain codebase cohesion and allow multiple developers to work on discrete domains without merge conflicts.
*   **Lessons Learned:** The domain isolation proved invaluable during the implementation of the settlement engine. It ensured that changes in the user identity domain never inadvertently impacted the financial ledger.

### B. Application-Layer Graph Traversal (BFS) vs. Database Recursion
*   **The Problem:** The system relies on a deeply nested binary tree structure. Executing deep recursive queries or utilizing Common Table Expressions (CTEs) in relational databases often results in severe CPU spikes and deadlocks under heavy write loads.
*   **My Decision:** Instead of delegating graph traversal to the database, I implemented a Breadth-First Search (BFS) algorithm in the application layer using specialized PHP Data Structures. The backend bulk-loads relation maps and executes the traversal in memory.
*   **Trade-offs Considered:** Moving traversal logic to the application layer increases memory consumption on the application servers. However, application servers are horizontally scalable, whereas relational database CPU capacity is far more difficult and expensive to scale vertically.
*   **Lessons Learned:** The in-memory BFS approach drastically reduced database load during high-traffic placement operations. In future iterations handling millions of nodes, I would migrate the hierarchical data structure to a dedicated Graph Database (e.g., Neo4j) to offload this complexity entirely.

### C. Pessimistic Database Locking for the Financial Ledger
*   **The Problem:** Concurrent requests handling internal ledger transfers or external withdrawals can trigger race conditions, resulting in the dreaded "double-spend" vulnerability.
*   **My Decision:** I implemented strict pessimistic row-level locking (`lockForUpdate`) on all wallet and ledger tables during financial mutations.
*   **Trade-offs Considered:** Pessimistic locking creates potential bottlenecks and lock contention if transactions take too long to process. To mitigate this, I designed the financial transaction scopes to be extremely lightweight, ensuring locks are held for milliseconds.
*   **Lessons Learned:** Strict ACID compliance at the database level is non-negotiable for financial ledgers. Implementing these constraints upfront prevented concurrency bugs during peak usage events.

## 3. Major Engineering Problems Solved

### A. High-Throughput Reconciliation Engine
*   **The Challenge:** Periodically calculating point volumes, evaluating cycle thresholds, computing carry-over balances, and managing payout ceilings across a deeply connected hierarchical network.
*   **My Implementation:** I engineered a batch processing engine that isolates sub-trees and evaluates compensation rules dynamically. The engine uses modulo arithmetic and cyclical threshold checks to determine payout tiers and flash-out (ceiling limit) events.
*   **Engineering Reasoning:** I separated the mathematical computation phase from the database writing phase. The engine calculates all resulting state mutations in memory first, then applies them via bulk database updates within a single overarching transaction, minimizing lock duration.

### B. Immutable Audit & Event Logging
*   **The Challenge:** In a system handling financial payouts and hierarchical modifications, administrators needed a secure, reliable way to track who changed what, and when.
*   **My Implementation:** I decoupled the auditing system using an event-driven observer pattern. Every critical state change triggers an event that records the `oldModel` and `newModel` JSON states, the actor ID, and the IP address into a centralized immutable log.
*   **Engineering Reasoning:** Rather than scattering logging commands throughout the business logic, the observer pattern keeps the domain services clean and ensures auditing cannot be accidentally bypassed by developers.

### C. Complex Multi-Stage Provisioning
*   **The Challenge:** Onboarding required coordinating document uploads, One-Time Password (OTP) verifications, external payment gateways, and final graph placement atomically. If a user paid but the graph placement failed, the system would enter an inconsistent state.
*   **My Implementation:** I architected a linear state-machine for the onboarding wizard. State progression is strictly gated. Webhooks from the payment gateway act as asynchronous state triggers, and the final placement step wraps all database operations within a fail-safe transaction block.

## 4. Engineering Retrospective & Growth

Reflecting on the system's architecture, there are key areas I would optimize today to handle higher scale:

1.  **Asynchronous Event Queues:** Currently, certain heavy tasks (like processing large batch settlements or triggering verification emails) are handled synchronously or in tight loops. I would refactor these processes into a distributed messaging queue (e.g., RabbitMQ or Kafka) to ensure maximum API responsiveness and reliable retry mechanisms.
2.  **Dedicated Graph Infrastructure:** The relational database performs admirably using the in-memory BFS optimization, but relational structures are fundamentally mismatched for deep, highly dynamic graph data. Migrating the network topology to a Graph Database would simplify the relationship modeling and speed up downline/upline aggregations natively.
3.  **Test-Driven Development (TDD) for Financial Logic:** While the domain boundaries are clean, relying heavily on manual QA for complex mathematical settlement algorithms introduces regression risks. I would now mandate strict, automated unit testing covering all boundary cases, cycle caps, and floating-point math scenarios within the financial engine.

This project was a rigorous exercise in designing robust, transactional enterprise systems. It demonstrates my ability to navigate the complexities of distributed data structures, strict financial integrity, and scalable modular architecture.
