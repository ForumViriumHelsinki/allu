---
title: Scheduled Background Jobs
status: reference
verified_against: e3e24f8ed
verified_on: 2026-09-11
---

# Scheduled Background Jobs

Canonical reference for every cron-driven job in Allu. Other documents link here
rather than repeating schedules — [data retention](data-retention.md) covers what
the retention jobs *do*, this file covers *when they run*.

Two services run scheduled work:

- **`scheduler-service`** — 10 jobs, all declared in
  `backend/scheduler-service/src/main/java/fi/hel/allu/scheduler/controller/ScheduleRunner.java`
  via `@Scheduled(cron = "${...}")`.
- **`allu-etl`** — 2 jobs, declared in
  `backend/allu-etl/src/main/java/fi/hel/allu/etl/EtlRunner.java`.

## Where a schedule actually comes from

Each `@Scheduled` annotation reads a property, and that property is resolved from
a different file depending on how the service is running:

| Running as | Property source |
| --- | --- |
| Local dev / `docker-compose` | `backend/<service>/src/main/resources/application.properties` |
| Deployed (test/staging/production) | `deployment/roles/backend_deploy/templates/scheduler-service.properties`, filled from `deployment/group_vars/<env>/backend.yml` |

The two columns below differ for almost every job, so **read the column that
matches the environment you are asking about.** Quoting a local-dev value as if
it were production is the specific mistake this table exists to prevent.

## `scheduler-service`

| Job method | Property | Deployed (production) | Local dev | Action |
| --- | --- | --- | --- | --- |
| `remindApplicants` | `applicantReminder.cronstring` | `0 0 1 * * MON-FRI` | `*/15 * * * * MON-FRI` | Reminder emails for pending requirements or expiry |
| `sendInvoices` | `invoice.cronstring` | `0 0 7 * * *` | `*/15 * * * * MON-FRI` | Builds and uploads SAP IDoc XML billing batches |
| `updateCustomers` | `customer.update.cronstring` | `0 0/30 * * * ?` | `*/15 * * * * MON-FRI` | Polls SAP for customer master-data updates |
| `sendCustomerNotifications` | `customer.notification.cronstring` | `0 0 7 * * *` | `*/15 * * * * MON-FRI` | Internal digest of customer changes |
| `syncSearchData` | `search.sync.cronstring` | `0 0 23 * * *` † | `0 0 1 * * *` | Re-indexes PostgreSQL data into Elasticsearch |
| `updateApplicationStatuses` | `application.status.update.cronstring` | `0 0 * * * *` † | `0 1 * * * *` | Hourly sweep moving ended permits to `FINISHED` / `ARCHIVED` |
| `updateCityDistricts` | `cityDistricts.update.cronstring` | `0 30 0 * * SUN` | `30 0 * * * *` | Syncs district polygons from the city GeoServer |
| `checkAnonymizableApplications` | `anonymization.update.cronstring` | `0 0 3 * * *` | `0 */5 * * * *` | Finds cable reports past 2 years and strips PII |
| `sendRemovedSapCustomerNotifications` | `removed.customers.notification.cronstring` | `0 15 3 * * *` | `0 15 3 * * *` | Emails finance about deleted SAP accounts |
| `purgeObsoleteCustomers` | `customer.purge.cronstring` | `0 0 4 * * *` | `0 0 4 * * *` | Hard-deletes soft-deleted customers older than 5 years |

† These two are **hardcoded in the Ansible template** rather than templated from
`group_vars`, so they cannot be changed per environment without editing
`scheduler-service.properties`.

## `allu-etl`

| Job method | Property | Deployed (production) | Local dev | Action |
| --- | --- | --- | --- | --- |
| `run` | `etl.cronstring` | `0 0 */1 * * *` | `0 * * * * *` | Loads operative data into the `allureport` warehouse |
| `runCleanup` | `etl.cleanup.cronstring` | `0 45 */6 * * *` | `0 45 */6 * * *` | `cleanup.sql` — removes warehouse rows deleted upstream (ALLU-234) |

Production values come from `deployment/group_vars/production/allu-reporting.yml`.

## Disabling a job

Set the cron to a date that never occurs. The repo's own convention, documented
in `deployment/group_vars/production/backend.yml`:

```
customer_purge_cronstring: "0 0 3 31 2 *"
```

February 31st never arrives, so the job is registered but never fires.

## Known dead config

`deletable.customer.scan.cronstring` is templated in
`deployment/roles/backend_deploy/templates/scheduler-service.properties`
(default `0 0 3 * * *`) but no `@Scheduled` method binds it. Setting it has no
effect.
