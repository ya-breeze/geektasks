# Todo Task Management Application — Requirements

A lightweight, multi-user, web-based todo app with time tracking and mobile synchronization. Stack: Go (Golang) backend, GORM ORM, SQLite DB (default), SSR templates with Alpine.js for interactivity. Mobile app is out-of-repo but will sync via the API defined here.

---

## 1) User Stories

Authentication and Accounts
- As an app admin, I want to precreate user accounts with passwords so that users can sign in without public registration.
- As a user, I want to sign in with my email and password so that I can securely access my tasks.
- As a user, I want to sign out so that my session ends on shared devices.

Tasks, Tags, and Lists
- As a user, I want to create tasks with a title and optional description so that I can track work items.
- As a user, I want to edit a task’s title, description, due date, status, tags, and list so that I can maintain accurate details.
- As a user, I want to delete tasks so that I can keep my list clean.
- As a user, I want to tag tasks and filter by tags so that I can organize my work.
- As a user, I want to create, rename, and delete custom lists so that I can organize tasks by context or project.
- As a user, I want to select a list when creating/editing a task so that each task belongs to exactly one list.
- As a user, I want a list sidebar to filter tasks by list and see task counts per list so that navigation is efficient.
- As a user, I want new tasks to go to an Inbox by default when I don’t choose a list so that they aren’t lost.
- As a user, I want well-known lists (Inbox, Today, This Week) visible in the UI so that I can quickly find what matters.

Time Tracking
- As a user, I want to start, pause, and stop a task timer so that I can track time spent.
- As a user, I want only one active timer at a time so that time isn’t double-counted.
- As a user, I want to edit or add manual time entries so that I can fix mistakes.
- As a user, I want to see summaries of time by task and by day/week/month so that I understand where my time goes.

Synchronization (Mobile)
- As a mobile user, I want to sync my tasks, tags, and time entries so that my data is consistent across devices.
- As a mobile user, I want incremental sync since the last time so that sync is fast.
- As a mobile user, I want last-write-wins conflict resolution so that I don’t lose changes.
- As a mobile user, I want deletes to be respected and fully removed after 30 days so that data stays tidy.

Non-goals
- No task sharing/collaboration; every user’s tasks are private.
- No comments on tasks.

---

## 2) Product Requirements Document (PRD)

### 2.1 Scope
- In scope: Task CRUD, tags, time tracking (start/pause/stop, manual entries), user sign-in, admin precreation of accounts (no public registration UI), JWT-based auth, mobile sync via REST, basic time reports, English-only UI.
- Out of scope: Task sharing, comments, accessibility requirements, admin web UI, mobile client implementation.

### 2.2 Functional Requirements

Tasks
- CRUD tasks with fields: id, user_id, list_id (nullable = Inbox), title (required), description (optional), status (todo|in-progress|done), due_date (optional), archived_at (nullable), created_at, updated_at, tags (N:M via Tag/TaskTag).
- Soft-delete behavior for UI archive is optional; explicit Delete removes task and creates a tombstone for sync; permanent removal after 30 days.

Tags
- CRUD tags (name unique per user). Assign/unassign tags to tasks. Filter by one or more tags.

Lists
- CRUD lists with fields: id, user_id, name (required), color (optional), created_at, updated_at.
- Tasks must belong to exactly one list conceptually. Inbox is a virtual list represented by tasks with list_id = NULL.
- Deleting a custom list reassigns its tasks to Inbox (sets list_id = NULL).
- Well-known virtual lists (no database rows):
  - Inbox: tasks where list_id is NULL (default for new tasks if list not specified)
  - Today: tasks due today
  - This Week: tasks due within the current week

Time Tracking
- One active timer per user at a time; starting a timer on task A pauses any running timer.
- TimeEntry records (id, task_id, user_id, started_at, ended_at, duration_sec derived; created_at, updated_at).
- Manual add/edit/delete of time entries allowed (audit via updated_at).

Reports
- Time summaries grouped by task or by tags, within a date range. CSV export endpoint.

Authentication & Authorization
- Users precreated by admin (CLI or seed). No public registration endpoint in web UI.
- JWT-based authentication for both web and mobile. Web stores JWT in httpOnly, secure cookie.
- Authorization: users can only access their own resources (tasks, tags, time entries). No sharing.
- Password reset: admin-only mechanism (e.g., CLI) in MVP.

Synchronization
- Incremental pull by updated_at and push of created/updated/deleted items.
- Conflict resolution: last-write-wins based on updated_at (server authoritative timestamping on write).
- Deletes: create tombstone records for 30 days to propagate delete to clients; hard delete after 30 days.
- Devices tracked minimally for last_sync_at.

### 2.3 User Interface (SSR + Alpine.js)

Pages
- Sign In page.
- Task List: list sidebar navigation; filter by list, status, and tags; quick actions (start/pause timer, mark done); show per-list task counts.
- Task Detail: title/description/status/tags/list, timer controls, time entries list (with add/edit/delete), due date.
- Lists: management page to create/rename/delete custom lists (virtual lists are not deletable and have no edit UI).
- Reports: date range, group by task/day/month, CSV export.
- Settings: profile (name, timezone), manage API token display if needed.

Alpine.js components
- x-task-list: filters, sorting, inline status changes.
- x-task-item: start/pause timer, tag chips.
- x-timer: shows active task and elapsed time, updates via setInterval, debounced writes.
- x-report: fetch JSON report and render table.

### 2.4 Data Models and Relationships

- User 1—N List; List 1—N Task
- Task N—M Tag via TaskTag
- Task 1—N TimeEntry; User 1—N TimeEntry
- User 1—N Device (for sync cursor)
- Tombstone records track deletions for 30 days

Key fields (high-level)
- User: id (UUID), email (unique), password_hash (bcrypt), name, timezone, created_at, updated_at
- List: id (UUID), user_id, name, color (nullable), created_at, updated_at
- Task: id (UUID), user_id, list_id (nullable = Inbox), title, description, status, due_date (nullable), archived_at (nullable), created_at, updated_at
- Tag: id (UUID), user_id, name (unique per user), created_at, updated_at
- TaskTag: task_id, tag_id (composite PK)
- TimeEntry: id (UUID), task_id, user_id, started_at, ended_at (nullable), duration_sec (derived), created_at, updated_at
- Device: id (UUID), user_id, name, last_sync_at, created_at, updated_at
- Tombstone: id (UUID), user_id, entity_type (list|task|tag|time_entry), entity_id, deleted_at

### 2.5 API Requirements (Mobile Synchronization)

- Auth: JWT Bearer tokens (v1 prefix). HTTPS required.
- Pull-only changed resources since timestamp; include tombstones.
- Push includes arrays of upserts and deletes with client-generated UUIDs; server idempotency via Idempotency-Key header.
- Include lists in pull/push sets (custom lists only; virtual lists are UI-only and not part of API payloads).
- LWW conflict rule on updated_at.

### 2.6 Non-Functional Requirements

- Security: HTTP, HTTPS by proxy, bcrypt password hashing, CSRF protection for SSR forms.
- Reliability: SQLite in WAL mode; graceful shutdown.
- Maintainability: single-binary deployment; configuration via env vars.
- Internationalization: English only.
- Data retention: hard delete 30 days after deletion (tombstones purged).

---

## 3) Technical Implementation Details

### 3.1 Architecture
- Monolithic Go server.
- SSR with html/template; static assets; Alpine.js for interactivity.
- JSON REST API under /api/v1 for mobile and AJAX.
- GORM with SQLite (default). Foreign keys ON; WAL mode.
- Auth: JWT issuance on login; web stores JWT in secure cookie; API uses Authorization: Bearer.

### 3.2 Database (GORM Conventions)
- UUID primary keys (string) for sync friendliness.
- AutoMigrate all models on startup.
- PRAGMAs: journal_mode=WAL; foreign_keys=ON.
- Foreign key: tasks.list_id → lists.id ON DELETE SET NULL (deleting a custom list moves tasks to Inbox).
- Tombstones table to track deletions; daily cleanup job to hard-delete rows older than 30 days and purge associated data.

### 3.3 Endpoints Summary
- Auth: POST /api/v1/auth/login, POST /api/v1/auth/logout (web cookie clear)
- Lists: GET/POST /api/v1/lists; PATCH/DELETE /api/v1/lists/{id}
- Tasks: GET/POST /api/v1/tasks (filter by listId optional); GET/PATCH/DELETE /api/v1/tasks/{id}
- Timer: POST /api/v1/tasks/{id}/timer/{start|pause|stop}
- Time Entries: GET /api/v1/time-entries (filter by taskId optional); POST /api/v1/time-entries; PATCH/DELETE /api/v1/time-entries/{id}
- Tags: GET/POST /api/v1/tags; PATCH/DELETE /api/v1/tags/{id}; POST/DELETE /api/v1/tasks/{id}/tags/{tagId}
- Sync: GET /api/v1/sync/pull?since=ts; POST /api/v1/sync/push
- Reports: GET /api/v1/reports/time; GET /api/v1/reports/time/export (CSV)

### 3.4 Frontend (SSR + Alpine.js)
- Templates: layout.tmpl, auth/login.tmpl, tasks/index.tmpl, tasks/show.tmpl, lists/index.tmpl, reports/index.tmpl, settings/index.tmpl.
- List sidebar/navigation component (x-list-nav) shows custom lists plus virtual lists (Inbox, Today, This Week) with task counts.
- Progressive enhancement: SSR forms with data-action for Alpine to intercept; server handles non-JS fallback.
- CSRF: hidden input in forms; X-CSRF-Token header for fetch.

### 3.5 Sync Protocol
- Client-generated UUIDs on create; server preserves IDs.
- RFC3339 timestamps; server returns server_time for skew handling.
- Pull returns changed entities (lists, tasks, tags, time_entries) and tombstones since `since`.
- Push is idempotent via Idempotency-Key; per-item status in response.
- LWW: compare updated_at; server sets authoritative updated_at on successful write.
- Deletes: create tombstone (including entity_type 'list'); included in pull for 30 days; then hard delete.

### 3.6 Deployment & Configuration
- Single Go binary. Static files embedded via `embed` (optional).
- Env vars: PORT (8080), DATABASE_URL (sqlite file path), SESSION_SECRET, JWT_SECRET, GORM_LOG_LEVEL, CSRF_ENABLED (true/false).
- Run behind reverse proxy (TLS termination) or enable TLS directly.
- Backups: periodic snapshot of SQLite DB file.

### 3.7 Development Setup
- Requirements: Go 1.25+, SQLite3 CLI (optional), Make.
- Steps: copy .env.example → .env; set SESSION_SECRET and JWT_SECRET; `go run ./cmd/server`.
- Integration tests with temp SQLite file.

### 3.8 Data Retention & Cleanup
- Nightly job: purge tombstones older than 30 days; cascade hard delete corresponding rows if still present.
- Ensure sync clients that have not pulled within 30 days may not receive delete notifications (documented limitation).

---

## 4) OpenAPI Specification (v1)

```yaml
openapi: 3.0.3
info:
  title: Todo+Time API
  version: 1.0.0
servers:
  - url: https://api.example.com/api/v1
security:
  - bearerAuth: []
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    UUID:
      type: string
      format: uuid
    Timestamp:
      type: string
      format: date-time
    LoginRequest:
      type: object
      required: [email, password]
      properties:
        email: { type: string, format: email }
        password: { type: string, format: password }
    LoginResponse:
      type: object
      required: [token]
      properties:
        token: { type: string }
    Task:
      type: object
      required: [id, title, status, created_at, updated_at]
      properties:
        id: { $ref: '#/components/schemas/UUID' }
        list_id: { $ref: '#/components/schemas/UUID', nullable: true, description: 'NULL = Inbox (virtual list)' }
        title: { type: string }
        description: { type: string, nullable: true }
        status: { type: string, enum: [todo, in-progress, done] }
        due_date: { $ref: '#/components/schemas/Timestamp', nullable: true }
        archived_at: { $ref: '#/components/schemas/Timestamp', nullable: true }
        tags:
          type: array
          items: { $ref: '#/components/schemas/Tag' }
        created_at: { $ref: '#/components/schemas/Timestamp' }
        updated_at: { $ref: '#/components/schemas/Timestamp' }
    List:
      type: object
      required: [id, name, created_at, updated_at]
      properties:
        id: { $ref: '#/components/schemas/UUID' }
        name: { type: string }
        color: { type: string, nullable: true }
        created_at: { $ref: '#/components/schemas/Timestamp' }
        updated_at: { $ref: '#/components/schemas/Timestamp' }
    Tag:
      type: object
      required: [id, name, created_at, updated_at]
      properties:
        id: { $ref: '#/components/schemas/UUID' }
        name: { type: string }
        created_at: { $ref: '#/components/schemas/Timestamp' }
        updated_at: { $ref: '#/components/schemas/Timestamp' }
    TimeEntry:
      type: object
      required: [id, task_id, started_at, created_at, updated_at]
      properties:
        id: { $ref: '#/components/schemas/UUID' }
        task_id: { $ref: '#/components/schemas/UUID' }
        started_at: { $ref: '#/components/schemas/Timestamp' }
        ended_at: { $ref: '#/components/schemas/Timestamp', nullable: true }
        duration_sec: { type: integer, format: int64, nullable: true }
        created_at: { $ref: '#/components/schemas/Timestamp' }
        updated_at: { $ref: '#/components/schemas/Timestamp' }
    Tombstone:
      type: object
      required: [id, entity_type, entity_id, deleted_at]
      properties:
        id: { $ref: '#/components/schemas/UUID' }
        entity_type: { type: string, enum: [list, task, tag, time_entry] }
        entity_id: { $ref: '#/components/schemas/UUID' }
        deleted_at: { $ref: '#/components/schemas/Timestamp' }
    SyncPullResponse:
      type: object
      properties:
        server_time: { $ref: '#/components/schemas/Timestamp' }
        lists:
          type: array
          items: { $ref: '#/components/schemas/List' }
        tasks:
          type: array
          items: { $ref: '#/components/schemas/Task' }
        tags:
          type: array
          items: { $ref: '#/components/schemas/Tag' }
        time_entries:
          type: array
          items: { $ref: '#/components/schemas/TimeEntry' }
        tombstones:
          type: array
          items: { $ref: '#/components/schemas/Tombstone' }
    SyncPushRequest:
      type: object
      properties:
        lists: { type: array, items: { $ref: '#/components/schemas/List' } }
        tasks: { type: array, items: { $ref: '#/components/schemas/Task' } }
        tags: { type: array, items: { $ref: '#/components/schemas/Tag' } }
        time_entries: { type: array, items: { $ref: '#/components/schemas/TimeEntry' } }
        deletes:
          type: array
          items: { $ref: '#/components/schemas/Tombstone' }
paths:
  /auth/login:
    post:
      summary: Login and obtain JWT
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/LoginRequest' }
      responses:
        '200':
          description: JWT issued
          content:
            application/json:
              schema: { $ref: '#/components/schemas/LoginResponse' }
        '401': { description: Invalid credentials }
  /tasks:
    get:
      summary: List tasks
      parameters:
        - in: query
          name: listId
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: array
                items: { $ref: '#/components/schemas/Task' }
    post:
      summary: Create task
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/Task' }
      responses:
        '201': { description: Created }
  /tasks/{id}:
    get:
      summary: Get task
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Task' }
        '404': { description: Not found }
    patch:
      summary: Update task
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/Task' }
      responses:
        '200': { description: Updated }
    delete:
      summary: Delete task (create tombstone)
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '204': { description: Deleted }
  /tasks/{id}/timer/start:
    post:
      summary: Start timer for task
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '200': { description: Started }
  /tasks/{id}/timer/pause:
    post:
      summary: Pause active timer for task
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '200': { description: Paused }
  /tasks/{id}/timer/stop:
    post:
      summary: Stop active timer for task
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '200': { description: Stopped }
  /time-entries:
    get:
      summary: List time entries
      parameters:
        - in: query
          name: taskId
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: array
                items: { $ref: '#/components/schemas/TimeEntry' }
    post:
      summary: Create time entry (manual)
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/TimeEntry' }
      responses:
        '201': { description: Created }
  /time-entries/{id}:
    patch:
      summary: Update time entry
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/TimeEntry' }
      responses:
        '200': { description: Updated }
    delete:
      summary: Delete time entry (create tombstone)
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '204': { description: Deleted }
  /tags:
    get:
      summary: List tags
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: array
                items: { $ref: '#/components/schemas/Tag' }
    post:
      summary: Create tag
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/Tag' }
      responses:
        '201': { description: Created }
  /tags/{id}:
    patch:
      summary: Update tag
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/Tag' }
      responses:
        '200': { description: Updated }
    delete:
      summary: Delete tag (create tombstone)
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '204': { description: Deleted }
  /lists:
    get:
      summary: List lists
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: array
                items: { $ref: '#/components/schemas/List' }
    post:
      summary: Create list
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/List' }
      responses:
        '201': { description: Created }
  /lists/{id}:
    patch:
      summary: Update list
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/List' }
      responses:
        '200': { description: Updated }
    delete:
      summary: Delete list (create tombstone)
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '204': { description: Deleted }
  /tasks/{id}/tags/{tagId}:
    post:
      summary: Attach tag to task
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
        - in: path
          name: tagId
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '204': { description: Attached }
    delete:
      summary: Detach tag from task
      parameters:
        - in: path
          name: id
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
        - in: path
          name: tagId
          required: true
          schema: { $ref: '#/components/schemas/UUID' }
      responses:
        '204': { description: Detached }
  /sync/pull:
    get:
      summary: Pull changes since timestamp
      parameters:
        - in: query
          name: since
          required: true
          schema: { $ref: '#/components/schemas/Timestamp' }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema: { $ref: '#/components/schemas/SyncPullResponse' }
  /sync/push:
    post:
      summary: Push changes (idempotent)
      parameters:
        - in: header
          name: Idempotency-Key
          required: false
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/SyncPushRequest' }
      responses:
        '200': { description: Processed }
  /reports/time:
    get:
      summary: Time report
      parameters:
        - in: query
          name: from
          required: true
          schema: { $ref: '#/components/schemas/Timestamp' }
        - in: query
          name: to
          required: true
          schema: { $ref: '#/components/schemas/Timestamp' }
        - in: query
          name: groupBy
          schema: { type: string, enum: [task, day, month] }
      responses:
        '200': { description: OK }
```

---

## 5) Notes
- API base path uses 'v1' prefix. Future versions can coexist under /api/v2.
- Mobile client is separate; this repo provides the server API and web app only.
- Admin operations (user creation/reset) are via CLI or one-off scripts; no admin UI included in MVP.



## 6) Implementation Plan (MVP-first milestones)

Each milestone builds on the previous, adds one major capability, and produces a deployable, usable application.

Step 1: Foundations + Authentication
- New functionality:
  - Project skeleton (monolithic Go server) using standard net/http, chi router, and html/template; base middleware (request logging, recovery, CSRF)
  - SQLite with GORM (WAL mode) and AutoMigrate for initial schema
  - Auth: login/logout with JWT (Bearer for API, cookie for web), password hashing with bcrypt; basic rate limiting on auth routes
  - Extract OpenAPI spec from this document to a standalone openapi.yaml at repo root (kept as the single source of truth)
  - Project structure initialized with specified libraries and go.mod pinned to major versions; Makefile/scripts for build, test, lint; dev tooling configured (golangci-lint, goimports, oapi-codegen)
  - Admin-only CLI to precreate users
- User can: Sign in and out; see an empty home/dashboard page.
- Value: Establishes a secure, tooled baseline so subsequent features are gated, testable, and deployable (served over HTTP behind an HTTPS reverse proxy).

Step 2: Basic Tasks (CRUD)
- New functionality: Task model and SSR UI for create/read/update/delete; statuses; due date; basic validations; list view with pagination.
- User can: Maintain a personal list of tasks; mark tasks done; edit details.
- Value: Delivers the core todo utility so the app is already useful.

Step 3: Tags
- New functionality: Tag entity, assign/unassign tags to tasks, filter list by tags in the UI; tag CRUD.
- User can: Organize tasks by tags and quickly filter by context.
- Value: Improves organization and prioritization without adding heavy complexity.

Step 4: Time Tracking (Timers + Manual Entries)
- New functionality: Start/pause/stop a single active timer per user; automatic time entry creation; manual add/edit/delete of time entries; display time totals on task detail.
- User can: Track time spent on tasks via a live timer or by entering it manually.
- Value: Introduces the time-tracking value proposition, enabling productivity insights.

Step 5: Reporting (Task/Tag)
- New functionality: Reports page with date-range filter; summaries grouped by task or by tags; CSV export endpoint.
- User can: Review and export how time is distributed across tasks and tags.
- Value: Turns raw time data into actionable insights/shareable outputs.

Step 6: Public JSON API (Foundations for Mobile)
- New functionality: JWT-protected JSON endpoints for auth session check, tasks, tags, and time entries (CRUD), matching the OpenAPI schemas; idempotency for creates via client-supplied UUIDs.
- User can: Integrate with lightweight scripts or prototype mobile clients against stable endpoints.
- Value: Enables programmatic access and sets the stage for synchronization.

Step 7: Mobile Sync v1 (Pull)
- New functionality: Incremental pull endpoint returning changed tasks, tags, time entries, and tombstones since a timestamp; server_time for clock skew handling.
- User can: Use a mobile client to download and refresh local data efficiently.
- Value: Provides read-side mobility and offline catch-up with minimal risk.

Step 8: Mobile Sync v2 (Push)
- New functionality: Push endpoint accepting upserts and deletes; Idempotency-Key support; last-write-wins conflict resolution by updated_at; per-item status responses.
- User can: Make changes on mobile and sync them back to the server safely.
- Value: Completes two-way sync for core entities, enabling full cross-device workflows.

Step 9: Lists (Custom + Virtual)
- New functionality: List entity (CRUD) and UI (sidebar, list management page, list selection on task forms); virtual lists in UI: Inbox (list_id = NULL), Today (due today), This Week (due within current week); deleting a custom list reassigns its tasks to Inbox.
- User can: Organize tasks into lists, navigate by list, and see per-list counts including virtual lists.
- Value: Adds higher-level organization and quick navigation views without complicating the data model.

Step 10: Sync Lists
- New functionality: Include lists in sync pull/push payloads and tombstones; task list_id synchronized; virtual lists remain UI-only.
- User can: Manage lists on one device and see them reflected everywhere.
- Value: Unifies organization across devices to complete the lists feature.

Deployment posture for all steps
- Serve the app over HTTP behind an HTTPS reverse proxy; use environment-based configuration; keep SQLite in WAL mode; ship each step as a single binary plus assets.


## 7) Implementation Best Practices

Architecture & Design Patterns
- Enforce clear separation of concerns:
  - Handlers (HTTP/SSR/API) handle routing, request/response shaping, validation, and mapping to services
  - Services encapsulate business logic and orchestration
  - Repositories (data access) isolate GORM/SQL details and return domain models
- Define interfaces for major components (AuthService, TaskService, SyncService, ListRepository, TaskRepository, etc.) to allow mocking in tests and future swaps
- Use lightweight dependency injection (constructor injection) to pass interfaces into handlers/services
- Follow SOLID principles, especially:
  - Single Responsibility: each type/module has one reason to change
  - Dependency Inversion: depend on abstractions (interfaces), not concretions
- Keep DTOs separate from DB models when API/SSR payloads differ; map explicitly

Testing Strategy
- Aim for high coverage on business logic (services); write table-driven unit tests with clear arrange/act/assert
- Integration tests for repositories using temporary SQLite files (in WAL mode) and AutoMigrate per test suite
- API tests: success and error cases (validation errors, not found, authorization failures, conflict), including CSRF on SSR form posts
- Authentication/Authorization tests: JWT validation (expiry, signature, audience), cookie handling, route guards by user ownership
- Sync tests: pull/push flows, idempotency with repeated requests, last-write-wins conflict scenarios, and tombstone propagation/purge after 30 days
- Timer logic tests: ensure only one active timer per user; verify correct durations across pause/stop operations

Code Quality & Maintainability
- Adhere to Go idioms and consistent naming (mixedCaps for exported names, error values end with Err prefix variable names, etc.)
- Centralize error handling; wrap errors with context; use structured logging (e.g., log/slog) with request IDs and user IDs where applicable
- Validate inputs at API boundaries (use request DTOs and explicit validation rules) and on SSR forms (server-side enforcement)
- Use database transactions for multi-step operations (e.g., timer state transitions, sync batch upserts/deletes)
- Implement graceful shutdown (context cancellation, HTTP server Shutdown, draining in-flight requests, closing DB)
- Propagate context.Context through layers for timeouts, cancellation, and request-scoped values

Security Considerations
- Validate and sanitize all inputs; reject unexpected fields; enforce length and format constraints
- Use parameterized queries via GORM/query builder; never build SQL with string concatenation
- Apply rate limiting on authentication endpoints and sync push endpoints
- JWT: verify signature, issuer/audience (if used), expiry/nbf; rotate secrets safely; short-lived tokens recommended
- Web sessions: use httpOnly cookies; set Secure when served behind HTTPS proxy; set SameSite=Lax (or Strict where feasible)
- CSRF: include tokens on state-changing SSR requests; verify on server

Performance & Scalability
- Add indexes for common queries:
  - tasks: (user_id), (list_id), (due_date), (status), (updated_at)
  - tags: (user_id, name UNIQUE)
  - task_tags: (task_id, tag_id) composite primary key; indexes on (task_id) and (tag_id)
  - time_entries: (user_id), (task_id), (started_at), (updated_at)
  - lists: (user_id, name UNIQUE)


## 8) Technology Stack & Dependencies

Core Framework & HTTP
- Web server: standard library net/http
- Router: github.com/go-chi/chi/v5 (v5.x)
- Templates: standard library html/template

Database & ORM
- ORM: gorm.io/gorm (v1.25.x)
- SQLite driver: gorm.io/driver/sqlite (v1.5.x)
- Migrations: Start with GORM AutoMigrate for MVP; for versioned changes adopt github.com/golang-migrate/migrate/v4 (v4.x) with SQL migrations. Keep AutoMigrate limited to additive changes.

Authentication & Security
- JWT: github.com/golang-jwt/jwt/v5 (v5.x)
- Password hashing: golang.org/x/crypto/bcrypt (module golang.org/x/crypto v0.x)
- CSRF: github.com/go-chi/csrf (v2.x) middleware for SSR and cookie-based sessions
- Rate limiting: github.com/go-chi/httprate (v0.x) for per-route limits (e.g., /auth/* and /sync/push)

Validation & Serialization
- Input validation: github.com/go-playground/validator/v10 (v10.x)
- JSON: standard library encoding/json (optionally replace with github.com/goccy/go-json for faster JSON in the future; not required for MVP)

Logging & Configuration
- Logging: standard library log/slog (Go 1.25+). Optional pretty dev handler: github.com/lmittmann/tint (v1.x)
- Configuration: github.com/caarlos0/env/v10 (v10.x) to load and validate environment variables into typed structs

Testing
- Unit/integration testing: standard testing package
- Assertions/mocks: github.com/stretchr/testify (v1.9.x) for assert/require
- HTTP testing: net/http/httptest for handlers; use temporary SQLite database files (os.CreateTemp) per test suite; run GORM AutoMigrate in test setup

Development Tools
- OpenAPI codegen (optional): github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen (v2.x) for generating types/clients/servers from openapi.yaml
- Linting: github.com/golangci/golangci-lint/cmd/golangci-lint (v1.60.x)
- Formatting: gofmt (built-in) and golang.org/x/tools/cmd/goimports (v0.x)
- Debugging: github.com/go-delve/delve/cmd/dlv (latest compatible)

Versioning guidance
- Pin to the indicated major version lines (e.g., chi v5.x, jwt v5.x) and update conservatively. Use go.mod to record exact versions; consider a tools.go file to pin CLI tools.

- Implement pagination on list endpoints (tasks, tags, time entries, lists) with stable sort (e.g., created_at desc, id tie-breaker)
- Monitor and log basic performance metrics (request latency, error rates, DB timings); enable pprof only in development

## 9) Recommended Project Layout & Tooling

Project layout
```
.
├── cmd/
│   └── server/                 # main entrypoint (main.go)
├── internal/                   # app-internal code (not importable by others)
│   ├── api/                    # OpenAPI types/handlers (generated + adapters)
│   ├── http/                   # HTTP wiring (routes, middleware, SSR handlers)
│   ├── services/               # business logic (AuthService, TaskService, ...)
│   ├── repository/             # GORM repositories (TaskRepository, ...)
│   ├── models/                 # domain + GORM models
│   ├── auth/                   # JWT, password hashing, CSRF helpers
│   ├── sync/                   # mobile sync orchestration (pull/push)
│   ├── config/                 # config structs and loader (caarlos0/env)
│   └── logging/                # slog setup
├── pkg/                        # optional reusable utilities (if any)
├── web/
│   ├── templates/              # html/template files (SSR)
│   └── static/                 # static assets (css/js/images); Alpine.js
├── migrations/                 # SQL migrations (golang-migrate)
├── openapi.yaml                # extracted OpenAPI spec (single source of truth)
├── Makefile                    # dev workflows (build/test/lint/generate)
├── tools.go                    # pins CLI tools in go.mod
├── go.mod
└── go.sum
```

Notes
- Keep internal/ as the main workspace and avoid circular deps by layering: http -> services -> repository.
- api/ may contain generated code (oapi-codegen) and thin adapters to services.
- models/ holds DB models and domain types; keep API DTOs in api/ to avoid leaking DB details.
- migrations/ contains idempotent, forward-only SQL files; use AutoMigrate only for additive early changes.
