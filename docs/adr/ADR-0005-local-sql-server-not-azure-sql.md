# ADR-0005: SQL Server in Docker for local dev; Azure SQL Database only in the live environment

- **Status:** Accepted
- **Date:** 2026-09-05
- **Repo(s) affected:** `Lakbay.Cms`, `Lakbay.Booking`

## Context

Randolf asked directly whether Azure SQL Database can be "set up locally
for free," given the whole platform is being built local-only for now, no
environments deployed yet. Azure SQL Database is a cloud-only PaaS
product — Microsoft doesn't ship a local/offline edition of it, so there
is no version of "run Azure SQL on this machine." What it does offer,
confirmed 2026-09-05: a genuine free tier (100,000 vCore-seconds and 32GB
storage per database per month, serverless, up to 10 databases per
subscription, no expiration) — real, but still cloud-hosted, so it doesn't
answer the offline/local-only requirement this project has held since the
[[working-style]] "code needs to survive without AI or internet" entry.

## Decision

Local development and testing for `Lakbay.Cms` and `Lakbay.Booking` runs
against **SQL Server, Developer Edition, in Docker**
(`mcr.microsoft.com/mssql/server` — free, full-featured, non-production
license) via each repo's `Docs/DEVELOPER_HANDBOOK.md` Docker Compose
stack. **Azure SQL Database is used only once a live/staging environment
exists** (`02_BUILD_PLAN.md` Phase 5) — same connection-string shape, same
EF Core migrations, no application code changes between the two.

## Alternatives considered

- **SQL Server Express** instead of Developer Edition — rejected:
  Developer Edition is equally free for non-production use and has no
  database-size or feature ceiling to hit accidentally during development;
  Express's 10GB-per-database cap and missing features (e.g. some
  partitioning/columnstore behavior) have no upside for a dev machine.
- **SQLite** (as Ophir Mineral Ventures uses for its own Umbraco install)
  — rejected specifically for Lakbay: Ophir is a low-traffic brochure site
  where SQLite's simplicity is a genuine win; Lakbay's booking/availability
  writes and Umbraco's own SQL Server-first support make SQL Server the
  better-matched local stand-in for what production (Azure SQL Database)
  actually is.
- **Try to reach a real Azure SQL Database from day one** (using its free
  tier) — rejected for local development specifically: it's still a cloud
  dependency, which conflicts with the stated "must work fully offline"
  requirement for local dev. Worth revisiting for a free staging tier once
  Phase 5 needs a real deployed environment, but that's a different use
  case from local dev.

## Consequences

- Local dev has zero cloud dependency for the database layer, once the
  SQL Server Docker image is pulled once — consistent with every other
  offline-runnable Docker Compose stack already planned per repo.
- The move to Azure SQL Database in Phase 5 should be close to
  connection-string-only, since Azure SQL Database and SQL Server share
  the same T-SQL surface for ordinary EF Core/Umbraco usage. A small,
  genuinely different surface exists (no SQL Agent, no cross-database
  queries, some compatibility-level differences) — if either repo ever
  needs one of those, that's a signal to verify against a real Azure SQL
  Database earlier than Phase 5, not an assumption to make now.
- `Lakbay.Cms`'s and `Lakbay.Booking`'s `Docs/DEVELOPER_HANDBOOK.md` (once
  written, per Phase 0) must include the exact Docker Compose service
  definition for SQL Server, not just say "install SQL Server" — the
  offline-runnable bar this project holds itself to.
