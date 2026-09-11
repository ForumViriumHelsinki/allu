---
title: Allu documentation index
status: index
---

# Allu documentation

Architecture and domain documentation for Allu, the City of Helsinki system for
managing usage rights, notifications, and billing for public areas.

This file is the **only** index of what lives here. If you add a document, add a
row below and nowhere else — the previous layout listed every file in three
places (this index, a Mermaid graph, and the repository's `AGENTS.md`), which
meant three edits per addition and three chances to drift.

## `architecture/` — how the platform is built

Cross-cutting services and mechanisms.

| Document | Covers |
| --- | --- |
| [api-gateways.md](architecture/api-gateways.md) | The three client-facing REST APIs: internal UI, external B2B, field supervision. Auth, roles, mutation rules, milestone reporting. |
| [environments.md](architecture/environments.md) | Production/staging/test/local environments, public URLs, Apache proxy routing, the full internal port map, external integrations. |
| [data-retention.md](architecture/data-retention.md) | The four-tier lifecycle: archiving, GDPR anonymization, customer hard purge, warehouse cleanup. |
| [scheduled-jobs.md](architecture/scheduled-jobs.md) | Every cron-driven job, with deployed and local-dev schedules. |
| [subsystems-overview.md](architecture/subsystems-overview.md) | Shorter treatments: SAP financials, GIS/MapProxy, PDF generation, ETL warehouse, frontend, test tiers. |

## `domain/` — what the platform manages

Application types and their data models.

| Document | Covers |
| --- | --- |
| [application-types.md](domain/application-types.md) | All eight `ApplicationType` values, their pricing engines and specialties. |
| [excavation-announcement.md](domain/excavation-announcement.md) | *Kaivuilmoitus* in depth: extension fields, stakeholders, geometry, kinds/specifiers, supervision lifecycle, pricing. |
| [excavation-revisions.md](domain/excavation-revisions.md) | Revision chains (*korvaava päätös*), the `tyon_tarkoitus` progress-journal convention, and query recipes for history. |

## `fvh/` — Forum Virium Helsinki working documents

Not descriptions of Allu. See [fvh/README.md](fvh/README.md) for why they are
separated.

| Document | Covers |
| --- | --- |
| [excavation-timeline-service.md](fvh/excavation-timeline-service.md) | Design guide for an FVH map + timeline service over excavation data. |
| [upstream-proposals.md](fvh/upstream-proposals.md) | Four proposed upstream changes, drafted while designing the above. |

---

## Conventions

### Frontmatter

Every document carries provenance, because all of this describes a codebase that
is actively changing:

```yaml
---
title: Data Retention, Archiving & Pruning
status: reference        # reference | design | proposal | index
verified_against: e3e24f8ed
verified_on: 2026-09-11
---
```

`verified_against` is the commit the claims were checked against. When you
re-verify a document, update both fields. A stale pair is a visible signal; no
pair at all is not.

### One owner per fact

Several facts are relevant to more than one document. Each has exactly one home,
and the others link to it rather than restating it:

| Fact | Canonical home |
| --- | --- |
| Cron schedules | [architecture/scheduled-jobs.md](architecture/scheduled-jobs.md) |
| Internal service ports, deployment topology | [architecture/environments.md](architecture/environments.md) |
| Revision chains and the `tyon_tarkoitus` journal | [domain/excavation-revisions.md](domain/excavation-revisions.md) |
| External API request/response shapes | [architecture/api-gateways.md](architecture/api-gateways.md) |
| The eight application types | [domain/application-types.md](domain/application-types.md) |

### Where a new document goes

- Describes a **service or mechanism** → `architecture/`
- Describes an **application type or its data** → `domain/`
- Describes **something FVH wants to build or propose** → `fvh/`

A short treatment starts as a section in
[subsystems-overview.md](architecture/subsystems-overview.md) and graduates to
its own file once it outgrows a page.

## Coverage gaps

Documented in depth: excavation announcements, the API surfaces, environments,
retention, scheduled jobs.

Covered only as summaries: the other seven application types, SAP/invoicing,
GIS, PDF generation, ETL, and the frontend. The imbalance reflects what the
initial investigation needed, not the relative importance of the subsystems.
