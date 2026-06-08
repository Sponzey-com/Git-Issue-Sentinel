# Git Issue Sentinel

[Korean](./README.ko.md)

Git Issue Sentinel is a self-host-first issue intelligence platform for collecting, viewing, searching, classifying, and eventually managing issues from Git-based collaboration systems and connected project management tools.

This project is currently in the planning and design phase. Implementation should follow the decisions in `PROJECT.md`, `AGENTS.md`, and `RESEARCH.md`.

## Core Direction

| Item | Decision |
|---|---|
| Product priority | Self-host first, SaaS later |
| MVP provider | GitHub |
| MVP scope | Read-only GitHub Issues ingestion, normalization, and unified list view |
| Frontend | Latest stable Vue.js, based on the Vue 3 stable line |
| Build/development tools | Vite + TypeScript + Vue SFC + Composition API |
| Issue list/board | Headless issue table based on `@tanstack/vue-table` |
| Kanban board | Drag-and-drop based on `SortableJS`; use a stable Vue 3 wrapper or a direct composable wrapper |
| Architecture | Layered Architecture, Clean Architecture |
| Development method | Tidy First, TDD |
| Configuration policy | Minimize external config files; accept environment values only at process startup |
| Logging | Three log profiles: `product`, `debug`, `dev` |

## Problem

Development teams often spread issue work across GitHub, GitLab, Gitea, Forgejo, Bitbucket, Azure DevOps, SourceHut, Jira, Linear, Plane, OpenProject, and other systems. As repositories and teams grow, several problems become harder to manage.

- Teams cannot see issues from multiple repositories in one place.
- Labels, assignees, states, and priorities differ by provider.
- Stale issues, unassigned issues, and release blockers are hard to find.
- Git issues are weakly connected to external project management tools.
- Self-hosting, private network, audit log, and data retention requirements may not fit hosted SaaS tools.

Git Issue Sentinel addresses this by hiding provider-specific API differences behind adapters and handling issues through a normalized internal issue model.

## MVP

The MVP scope is intentionally narrow.

**Goal:** Build a read-only issue inbox that ingests GitHub Issues and shows them in a searchable, filterable unified list.

Required MVP features:

- GitHub connection
- GitHub repository selection
- Open/closed/all issue ingestion
- Pagination handling
- Rate-limit handling
- Preservation of source issue ID, number, URL, repository, state, labels, assignees, author, and timestamps
- Raw payload storage
- Normalized issue storage
- Idempotent sync without duplicate issues
- All Issues list
- Basic search and filters
- Issue detail view
- Sync status display

Excluded from the MVP:

- Issue create/update/delete
- Comment writing
- Label, assignee, or state changes
- Two-way sync
- Webhook-based realtime sync
- AI triage
- Jira/Linear/Plane/OpenProject integration
- SaaS operations features

## Final Goal

The final product is an operations hub for all Git-based issue sources.

Long-term goals:

- Support GitHub, GitLab, Gitea, Forgejo, Bitbucket, Azure DevOps, and SourceHut
- Support self-hosted enterprise instances
- Unified issue inbox
- Issue board and kanban board
- Provider write-back
- Two-way sync through webhooks and polling
- Jira, Linear, Plane, OpenProject, and Redmine integrations
- Slack, Teams, and email notifications
- AI-based summaries, duplicate detection, label recommendations, assignee recommendations, and priority recommendations
- Automation rules
- SSO, RBAC, audit logs, and token encryption
- Self-host single-node, enterprise, and air-gapped deployments
- SaaS only after the self-host product is stable

## Technology Choices

## Frontend

Use the latest stable Vue.js version. As of 2026-06-08, the official Vue documentation is centered on Vue 3 and recommends Single-File Components with the Composition API for full applications. At scaffolding time, use the stable tags for `vue@latest` and `create-vue@latest`, then pin the resolved versions in the lockfile.

Recommended stack:

- Vue 3 stable
- TypeScript
- Vite
- Vue Router
- Pinia
- Vue SFC
- Composition API
- Vitest
- Playwright or Cypress

## Issue List / Board

Use `@tanstack/vue-table` as the baseline for issue lists and list-style boards.

Why:

- Provides a Vue adapter
- Headless table design keeps UI ownership inside the project
- Supports sorting, filtering, pagination, column visibility, and row selection
- Fits the MVP need to handle large issue lists
- Keeps table state in the UI layer, which aligns with Clean Architecture

Rules:

- TanStack Table is only a frontend presentation/state tool.
- Domain models and sync logic must not depend on TanStack Table.
- When server-side filtering or sorting is needed, convert table state into an API query object.

## Kanban Board

Implement the kanban board with `SortableJS`. For Vue, choose a stable Vue 3 wrapper, or write a direct composable wrapper around SortableJS if wrapper maintenance is not good enough.

Why:

- SortableJS is a proven drag-and-drop list library.
- The SortableJS ecosystem includes Vue wrapper projects.
- It fits common kanban needs such as moving cards across columns, sorting cards inside a column, drag handles, and touch-device support.
- It makes it practical to separate board domain policy from drag-and-drop UI behavior.

Rules:

- SortableJS belongs only in the UI interaction layer.
- Issue state changes, label changes, and provider write-back are handled by application use cases.
- Drag results should be converted into explicit commands such as `MoveIssueOnBoardCommand`.
- Do not implement the kanban board in the MVP; introduce it in V1/V2 or later.

## Backend Direction

The backend language and framework are not decided yet. The structure should still follow these principles.

```text
apps/
  web/
  api/
  worker/
packages/
  core/
  adapters/
    github/
    gitlab/
    gitea/
    forgejo/
    bitbucket/
    azure-devops/
    sourcehut/
  db/
  auth/
  crypto/
  search/
  queue/
  shared/
```

Required MVP backend pieces:

- GitHub provider adapter
- Provider connection management
- Repository discovery
- Issue sync job
- Pagination and rate-limit handling
- Normalized issue model
- Issue upsert
- Issue list API
- Issue detail API
- Sync status API

## Architecture Rules

This project follows `AGENTS.md`.

Required:

- Layered Architecture
- Clean Architecture
- Tidy First
- TDD
- Provider adapter boundary
- Domain/infrastructure separation
- Environment values accepted only at the composition root
- Explicit argument/dependency passing
- No token or secret logging

Forbidden:

- Calling provider APIs directly from the domain layer
- Reading `process.env` directly in use cases
- Injecting or changing environment values during runtime
- Storing provider tokens in global singletons
- Calling provider APIs directly from UI components
- Quietly adding write-back behavior inside the MVP scope

## Configuration Policy

External configuration files should be minimized.

Allowed configuration:

- Build/test tool configuration
- DB migrations
- Docker Compose or deployment manifests
- Documentation needed for OAuth app registration
- Test fixtures

Environment values are read only at process startup.

Allowed:

```ts
const runtimeConfig = loadRuntimeConfig(process.env);
const app = createApp({ config: runtimeConfig, adapters, db, queue, logger });
```

Forbidden:

```ts
process.env.GITHUB_TOKEN = token;
process.env.LOG_LEVEL = "debug";
```

User-specific provider tokens, repository-specific sync options, tenant permissions, and sync cursors must be passed through DB records, command inputs, job payloads, or explicit dependencies instead of environment variables.

## Logging Policy

There are three log profiles.

| Profile | Purpose | Environment |
|---|---|---|
| `product` | Minimal product logs | Default for production/self-host |
| `debug` | Field diagnostics | Customer-site incident analysis |
| `dev` | Development and test inspection | Local/test |

Always forbidden:

- Token
- Secret
- OAuth code
- Authorization header
- Full issue body/comment
- Full raw payload
- Personal data

The `log profile` is decided at process startup and must not be changed by mutating environment variables at runtime.

## Self-host-first Strategy

The initial product is designed for self-hosting.

Self-host MVP baseline:

- Runnable on a single server or with Docker Compose
- Local PostgreSQL support
- GitHub token or GitHub App connection
- Minimal dependency on external SaaS
- Private repository issues should not leave the deployment boundary
- Clear log/profile/config policy

SaaS should be considered only after:

- The self-host MVP is stable
- Token encryption and tenant isolation are verified
- Sync jobs, rate limits, and retries are stable
- Audit logs and retention policy foundations exist
- Extensibility beyond GitHub is verified

## Provider Roadmap

| Phase | Provider | Scope |
|---|---|---|
| MVP | GitHub | Read-only issue sync/list/detail |
| V1 | GitHub | Comments/events, saved views, webhook review |
| V1/V2 | GitLab | Read-only issue sync |
| V1/V2 | Gitea/Forgejo | Self-host forge read-only sync |
| V2 | GitHub | Write-back |
| V3 | Jira/Linear/Plane/OpenProject | Export/link/sync |
| Later | Bitbucket, Azure DevOps, SourceHut | Extended providers |

## Decisions Needed Before Development

- Backend language/framework
- DB migration tool
- Job queue strategy
- GitHub authentication strategy: GitHub App first or PAT-based MVP
- Raw payload retention period
- Token encryption strategy
- Local self-host deployment method
- Filter/query shape for the issue list API
- Initial UI design system
- Phase for kanban board introduction

## Documents

- [PROJECT.md](./PROJECT.md): Project scope, features, providers, data model, roadmap
- [AGENTS.md](./AGENTS.md): Implementation rules, architecture, TDD, configuration/logging principles
- [RESEARCH.md](./RESEARCH.md): OSS market research, competing products, pricing/features analysis

## References

- Vue official docs: <https://vuejs.org/guide/introduction.html>
- TanStack Table Vue docs: <https://tanstack.com/table/latest/docs/framework/vue>
- TanStack Table v8 introduction: <https://tanstack.com/table/v8/docs/introduction>
- SortableJS GitHub: <https://github.com/SortableJS>
- SortableJS demo/docs: <https://sortablejs.github.io/Sortable/>
