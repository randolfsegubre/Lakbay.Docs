# ADR-0023: A WCF service simulates legacy CTI screen-pop for the agent desktop

- **Status:** Accepted
- **Date:** 2026-09-08
- **Repo(s) affected:** `Lakbay.AgentDesktop` (new `Lakbay.AgentDesktop.TelephonyBridge` project within the repo)

## Context

Real call centers integrate a desktop agent tool with telephony/CTI
(computer-telephony integration) middleware so that when a call arrives,
the agent's screen automatically shows who's calling and their history —
"screen pop" — before the agent even says hello. This integration point
is frequently WCF in practice: many PBX/CTI middleware vendors (legacy
Genesys, Avaya, and in-house telephony gateways) shipped WCF-based SDKs
during the 2010s and those integrations are still running unchanged in
production at plenty of enterprises today, which is exactly why WCF shows
up as a named requirement on some .NET job postings even now.

`Lakbay.AgentDesktop` has no real PBX to integrate with. Building a
believable simulation of this exact integration shape — a WCF service
that mimics what a real CTI gateway would push to a desktop client — is
the most honest way to demonstrate real WCF service-hosting and
consumption experience without fabricating a hardware integration that
doesn't exist.

## Decision

A separate project inside the `Lakbay.AgentDesktop` repo,
`Lakbay.AgentDesktop.TelephonyBridge`:

- A **WCF service** (`System.ServiceModel`, hosted via a Windows Service
  or a console host locally) exposing `ITelephonyBridge`:
  - `SubscribeToIncomingCalls()` — a duplex contract (`IsOneWay=false`,
    callback contract `ITelephonyBridgeCallback`) that pushes an
    `IncomingCallNotification` (caller phone number, a simulated
    "matched customer" lookup, call-start timestamp) to the desktop
    client the moment a (simulated) call arrives.
  - `EndCall(callId)` — logs call end, used to close out the active call
    context in the agent desktop UI.
- A **call simulator** (a small console/WinForms tool, or a "Simulate
  Incoming Call" button in a debug panel of `Lakbay.AgentDesktop` itself)
  stands in for the real PBX — it's what would be replaced by an actual
  telephony gateway in production, and the ADR's job is to make that
  replacement boundary explicit and clean.
- Transport: `NetTcpBinding` locally (matches how a real on-prem CTI
  gateway would typically be reached — same LAN, not internet-facing),
  documented as the one piece of this platform that is deliberately
  **not** cross-platform and **not** cloud-hosted, by design — this
  mirrors how real telephony middleware usually stays on-prem.

## Alternatives considered

- **A REST/gRPC endpoint instead of WCF** — rejected outright for this
  component specifically: the entire point is demonstrating real WCF
  service-hosting experience, which a REST or gRPC endpoint would not do,
  no matter how architecturally cleaner it might be in isolation.
- **SignalR for the push instead of a WCF duplex contract** — SignalR is
  already used elsewhere in this system (ADR-0025, `Lakbay.AgentOps`) for
  real-time availability push, and reusing it here would be simpler — not
  chosen for this specific integration point because it wouldn't
  demonstrate WCF, and because a duplex WCF contract is a more realistic
  match for how legacy CTI SDKs from this era were actually shaped.
- **Skip the simulator, just have the desktop app call a "get next fake
  call" endpoint on a timer** — rejected: a push-based duplex contract is
  the architecturally honest shape for "the phone rings," and polling
  would misrepresent how the real integration works.

## Consequences

- `Lakbay.AgentDesktop.TelephonyBridge` is the platform's only WCF
  component and its only explicitly on-prem/LAN-only piece — every other
  service in this system is designed for cloud deployment (ADR-0005's SQL
  Server exception aside, which is itself dev-only). This asymmetry is
  intentional and should be called out, not "fixed" toward consistency.
  in a later phase.
- WCF is being consumed with `System.ServiceModel.Duplex`/`NetTcpBinding`
  on modern .NET, not .NET Framework — confirms real WCF still works
  under current .NET, a fact worth stating plainly since WCF's status on
  modern .NET is a common point of confusion (the WCF *server-side*
  framework itself is not part of the shared framework anymore; the
  community-maintained `CoreWCF`/`System.ServiceModel.*` client and
  server packages are what make this possible).
- If a real telephony integration is ever pursued, this ADR's simulator
  boundary is exactly what gets replaced — the `ITelephonyBridge` contract
  itself should not need to change.
