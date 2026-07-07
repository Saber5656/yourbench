# ADR-005: Local-only single-user security posture

Date: 2026-07-08
Status: Accepted

## Context

mybench stores the user's *real* prompts — potentially the most sensitive text
on the machine — plus references to paid API keys. Owner selected fully local,
single-user operation (Q7-A) and human-only voting (Q8-A). The project will be
OSS; the security model must be explicit so contributors don't erode it.
Threat model details: DESIGN.md §13.

## Decision

1. **Network exposure:** the web server binds `127.0.0.1` only. There is no
   flag to bind other interfaces in v1. Team/remote use is explicitly v2+ and
   will require authentication design first.
2. **Browser-boundary defenses ship in v1** even though the server is
   loopback-only, because *other local processes and hostile web pages* can
   reach loopback: Host-header allowlist (DNS-rebinding defense), Origin check
   + double-submit CSRF token on every state-changing request, strict CSP with
   no inline scripts, `X-Content-Type-Options`, `X-Frame-Options: DENY`,
   `Referrer-Policy: no-referrer`.
3. **Untrusted content rule:** every model output is treated as an attack
   payload — sanitized markdown rendering in the browser (raw HTML disabled +
   nh3 allowlist) and ANSI-escape stripping before terminal display.
4. **No telemetry, ever.** The only network connections the process makes are
   to explicitly configured provider base URLs. The web UI loads no external
   resources (fonts, CDNs, analytics).
5. **Data at rest:** data dir `0700`, DB file `0600`. No encryption at rest in
   v1 (OS user boundary + FileVault-class disk encryption assumed); documented
   as a limitation.
6. **Secrets:** per ADR-004, env-var references only; redaction filter in
   logging; exports contain no key material by construction.

## Consequences

- "Local-only" is enforced by code and tests, not by documentation promises;
  issues 22/32 carry the acceptance criteria.
- Multi-user features cannot be bolted on casually — lifting the bind
  restriction requires a new ADR with an auth design.
- Users on shared machines rely on OS user isolation; a same-user attacker is
  out of scope by definition.
