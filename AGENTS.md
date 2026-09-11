# AGENTS.md — Allu

> Guidance for AI coding agents (and human contributors) working in the **Allu** repository.
> Allu is the Helsinki City system for managing usage rights, notifications, and billing for public areas (streets, sidewalks, green areas, events, land for rent, etc.).

This file supplements [README.md](README.md) and [PLAN.md](PLAN.md). Start with those first.

Architecture and domain documentation lives in [`docs/`](docs/README.md) — that
index is the only list of what is documented; this file does not duplicate it.

---

## 1. Repository Overview

| Aspect       | Detail |
|--------------|--------|
| **Purpose**  | Public-area usage management for the City of Helsinki |
| **Owner**    | City of Helsinki (Open Data / Solution Office) |
| **License**  | No project-wide licence is declared. `LICENSE` contains third-party attributions only (Leaflet.draw, TurfJS, …), and GitHub reports the repository as `NOASSERTION`. Do not describe Allu as MIT-licensed. |
| **Remotes**  | `origin` → `ForumViriumHelsinki/allu` (fork), `upstream` → `City-of-Helsinki/allu` (canonical, actively developed by Gofore) |
| **Language** | Java 17 (backend), TypeScript/Angular 18 (frontend), Bash/Ansible (deployment) |
| **Architecture** | Monorepo: `backend/`, `frontend/`, `deployment/` |

### Directory layout

```
allu/
├── backend/          # Java / Spring Boot multi-module Maven project
├── frontend/         # Angular 18 single-page application
├── deployment/       # Ansible playbooks, Dockerfiles, env inventories
├── docs/             # Architecture and domain documentation — see docs/README.md
│   ├── architecture/ # Services and mechanisms (APIs, environments, retention, jobs)
│   ├── domain/       # Application types and their data models
│   └── fvh/          # Forum Virium plans and upstream proposals — NOT descriptions of Allu
├── PLAN.md           # In-progress Angular Material MDC migration plan
└── README.md         # Project overview + local setup instructions
```

---

## 2. Backend (`/backend`)

### Stack

- **Spring Boot 2.7.18** / Java 17
- **Maven** multi-module (build with `mvn clean install`)
- **QueryDSL** 5.1.0 for type-safe queries
- **PostgreSQL** (primary + reporting DB port 5433)
- **Elasticsearch** (search backend)
- **wkhtmltopdf** for PDF generation
- **SAP interface** for billing/invoice integration

### Module map (from `backend/pom.xml`)

| Module | Role |
| --- | --- |
| `common-domain` / `common` | Shared domain logic and infrastructure |
| `service-core-domain` / `service-core` | Core service layer |
| `model-domain` / `model-querydsl-support` / `model-service` | Entity/model layer with QueryDSL support |
| `search-domain` / `search-service` | Elasticsearch-backed search |
| `external-domain` / `external-service` | Integrations with external systems |
| `pdf-domain` / `pdf-service` | PDF generation (via wkhtmltopdf) |
| `scheduler-service` | Background/cron tasks (notifications, purging, ETL cleanup) |
| `sap-interface` / `sap-import` | SAP system integration for billing data |
| `mail` | Email notification generation |
| `allu-ui-service` | Main REST API for the frontend (port 9000) |
| `allu-ui-service-test` | E2E/load tests (TypeScript-based, in a subdirectory) |
| `supervision-api` | Field supervision API, port 9050 (Swagger UI at `:9050/swagger-ui.html`) |
| `allu-etl` | ETL pipelines for data loading |

### Build & run

```bash
# Build (from /backend)
mvn clean install

# Run locally via Docker (requires built jars)
docker-compose up    # from /backend

# Minimum services to start:
#   allu-ui-service, model-service, search-service, external-service
```

### Key config

- Per-service config: `<service>/src/main/resources/application.properties`
- External credentials (WFS username/password) are **not** in config — they come from `deployment/group_vars/test/credentials.yml` (Ansible-managed).
- `wkhtmltopdf` must be installed and its path set in `pdf-service/src/main/resources/application.properties` → `pdf.generator=...`

### Testing

- **Unit/integration tests**: standard Maven Surefire (`*Test.java`, `*Spec.java`, `*IT.java` patterns).
- **E2E/load tests**: `backend/allu-ui-service-test/` — a TypeScript project. Run `npm install` then `TEST_TARGET=http://localhost:3000 npm test`.
- Test data seeding requires `allu-ui-service` + `model-service` + `search-service` + `frontend` running.
- The `massdata` test inserts 10,000 applications — it is slow; not for CI.

---

## 3. Frontend (`/frontend`)

### Stack

- **Angular 18.2.14** (SPM, not standalone components — uses NgModules)
- **Angular Material** — **in active MDC migration** (see PLAN.md)
  - Legacy components (`MatLegacy*`) are being replaced with MDC equivalents
  - Do not introduce new `MatLegacy*` usage
  - Migration is stepwise: form-field → input → select → autocomplete
- **Flex Layout** 15.0.0-beta.42 (for responsive layouts)
- **Karma + ChromeHeadless** for unit tests
- **ESLint** (burn-down complete — 0 problems required)
- **Node 18–20**, **npm 9–10**

### Build & run

```bash
# Install
npm install

# Dev server (with proxy to backend at :3000)
npm run start

# Hot Module Reload
npm run hmr

# Production build
npm run build-prod

# Tests (headless Chrome required)
CHROME_BIN=/path/to/chromium npm test -- --watch=false --browsers=ChromeHeadless

# Lint (must report 0 problems)
npm run lint
```

### Feature modules (`/src/app/feature/`)

The frontend is organized by domain feature. Key areas:

```
admin/              # Admin functions (e.g. external-user management)
allu/               # Root app module, routing, shared infra
application/         # Application/permit management
auth/ login/        # Authentication (OAuth2 flow)
common/             # Shared components (input-box, etc.)
customerregistry/   # Customer registry lookups (autocomplete)
decision/           # Decision/proposal handling
information-request/ # Information request workflow
map/ mapsearch/     # Map integration (WFS-backed)
notification/        # Notification management
pdf/               # PDF download/viewing
project/            # Project (application-group) management
supervision-workqueue/ workqueue/  # Work queue / supervision
search/ searchbar/  # Search functionality
customer-related (sidebar, toolbar, etc.
```

### CI (GitHub Actions — `.github/workflows/frontend-ci.yml`)

- **Lint** job: `npm ci` + `npm run lint` (0 problems)
- **Build & Test** job: `npm ci` + `npm run build` + `ng test` (ChromeHeadless, no sandbox)
- Both run on `ubuntu-latest`, Node 22.20.0
- Branches that trigger CI: `master`, `release-*` (and PRs)
- **Branch protection** must be configured in GitHub UI to enforce these (not possible via committed file).

---

## 4. Deployment (`/deployment`)

### Stack

- **Ansible** playbooks (not Terraform/CDK — this is Ansible-driven infra)
- **Docker** for local dev and build
- Three environments: `test`, `staging`, `production` (inventory files: `*.inventory`)
- **Zabbix** for monitoring

### Key deploy scripts

| Script | Purpose |
| --- | --- |
| `install_all_to_{test,staging,production}.sh` | Full install to an environment |
| `deploy_{backend,frontend}_to_test.sh` | Per-service deploy to test |
| `recover_{production,staging}_database.sh` | DB recovery |

### Docker services (local)

- `allu_database` (PostgreSQL :5432)
- `allu-node` (Elasticsearch :9200)
- `allu-reporting-database` (PostgreSQL :5433)

### Ansible roles

```
roles/
├── database_build/       # DB container build (PostgreSQL + Elasticsearch)
├── reporting_database_build/  # Separate reporting DB
├── backend_build/        # Backend service Docker images
├── frontend_build/       # Frontend build
├── etl_build/            # ETL service build
├── elasticsearch_build/  # Elasticsearch-specific config
├── sftpserver_build/  # SFTP server config
└── zabbix/              # Monitoring config
```

---

## 5. Git Conventions

### Branch naming

- Use **Jira ticket numbers**: `ALLU-234`, or prefixed: `severij/ALLU-218`
- Some branches use numeric-only: `#336-wrong-location-order`
- Main integration branch: `master`
- Release branches: `release-*`

### Commit message convention

- Format: `ALLU-<n>: <description>` (e.g. `ALLU-181: Update Spring Security...`)
- Some earlier commits used: `ALLU-<n>-<short-slug>`
- Prefer the `ALLU-n: description` form for new commits.

### PR workflow

- PRs are merged into `master`
- Frontend CI must pass (Lint + Build & Test) but enforcement requires GitHub UI branch protection
- No automated CI gate for the **backend** — no CI workflow found for Java/Maven

---

## 6. Active Work / Known State

### Angular Material MDC Migration (PLAN.md)

- **Status**: In progress (4 steps planned)
- **Goal**: Migrate from `MatLegacy*` (Angular Material legacy) to MDC-based components
- **Steps**: form-field → input → select → autocomplete
- **Each step**: Run `ng generate @angular/material:mdc-migration`, review diff, commit
- **Watch for**: SCSS class selector changes (`.mat-form-field` → `.mat-mdc-form-field`, etc.)
- **Do not** introduce new `MatLegacy*` imports anywhere

### Spring Security Modernization (recent commits)

- `@EnableGlobalMethodSecurity` → `@EnableMethodSecurity` (done)
- `WebSecurityConfigurerAdapter` → `SecurityFilterChain` (done)
- Constructor DI (not field injection) in security configs (done)

### Test infrastructure

- Frontend: Karma + ChromeHeadless, ESLint burn-down complete
- Backend: Maven Surefire; **no automated CI workflow for Java** (gap — agents should note this)
- E2E tests in `backend/allu-ui-service-test/` (TypeScript, manual run)

---

## 7. Quick Reference — Commands

| Task | Command | Location |
| --- | --- | --- |
| Build backend | `mvn clean install` | `/backend` |
| Run backend locally | `docker-compose up` | `/backend` |
| Build frontend | `npm run build-prod` | `/frontend` |
| Run frontend dev | `npm run start` | `/frontend` |
| Run frontend tests | `npm test -- --watch=false --browsers=ChromeHeadless` | `/frontend` |
| Lint frontend | `npm run lint` | `/frontend` |
| Start DB + ES | `docker compose up` | `/deployment/roles/database_build/files/database_docker` |
| Run E2E tests | `TEST_TARGET=http://localhost:3000 npm test` | `/backend/allu-ui-service-test` |
| Swagger (supervision) | `http://localhost:9050/swagger-ui.html` | — |
| Swagger (external API) | `http://localhost:9040/swagger-ui.html` | — |
| App login | `http://localhost:3000/login` (user: `allute`) | — |

---

## 8. Agent-Specific Notes

1. **This is a fork, not the canonical repository.** `origin` is `ForumViriumHelsinki/allu`; `upstream` is `City-of-Helsinki/allu`, which Gofore develops actively for the city. Changes here may be **local experimentation**, **Forum Virium work**, or **intended to go upstream** — check with the user before assuming ownership or a branching strategy. Anything under `docs/fvh/` is Forum Virium's own material and is not upstream-bound.

2. **No backend CI pipeline** — when editing Java code, run `mvn clean install` locally before declaring done. There is no GitHub Actions gate for it.

3. **Material MDC migration is active** — before touching any frontend component that uses `MatLegacy*` classes, read PLAN.md. Do not add new `MatLegacy*` usage.

4. **Environment config** — WFS credentials and SAP credentials are **not in the repo** (they are in Ansible `group_vars`). Do not hardcode them.

5. **Test data** — the `allu-ui-service-test` module can seed 10k applications. This is slow (~minutes). Avoid running `massdata` in CI or on small VMs.

6. **PostgreSQL locale** — if the database Docker build fails with a `locale.md` error, the fix is documented in the README: replace the `xargs`-based locale command with a direct `localedef` call for `fi_FI`.

7. **Node version** — CI uses Node 22.20.0. Local dev uses Node 20. Both should work; the `package.json` allows Node 18–20 but 22 is what CI runs.

8. **wkhtmltopdf** — required for PDF generation. The path in `application.properties` is environment-specific.

---

## 9. File Structure Summary

```
allu/
├── backend/
│   ├── allu-ui-service/         # Main API (Spring Boot, port 9000)
│   ├── model-service/           # Model/entity layer
│   ├── search-service/          # Elasticsearch search API
│   ├── external-service/        # External system integrations
│   ├── scheduler-service/       # Background jobs / cron
│   ├── pdf-service/             # PDF generation
│   ├── sap-interface/           # SAP billing integration
│   ├── supervision-api/         # Supervision/monitoring REST API
│   ├── allu-etl/                # ETL data processing
│   ├── allu-ui-service-test/    # E2E/load tests (TypeScript!)
│   ├── common(-domain)/         # Shared domain code
│   ├── service-core(-domain)/   # Core service layer
│   ├── model-domain/            # Entity definitions
│   ├── search-domain/           # Search domain models
│   ├── external-domain/         # External integration domain
│   ├── pdf-domain/              # PDF domain
│   ├── mail/                    # Email generation
│   └── sap-import/              # SAP import processing
├── frontend/
│   ├── src/app/
│   │   ├── feature/             # Domain-organized feature modules
│   │   ├── http-interceptors/   # HTTP request/response interceptors
│   │   ├── model/               # TypeScript data models
│   │   ├── service/             # Angular services (HTTP calls)
│   │   ├── pipe/                # Custom Angular pipes
│   │   ├── util/                # Utility functions
│   │   └── typings/             # TypeScript definitions
│   ├── src/assets/              # SCSS partials (forms, inputs, etc.)
│   └── karma.conf.js            # Karma test config
└── deployment/
    ├── *.yml                    # Ansible playbooks (build/deploy per env)
    ├── *.inventory              # Ansible host inventories
    └── roles/                   # Docker build configs per service
```
