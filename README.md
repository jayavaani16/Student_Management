# Student Management System

Assessment implementation for student records, class/section master data, authentication, and audit history. The repository contains a Spring Boot 4 / Java 21 API, a React/Vite UI, and PostgreSQL DDL.

## Requirements and implementation map

| Assessment requirement | Implementation |
|---|---|
| Login, field validation, authentication, protected screens | `POST /api/auth/login`, BCrypt password hashes, signed short-lived JWTs, React protected routes |
| Student add/view/edit/deactivate | Student list, form, profile, and REST endpoints; DELETE performs soft deactivation |
| Search, class/grade, section, status, admission-date filters | Server-side query parameters on `GET /api/students` |
| Sort and pagination; never load the full student table in the browser | Allow-listed server sorting, zero-based pages, page-size cap of 100; UI requests one page at a time |
| Class/section master data | Admin create/update; authenticated users can retrieve active records |
| Server and UI validation | Jakarta Bean Validation, date/business checks, database constraints, browser form validation |
| Normalized schema, constraints, indexes | `db/schema.sql` |
| Consistent errors and sensitive-information protection | `ApiResponse`, `ApiErrorResponse`, centralized exception advice, generic server errors |
| ID-based record authorization | ADMIN can access school records; STAFF can access only records they own, enforced in the service for list and ID endpoints |
| Audit of login/create/update/deactivate | `audit_logs`; student mutation and audit write share a transaction |
| Concurrency | JPA `@Version`, required version on updates, HTTP 409 for stale writes |
| Transactional consistency | Student and audit writes use the same database transaction |
| Duplicate submission | UI disables submit while pending; normalized unique student ID/email constraints are the final duplicate guard |
| API, architecture, setup, design decisions | `docs/API.md`, `docs/architecture.md`, this README |

## Prerequisites

- Java 21
- PostgreSQL 14 or later; the `pg_trgm` extension must be available to the database user
- Node.js 20.19+ or 22.12+
- PowerShell on Windows (commands below use PowerShell syntax)

## Database setup

Create the database, then execute the schema script from the repository root:

```powershell
createdb -U postgres student_management_db
psql -U postgres -d student_management_db -f .\db\schema.sql
```

The script creates `users`, `students`, `class_sections`, and `audit_logs`, adds constraints and search indexes, and inserts a few class/section examples. It does not create an administrator with a default password.

## Configuration and run

Set environment variables in the same PowerShell window used to run the API. Supply your PostgreSQL password and a random JWT signing secret of at least 32 characters. To bootstrap a local administrator on first startup, provide an email and a password of at least 12 characters. The password is BCrypt-hashed before storage; an existing account is never overwritten.

```powershell
$env:DB_URL = "jdbc:postgresql://localhost:5432/student_management_db"
$env:DB_USERNAME = "postgres"
$env:DB_PASSWORD = "<your-local-database-password>"
$env:JWT_SECRET = "<random-secret-at-least-32-characters>"
$env:APP_ADMIN_EMAIL = "admin@example.test"
$env:APP_ADMIN_PASSWORD = "<local-only-password-at-least-12-characters>"
$env:CORS_ALLOWED_ORIGIN = "http://localhost:5173"
.\mvnw.cmd spring-boot:run
```

Set the administrator credentials only for local development; use a secret manager and a managed identity/rotation process in a deployed environment. Do not commit `.env` files or credentials. If no bootstrap email and password are set, create an administrator through your organization's secure database provisioning process.

In another PowerShell window:

```powershell
Set-Location .\student-management-ui
Copy-Item .\.env.example .\.env
npm install
npm run dev
```

The UI defaults to `http://localhost:8080/api`. Override `VITE_API_URL` in the UI `.env` when the API runs elsewhere. Open the Vite URL and sign in with the administrator values configured for the API.

## Verification commands

```powershell
# Backend compile and tests
.\mvnw.cmd test

# Frontend production build and dependency audit
Set-Location .\student-management-ui
npm run build
npm audit
```

The backend context test uses an isolated in-memory H2 database. Production runs use PostgreSQL and the environment variables above.

## Design and operational notes

### Architecture choice

The UI, REST API, business services, repositories, and relational database are separated. DTOs prevent direct entity serialization, services own validation/authorization/transactions, and controllers expose HTTP semantics. React's page-level data requests make loading and failure states explicit.

### Data model and indexes

Users hold BCrypt hashes and roles. Students reference one class/section and one owner user. Class/section pairs are unique; student IDs and emails are unique. UUIDs are used as external identifiers. Search/filter indexes cover class/status/admission date, owner access, admission sorting, and trigram indexes for case-insensitive substring name/email search. The trigram extension/indexes are PostgreSQL-specific.

### Search, pagination, and scale

The browser sends one page (default 20, maximum 100). The API performs predicates, sort, `LIMIT`/`OFFSET`, and a count query in PostgreSQL; it never serializes all students. Search values are bound query parameters, not concatenated SQL. For multi-million-row deployments, use `EXPLAIN (ANALYZE, BUFFERS)`, monitor count-query cost, table/index bloat and connection-pool wait, keep page sizes bounded, and consider cursor/keyset pagination for deep navigation if the product can move away from arbitrary page numbers. Partitioning, read replicas, and a dedicated search index are later options, not assumed in this starter.

### Authentication and authorization

The login endpoint compares BCrypt hashes and returns a short-lived HMAC-SHA256 signed bearer token. All other API calls require the token. Roles are `ADMIN` and `STAFF`; class-section writes are admin-only. Student list and read/write operations enforce ownership in the service: an ordinary staff member cannot access a different owner's student by changing a UUID in the URL. This is authorization, not just hiding links in the UI.

### SQL injection and errors

JPA Specifications and repository methods parameterize values. Sort fields are explicitly allow-listed. Bean validation and service checks reject invalid dates, unknown/inactive sections, and duplicates; database constraints are the final guard. The exception advice returns safe, consistent messages and logs unexpected details only on the server. Do not enable SQL parameter logging with student data in production.

### Concurrency and transactions

Updates require the version read with the student. The optimistic `@Version` check rejects a stale save with HTTP 409; clients reload and merge the user's changes rather than silently overwriting. Student changes and audit events commit or roll back together. No student-document table or file upload is defined by this assessment; if one is added, create its record in the same transaction, store binary content in object storage, and use an outbox/compensation strategy if the external storage step cannot participate in the database transaction.

### Duplicate submissions

The UI disables its submit button while an operation is pending. A repeated student payload is additionally blocked by normalized, unique student ID and email constraints. If future create operations trigger non-idempotent external effects, add a persisted `Idempotency-Key` contract with request hashing and replayed responses; UI locking alone is not a server-side guarantee.

### Slow API investigation

Correlate browser timings and request IDs with API latency, structured logs, JVM/GC and pool metrics; split network, API/service, SQL execution, lock wait, and downstream time. Capture database `EXPLAIN ANALYZE` plans and check index use before changing queries. Reproduce with representative data and compare before/after metrics; do not infer database slowness from the dashboard's total elapsed time.

### Scaling and production follow-ups

Run stateless API instances behind a load balancer, centralize secrets, configure TLS and rate limiting, add login throttling, metrics/tracing, backups and restore drills, retention for audit logs, migrations, CI checks, and integration tests against PostgreSQL. Consider keyset pagination for deep pages, query/read replicas, and tenant-aware authorization if deployment spans multiple schools.

## Known limitations

- User provisioning and password reset are administrative database workflows; no public registration or role-management API is exposed.
- The application has record-owner authorization for STAFF and school-wide ADMIN access; it does not model student-specific guardians or per-field disclosure rules.
- The assessment mentions a student-document step as a transaction scenario but does not define document fields or upload requirements; it is described above rather than inventing a document feature.
- Email notifications, advanced analytics, and production observability integrations are out of scope.
