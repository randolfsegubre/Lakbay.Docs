# ADR-0027: Agent Channel's confirm-booking call moves to gRPC

- **Status:** Accepted
- **Date:** 2026-09-10
- **Repo(s) affected:** `Lakbay.Booking`, `Lakbay.AgentOps`

## Context

`Lakbay.AgentOps`'s `AgentDesktopController.ConfirmBooking` (ADR-0021's
BFF pattern) called `Lakbay.Booking`'s `POST /api/bookings/confirm` REST
endpoint via a plain `IHttpClientFactory` client — the same integration
point `Lakbay.Web`'s online checkout uses. That transport had never
actually been a deliberate decision; it was just what Phase 0 scaffolded
first. An agent is live on the phone with a customer when this call
happens and needs an immediate confirm/fail response, which is exactly
the latency-sensitive, service-to-service scenario gRPC is built for —
and a named job-posting skills gap (gRPC) made this a genuine, useful
place to close it with real code rather than only studying it.

## Decision

`Lakbay.Booking.Api` gains a gRPC server **alongside**, not instead of,
its existing REST endpoints — `Lakbay.Web`'s online checkout is
completely untouched by this change. A new `Protos/bookingconfirm.proto`
(`BookingConfirmService.ConfirmAgentBooking`) and a thin
`BookingConfirmGrpcService` adapter map the incoming protobuf request
straight into the same `ConfirmBookingCommand` the REST endpoint already
sends through `IMediator` — ADR-0011's atomic-decrement guarantee has
exactly one implementation regardless of which transport reached it.
`Channel.Agent` is hardcoded server-side inside the gRPC service, never
accepted from the caller: this RPC exists only for `Lakbay.AgentOps`,
never a browser.

`Lakbay.AgentOps`'s `AgentDesktopController` calls it via
`Grpc.Net.ClientFactory` (`AddGrpcClient<BookingConfirmService.BookingConfirmServiceClient>`).
`RpcException` is caught and mapped to the same `AgentDesktopBookingResult`
shape the desktop client already expects — this is invisible to
`Lakbay.AgentDesktop`, no contract change on that side.

**gRPC requires TLS/ALPN for HTTP/2 protocol negotiation on a shared
origin** — `Booking:BaseUrl` must point at `Lakbay.Booking`'s HTTPS
endpoint, not its plain-HTTP one. Kestrel only serves HTTP/1.1 on a
plain-HTTP endpoint by default; ALPN-based negotiation (what lets one
origin serve both REST/HTTP1.1 and gRPC/HTTP2) only happens over TLS.

## Alternatives considered

- **Keep REST, add a short timeout** — rejected: doesn't close the named
  skills gap, and REST/JSON's per-call overhead is real (if modest at
  this scale) versus gRPC's binary framing for a call an agent is
  actively waiting on.
- **A second plain-HTTP port for h2c (HTTP/2 cleartext) instead of TLS** —
  rejected: more moving parts (a second Kestrel endpoint to configure and
  keep documented) for no real benefit in a dev/prod setup where HTTPS is
  already the standard for every other endpoint on this origin.

## Consequences

- A csproj wiring detail worth remembering: `Lakbay.Booking.Api.csproj`'s
  `<Protobuf>` item must be `GrpcServices="Both"`, not `"Server"` —
  `Lakbay.Booking.Tests` already has a `ProjectReference` to this project
  for its existing handler-level tests, so a second, independent
  client-stub codegen pass (the pattern `Lakbay.AgentOps` uses, which has
  no such reference) would collide with the message types already
  compiled into that assembly.
- Any future session wiring a new cross-repo gRPC call on this platform
  should set `BaseUrl` to the HTTPS origin from the start, not discover
  the ALPN requirement the hard way (see `Lakbay.Docs/docs/05_DEVLOG.md`'s
  2026-09-12 entry — this exact mistake shipped once and only got caught
  by an actual live cross-process call, not by either side's own unit
  tests, since both sides "build clean" independent of whether the
  configured origin can actually negotiate HTTP/2).
