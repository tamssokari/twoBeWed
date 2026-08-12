# Security, privacy, and compliance

## Decided (pilot)

| Topic | Decision |
|---|---|
| **Compliance lens** | **Both PIPEDA (Canada) and GDPR** — design for the stricter applicable control where they overlap |
| **Authn** | **SSO and/or passwordless out of the gate** — do **not** roll a custom MFA implementation; prefer IdP-managed auth (e.g. OIDC SSO, magic link / WebAuthn / passkeys via a provider) |
| **Product ownership / hosting** | This is a **machineaid product**; machineaid operates staging and production |

## Authn stance

### Prefer

- OIDC SSO (Google / Microsoft / enterprise IdP as needed)
- Passwordless (magic link and/or passkeys) through a reputable auth vendor or platform feature
- Short-lived access tokens + refresh; revoke on logout and risk events
- MFA **as provided by the IdP / passwordless factor**, not a bespoke TOTP/SMS stack in our domain

### Avoid (pilot)

- Homegrown password + homegrown MFA
- Long-lived API keys in the browser
- Sync/auth bypass that skips server re-validation on apply

Offline: device may use a cached session with expiry; sensitive money commands may require recent online auth (policy knob later).

## Privacy (PIPEDA + GDPR alignment)

Minimum bar for guest/logistics/money PII:

| Control | Approach |
|---|---|
| Lawful basis / purpose | Ops for contracted event delivery; document purposes in privacy notice |
| Data minimization | Collect what planners need for logistics/billing; policy can hide fields by event type |
| Access control | RBAC + row scope (host/guest later); money visibility via policy |
| Retention | Configurable retention (esp. audit + guest PII) post-event; Hypeluxe default TBD in policy pack |
| Export / delete | Workspace admin flows for subject access / deletion where legally required; audit may retain redacted stubs |
| Subprocessors | Auth provider, hosting, DB — listed in machineaid DPA / privacy docs |
| Cross-border | machineaid hosting region(s) documented; SCCs / appropriate transfer tools if EU personal data leaves EEA |
| Breach | machineaid incident process; notify per PIPEDA/GDPR timelines |

Exact legal copy and DPA templates are **ops/legal artifacts**, not domain code — but product must support retention, export, and access constraints.

## Hosting & operations

- **machineaid** owns the multi-tenant SaaS (Hypeluxe is a customer/tenant, not the host)
- Staging and prod under machineaid accounts
- Secrets via managed secret store; no secrets in tenant packs or client bundles
- Backups encrypted; restore drill on CD roadmap (not yet scheduled)

## Threat themes (initial)

1. Stolen device with offline local DB (encrypt at rest where platform allows; remote session revoke)
2. Sync replay / forged commands (authz + correlationId idempotency + server FSM re-check)
3. Privilege escalation across workspaces (strict `workspaceId` on every command)
4. PII leakage in logs/traces (redaction; ids not emails in OTel)
5. Invoice/payment tampering (FSM + audit; issued invoice immutability)

## Explicitly out of scope for this doc

- Formal SOC2 cert timeline (may follow; not a pilot blocker unless sales requires)
- Customer-managed encryption keys (CMEK) — later if enterprise needs

## Related

- [identity-and-access.md](./identity-and-access.md) — roles and authz
- [audit-and-attribution.md](./audit-and-attribution.md) — business attribution
- [platform-engineering.md](./platform-engineering.md) — observability correlation
- [decisions.md](./decisions.md) — decided vs open product calls
