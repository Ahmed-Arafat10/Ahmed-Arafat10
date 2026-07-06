# DT Requirements — Engineering Case Study
> [!NOTE]
> **Confidentiality Notice**
>
> This document is an engineering case study describing my architectural decisions, engineering challenges, technical trade-offs, and personal contributions while working on this project.
>
> To respect confidentiality obligations, proprietary implementation details, internal source code, business logic, infrastructure, workflows, and other confidential information have been intentionally omitted or generalized.
>
> The purpose of this document is to demonstrate my engineering approach and technical decision-making rather than disclose how the original product was implemented.

## 1. Engineering Context & Core Challenges

This project involved architecting a large-scale compliance and digital transformation tracking platform. The system enables massive organizations to ingest complex regulatory frameworks, upload unstructured evidence documents (PDFs, DOCX), and automatically evaluate compliance using a combination of algorithmic aggregation, distributed search, and external AI analysis.

**The Core Engineering Challenges I Addressed:**
*   **Complex Data Ingestion:** Designing a robust parsing engine to ingest, validate, and normalize highly complex, multi-lingual (Arabic/English) spreadsheet taxonomies into a deeply nested relational database tree.
*   **Distributed Full-Text Search:** Architecting a high-performance pipeline to extract and index unstructured textual data from thousands of uploaded documents into a distributed search cluster, offloading heavy read I/O from the primary relational database.
*   **Non-Deterministic Service Integration:** Integrating slow, non-deterministic third-party AI assessment services into strict, synchronous compliance workflows without causing API timeouts or breaking relational data constraints.
*   **Algorithmic Tree Aggregation:** Building a dynamic evaluation engine to recursively compute organizational compliance scores across n-level deep sub-indices.

## 2. Architectural Decisions & Trade-offs

### A. Microservices Approach for Search
*   **The Problem:** Traditional SQL databases (MySQL/PostgreSQL) suffer severe performance degradation when executing `FULLTEXT` searches across thousands of massive, unstructured PDF text dumps.
*   **My Decision:** I adopted a microservices approach for the search domain. The core PHP application handles transactions and access control, but it ships parsed document chunks (paragraphs, tables) to a dedicated Node.js/Elasticsearch cluster via synchronous HTTP calls.
*   **Trade-offs Considered:** Introducing Elasticsearch increases infrastructure complexity, deployment overhead, and requires eventual consistency tracking. However, the architectural split provides orders of magnitude faster full-text search capabilities and isolates heavy analytical reads from critical transactional writes.

### B. Asynchronous Event-Driven AI Processing
*   **The Problem:** Synchronously querying external AI models for document analysis caused the primary web threads to hang, leading to massive API timeouts and a degraded user experience.
*   **My Decision:** I decoupled the AI analysis into a background queue system. When a document is uploaded, an asynchronous job is dispatched. This job handles the AI API request, implements exponential backoff for rate limits, and updates the database upon completion.
*   **Trade-offs Considered:** Background processing introduces eventual consistency—users do not see their AI score immediately. To mitigate UX issues, I implemented websocket/polling statuses to indicate when a document is "Under Analysis."

### C. Strict Database Transaction Wrapping
*   **The Problem:** The ingestion of massive compliance Excel sheets involves hundreds of sequential database inserts. If a parsing error occurs halfway through, the database is left in a corrupted, partial state.
*   **My Decision:** I enforced strict database-level transactions around all bulk ingestion and complex deletion operations.
*   **Trade-offs Considered:** Transaction wrapping slightly increases memory overhead and locks tables for longer durations during imports. However, guaranteeing atomicity (all-or-nothing insertion) is a non-negotiable requirement for enterprise compliance data integrity.

## 3. Major Engineering Problems Solved

### A. Bridging Non-Deterministic AI with Strict Relational Models
*   **The Challenge:** External AI models return unstructured, natural language responses. However, the compliance database requires strict, numerical percentage scores to calculate aggregate health statuses.
*   **My Implementation:** Inside the asynchronous queue worker, I engineered custom Regex-based parsers and fallback heuristics. These parsers reliably extract the strict numerical scores hidden within localized, unstructured Arabic text responses, sanitizing the data before persisting it to the relational database.
*   **Engineering Reasoning:** You cannot trust external, non-deterministic APIs to directly mutate internal state. Building a robust parsing layer ensures data integrity remains uncompromised even if the AI's output format fluctuates.

### B. Algorithmic Compliance Tree Aggregation
*   **The Challenge:** Organizational compliance health must be calculated dynamically based on the completion status of thousands of fluctuating, n-level deep sub-requirements.
*   **My Implementation:** I developed an algorithmic aggregation engine that recursively traverses the compliance tree. It rolls up evidence completion percentages from the leaf nodes, applies weighted logic, and calculates top-level status values (Red/Yellow/Green) efficiently.
*   **Engineering Reasoning:** Pre-calculating and storing these values on every write is inefficient due to frequent updates. Computing them dynamically via an optimized, recursive tree traversal ensures the dashboard always reflects the real-time state of the organization.

### C. Automated Multilingual Report Generation
*   **The Challenge:** Auditors require natural language summaries explaining exactly what percentage of an index is complete and what specific documents are missing, in both English and Arabic.
*   **My Implementation:** I designed a dynamic template engine that injects variable data (calculated percentages, missing relational constraints) into localized linguistic templates, automatically generating human-readable compliance commentary.

## 4. Engineering Retrospective & Growth

Reflecting on the system's architecture, there are key areas I would optimize today to handle even higher scale:

1.  **Refactoring the Primary Domain Service:** As the system grew, the core `ComplianceService` evolved into a "God Object," handling UI logic, zip archive generation, and Elasticsearch syncing. I would now mandate strict adherence to SOLID principles, decomposing this monolith into discrete, single-responsibility Action classes (e.g., `ExportEvidenceAction`, `SyncSearchIndexAction`).
2.  **Event Sourcing for Audit Trails:** Compliance platforms require rigorous auditability. While the current system tracks basic mutations, implementing an Event Sourcing pattern would allow administrators to "time travel" and view the exact state of the compliance tree at any specific date in the past.

This project demonstrates my capability to architect enterprise-grade systems that orchestrate complex integrations, handle massive unstructured datasets, and bridge the gap between non-deterministic AI models and strict relational databases.
