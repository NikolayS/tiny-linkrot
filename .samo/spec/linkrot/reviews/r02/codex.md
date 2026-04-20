# Reviewer A — Codex

## summary

The main problem is not feature coverage; it is that the safety boundary is under-specified at exactly the places attackers exploit: transport resolution, proxy mediation, filesystem traversal, and total-work containment. As written, the spec claims stronger SSRF/CI hardening than the implementation constraints currently justify.

## weak-implementation

- (major) §16's SSRF control is not implementable safely as written unless the actual TCP/TLS connection is pinned to the already-vetted IP address. If the implementation resolves the host, approves the result, and then hands the hostname to Bun `fetch`, the runtime can re-resolve at connect time or on redirect hops, reintroducing DNS-rebinding/TOCTOU bypasses. The spec needs an explicit transport requirement that dials validated addresses directly while preserving Host/SNI semantics, or the 'blocked destination' guarantee is not real.

## missing-risk

- (major) Proxy behavior is undefined even though this is meant for CI. If Bun honors `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`, requests may bypass direct-destination checks, leak probed URLs to proxy infrastructure, and make SSRF/per-host accounting meaningless. 'Proxies are a non-goal' is not enough here; the spec needs an explicit allow/deny policy and tests for proxy-env interaction.
- (major) Recursive scan semantics do not say what happens with symlinks/reparse points. In an untrusted repo, a symlinked directory can escape the requested root, create scan loops, or drag in large/sensitive host paths; that is an ops and data-exposure risk, not just a performance issue. The spec should require root confinement and no-follow behavior by default.
- (major) The design bounds concurrency but not total work. Because extraction completes before probing and there is no cap on files, unique URLs, URL length, redirect metadata retained, or DNS answer fan-out, a malicious docs change can still DoS CI through memory growth and extreme wall-clock work while remaining within the per-request timeout model. A hardened spec needs explicit work-budget limits and fail-fast behavior.
- (minor) §17 uses a fixed denylist of sensitive query-parameter names for redaction. That is brittle for CI logs: site-specific secrets, OAuth codes/state values, signed path segments, and other opaque bearer material can still be emitted to stdout/stderr and artifacts. The safer default is a stricter rendering mode that never prints raw query values unless explicitly requested.

## unnecessary-scope

- (major) `--allow-internal` is an all-or-nothing escape hatch that disables the main SSRF safeguard for the entire run. In practice, one legitimate internal link would force operators to reopen access to every attacker-controlled link in the same Markdown set. That is broader than necessary; a scoped internal allowlist (host/CIDR/path root) would satisfy the use case without dropping the safety boundary globally.

## suggested-next-version

Define a hardening-only next revision: require IP-pinned/no-proxy transport semantics, root-confined non-symlink scanning, explicit work-budget caps, and replace global `--allow-internal` plus denylist redaction with narrower, safer controls.

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "weak-implementation",
      "text": "§16's SSRF control is not implementable safely as written unless the actual TCP/TLS connection is pinned to the already-vetted IP address. If the implementation resolves the host, approves the result, and then hands the hostname to Bun `fetch`, the runtime can re-resolve at connect time or on redirect hops, reintroducing DNS-rebinding/TOCTOU bypasses. The spec needs an explicit transport requirement that dials validated addresses directly while preserving Host/SNI semantics, or the 'blocked destination' guarantee is not real.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "Proxy behavior is undefined even though this is meant for CI. If Bun honors `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`, requests may bypass direct-destination checks, leak probed URLs to proxy infrastructure, and make SSRF/per-host accounting meaningless. 'Proxies are a non-goal' is not enough here; the spec needs an explicit allow/deny policy and tests for proxy-env interaction.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "Recursive scan semantics do not say what happens with symlinks/reparse points. In an untrusted repo, a symlinked directory can escape the requested root, create scan loops, or drag in large/sensitive host paths; that is an ops and data-exposure risk, not just a performance issue. The spec should require root confinement and no-follow behavior by default.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "The design bounds concurrency but not total work. Because extraction completes before probing and there is no cap on files, unique URLs, URL length, redirect metadata retained, or DNS answer fan-out, a malicious docs change can still DoS CI through memory growth and extreme wall-clock work while remaining within the per-request timeout model. A hardened spec needs explicit work-budget limits and fail-fast behavior.",
      "severity": "major"
    },
    {
      "category": "unnecessary-scope",
      "text": "`--allow-internal` is an all-or-nothing escape hatch that disables the main SSRF safeguard for the entire run. In practice, one legitimate internal link would force operators to reopen access to every attacker-controlled link in the same Markdown set. That is broader than necessary; a scoped internal allowlist (host/CIDR/path root) would satisfy the use case without dropping the safety boundary globally.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "§17 uses a fixed denylist of sensitive query-parameter names for redaction. That is brittle for CI logs: site-specific secrets, OAuth codes/state values, signed path segments, and other opaque bearer material can still be emitted to stdout/stderr and artifacts. The safer default is a stricter rendering mode that never prints raw query values unless explicitly requested.",
      "severity": "minor"
    }
  ],
  "summary": "The main problem is not feature coverage; it is that the safety boundary is under-specified at exactly the places attackers exploit: transport resolution, proxy mediation, filesystem traversal, and total-work containment. As written, the spec claims stronger SSRF/CI hardening than the implementation constraints currently justify.",
  "suggested_next_version": "Define a hardening-only next revision: require IP-pinned/no-proxy transport semantics, root-confined non-symlink scanning, explicit work-budget caps, and replace global `--allow-internal` plus denylist redaction with narrower, safer controls.",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
