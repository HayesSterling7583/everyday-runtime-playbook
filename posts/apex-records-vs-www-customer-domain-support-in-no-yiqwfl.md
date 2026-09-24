# Apex Records vs WWW Customer Domain Support in Node.js — 2026 Operations

**TL;DR:** For a fintech app that assigns every tenant a hostname, accept both the customer's apex and `www` only when the onboarding contract names one canonical host and redirects the other. If the platform cannot keep stable address records for the apex, make `www` the documented attachment point and treat apex forwarding as a separate customer-controlled concern. The deciding factor is operational coupling: an apex address record binds the customer to an address lifecycle, while a delegated hostname can stay behind a naming layer.

This is less about which spelling looks cleaner and more about who gets paged when an endpoint changes. The useful documentation artifact is therefore a state machine: requested, DNS observed, verified, certificate ready, and serving. Do not collapse those states into a green “connected” badge.

## Should customer domains support apex records or www?

Start with one explicit input, such as `pay.example.com` or `example.com`, rather than silently claiming both. Normalize case, remove a trailing dot for storage, and keep the original value for audit display. Then show the exact record shape expected by the platform and which party owns changes to it.

For `www`, the customer can point the hostname at a platform-controlled DNS name. That extra naming layer lets the platform change the addresses behind its own name without asking every tenant to edit DNS. An apex address record has no such indirection: the documented addresses become part of the platform's customer-facing contract, so address rotation requires coordination and a deliberate overlap window. This approach has a real limitation: a `www`-only application does not make the bare domain work by itself. If customers must use the apex directly, the platform needs to operate apex support or the customer needs a separately managed forwarding service. Neither choice is free of ownership.

Short answer: choose `www` when minimizing lifecycle coupling matters most. Choose apex support when the bare domain is a real product requirement and the team is prepared to operate its address contract, redirect behavior, certificate issuance, and verification states as one workflow.

## Walk the hostname through Node.js

The request path is small on purpose. DNS gets traffic to an ingress, TLS proves the requested name, and the application maps the normalized `Host` value to a tenant. The database record also stores the intended canonical hostname, so redirect policy is data rather than scattered conditionals.

```ts
type DomainState = "requested" | "dns_observed" | "verified" | "certificate_ready" | "serving";

type TenantDomain = {
  tenantId: string;
  hostname: string;
  canonicalHostname: string;
  state: DomainState;
};

function normalizeHostname(value: string): string {
  return value.trim().toLowerCase().replace(/\.$/, "");
}

function resolveTenant(
  rawHost: string | undefined,
  domains: ReadonlyMap<string, TenantDomain>,
): { tenantId: string; redirectTo?: string } | undefined {
  if (!rawHost) return undefined;

  const hostname = normalizeHostname(rawHost.split(":", 1)[0]);
  const domain = domains.get(hostname);
  if (!domain || domain.state !== "serving") return undefined;

  if (hostname !== domain.canonicalHostname) {
    return { tenantId: domain.tenantId, redirectTo: `https://${domain.canonicalHostname}` };
  }

  return { tenantId: domain.tenantId };
}
```

This example deliberately does not infer a tenant from a suffix or trust an arbitrary forwarded host. The ingress should pass an already validated hostname under a defined proxy trust policy, and the application should require an exact entry in its domain map. Unknown names fail closed. That matters in fintech because a friendly fallback page can turn a configuration mistake into a convincing page on the wrong tenant.

Verification must also be separate from routing. A DNS lookup can establish that the expected record is visible; it does not prove that the certificate is ready or that every resolver has the same cached view. Serve a tenant only after the complete state reaches `serving`, and make retries idempotent so an onboarding worker can resume after a timeout without creating a second domain claim.

## The coupling belongs in the runbook

The comparison becomes concrete when the documentation names the change owner and failure boundary.

| Concern | Apex address record | `www` naming target |
|---|---|---|
| Platform address change | Customer DNS may need coordinated edits | Platform updates addresses behind its target name |
| Canonical URL | Bare domain can remain canonical | `www` can be canonical; apex forwarding is separate |
| Verification | Observe the documented address set | Observe the documented target name |
| Incident ownership | Shared across platform and customer zone | Platform owns its target; customer owns the pointing record |
| Rollback | Keep old and new addresses valid during transition | Roll back the platform-controlled target |

The table is not a scorecard. Apex support moves work into address lifecycle management; `www` moves one visible character into URLs and may leave apex forwarding outside the application path. A tenant that uses its bare domain in statements, QR codes, or regulated customer communications may value that trade-off differently from a tenant launching a new payment portal. The `www` design is unsuitable when the bare domain is the contractual public identifier and an external redirect is prohibited; apex address records are a poor fit when the platform cannot maintain a stable address lifecycle. Those are hard boundaries, not setup preferences.

Five states and two ownership boundaries are enough to make the contract testable. Document them with examples, but keep volatile addresses out of prose that will linger in screenshots and tickets. Put current values in a machine-readable configuration surface, and put invariants in the durable guide: allowed hostname forms, verification meaning, redirect status, certificate state, timeout behavior, and who changes what during migration. A useful acceptance test starts with one tenant whose apex is canonical and one whose `www` name is canonical, exercises both aliases, then proves that crossing those records never changes the selected tenant. That concrete pair catches a class of mapping errors that a single happy-path domain cannot expose.

## Mail records make an apex mistake expensive

An apex is rarely dedicated to the web application. It can also carry mail-related DNS records. DMARC, for example, publishes policy at a `_dmarc` subdomain and defines organizational-domain processing for mail authentication. Web onboarding must leave unrelated records alone; “replace the zone” is not an acceptable setup instruction.

Keep the boundary narrow. Ask for only the record needed to route the chosen web hostname and a separate proof record if ownership verification requires one. Never tell a customer to delete records merely because they do not appear in a web-domain wizard. The cost of a sloppy instruction is not limited to a failed page load; it can alter another system's security posture.

This is the trap.

The support runbook should capture the queried name, record type, observed answer, resolver vantage point, verification timestamp, certificate state, and canonical redirect result. It should not copy the customer's whole zone into logs. That gives an operator enough evidence to distinguish stale DNS, an incorrect target, and an application mapping problem without collecting unrelated mail configuration.

## Operate the transition, not just setup

Before launch, test four paths independently: the canonical hostname, its optional alias, an unknown hostname, and a hostname that is verified but not yet serving. Check the redirect destination and status, confirm that query strings survive the redirect when the application needs them, and ensure an unknown host cannot select a default tenant. Test certificate readiness separately from DNS visibility.

For an apex address migration, publish the new destination through the existing operational process, allow an overlap in which both old and new addresses can serve the tenant, observe traffic at the old destination, and retire it only after the agreed migration condition is met. The exact waiting period belongs to the deployment plan because cached DNS behavior depends on the records and resolvers involved; a universal number in evergreen documentation would be false precision.

Track state transitions and their age, not a single success counter. Useful signals include domains stuck after DNS observation, certificate failures by reason, requests for unknown hosts, redirects by source and destination, and traffic still reaching a retiring address. These signals expose where responsibility sits while keeping the control plane vendor-neutral.

Cost follows the same boundary. Supporting apex records means maintaining stable ingress addresses or coordinating every change, plus monitoring both destinations during migrations. Supporting only `www` reduces that coordination but adds documentation and forwarding questions. **Pick the failure mode your small team can explain at 2 a.m.** For most automated tenant onboarding, that means `www` first and apex only as an explicit, fully operated capability, not a checkbox.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
