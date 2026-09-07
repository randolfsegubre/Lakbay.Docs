# Local Infrastructure — every Docker container, what it's for, how it fits

Written 2026-09-08 (Phase 3 session) after standing all of this up for
real. If Docker Desktop's **Containers** tab looks unfamiliar, this is the
"what am I looking at" reference — one section per container, in the order
you'd normally start them.

**The rule this platform follows everywhere:** Docker only ever runs
*infrastructure* (databases, message brokers) — never the applications
themselves. Every app (`Lakbay.Cms`, `Lakbay.AvailabilityApi`, `Lakbay.Web`,
the Sync function) runs natively via `dotnet run` / `npm run dev` / `func
start`, so you always get fast hot-reload and a normal debugger attach.
If you ever see an `ASPNETCORE_ENVIRONMENT` or a `.csproj` mentioned
*inside* a `docker-compose.yml` in this platform, something has gone off
the established pattern — flag it.

## Quick map: container → repo → purpose

| Container name | Image | Started by | Used by |
|---|---|---|---|
| `lakbay_sqlserver` | `lakbaycms-db` (custom build, SQL Server base) | `Lakbay.Cms/docker-compose.yml` | `Lakbay.Cms` (`umbracoDb`), `Lakbay.Booking` (`lakbayBookingDb`, once Phase 4 starts) |
| `lakbay_mongo` | `mongo:8` | `Lakbay.AvailabilityApi/docker-compose.yml` | `Lakbay.AvailabilityApi.Api` (reads), `Lakbay.AvailabilityApi.Sync` (writes) |
| `lakbay_servicebus` | `mcr.microsoft.com/azure-messaging/servicebus-emulator` | `Lakbay.AvailabilityApi/docker-compose.yml` | `Lakbay.Cms` (publishes), `Lakbay.AvailabilityApi.Sync` (consumes) |
| `lakbay_servicebus_sqledge` | `mcr.microsoft.com/azure-sql-edge` | `Lakbay.AvailabilityApi/docker-compose.yml` | Internal to `lakbay_servicebus` only — not used directly by anything |

Two other things you might see in the Containers tab that **aren't part of
this platform**: `testcontainers-ryuk-*` and a randomly-named Mongo
container (e.g. `vigilant_khayyam`) — those are spun up and torn down
automatically by `Lakbay.AvailabilityApi.Tests` (Testcontainers) every time
you run `dotnet test`. Safe to ignore; they clean themselves up within
seconds of the test run finishing. If one is ever still sitting there
minutes later, the test run was killed mid-way — safe to remove manually.

---

## `lakbay_sqlserver` — SQL Server (Developer Edition, free)

**What it is:** a single SQL Server instance, built from a small custom
Dockerfile in `Lakbay.Cms/Database/` (adapted from the official `dotnet
new umbraco-compose` template — trimmed to database-only, since the
template's default also containerizes the Umbraco app itself, which this
platform deliberately doesn't do).

**Why it exists:** `Lakbay.Cms` (Umbraco) needs a relational database for
its own content storage, and `Lakbay.Booking` will need one too once
Phase 4 starts (orders, availability). Rather than running two SQL Server
containers, both databases live on **one shared local instance** — purely
a local-dev convenience; they stay two fully separate databases
(`umbracoDb`, `lakbayBookingDb`), never sharing tables, matching ADR-0003
("`Lakbay.Booking` never stores its data in `Lakbay.Cms`'s database").

**How it relates to the app:** `Lakbay.Cms.Web` connects via a
`ConnectionStrings:umbracoDbDSN` value set through **`dotnet
user-secrets`**, never `appsettings.json` — the real password never has a
chance to land in a commit. Same pattern applies to `Lakbay.Booking` once
it's wired up, pointed at `lakbayBookingDb` on the same instance.

**Setup:**
```bash
cd Lakbay.Cms
cp .env.example .env
# edit .env — set DB_PASSWORD to any local-only value
docker compose up -d
docker inspect --format='{{.State.Health.Status}}' lakbay_sqlserver   # wait for "healthy"

cd src/Lakbay.Cms.Web
dotnet user-secrets set "ConnectionStrings:umbracoDbDSN" \
  "Server=localhost,1433;Database=umbracoDb;User Id=sa;Password=<your DB_PASSWORD>;TrustServerCertificate=true"
dotnet user-secrets set "ConnectionStrings:umbracoDbDSN_ProviderName" "Microsoft.Data.SqlClient"
```

**Port:** `1433` (host) → `1433` (container) — the standard SQL Server
port, exposed so any SQL client (Azure Data Studio, SSMS, `sqlcmd`) can
connect directly for inspection: `Server=localhost,1433;User
Id=sa;Password=...`.

---

## `lakbay_mongo` — MongoDB

**What it is:** stock `mongo:8`, no authentication (deliberate — this is
genuinely local-only, disposable data; nothing secret to protect, unlike
the SQL Server password).

**Why it exists:** `Lakbay.AvailabilityApi` is a denormalized, read-optimized
copy of the catalog (ADR-0007 — modeled on Hotelplan's `api-sphinx`
pattern). MongoDB's document model is a natural fit for that denormalized
shape (a `Product` document embeds its `Destination` inline, for
instance — no join needed at query time).

**How it relates to the app:** `Lakbay.AvailabilityApi.Api` (the GraphQL
query service) only ever *reads* from it. `Lakbay.AvailabilityApi.Sync`
(the Azure Function) is the only writer — it applies updates it receives
from `lakbay_servicebus` (see below). This split is ADR-0009: a burst of
sync writes should never compete with shoppers' queries for the same
process's CPU/threads.

**Setup:**
```bash
cd Lakbay.AvailabilityApi
docker compose up -d
docker inspect --format='{{.State.Health.Status}}' lakbay_mongo   # wait for "healthy"
```

**Port:** `27017` (host) → `27017` (container) — the standard Mongo port.
Connect directly with `mongosh "mongodb://localhost:27017"` or `docker exec
-it lakbay_mongo mongosh` for inspection. Database name: `lakbay_availability`.

**A real gotcha worth knowing:** `Lakbay.AvailabilityApi.Api`'s
`CatalogSeeder` used to auto-seed placeholder data into this database on
every boot (Phase 1). That's been retired (2026-09-08) — `Lakbay.Cms` is
now the only thing that populates real catalog data here, via the sync
pipeline below. If this database is empty, `Lakbay.Web`'s storefront will
render with zero products — that's expected until you've authored and
published something in `Lakbay.Cms`, not a bug.

---

## `lakbay_servicebus` + `lakbay_servicebus_sqledge` — the Azure Service Bus Emulator

**What they are:** Microsoft's official local emulator for Azure Service
Bus (`mcr.microsoft.com/azure-messaging/servicebus-emulator`), plus its
**required** companion database (`azure-sql-edge`) that the emulator uses
to persist its own internal state. You never talk to `sqledge` directly —
it exists purely so `lakbay_servicebus` has somewhere to keep its queue
metadata.

**Why they exist:** this is the actual message transport for ADR-0013 —
how `Lakbay.Cms` tells `Lakbay.AvailabilityApi` "a product changed,
here's the new data" without either service calling the other directly
(coupling `Lakbay.Cms`'s publish path to `Lakbay.AvailabilityApi`'s uptime
is exactly what this avoids). In production this would be a real Azure
Service Bus namespace; locally, the emulator gives the exact same wire
protocol with zero Azure account needed.

**How it relates to the app — the actual data flow:**

```
Editor publishes a Product/Destination/ProductLine node in Lakbay.Cms
  → CatalogPublishSyncHandler (a ContentPublishedNotification handler)
  → builds a CatalogSyncEvent matching Lakbay.Contracts' shapes
  → sends it to the "lakbay-catalog-sync" queue on lakbay_servicebus
  → Lakbay.AvailabilityApi.Sync's CatalogSyncFunction (a [ServiceBusTrigger])
    wakes up, consumes the message
  → applies it to lakbay_mongo (field-scoped $set — see ADR-0014 for why
    it's not a blind whole-document overwrite)
  → Lakbay.AvailabilityApi.Api's GraphQL now reflects the change
  → Lakbay.Web shows it on next query
```

The queue itself (`lakbay-catalog-sync`) is defined in
`Lakbay.AvailabilityApi/servicebus-emulator-config.json` — the emulator
reads this file at startup and creates exactly the queues listed in it
(nothing is created dynamically at runtime, unlike a real Azure
namespace where you could create queues via the portal/CLI).

**Setup:** started together with `lakbay_mongo` — same
`docker compose up -d` in `Lakbay.AvailabilityApi`. Both `Lakbay.Cms` and
`Lakbay.AvailabilityApi.Sync` connect using the exact same well-known
local connection string (this is Microsoft's documented fixed value for
the emulator — not a real secret, safe to see in plain config):

```
Endpoint=sb://localhost;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SAS_KEY_VALUE;UseDevelopmentEmulator=true;
```

- `Lakbay.Cms.Web`: set via `dotnet user-secrets set "ConnectionStrings:ServiceBus" "<the string above>"`
- `Lakbay.AvailabilityApi.Sync`: already in `src/Lakbay.AvailabilityApi.Sync/local.settings.json`'s `ServiceBusConnection` key (gitignored — that file never leaves your machine)

**Ports:** `lakbay_servicebus` exposes `5672` (AMQP — the actual message
protocol clients use) and `5300` (an HTTP management/config endpoint).
`lakbay_servicebus_sqledge` exposes nothing to the host — it's reachable
only from `lakbay_servicebus` over Docker's internal network.

**Verifying it's actually working**, beyond "container is Up":
```bash
docker logs lakbay_servicebus --tail 20
# look for: "Emulator Service is Successfully Up!" and a line confirming
# "lakbay-catalog-sync" queue creation — if that queue name is missing,
# check servicebus-emulator-config.json got mounted correctly.
```

---

## Running everything together — the order that actually works

```bash
# 1. Infrastructure first
cd Lakbay.Cms && docker compose up -d
cd ../Lakbay.AvailabilityApi && docker compose up -d
# wait for all four containers to report healthy/up (docker ps)

# 2. The Sync function — start this BEFORE publishing anything in Cms,
#    or messages will sit queued (harmlessly) until it starts
cd Lakbay.AvailabilityApi/src/Lakbay.AvailabilityApi.Sync
func start

# 3. The query API
cd ../Lakbay.AvailabilityApi.Api
dotnet run

# 4. Cms
cd ../../../Lakbay.Cms/src/Lakbay.Cms.Web
dotnet run

# 5. The storefront
cd ../../../Lakbay.Web
npm run dev
```

See `07_MANUAL_SETUP_GUIDE.md` for first-time setup of each (user-secrets,
`.env` files, `npm install`, etc.) — this section assumes that's already
done once and you're just bringing the stack up for a normal dev session.

## Tearing it down / starting fresh

```bash
# Stop everything (keeps data):
cd Lakbay.Cms && docker compose down
cd ../Lakbay.AvailabilityApi && docker compose down

# Stop AND wipe all data (SQL Server content, Mongo catalog, Service Bus
# queue state) — use when you want a genuinely clean slate:
docker compose down -v   # -v removes the named volumes too
```

A half-clean state worth knowing about: if you only want to re-seed
`Lakbay.Cms`'s Products-tree content (not the whole database), don't wipe
SQL Server — instead run `Lakbay.Cms.Web` once with
`LAKBAY_FORCE_RESEED_CATALOG=true` set (see `CatalogContentTypeSeeder`'s
own doc comment) to delete and recreate just the seeded content nodes.
