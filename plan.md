# Comprehensive Development Plan: Production-Ready Wiki Website in Go

## Executive Summary

This plan provides a roadmap to build a production-ready, collaborative wiki using **Go 1.25** and modern best practices. The architecture is modular (hexagonal/clean) and container-first, with a clear separation between domain logic and infrastructure adapters.

### High-Level Architecture

- **Language:** Go 1.25 (released 2025-08-12; includes experimental JSON v2 for performance, experimental greentea GC, improved `net/http` pattern routing since Go 1.22, and `sync.WaitGroup.Go`)
- **Backend Framework:** Chi router built on standard library `net/http`. This aligns with “stdlib-first” guidance and 2025 trends (Chi ~12% usage; net/http is most common). Chi is lightweight, composable, and explicitly designed for use with `net/http` via `chi.NewRouter()`.
- **Frontend Options:**
  - **Option A (recommended as default):** Server-rendered with **templ** + **HTMX**. templ compiles to performant Go code and supports interop with `html/template`. HTMX enables dynamic interactions without a full SPA.
  - **Option B:** SPA with an API backend (e.g., React/Vue). Plan includes RESTful API contracts and endpoints for this path.
- **Database:** **PostgreSQL** (primary). Driver: **jackc/pgx v5** for performance and PostgreSQL-native features. Optional: SQLite for dev/local use.
- **Search:** **Bleve** (embedded, pure Go) for full-text search with fields, faceting, and BM25. Optional: external search services (e.g., Meilisearch) as a future path.
- **Authentication:** **go-chi/jwtauth v5** (uses `lestrrat-go/jwx`). Avoid archived `jwt-go`. Optional: OAuth/OIDC providers via go-pkgz/auth for extensibility.
- **Migration Tool:** **golang-migrate** (CLI + lib) for versioned migrations. **Atlas** is noted for improved error handling and schema-as-code; it can optionally be introduced later. Avoid overclaiming Atlas adoption.
- **Testing:** stdlib `testing` + **Testify** for assertions and mocking; **go test -race** and `testing/synctest` (Go 1.25). Use **testcontainers-go** for integration tests with real Postgres/Bleve.
- **Observability:** **OpenTelemetry Go** for traces/metrics/logs; **Prometheus** 3.0+ as a backend (native OTLP and delta temporality supported). Use `log/slog` (Go 1.21+) for structured logging.
- **Deployment:** Docker multi-stage builds targeting `scratch` or `distroless`; Kubernetes with RBAC/PSS/Network Policies; CI/CD via GitHub Actions.

### Technology Stack Justification

| Component | Recommended Tool | Evidence/Notes |
|-----------|----------------|-----------------|
| Web Framework | Chi (go-chi/chi v5) | Designed for `net/http`; idiomatic context use; composable middleware; widespread use (~12% in 2025 ecosystem per JetBrains). Aligns with “stdlib first” guidance. |
| Database Driver | jackc/pgx v5 | High performance, native PostgreSQL features, binary protocol, robust pooling (pgxpool). Benchmarks and guidance indicate significant speedups vs older drivers. |
| Templating | templ | Compiles to Go; type-safe; good developer tooling; 424+ snippets in docs; supports interop with `html/template`. |
| Markdown Parser | goldmark | CommonMark-compliant; AST-based; extensible; extension ecosystem (GFM, tables, footnotes, typographer). |
| Search | Bleve | Embedded full-text search engine in Go; supports indexing JSON, query DSL, faceting, BM25. Good fit for integrated search without an extra service. |
| Authentication | go-chi/jwtauth v5 | Built for Chi; uses `lestrrat-go/jwx`; well-maintained. Avoid archived jwt-go. |
| Migration Tool | golang-migrate (CLI + lib) | Widely used, battle-tested. Atlas offers improved error handling and declarative flows; can be phased in later. |
| Testing | Testify + testcontainers-go + stdlib | Testify is popular (27% per JetBrains 2025). testcontainers-go simplifies real-dependency integration tests. Go 1.25 `testing/synctest` for concurrent code tests. |
| Observability | OpenTelemetry Go + Prometheus 3.0+ | OTel is the emerging industry standard; Prometheus 3.0 has native OTLP ingestion and delta temporality support (per Grafana Labs and Prometheus docs). |
| Deployment | Docker (multi-stage to scratch/distroless) + Kubernetes | 2025 best practices: multi-stage, static comp (`CGO_ENABLED=0`), small images. Kubernetes RBAC/PSS and network policies; GitOps-friendly. |

---

## Phased Implementation Plan

### Phase 1: Project Foundation & Core CRUD (Weeks 1–4)

**Milestone:** Running server with basic page CRUD and templ rendering.

**Tasks:**
- Initialize project and Go module (`go mod init`). Target Go 1.25. Use `cmd/`, `internal/`, `pkg/`, `web/`, `configs/`, `deploy/`, `test/` (see project structure below).
- Set up Docker Compose for local dev (Postgres, app runner).
- Configure `pgxpool` with reasonable pool settings (see pgx Context7 docs). Add connection lifecycle hooks if needed.
- Design and apply initial schema via golang-migrate (CLI/lib): `pages` table (id, title, slug, content, created_at, updated_at, author_id, status).
- Implement domain types (Page entity) and repository interfaces.
- Use Chi for routes: `GET /pages`, `GET /pages/:slug`, `POST /pages`, `PUT /pages/:id`, `DELETE /pages/:id`.
- Write handlers and a service layer; wire with pgx queries using prepared statements or pgx’s conveniences.
- Use templ to render pages and forms (basic read/create/edit views). Add CSRF middleware for form submissions.
- Write unit and integration tests (Testify, testcontainers-go for DB). Aim for >80% coverage.

**Dependencies:** Chi v5, pgxpool v5, golang-migrate v4, templ, testify, testcontainers-go.

**Acceptance Criteria:**
- Pages can be created, viewed, edited, and deleted via web UI.
- All CRUD endpoints covered by tests; CI passes.
- DB migrations run reproducibly with golang-migrate.

**Risk:** None significant. Standard stack; low complexity.

---

### Phase 2: Authentication & Authorization (Weeks 5–7)

**Milestone:** Secure auth flow with JWT and role-based access.

**Tasks:**
- Add `users` table and migrations.
- Implement auth service (password hashing, registration, login) using bcrypt/scrypt.
- Configure go-chi/jwtauth for JWT verification/issuance:
  - Middleware: `jwtauth.Verifier` and `jwtauth.Authenticator`.
  - Use context for user claims (e.g., `jwtauth.FromContext`).
- Create middleware for role-based checks (Admin/Editor/Viewer) based on user claims stored in JWT or session.
- Set up sessions/cookies (use secure flags, SameSite) and optional refresh token rotation.
- Document authentication flow and public vs protected route groups in Chi.
- Add tests for auth endpoints and middleware.

**Dependencies:** go-chi/jwtauth v5, bcrypt/scrypt (stdlib or library), testify.

**Acceptance Criteria:**
- Users can register and login; JWT tokens issued and validated.
- Protected routes require valid JWT/role; unauthorized requests are rejected.
- Auth flow tested end-to-end.

**Risk:** Ensure secret management (env vars/secrets) and secure cookie practices; no archived auth libs (jwt-go is archived).

---

### Phase 3: Version Control & Diffs (Weeks 8–10)

**Milestone:** Full revision history, diff viewing, and rollback.

**Tasks:**
- Design `page_revisions` table (page_id, revision_number, content, author_id, created_at).
- On every page update, insert a revision snapshot atomically with the page update.
- Implement diff generation (line/word) using a Go diff library (e.g., github.com/sergi/go-diff or a suitable package).
- Add endpoints: `GET /pages/:slug/history`, `GET /pages/:slug/versions/:id`, `POST /pages/:slug/rollback/:version`.
- Display history and diffs in the templ UI.
- Index revisions for search if needed (via Bleve).

**Dependencies:** diff library (evaluate choices with up-to-date maintenance), testify.

**Acceptance Criteria:**
- Each page update creates a versioned revision.
- History and diffs viewable and rollback functional.
- Revision queries tested.

**Risk:** Storage growth; mitigate by retention policies and optional pruning.

---

### Phase 4: Search Implementation (Weeks 11–13)

**Milestone:** Full-text search across pages with faceting and highlight support.

**Tasks:**
- Evaluate Bleve integration: index page contents (title, content, tags, author).
- Create Bleve index mapping (text fields, analyzer, optional synonyms). Use Bleve CLI (`bleve create`, `bleve index`) for initial index.
- Implement search endpoints: `GET /search?q=...&tags=...`.
- Map queries to Bleve query types (match, phrase, conjunction, disjunction). Use Bleve context docs for query DSL and highlight.
- Add faceting by tags or namespaces.
- Consider re-indexing strategy: on page updates via background worker or scheduled job.
- Document index lifecycle and reindex procedure.

**Dependencies:** Bleve, testify.

**Acceptance Criteria:**
- Search returns relevant pages; facets supported; highlight in UI.
- Index kept in sync with page updates; reindexing documented.

**Risk:** Index consistency. Use idempotent updates and monitor index rebuilds.

---

### Phase 5: Collaboration Features (Weeks 14–16)

**Milestone:** Comments, notifications, watch system, and mentions.

**Tasks:**
- Add `comments`, `notifications`, `watches` tables.
- Implement comment CRUD per page (markdown rendering via goldmark).
- Design notification types (page edited, comment added, mentioned).
- Build watch/unwatch logic (opt-in per page or namespace).
- Implement user mention parsing (e.g., @username) in comments and page content.
- Add background job for email/in-app notifications (use workers and a queue or cron).
- Wire in templ views for comments and notifications UI.

**Dependencies:** goldmark (for comments), job queue (e.g., simple worker pool or library), testify.

**Acceptance Criteria:**
- Comments and mentions work; watchers receive notifications.
- Notification delivery pipeline functional and tested.

**Risk:** Notification delivery guarantees; at-least-once semantics with idempotency.

---

### Phase 6: Performance Optimization (Weeks 17–19)

**Milestone:** Sub-200ms median page loads, caching, CDN support.

**Tasks:**
- Profile and optimize slow DB queries (use pgx prepared statements, indexing).
- Add caching layer: Redis or in-memory (see cache tradeoffs). Cache hot pages, rendered HTML, and search results where appropriate.
- Configure CDN for static assets (CSS, JS, images). Use proper cache headers.
- Optimize Bleve queries; review analyzer and mappings.
- Set up OpenTelemetry instrumentation for HTTP and DB (otelhttp, DB instrumentation).
- Configure Prometheus scrapers; create dashboards for latency, throughput, error rates.
- Run load tests (k6 or similar) to confirm 1000+ concurrent users.

**Dependencies:** Redis (optional), OpenTelemetry Go, Prometheus, k6.

**Acceptance Criteria:**
- Page load times <200ms p95 under load.
- Observability stack (metrics/traces) in place; alerts configured.
- Cache hit rates acceptable; CDN configured.

**Risk:** Cache invalidation complexity; plan cache keys and TTLs carefully.

---

### Phase 7: Security Hardening (Weeks 20–22)

**Milestone:** OWASP Top 10 mitigations; security scanning.

**Tasks:**
- CSRF: Ensure CSRF tokens on all mutating endpoints (Chi middleware or custom).
- XSS: Goldmark’s HTML renderer escapes by default; review and test untrusted content rendering.
- SQL Injection: Use pgx parameterized queries exclusively (no string concatenation).
- Rate limiting: Implement rate limiting middleware (Chi-compatible middleware) per IP/user.
- Content Security Policy: Set CSP headers via middleware.
- Input validation: Validate and sanitize user inputs; use struct tags and validation libraries.
- Security scanning: Integrate `gosec`, `staticcheck`, and `govulncheck` in CI.
- Secrets management: Use environment variables or secret manager (no hardcoded secrets).

**Dependencies:** golang-migrate, pgx, goldmark, security scanners (gosec, staticcheck, govulncheck).

**Acceptance Criteria:**
- Security scanners pass in CI.
- Manual pen test or review passes; no critical vulnerabilities.
- OWASP risks mitigated (CSRF, XSS, injection, rate limiting).

**Risk:** Configuration errors; audit security headers and CSP.

---

### Phase 8: Deployment & Monitoring (Weeks 23–24)

**Milestone:** Production-ready containerization, Kubernetes deployment, monitoring.

**Tasks:**
- Docker multi-stage build: compile with `CGO_ENABLED=0`, strip symbols, use `scratch` or `distroless` final stage. Add CA certs if needed.
- Configure Kubernetes manifests: Deployment, Service, Ingress (with TLS), ConfigMap, Secret, HPA.
- Set RBAC with least-privilege, Pod Security Standards, Network Policies.
- Use readiness/liveness probes; configure resource requests/limits.
- Implement GitOps-friendly deploy (e.g., Argo CD or Flux optional).
- Set up OpenTelemetry collector/exporter to Prometheus; configure dashboards (Grafana).
- Document runbooks: scaling, failover, backup/restore.

**Dependencies:** Docker, Kubernetes, OpenTelemetry Go, Prometheus, optional GitOps tooling.

**Acceptance Criteria:**
- Application deploys via CI/CD; cluster resources defined as code.
- Observability dashboards operational; alerts defined.
- Backup/restore documented and tested.

**Risk:** Cluster misconfigurations; follow K8s security and RBAC best practices.

---

## Technical Decision Documentation

### Web Framework: Chi

**Options:**
- net/http stdlib with Go 1.22+ pattern routing (minimal, no framework)
- Chi
- Gin
- Echo
- Fiber

**Decision:** Chi (go-chi/chi v5)

**Rationale:**
- Chi is designed to work directly with `net/http`, aligning with Go’s “stdlib first” ethos.
- Provides composable middleware and clean context handling (examples from Chi docs).
- Widespread use and maintenance; ~12% adoption in 2025 ecosystem per JetBrains trends.
- Sufficient performance for typical web workloads; benefits from stdlib optimizations.

**Evidence:**
- Chi docs show JWT auth, middleware, and route groups examples using `chi.NewRouter` and context.

### Database Driver: jackc/pgx v5

**Options:**
- jackc/pgx v5 (native)
- sqlx + lib/pq (or older driver)
- GORM

**Decision:** jackc/pgx v5

**Rationale:**
- High performance; benchmarks and 2025 articles show significant speedups vs older drivers (up to 70% in some scenarios).
- Native PostgreSQL features: binary protocol, LISTEN/NOTIFY, COPY, JSONB.
- Built-in robust connection pooling (`pgxpool`), transaction helpers (`BeginFunc`), and context support.
- Well-maintained and recommended for PostgreSQL-only projects.

**Evidence:**
- pgx Context7 docs show connection pooling, transaction management, and query examples.

### Template Engine: templ

**Options:**
- templ
- html/template stdlib
- quicktemplate
- pongo2

**Decision:** templ

**Rationale:**
- Compiles to Go for high performance; type-safe; strong DX (tooling, editor support).
- 424+ code snippets in docs; active development and integration guides.
- Supports interoperability with `html/template` via `templ.ToGoHTML` and `templ.FromGoHTML` (see docs).
- Fits well with server-rendered approach and HTMX for interactivity.

**Evidence:**
- templ docs show component rendering and `html/template` interop.

### Markdown Parser: goldmark

**Options:**
- goldmark
- blackfriday (not CommonMark-compliant; maintenance concerns)
- gomarkdown

**Decision:** goldmark

**Rationale:**
- CommonMark-compliant and extensible; AST-based; extension ecosystem (GFM, tables, footnotes, typographer).
- Good performance; widely used and maintained.

**Evidence:**
- goldmark docs show configuration with extensions (GFM, auto heading IDs, renderer options).

### Search: Bleve

**Options:**
- Bleve (embedded)
- Meilisearch (external service)
- Postgres full-text search
- Elasticsearch

**Decision:** Bleve (embedded), with external service optional

**Rationale:**
- Pure Go, embedded; no separate service to operate initially.
- Supports rich query types, faceting, and ranking (BM25).
- Good fit for integrated search with moderate dataset sizes.
- For large scale or advanced semantic search, Meilisearch or Elasticsearch can be added later.

**Evidence:**
- Bleve docs show indexing, querying (match phrase, conjunction, highlight), synonyms, and CLI usage.

### Authentication: go-chi/jwtauth

**Options:**
- go-chi/jwtauth v5 (uses lestrrat-go/jwx)
- go-pkgz/auth (multi-provider OAuth/JWT)
- Custom implementation

**Decision:** go-chi/jwtauth v5

**Rationale:**
- Integrates cleanly with Chi; middleware for verifier and authenticator.
- Avoid archived jwt-go; relies on `lestrrat-go/jwx`, which is actively maintained.
- Supports standard JWT flows; easy to extend for refresh tokens.

**Evidence:**
- Chi docs show JWT authentication examples with `jwtauth.Verifier` and `Authenticator`.

### Migration Tool: golang-migrate (with Atlas as optional future)

**Options:**
- golang-migrate
- Atlas
- Goose

**Decision:** golang-migrate as baseline; Atlas considered for improved flows

**Rationale:**
- golang-migrate is battle-tested, widely used, supports CLI and library.
- Atlas provides better error handling (no dirty state), statement-level tracking, and declarative flows.
- Phased approach: start with golang-migrate; evaluate Atlas for declarative migrations or error recovery later.

**Evidence:**
- golang-migrate docs show CLI and lib usage for Postgres migrations.

### Testing: Testify + testcontainers-go

**Options:**
- Testify (assertions, mocks)
- Ginkgo + Gomega
- stdlib only

**Decision:** Testify + testcontainers-go

**Rationale:**
- Testify is popular (27% per JetBrains 2025); clean API for assertions and mocks.
- testcontainers-go provides real dependencies (Postgres containers) for integration tests.
- Use stdlib for core tests; augment with Testify and testcontainers for complex scenarios.

**Evidence:**
- Testify docs show assertions and mocks.
- testcontainers-go docs show container usage.

### Observability: OpenTelemetry Go + Prometheus

**Options:**
- OpenTelemetry Go + Prometheus
- Prometheus client library directly

**Decision:** OpenTelemetry Go + Prometheus

**Rationale:**
- OTel is the emerging vendor-neutral standard; Prometheus 3.0 has native OTLP ingestion and delta temporality support (per Prometheus docs).
- Use OTel instrumentation for HTTP, DB, and custom metrics.

**Evidence:**
- Prometheus 3.0 docs and Grafana blogs indicate native OTLP and delta temporality support in 2025.

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|-------|------------|----------|-------------|
| JWT library vulnerabilities (archived jwt-go) | Low (we avoid it) | High | Use go-chi/jwtauth; keep dependencies updated; run `govulncheck` in CI. |
| Search index drift | Medium | Medium | Implement idempotent reindexing; schedule consistency checks; version index schemas. |
| Database migration issues (dirty states) | Low-Medium | High | Use golang-migrate; consider Atlas for better error handling; test migrations thoroughly. |
| Performance under load | Medium | High | Profile early; use caching (Redis/in-memory); optimize queries and indexes; load test; observability. |
| Security vulnerabilities | Low-Medium | High | Follow OWASP guidelines; use security scanners; CSP, CSRF, XSS mitigations; secure headers. |
| Dependency abandonment | Low | Medium | Prefer actively maintained libraries with clear licenses; monitor updates. |
| Scaling Bleve | Medium | Medium | Monitor index size and performance; consider external search service for large datasets. |
| Container image size/security | Low | Medium | Use multi-stage builds to scratch/distroless; non-root user; scan images. |
| CI/CD pipeline failures | Low | Medium | Test CI jobs locally where possible; use caching; monitor workflow performance. |

---

## File and Package Structure

```
gowiki/
├── cmd/
│   └── wiki/
│       └── main.go              # application entry point
├── internal/
│   ├── auth/
│   │   ├── jwt.go              # JWT token management (go-chi/jwtauth)
│   │   ├── middleware.go        # auth/role middleware
│   │   └── service.go          # auth service (login, register, hashing)
│   ├── domain/
│   │   ├── page.go              # Page entity and value objects
│   │   ├── user.go              # User entity
│   │   ├── revision.go          # Revision entity
│   │   ├── comment.go           # Comment entity
│   │   └── search.go           # search criteria/results
│   ├── repository/
│   │   ├── postgres/
│   │   │   ├── page.go          # pgx queries for pages
│   │   │   ├── user.go          # pgx queries for users
│   │   │   └── revision.go      # pgx queries for revisions
│   │   ├── search/
│   │   │   └── bleve.go        # Bleve index/search
│   │   └── interface.go         # repository interfaces
│   ├── service/
│   │   ├── page.go              # page service (CRUD, revisions)
│   │   ├── user.go              # user service
│   │   ├── search.go            # search service
│   │   └── notification.go      # notifications service
│   ├── http/
│   │   ├── handler/
│   │   │   ├── page.go
│   │   │   ├── auth.go
│   │   │   ├── search.go
│   │   │   └── notification.go
│   │   ├── middleware/
│   │   │   ├── logging.go
│   │   │   ├── recovery.go
│   │   │   ├── csrf.go
│   │   │   └── cors.go
│   │   └── router.go            # Chi router setup
│   ├── config/
│   │   └── config.go            # configuration loading (env vars/viper)
│   └── worker/
│       └── notification.go       # background job for notifications
├── pkg/
│   └── util/
│       └── diff.go              # diff utility for revisions
├── web/
│   ├── static/                  # static assets (CSS, JS, images)
│   └── templ/
│       ├── layout.templ
│       ├── page.templ
│       ├── search.templ
│       └── auth.templ
├── db/
│   └── migrations/              # golang-migrate files
│       ├── 000001_init.up.sql
│       ├── 000001_init.down.sql
│       └── ...
├── configs/
│   ├── docker-compose.yml
│   └── kubernetes/            # K8s manifests (Deployment, Service, Ingress)
├── scripts/
│   ├── build.sh
│   ├── migrate.sh
│   └── test.sh
├── test/
│   └── integration/
│       └── suite_test.go        # testcontainers setup
├── .github/
│   └── workflows/
│       └── ci.yml              # GitHub Actions
├── .dockerignore
├── Dockerfile
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

---

## Frontend Options (Detailed)

### Option A: Server-Rendered with templ + HTMX

- **templ** compiles to Go for performance and type safety.
- **HTMX** adds interactivity (dynamic updates, partial loading) without a full SPA.
- Benefits: simpler architecture, fast page loads, minimal JS, easier SEO, fewer client-side moving parts.
- Integration: Use templ to render components; HTMX endpoints to fetch partials.

### Option B: SPA with API Backend

- Build a RESTful API (already planned) that returns JSON for all resources.
- SPA (React/Vue) consumes API; handles routing, state, and UI client-side.
- Benefits: rich interactivity, offline support, progressive web app (PWA).
- Tradeoffs: more complexity, additional build tooling, SEO requires SSR; plan includes OpenAPI/Swagger docs for the API.

---

## Development Infrastructure

### Development Environment

- Docker Compose for local development (app, Postgres, optional Redis).
- Hot reload using `air` or similar; watch for file changes and restart app.
- Makefile targets: `dev`, `build`, `test`, `migrate-up`, `migrate-down`, `lint`, `docker-build`.
- Pre-commit hooks: `go fmt`, `go vet`, `golangci-lint`, `govulncheck`.

### Testing Strategy

- Unit tests: >80% coverage; use `go test` and Testify.
- Integration tests: testcontainers-go with real Postgres; test endpoints end-to-end.
- E2E tests: critical user flows (page CRUD, auth, search, revision rollback) using a browser automation framework (e.g., Playwright) if needed.
- Load tests: k6 or similar; target 1000+ concurrent users.
- Security scanning: `gosec`, `staticcheck`, `govulncheck` in CI.

### CI/CD Pipeline

- GitHub Actions workflow:
  - Checkout, setup Go 1.25, cache dependencies.
  - Run linting (golangci-lint), security scanning (`govulncheck`), and tests.
  - Build Docker image (multi-stage).
  - Push image to registry; optionally deploy to staging.
- Use matrix strategy for multiple Go versions if needed.
- Cache dependencies and Docker layers for speed.

### Documentation

- API docs: OpenAPI/Swagger; generate from annotations or use a spec.
- Developer setup: `README.md` with quick start (Docker Compose, migration, running app).
- Architecture Decision Records (ADRs): document major decisions (framework choice, DB schema, search engine).
- User documentation: hosted on the wiki itself.

---

## Timeline Estimate

- Phase 1: 4 weeks
- Phase 2: 3 weeks
- Phase 3: 3 weeks
- Phase 4: 3 weeks
- Phase 5: 3 weeks
- Phase 6: 3 weeks
- Phase 7: 3 weeks
- Phase 8: 2 weeks

**Total:** ~24 weeks (6 months). Actual timeline may vary based on team size and complexity (e.g., SPA vs. server-rendered).
