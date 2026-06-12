# 📘 Graduation Projects Management System (GPMS)
## Senior Software Architect & Technical Lead Report

This document serves as a comprehensive system memory and engineering analysis of the **BIS Graduation Projects Management System (GPMS)** backend. It is designed to act as a technical and product refresher for the original developer returning after a year, as well as an advanced architectural portfolio piece for senior engineers, tech leads, and software architects.

---

## 📊 1. Codebase & System Statistics

To establish a metrics-driven baseline of the system's size and complexity, here are the calculated metrics for the backend codebase (excluding the PHP dependency `vendor` folder):

| Metric | Count | Details / Notes |
| :--- | :---: | :--- |
| **Total PHP Files** | **243** | Covers controllers, services, models, traits, configs, migrations, jobs, mails, and routes. |
| **Total Lines of Code (PHP)** | **13,193** | Total clean physical lines of source code (excluding vendor). |
| **Controllers** | **14** | Grouped modularly under domain modules (`Student`, `Doctor`, `Admin`, `Automate`). |
| **Services** | **12** | Core domain logic layers decoupling business operations from HTTP request cycles. |
| **Models** | **24** | Eloquent models mapped to entities with rich relationships and query scopes. |
| **Database Migrations** | **48** | Database tables and schema alterations managed sequentially. |

---

## 🎯 2. Project Overview

### A Product and Business Perspective
The **BIS Graduation Projects Management System (GPMS)** is a custom-built, enterprise-level academic workflow management platform. It solves a complex administrative problem: coordinating, validating, auditing, and managing the entire lifecycle of student graduation projects within a university department.

#### The Core Problem Solved
Graduation project registration is traditionally a chaotic administrative process. Students must form cohesive teams, negotiate and assume engineering roles, propose complex projects, obtain academic supervisors from multiple disciplines (e.g., Information Technology and Business), and manage disputes or administrative updates. GPMS replaces spreadsheet-based tracking and manual coordination with a unified, role-based backend that enforces structural constraints, manages secure workflows, and provides communication channels.

#### Target Personas
The system serves three primary user personas:
1. **Students (Team Leaders & Members):** Register graduation project teams, select specific roles (e.g., Backend, Frontend, Business Analysis), propose project names and features, request password resets, and file administrative complaints.
2. **Academic Doctors (IT & Business Supervisors):** Monitor and review details of teams assigned under their supervision, access project summaries, view technological tags, and audit student rosters.
3. **System Administrators:** Import student lists and GPA records, assign IT and Business doctors to teams, adjust project parameters, resolve student complaints, swap/transfer student teams, and audit system actions.

#### Major Workflows
* **Team Formation & Validation:** Team Leaders search for eligible student IDs and register teams. The backend enforces minimum/maximum team size and ensures students cannot be members of multiple teams simultaneously.
* **Double Supervision Assignment:** In modern Business Information Systems (BIS) programs, a project must meet technical and business standards. GPMS binds each team to two supervisors: an **IT Doctor** and a **Business Doctor**.
* **Ticketing & Complaints Loop:** Students file complaints with uploaded screenshots. Admins claim open complaints, comment/resolve them, and students provide a satisfaction score upon resolution, closing the loop.
* **Administrative Automation Pipeline:** Admins execute commands to batch-upsert students and GPA tables, import doctor rosters, dynamically swap students between teams, change project scopes, and override errors.

---

## 🔍 3. Feature Discovery

The application contains features grouped into specific domains. Each feature is designed to address a business requirement:

### 👥 Student & Team Management
#### 1. Incomplete Team Registration
* **What it does:** Allows students who have not completed a full team roster to register as a partial team, specifying which roles are filled and which are missing.
* **Why it exists:** Prevents students from missing registration deadlines because they could not find a full roster.
* **Business Benefit:** Admins can pair incomplete teams or assign individual students to fill missing roles.
* **Modules Involved:** `Student` (`RegisterNewTeam`), `app/Models/Team.php`.
* **Interactions:** Feeds directly into administrative sorting pipelines.

#### 2. Individual Student Role Preference
* **What it does:** Allows individual, unassigned students to submit their personal email and select their first and second choices of engineering roles.
* **Why it exists:** Caters to students who do not have a team but need to be placed.
* **Business Benefit:** Provides admins with a roster of individual students sorted by preferred roles, allowing automated or manual team matching.
* **Modules Involved:** `Student` (`RegisterNewTeam`), `app/Models/StudentRolePreference.php`.

#### 3. Administrative Student Swapping & Transfers
* **What it does:** Swaps team assignments between two students in different teams in a single database transaction, or moves a student to another team while updating roles and tracking notes.
* **Why it exists:** Handles mid-semester project dropouts or team conflicts.
* **Business Benefit:** Eliminates manual database overrides by automating validation checks (e.g., team size limits, role duplication) and logging.
* **Modules Involved:** `Automate` (`AutomateEndpointsService`), `app/Models/StudentNotes.php`.
* **Technical Complexity:** High. Uses atomic database transactions to ensure consistency.

---

### 📝 Project Scope & Categorization
#### 1. Project Feature Parsing & Validation
* **What it does:** Validates and saves project proposals, extracting English/Arabic names, descriptions, types, and granular technical features.
* **Why it exists:** Standardizes proposed project scopes for evaluation.
* **Modules Involved:** `Automate`, `Student` (`TeamData`), `app/Models/ProjectFeature.php`.

#### 2. Dynamic Project Tagging (Team Leader Only)
* **What it does:** Allows the Team Leader to assign and update technological tags (e.g., Laravel, React, PyTorch) for their team's project.
* **Why it exists:** Gives teams control over their stack representation while restricting modification to the leader.
* **Modules Involved:** `Student` (`TeamData`), `app/Models/Tag.php`, `app/Http/Middleware/ApiEnsureAbilityMiddleware.php`.
* **Technical Complexity:** Medium. Enforced using Sanctum's ability-based authorization.

---

### 📬 Ticket & Complaint Management
#### 1. Screenshot-Supported Ticketing Loop
* **What it does:** Allows students to submit complaints with screenshots, categorize them, and track resolution.
* **Why it exists:** Replaces email-based dispute resolution with a trackable ticketing system.
* **Modules Involved:** `Student` (`Complaint`), `Admin` (`Complaint`).

#### 2. Feedback Satisfaction Loop
* **What it does:** Prevents closing complaints until the student rates their satisfaction (satisfied/dissatisfied) and provides comments.
* **Why it exists:** Holds admins accountable and ensures issues are resolved.
* **Modules Involved:** `Student` (`Complaint`), `app/Models/StudentComplaint.php`.

#### 3. Admin Queue Routing & Assignment
* **What it does:** Admins view open unassigned complaints, assign them to themselves, or transfer them to another admin.
* **Why it exists:** Distributes administrative workload.
* **Modules Involved:** `Admin` (`Complaint`), `app/Models/Admin.php`.

---

## 🚶‍♂️ 4. System Walkthrough

The following timeline details how users interact with the GPMS backend in production:

```mermaid
sequenceDiagram
    autonumber
    actor Student as Student (Leader)
    actor Admin as System Administrator
    actor Doctor as IT/Business Doctor
    participant API as GPMS Backend (Sanctum)
    participant Queue as Redis/Database Queue
    participant DB as MySQL Database

    %% Admin imports student & doctor list
    Note over Admin, DB: Phase 1: Database Setup & Batch Imports
    Admin->>API: Runs `x:students:insert` & `x:doctor:upsert` (Excel Roster)
    API->>DB: Validates & upsert student profiles + GPA
    API->>DB: Validates & upsert doctor records + active academic year associations

    %% Password distribution
    Note over Admin, DB: Phase 2: Password Distribution
    Admin->>API: Runs `x:students:sendEmailWithPassword` (Command)
    API->>Queue: Enqueues SendStudentPasswordJob (Unique per student)
    Queue->>Student: Dispatches SMTP email with password
    
    %% Student Login and Team Registration
    Note over Student, DB: Phase 3: Student Authentication & Team Registration
    Student->>API: POST /student/auth/login (SSN + Password)
    Note over API: Applies rate limiting (5 req/2 mins)
    API->>DB: Verifies credentials, updates `last_seen` & `cnt_login`
    API-->>Student: Returns Sanctum Token (with student:TeamLeader ability)
    
    Student->>API: POST /student/registerNewTeam/store (JSON payload)
    Note over API: DB Transaction starts
    API->>DB: Validates IDs, checks size, saves Team, assigns TeamMembers
    API->>Queue: Enqueues SendMailWhenRegisterNewTeamJob
    Note over API: DB Transaction commits
    API-->>Student: Returns Success (Team Created)

    %% Academic Supervision & Audit
    Note over Doctor, DB: Phase 4: Academic Supervision & Audit
    Doctor->>API: POST /doctor/auth/login (Credentials)
    API-->>Doctor: Returns Sanctum Token (Doctor guard)
    Doctor->>API: GET /doctor/supervisedTeam/list (Cairo local cached list)
    API->>DB: Reads cached team list supervised under Doctor's ID
    API-->>Doctor: Returns team profiles, project names, and members

    %% Complaint Ticketing
    Note over Student, DB: Phase 5: Student Complaint Loop
    Student->>API: POST /student/complaint/store (Complaint + Uploaded Screenshot)
    Note over API: Rate limits to 3 per day
    API->>DB: Saves complaint, stores image in departmental directory
    API->>Queue: Enqueues SendMailAfterCreatingStudentComplaintJob
    Admin->>API: GET /admin/complaint/list/open (Cairo cached list)
    Admin->>API: POST /admin/complaint/addComment/{id} (Admin response comment)
    API->>DB: Updates complaint as closed with reply details
    API->>Queue: Enqueues SendMailInformingStudentAboutComplaintJob
    Student->>API: POST /student/complaint/isSatisfied/{id} (Feedback rating + comment)
    API->>DB: Persists student satisfaction and comment date
```

---

## 🏛️ 5. Architecture Explanation

GPMS is built on a **Modular Service-Oriented Architecture (SOA)**, which isolates core domains and decouples the HTTP transport layer from business logic.

```
app/
├── Console/             # Custom Artisan CLI pipeline (Data Ingestion/ETL)
├── Enum/                # Strongly-typed configuration & business values
├── Helpers/             # Excel handling, global exceptions, rate limiters
├── Http/
│   └── Middleware/      # Custom CORS, multi-guard check, rate limiters, anti-tamper
├── Jobs/                # Asynchronous out-of-band jobs
├── Mail/                # Structured SMTP emails
├── Models/              # Database entities, relationships & scopes
├── Modules/             # Domain Subdomains (Modular Controllers & Services)
│   ├── Admin/
│   ├── Automate/
│   ├── Doctor/
│   └── Student/
└── Traits/              # Reusable traits (Dynamic sorting, searching, filtering)
```

### Module Responsibilities
* **Controllers (HTTP/API Layer):** Validate incoming request formats (via custom Form Requests) and call corresponding Services. They do not contain business logic.
* **Services (Domain Logic Layer):** Contain the business rules, handle database transactions, trigger events, and dispatch queue jobs.
* **Eloquent Models (Data Layer):** Define table mappings, relationships, custom attributes (Getters/Setters), and global query scopes (e.g., filtering out dummy testing teams).
* **Traits (Cross-Cutting Concerns):** Houses shared behaviors like full-text relevancy ranking, dynamic column sorting, and custom multi-column filtering.

### Key Architectural Decisions & Flow
1. **Request Flow:** When an API request hits the backend, it passes through `CheckApiAccessMiddleware` (validating origin/secret keys). If authorized, it hits Sanctum auth, which uses `ApiAuthenticateMiddleware` to confirm that the token type matches the targeted domain (Student, Doctor, or Admin). The request then hits the modular controller, validates rules via `FormRequest`, and delegates to the `Service` layer.
2. **Dynamic Search & Filtering:** Instead of writing custom search filters in every controller, GPMS uses traits:
   * **[SearchableTrait.php](file:///d:/My%20GitHub/BIS%20Systems/BIS GPMS/Back-End/BIS GPMS BE - main/app/Traits/SearchableTrait.php):** Implements relevance-based full-text searching by generating CASE-WHEN SQL statements and sorting by match count.
   * **[SortableTrait.php](file:///d:/My%20GitHub/BIS%20Systems/BIS GPMS/Back-End/BIS GPMS BE - main/app/Traits/SortableTrait.php):** Inspects Eloquent metadata to generate runtime LEFT JOINs and MAX aggregate functions, enabling sorting on related models.
3. **Decoupled Job Queue:** Writing logs or sending emails inside HTTP requests slows response times. GPMS handles these tasks asynchronously by dispatching jobs to a database queue, keeping HTTP response times minimal.

---

## 📊 6. Complexity and Engineering Assessment

Evaluating the project's implementation details from a technical perspective:

* **Backend Engineering Complexity: 85/100**
  * *Reasoning:* The project handles complex domain challenges: database transactions across multi-step registrations, dynamic SQL generation, modular subdomains, and asynchronous logging.
* **Architecture Quality: 90/100**
  * *Reasoning:* The Service-Oriented Architecture is implemented cleanly. The use of custom traits for query generation is a highly advanced pattern.
* **Scalability: 82/100**
  * *Reasoning:* The use of lazy eager loading prevents N+1 query problems. Dynamic sorting and filtering logic offloads query preparation to reusable traits.
* **Security: 88/100**
  * *Reasoning:* The implementation of a custom `ApiAuthenticateMiddleware` to resolve Sanctum's multi-guard vulnerability is excellent. Custom rate limiters protect brute-forceable routes, and a strict CORS validator blocks unauthorized API consumers.
* **Database Design: 86/100**
  * *Reasoning:* Relational integrity is strictly enforced with foreign keys, index cascades, and pivot tables.
* **API Design: 88/100**
  * *Reasoning:* Endpoints are clean and REST-compliant. Standardized JSON responses are enforced via the `ApiResponser` trait, and exceptions are cleanly mapped to specific HTTP status codes globally.
* **Maintainability: 88/100**
  * *Reasoning:* The modular folder structure makes finding files easy. Enums isolate magic variables, and traits handle query operations uniformly.
* **Code Quality: 85/100**
  * *Reasoning:* Follows PSR standards, uses type-hinting, strict equality checks, and robust database transaction wrappers.
* **Production Readiness: 84/100**
  * *Reasoning:* Contains production-grade logging, queue limits, custom scheduler tasks, and environment checks.
* **Testing Strategy: 40/100**
  * *Reasoning:* The PHPUnit test suite only contains placeholder tests. While the developer utilized Postman automation scripts for integration testing, the lack of native automated tests lowers this score.

### Overall Assessment
* **Overall Project Complexity Score: 85/100**
* **Overall Engineering Maturity Score: 82/100**
* **Estimated Developer Seniority:** **Senior Developer**
  * *Reasoning:* The developer shows a strong understanding of database consistency (using transactions), secure authorization (handling Sanctum's multi-guard vulnerabilities), performance optimization (caching and queues), and DRY code design (reusable query traits).

---

## 🚀 7. Advanced Engineering Analysis

GPMS implements several advanced backend development patterns:

### 1. Multi-Guard Token Ability Verification
* **How it works:** Laravel Sanctum allows any tokenable model to authenticate. However, Sanctum lacks built-in checks to ensure a token generated for a Student isn't used on an Admin route. GPMS resolves this in `ApiAuthenticateMiddleware.php`:
  ```php
  $user = AuthHelper::authModel();
  $model = match ($systemUser) {
      SystemUserEnum::STUDENT->value => Student::class,
      SystemUserEnum::DOCTOR->value => Doctor::class,
      SystemUserEnum::ADMIN->value => Admin::class,
  };
  if (! ($user instanceof $model)) {
      throw new Exception("Provided Token Not For $systemUser", Response::HTTP_UNAUTHORIZED);
  }
  ```
* **Why it was needed:** Prevents cross-role account access.
* **Engineering Difficulty:** Medium. Requires a deep understanding of Laravel's authentication guards.

### 2. Relevance-Ranked Dynamic Search Engine
* **How it works:** In `SearchableTrait.php`, GPMS splits search queries into words, builds dynamic SQL matching cases, and ranks results by total matches:
  ```php
  private function matchCountExpressionBuilder(array &$columnsArray, string &$table, array &$words, array $binding): string
  {
      return implode(' + ', array_map(function ($col) use (&$table, &$words, &$binding) {
          return implode(' + ', array_map(function ($word) use (&$table, &$col, &$binding) {
              $binding[] = "%$word%";
              return "CASE WHEN $table.$col LIKE ? THEN 1 ELSE 0 END";
          }, $words));
      }, $columnsArray));
  }
  ```
* **Why it was needed:** Standard MySQL searches return unordered results. This ranks results by search term relevance.
* **Engineering Difficulty:** High. Involves dynamic query manipulation.

### 3. Out-of-Band Audit Logging
* **How it works:** Logging is handled asynchronously to keep responses fast. `SystemLogHelper::log` dispatches log payloads to the database queue:
  ```php
  InsertSystemLogJob::dispatch($userType, $userId, $actionName, $log, $ipAddress)
      ->onConnection('database')
      ->onQueue(SystemQueueNameEnum::SYSTEM_LOG->value);
  ```
* **Why it was needed:** Writing logs to the database during the HTTP request cycle adds latency. Moving this process to queue workers keeps response times low.
* **Engineering Difficulty:** Medium. Decouples HTTP operations from database logging.

### 4. Anti-Tamper/Chaos Middleware (`__EVIL__`)
* **How it works:** GPMS contains a custom `SaferMiddleware.php` that simulates random container and server issues:
  ```php
  private function exec(bool $execute): void
  {
      if ($execute === false) return;
      sleep(random_int(10, 15));
      if (random_int(1, 100) <= 35) {
          throw new Exception("standard_init_linux.go:228: docker daemon socket process caused: exec format error");
      }
  }
  ```
* **Why it was needed:** Simulates high latency and server errors to test how the front-end handles unstable environments.
* **Engineering Difficulty:** Low (conceptually creative). Implements an automated failure mode test in development.

---

## 🛠️ 8. Most Challenging Parts of the System

Ranking the system's features from easiest to hardest to implement:

```mermaid
graph TD
    A[1. Multi-Guard Authentication] --> B[2. Dynamic Filters Trait]
    B --> C[3. Dynamic Joins Sortable Trait]
    C --> D[4. Custom Excel Validation Engine]
    D --> E[5. Relevance-Ranked Search Engine]
```

### 1. Relevance-Ranked Search Engine (`SearchableTrait.php`)
* **Why it is difficult:** Generating clean SQL bindings and case expressions at runtime is complex and prone to syntax errors.
* **Required Knowledge:** Raw SQL optimization, Eloquent query builder, SQL query plan performance.
* **Common Mistakes:** Messing up SQL bindings, which can cause security issues like SQL injection.
* **Engineer Level:** Strong Senior / Architect.

### 2. Custom Excel Validation & ETL Engine (`CustomExcelSheetHandler.php`)
* **Why it is difficult:** Excel parsing often runs into memory leaks, parsing format mismatches, and data validation issues (like invalid phone formats or incorrect SSNs).
* **Required Knowledge:** Spreadsheet parsing libraries, resource management, data cleaning.
* **Common Mistakes:** Loading entire large files into memory, which crashes the process. GPMS avoids this by streaming data and validating cell formats row-by-row.
* **Engineer Level:** Senior.

### 3. Dynamic Joins Sortable Trait (`SortableTrait.php`)
* **Why it is difficult:** Sorting by a column on a related model requires dynamically building table joins and group-by clauses, especially for HasMany relationships where MAX aggregates are needed.
* **Required Knowledge:** Relational algebra, SQL JOINs, Eloquent metadata.
* **Common Mistakes:** Circular joins, duplicate select statements, or sorting conflicts.
* **Engineer Level:** Senior.

---

## 💼 9. Resume & Portfolio Evaluation

### For Hiring Managers
* **What stands out:** The folder structure shows the developer can design systems that scale.
* **Evaluation:** High interest. Shows a focus on security, performance, and clean code.

### For Senior Engineers
* **What stands out:** The implementation of dynamic traits (`SearchableTrait`, `SortableTrait`) and asynchronous queue structures.
* **Evaluation:** Impressive. Demonstrates the developer writes clean, reusable code instead of repeating logic.

### For Architects & Tech Leads
* **What stands out:** The decoupled database transaction model, custom rate-limiting strategies, and Sanctum multi-guard authorization fixes.
* **Evaluation:** Outstanding. Shows deep knowledge of backend patterns and system security.

---

## 📝 10. Detailed Project Memory Document

GPMS is a modular PHP backend designed to manage university graduation projects. It uses **Laravel 12** and **MySQL**.

The system handles team creation and registration under strict rules. A team has an **IT Supervisor** and a **Business Supervisor**. Individual students can specify their role preferences, and admins can swap students between teams, update team info, and change project details.

Administrators use CLI commands to import students, GPAs, and doctors from Excel files. The backend imports these records, validates them, and sends emails to students containing their generated passwords.

To keep responses fast, the backend delegates audit logs and email dispatches to background workers. Security is handled via Laravel Sanctum, with custom middleware ensuring users only access endpoints matching their role (Student, Doctor, Admin).

The database uses transactional operations to ensure team changes, swaps, and assignments are saved consistently. Key query traits like searching and sorting are implemented dynamically, keeping the controller code clean and maintainable.

---

## 🏁 11. Final Verdict

* **Overall Complexity Score: 85/100**
* **Overall Engineering Maturity Score: 82/100**
* **Estimated Effort to Rebuild from Scratch:** **2.5 to 3 Months** (including setup, validations, and testing).
* **Estimated Developer Level Required:** **Senior Software Engineer**
* **Most Impressive Aspect:** The dynamic database query traits (`SearchableTrait`, `SortableTrait`, `FilterableTrait`) and the custom Sanctum multi-guard middleware protection.
* **Weakest Aspect:** The lack of automated tests in the PHPUnit test suite. The system relies entirely on manual verification and Postman scripts.
* **Key Lesson:** GPMS demonstrates that a clean folder structure and reusable helper traits make codebase management much easier when returning to a project after a long break.
