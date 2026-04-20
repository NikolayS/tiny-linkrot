# Reviewer A — Codex

## summary

The SSRF posture is materially better than earlier rounds, but the remaining gaps are in policy completeness and implementation semantics: redirect hops can escape host policy, the recommended strict-redaction mode suppresses real findings, output/header sizes are still unbounded, hostname-based internal exceptions are too permissive, and transport pinning is not tight enough unless connection reuse is explicitly constrained. The spec is also carrying more policy surface than a first secure release needs.

## missing-risk

- (major) Redirect handling is still under-specified from a policy perspective. §16.5 only requires the SSRF re-check on each hop; it does not say that `--allow-host` / `--deny-host` from §16.8 are re-applied to every redirected host. That lets an allowed seed URL bounce the checker onto a different public host that the operator meant to exclude, defeating destination scoping. Bind the full host-policy evaluation order to every hop, not just the initial URL.
- (major) The spec hardens control characters but not output or header size. There is no explicit cap on total response-header bytes, `Location` length, status text, warning text, or rendered URL length once a server is being probed. A hostile endpoint can still force large allocations or flood CI logs/artifacts even within the request timeout. Add hard byte limits for received headers and deterministic truncation rules for all rendered server-controlled fields.
- (major) `--allow-internal-host HOSTNAME` (§16.7, §18) is too broad for an SSRF escape hatch. A hostname allowance blesses whatever private addresses that name resolves to now or later, so DNS drift or rebinding turns a supposedly scoped exception into a moving target. For a safety-sensitive bypass, require IP/CIDR-based allowances for private-space access, or at minimum constrain hostname allowances to an explicitly approved private CIDR set.

## weak-implementation

- (major) `--strict-redaction` (§17.3) is recommended for CI, but it changes what gets probed: multiple distinct URLs collapse to one request and the first occurrence in scan order decides the on-the-wire target. That can hide a broken secret-bearing URL behind a healthy sibling and makes correctness depend on file ordering. Redaction should affect rendering only; any lossy dedup mode should be a separate explicitly accuracy-reducing option, not the recommended safety setting.
- (major) The pinning story in §16.4 depends on 'resolve once, connect to that exact IP', but the spec never constrains connection pooling / HTTP/2 origin coalescing / socket reuse. A client can satisfy the API seam while reusing an existing TLS connection for a later hop or origin, which breaks the one-resolution-one-connect invariant the SSRF defense relies on. Specify fresh-socket-per-hop behavior or explicitly disable cross-host reuse and coalescing in the transport layer.

## unnecessary-scope

- (minor) For a 'small Bun CLI', v0.4 is carrying a large safety-policy DSL and transport platform surface: host-match prefixes (`=`, `.`, `*.`), port-scoped rules, CIDR on one flag, strict redaction heuristics, per-host backpressure, multiple budgets, and a Bun/Node hybrid pinned transport. That scope expansion is itself an ops risk because it multiplies parser, testing, and startup-failure paths before the core checker exists. Cut the first secure release to the minimum safe set and defer the matching DSL and lossy redaction mode.

## suggested-next-version

Narrow the next revision around a smaller, stricter core: re-apply full host policy on every redirect hop, make redaction render-only, require IP/CIDR-based internal allowances, add explicit header/output byte caps and truncation, and specify fresh pinned sockets per hop with no cross-host reuse/coalescing. Defer the host-matching DSL and the lossy strict-redaction dedup behavior until the basic secure transport path is implemented and proven.

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "missing-risk",
      "text": "Redirect handling is still under-specified from a policy perspective. §16.5 only requires the SSRF re-check on each hop; it does not say that `--allow-host` / `--deny-host` from §16.8 are re-applied to every redirected host. That lets an allowed seed URL bounce the checker onto a different public host that the operator meant to exclude, defeating destination scoping. Bind the full host-policy evaluation order to every hop, not just the initial URL.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "`--strict-redaction` (§17.3) is recommended for CI, but it changes what gets probed: multiple distinct URLs collapse to one request and the first occurrence in scan order decides the on-the-wire target. That can hide a broken secret-bearing URL behind a healthy sibling and makes correctness depend on file ordering. Redaction should affect rendering only; any lossy dedup mode should be a separate explicitly accuracy-reducing option, not the recommended safety setting.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "The spec hardens control characters but not output or header size. There is no explicit cap on total response-header bytes, `Location` length, status text, warning text, or rendered URL length once a server is being probed. A hostile endpoint can still force large allocations or flood CI logs/artifacts even within the request timeout. Add hard byte limits for received headers and deterministic truncation rules for all rendered server-controlled fields.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "`--allow-internal-host HOSTNAME` (§16.7, §18) is too broad for an SSRF escape hatch. A hostname allowance blesses whatever private addresses that name resolves to now or later, so DNS drift or rebinding turns a supposedly scoped exception into a moving target. For a safety-sensitive bypass, require IP/CIDR-based allowances for private-space access, or at minimum constrain hostname allowances to an explicitly approved private CIDR set.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "The pinning story in §16.4 depends on 'resolve once, connect to that exact IP', but the spec never constrains connection pooling / HTTP/2 origin coalescing / socket reuse. A client can satisfy the API seam while reusing an existing TLS connection for a later hop or origin, which breaks the one-resolution-one-connect invariant the SSRF defense relies on. Specify fresh-socket-per-hop behavior or explicitly disable cross-host reuse and coalescing in the transport layer.",
      "severity": "major"
    },
    {
      "category": "unnecessary-scope",
      "text": "For a 'small Bun CLI', v0.4 is carrying a large safety-policy DSL and transport platform surface: host-match prefixes (`=`, `.`, `*.`), port-scoped rules, CIDR on one flag, strict redaction heuristics, per-host backpressure, multiple budgets, and a Bun/Node hybrid pinned transport. That scope expansion is itself an ops risk because it multiplies parser, testing, and startup-failure paths before the core checker exists. Cut the first secure release to the minimum safe set and defer the matching DSL and lossy redaction mode.",
      "severity": "minor"
    }
  ],
  "summary": "The SSRF posture is materially better than earlier rounds, but the remaining gaps are in policy completeness and implementation semantics: redirect hops can escape host policy, the recommended strict-redaction mode suppresses real findings, output/header sizes are still unbounded, hostname-based internal exceptions are too permissive, and transport pinning is not tight enough unless connection reuse is explicitly constrained. The spec is also carrying more policy surface than a first secure release needs.",
  "suggested_next_version": "Narrow the next revision around a smaller, stricter core: re-apply full host policy on every redirect hop, make redaction render-only, require IP/CIDR-based internal allowances, add explicit header/output byte caps and truncation, and specify fresh pinned sockets per hop with no cross-host reuse/coalescing. Defer the host-matching DSL and the lossy strict-redaction dedup behavior until the basic secure transport path is implemented and proven.",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
