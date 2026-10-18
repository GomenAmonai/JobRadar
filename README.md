# JobRadar

[![CI](https://github.com/GomenAmonai/JobRadar/actions/workflows/ci.yml/badge.svg)](https://github.com/GomenAmonai/JobRadar/actions/workflows/ci.yml)
![.NET 10](https://img.shields.io/badge/.NET-10-512BD4?logo=dotnet&logoColor=white)
![Kafka](https://img.shields.io/badge/Kafka-KRaft-231F20?logo=apachekafka&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)
![React](https://img.shields.io/badge/React-18-149ECA?logo=react&logoColor=white)

JobRadar is a real-time remote-job platform for .NET and backend engineers. It combines a vacancy aggregator, a candidate application tracker, and an employer workspace in one product.

Vacancies are collected from public job boards, transported through Kafka, normalized and persisted by a .NET worker, exposed through a REST API, and streamed to the browser with SignalR.

**Value proposition:** one codebase that demonstrates end-to-end backend engineering skills: event-driven ingestion, secure API design, real-time updates, and production-oriented deployment practices.

> **Live demo:** [job-finder-eight-cyan.vercel.app](https://job-finder-eight-cyan.vercel.app)
>
> The demo runs entirely in the browser with a simulated API, so it does not require the backend stack. Use the controls in the application to sign in as a demo candidate or employer.

![JobRadar vacancy feed with filters and live updates](docs/screenshots/feed-light.png)
![JobRadar vacancy feed in dark theme](docs/screenshots/feed-dark.png)

## Contents

- [Key capabilities](#key-capabilities)
- [User roles](#user-roles)
- [Architecture](#architecture)
- [Technology](#technology)
- [API](#api)
- [Security and reliability](#security-and-reliability)
- [Running locally](#running-locally)
- [Deployment](#deployment)
- [Testing](#testing)
- [Frontend](#frontend)
- [Demo mode](#demo-mode)
- [Project status and limitations](#project-status-and-limitations)
- [Roadmap](#roadmap)

## Key capabilities

- **Real-time vacancy feed** with filtering by market, level, stack, and search text.
- **Idempotent ingestion under concurrency.** A unique `(Source, ExternalId)` key and update-first upsert strategy prevent duplicate vacancies, including during concurrent writes. The integration test exercises 50 concurrent writers and verifies a single stored row and insert.
- **Event-driven processing.** Collectors publish raw vacancies to Kafka; the worker normalizes and persists them, then publishes change events for the API and frontend.
- **Candidate application tracking.** Candidates can save filters, track applications, and move applications through a controlled status workflow.
- **Employer workspace.** Employers can publish native JobRadar vacancies, review incoming applications, and update their status.
- **Optimistic concurrency.** Interactive updates use PostgreSQL `xmin`; stale writes return `409 Conflict` instead of silently overwriting newer data.
- **Live notifications.** SignalR delivers vacancy changes, saved-filter matches, and application status updates.
- **Automated quality checks.** Unit and integration tests run in CI against real PostgreSQL through Testcontainers.

Aggregated vacancies remain hosted on their original job boards. Applying to an aggregated vacancy adds it to the candidate's personal tracker; applications to vacancies posted directly on JobRadar are delivered to the employer.

## User roles

### Guest

- Browse the public vacancy feed.
- Filter vacancies by market, level, stack, and search text.
- Receive live feed updates.

### Candidate

- Register and authenticate with a JWT-based account.
- Save reusable vacancy filters.
- Add vacancies to a personal application pipeline.
- Track application status transitions.
- Receive notifications when a vacancy matches a saved filter or an employer changes an application status.

### Employer

- Publish vacancies directly on JobRadar.
- Review incoming applications.
- Move applications through the employer workflow.
- Notify candidates about status changes in real time.

## Architecture

```text
Collectors                 Kafka                    Worker (.NET)                 Client

Remotive       ────────►   vacancies.raw   ─────►  VacancyConsumer
RemoteOK                                              ├─ normalize market, level,
                                                      │  stack, salary and dedup key
                                                      ├─ idempotent upsert ───────► PostgreSQL
                                                      ├─ processing failures ─────► vacancies.dead-letter
                                                      └─ publish changes ─────────► vacancies.changed
                                                                                         │
                                                      API: VacancyChangedConsumer ◄──────┘
                                                      └─ SignalR hub ─────────────► React
                                                      API: REST ──────────────────► React SPA
```

The worker and API run as separate processes. The worker does not depend on SignalR, and the API does not depend on the ingestion implementation. Kafka provides the event-driven boundary between them.

The solution follows Clean Architecture, with dependencies directed inward:

| Project | Responsibility |
|---|---|
| `JobRadar.Domain` | Entities and pure domain logic, including application status transitions |
| `JobRadar.Application` | Contracts, DTOs, normalization, salary parsing, and deduplication keys |
| `JobRadar.Infrastructure` | EF Core, migrations, persistence, authentication, filters, and applications |
| `JobRadar.Api` | Minimal API, SignalR, JWT authentication, authorization, rate limiting, OpenTelemetry, and Scalar |
| `JobRadar.Worker` | Kafka consumers, collectors, background processing, and resilience policies |
| `frontend/` | React, Vite, TypeScript, TanStack Query, SignalR, and the browser UI |

## Technology

- .NET 10 and ASP.NET Core Minimal APIs
- Entity Framework Core 10 and PostgreSQL 16
- Apache Kafka in KRaft mode with Confluent.Kafka
- SignalR for real-time browser updates
- React 18, Vite, and TypeScript
- TanStack Query for client-side data fetching and cache management
- OpenTelemetry for traces and metrics
- Docker Compose for local infrastructure
- xUnit and Testcontainers for automated testing
- GitHub Actions for continuous integration

## API

The interactive API reference is available at `/scalar/v1` in the Development environment.

| Method | Route | Access |
|---|---|---|
| `GET` | `/vacancies?market=&level=&stack=&q=&page=&pageSize=` | Public |
| `POST` | `/auth/register`, `/auth/login`, `/auth/refresh` | Public; rate-limited |
| `GET` | `/auth/me` | Authenticated |
| `GET/POST/PUT/DELETE` | `/me/filters[/{id}]` | Authenticated |
| `POST` | `/vacancies/{id}/applications` | Authenticated |
| `GET/PATCH/DELETE` | `/me/applications[/{id}][/status]` | Authenticated |
| `POST` | `/employer/vacancies` | `Employer` role |
| `GET` | `/employer/applications` | `Employer` role |
| `PATCH` | `/employer/applications/{id}/status` | `Employer` role |
| `GET` | `/hubs/vacancies` | SignalR; token supplied through `?access_token=` |

### SignalR events

- `VacancyChanged` — broadcast when a vacancy is inserted or updated.
- `MatchedVacancy` — sent to a candidate when a new vacancy matches a saved filter.
- `ApplicationStatusChanged` — sent to a candidate when an employer changes an application status.

## Security and reliability

- **JWT validation:** issuer, audience, lifetime, and signature validation are enabled for access tokens.
- **Refresh-token protection:** refresh tokens are opaque; only their SHA-256 hashes are stored. Rotation revokes the token chain when reuse is detected.
- **Authorization boundaries:** authenticated queries are scoped to the current user, and employer endpoints require the `Employer` role.
- **Production hardening:** ProblemDetails responses omit stack traces outside Development; HSTS, HTTPS redirection, security headers, and configured CORS are enabled.
- **Rate limiting:** authentication endpoints are limited to 10 requests per minute per IP; the public vacancy endpoint is limited to 120 requests per minute per IP.
- **Input protection:** search expressions are escaped and length-limited, page size is capped at 100, and password length is constrained.
- **Reliable delivery:** Kafka consumption is at least once, while idempotent persistence prevents duplicate vacancies. Messages that cannot be processed are sent to a dead-letter topic.
- **Observability:** OpenTelemetry captures ASP.NET Core, HTTP client, PostgreSQL, and runtime traces and metrics. OTLP export is enabled when `OTEL_EXPORTER_OTLP_ENDPOINT` is configured.

The security model was reviewed against relevant areas of the OWASP API Security Top 10, including object-level authorization, broken authentication, property-level authorization, resource consumption, function-level authorization, SSRF, and security misconfiguration.

## Running locally

### Prerequisites

- Docker
- .NET 10 SDK
- Node.js and npm

The .NET SDK version is pinned in [`global.json`](global.json).

### Start the stack

```bash
# 1. Start PostgreSQL, Kafka, and Kafka UI.
#    PostgreSQL is exposed on host port 5433; Kafka UI is available on :8080.
docker compose up -d

# 2. Apply database migrations.
dotnet ef database update \
  --project src/JobRadar.Infrastructure \
  --startup-project src/JobRadar.Api

# 3. Start the worker.
dotnet run --project src/JobRadar.Worker

# 4. Start the API on http://localhost:5088.
dotnet run --project src/JobRadar.Api

# 5. Start the frontend on http://localhost:5173.
cd frontend
npm install
npm run dev
```

The frontend reads the API address from `VITE_API_URL` and defaults to `http://localhost:5088`. See [`frontend/env.example`](frontend/env.example) for configuration details.

> **Development configuration:** the PostgreSQL credentials and JWT key in the application settings are intended only for local development. Before deploying to a shared or production environment, move secrets to environment variables or a secret manager, add employer verification, define vacancy retention policies, and provide a dead-letter replay process.

## Deployment

The repository includes multi-stage Dockerfiles for each deployable service:

- `src/JobRadar.Api/Dockerfile`
- `src/JobRadar.Worker/Dockerfile`
- `frontend/Dockerfile`

The API supports platform-provided `$PORT` values and can apply migrations at startup when `RunMigrationsOnStartup=true` is enabled. For a Railway deployment walkthrough, see [`DEPLOY.md`](DEPLOY.md).

## Testing

```bash
# Unit tests: fast and Docker-free.
dotnet test tests/JobRadar.UnitTests

# Integration tests: require Docker and use Testcontainers.
dotnet test tests/JobRadar.IntegrationTests
```

The CI workflow also builds the frontend with TypeScript and Vite on every push and pull request. The test suite covers:

- **Unit behavior:** normalization, salary parsing, deduplication keys, relevance scoring, status transitions, and mapping.
- **Integration behavior:** concurrent upserts, vacancy querying and deduplication, authentication and refresh-token rotation, saved-filter concurrency conflicts, applications, employer flows, IDOR protection, and HTTP-level role enforcement through `WebApplicationFactory`.

## Frontend

The frontend is a React, Vite, and TypeScript single-page application. It uses plain CSS with design tokens, supports light and dark system themes, and changes the vacancy table into cards on narrow screens.

TanStack Query manages server state and cache invalidation. SignalR events invalidate the active vacancy query, while the typed API client handles token refresh automatically.

For frontend-specific setup, configuration, and project structure, see [`frontend/README.md`](frontend/README.md).

## Demo mode

The static frontend includes a browser-only demo mode so the deployed experience remains usable without a running API. [`frontend/src/api/demo/server.ts`](frontend/src/api/demo/server.ts) mirrors the real API's routes and behavior, including status codes, optimistic-concurrency versions, application status transitions, and the `Employer` role. State is stored in `localStorage`.

A simulated worker discovers a vacancy approximately every 25 seconds and emits the same event types as the SignalR hub. Components use the same interface in both modes; only the transport layer in `frontend/src/api/http.ts` changes.

| Command or setting | Behavior |
|---|---|
| `npm run dev` | Use the real API at `VITE_API_URL` |
| `npm run build` with `VITE_API_URL` | Build against the real API |
| `npm run build` without `VITE_API_URL` | Build the browser demo used by the Vercel deployment |
| `VITE_DEMO_MODE=true` | Force demo mode on |
| `VITE_DEMO_MODE=false` | Force demo mode off |

## Project status and limitations

- **Status:** active portfolio project with CI, automated tests, and containerized deployment assets.
- **Current limitation:** the public live demo is frontend-only and uses an in-browser simulated API instead of the hosted backend stack.
- **Current limitation:** aggregated vacancies stay on their original job boards; only vacancies posted directly on JobRadar support native employer-to-candidate workflow inside the platform.

## Roadmap

- Add a production-hosted backend environment for the live demo experience.
- Improve employer verification and moderation workflows for posted vacancies.
- Expand observability dashboards and operational runbooks for deployment environments.
