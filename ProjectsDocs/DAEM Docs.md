# DAEM — Engineering Case Study

> [!NOTE]
> **Confidentiality Notice**
>
> This document is an engineering case study describing my architectural decisions, engineering challenges, technical trade-offs, and personal contributions while working on this project.
>
> To respect confidentiality obligations, proprietary implementation details, internal source code, business logic, infrastructure, workflows, and other confidential information have been intentionally omitted or generalized.
>
> The purpose of this document is to demonstrate my engineering approach and technical decision-making rather than disclose how the original product was implemented.

## 1. Executive Summary & Engineering Role

As the Senior Software Engineer for this platform, my primary responsibility was designing and engineering a large-scale data aggregation, extraction, and strategic intelligence engine. The system's purpose was to ingest massive volumes of unstructured data, run complex benchmarking algorithms, and orchestrate semantic searches to validate compliance and generate strategic insights.

My focus was on building a fault-tolerant, scalable, and highly maintainable architecture. I was responsible for end-to-end system design, establishing the API boundaries, structuring the relational and NoSQL databases, and engineering the core parsing algorithms that formed the backbone of the ingestion pipeline.

## 2. Architectural Design & Trade-offs

I architected the system using a modular, Domain-Driven Design (DDD-lite) approach. Rather than organizing the codebase by technical concerns (e.g., all controllers together, all models together), I isolated features into cohesive business domains. 

### Microservices Orchestration
The system required specialized processing that a single monolithic stack could not efficiently handle. I designed an orchestrated architecture combining a core backend with specialized microservices:
*   **Core Backend:** Handled business logic, relational integrity, and API routing.
*   **Python OCR/AI Service:** Offloaded heavy text extraction and machine learning tasks.
*   **Node.js Middleware:** Managed high-throughput communication with the full-text search cluster.

**Trade-off:** Distributing the system introduced network latency and deployment complexity. I accepted this trade-off because attempting to run heavy OCR processes or intensive search aggregations within the main backend request lifecycle would have severely degraded API performance and compromised stability. 

### Double Indexing Strategy
To support both strict relational data and unstructured semantic search, I implemented a double-indexing architecture. 
*   **Primary Datastore (PostgreSQL):** Ensured ACID compliance, transaction safety, and strict normalization for financial data and benchmarking metrics.
*   **Search Datastore (Elasticsearch):** Indexed raw text paragraphs and metadata for rapid, high-volume semantic queries.

**Why I chose this:** Relational databases are fundamentally unsuited for fast, full-text semantic searching across millions of unstructured document chunks. Separating the read-heavy search operations from the write-heavy relational transactions protected the primary database from complex query exhaustion.

## 3. Core Engineering Challenges & Solutions

### Challenge A: Designing a Fault-Tolerant, Unstructured Data Extraction Pipeline
**The Problem:** The system needed to ingest highly irregular, unstructured text outputs from the OCR microservice and map them into a strict tabular relational schema. Real-world OCR data is inherently messy—containing typos, misalignments, inconsistent formatting, and varied date structures.

**My Solution:** I engineered a multi-pass parsing engine that moved away from deterministic string matching. Instead, I implemented statistical text-matching algorithms (such as Levenshtein distance) to evaluate confidence thresholds between the extracted text and the expected schema definitions. 

**Engineering Reasoning:** Relying on exact string matches (`==`) would have resulted in an unacceptably high failure rate, shifting the burden to manual user correction. By utilizing a threshold-based similarity algorithm, the system autonomously corrected minor OCR anomalies and dynamically categorized rows, significantly improving the ingestion pipeline's resilience.

**Lessons Learned:** Building fault-tolerance directly into the parsing layer is essential for data ingestion systems. In hindsight, I would enhance this by persisting the confidence scores of every match. This would allow the system to automatically flag low-confidence mappings for human review, creating a continuous feedback loop to fine-tune the matching algorithm.

### Challenge B: Real-Time Analytics & Rules Engine
**The Problem:** The system required the capability to run complex mathematical benchmarking (calculating weighted averages across large datasets) and evaluate these metrics to generate automated, dynamic narrative commentary.

**My Solution:** I designed a modular commentary and analytics engine. Rather than hardcoding evaluation logic, I built a conditional rules evaluator that dynamically parses logical operators and thresholds against the computed metrics, rendering text narratives based on the results.

**Trade-offs:** I initially favored transactional database integrity over eventual consistency to ensure accuracy of the aggregated metrics. The drawback was that synchronous calculation of large aggregates introduced API latency during peak ingestion.

**Lessons Learned:** Decoupling heavy aggregations into asynchronous background queues is critical for scaling. While the rules engine design proved highly extensible—allowing new conditions to be added without code changes—the underlying aggregation logic should have been moved to an event-driven message broker (e.g., RabbitMQ or Kafka) earlier in the architectural lifecycle.

### Challenge C: Managing External API Constraints & Distributed Scraping
**The Problem:** The platform relied on aggregating data from various external third-party APIs, many of which enforced aggressive rate limiting (e.g., HTTP 429 Too Many Requests) and had highly divergent payload structures.

**My Solution:** I engineered a centralized API consumption layer that normalized incoming payloads into a unified internal model. To handle rate limits, the service intelligently parses response headers to calculate dynamic wait times and implements an exponential backoff strategy, preventing thread-blocking and ensuring resilient data ingestion.

**Engineering Reasoning:** Direct integration with third-party services in the main business flow creates brittle systems. Abstracting the scraping and API consumption logic ensured that when an external provider changed their schema or rate limits, the core system remained unaffected.

## 4. System Design Principles & Maintainability

To ensure the codebase remained maintainable and scalable as complexity grew, I enforced several strict engineering principles:

*   **Boundary Isolation:** I strictly separated Controllers (HTTP layer), Form Requests (Validation), and Services (Business Logic). Controllers remained extremely thin, acting only as traffic directors.
*   **Trait-Based Utility Sharing:** For cross-cutting concerns like standardizing formats or shared parsing logic, I utilized composable traits. This prevented code duplication across isolated domain services while avoiding deep, rigid inheritance trees.
*   **Database Normalization:** I designed a highly normalized schema (over 80 tables) to prevent data duplication, utilizing foreign key constraints and transactional blocks to guarantee relational integrity.

## 5. Retrospective & Architectural Takeaways

Reflecting on the architecture, the decision to use a modular domain structure and separate the search infrastructure (double-indexing) was the system's greatest success. It mitigated the unstructured query bottlenecks and allowed independent scaling of the search capability.

**What I would do differently today:**
1.  **Test-Driven Development for Algorithms:** While the system structure was robust, the complex algorithmic parsing logic lacked comprehensive automated test coverage. I would mandate rigorous unit and integration testing for any text-similarity or mathematical aggregation engines from day one.
2.  **Asynchronous by Default:** I would adopt an event-driven architecture earlier. Moving sector recalculations and heavy analytics processing out of the synchronous request-response cycle and into distributed background workers would significantly improve the platform's ability to scale horizontally.

Ultimately, engineering this platform reinforced my approach to building robust enterprise systems: anticipate dirty data, isolate your domain boundaries, and choose the right datastore for the specific operational workload.
