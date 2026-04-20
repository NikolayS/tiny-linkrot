# Reviewer A — Codex

## summary

v0.3 improved the SSRF story, but several safety-critical rules still contradict each other or leave common attacker inputs underspecified. The biggest problems are the unusable scoped-internal allow flow, redaction/dedup behavior that can suppress real failures, non-failing treatment of direct internal URLs, and path-confinement wording that is easy to implement unsafely.

## weak-implementation

- (major) §§16.6, 16.8, and 18 make `--allow-internal-host` impossible as written. The declared evaluation order is `deny -> SSRF block -> scoped internal allow -> allow -> default`, but the scoped internal allow is supposed to bypass the SSRF block. An implementation that follows the binding order will never reach the scoped allow for blocked destinations.
- (major) §§17.2, 17.3, and 21 contradict each other on sensitive-parameter deduplication. Default mode says URLs differing only in sensitive query values collapse to one probe, but the tests then say `--strict-redaction=off` keeps them distinct even though `off` is already the default. Beyond the contradiction, collapsing by redacted query value can hide a genuinely broken URL behind a healthy sibling.
- (major) §22 says root confinement is enforced by checking whether a candidate real path has the root "as a prefix". If this is implemented as a raw string-prefix check, `/repo/docs-evil/file.md` passes for root `/repo/docs`. The spec needs component-aware containment semantics, not prefix wording.
- (major) §16 defines SSRF protection entirely in terms of DNS A/AAAA resolution before connect, but it does not define the required behavior for IP-literal URLs such as `127.0.0.1`, `[::1]`, or `169.254.169.254`. Those are common SSRF inputs and must be classified directly without resolver-dependent behavior.
- (major) §12 says a file read error aborts the run with exit 2 and no partial report, but §20 later says a fatal single-file failure, explicitly including I/O failure and decoder failure, is treated as a skip and the rest of the run continues. That is a control-plane contradiction with materially different CI behavior.
- (minor) §18 makes bare `example.com` match the apex and all subdomains. For safety-sensitive allow/deny flags, that default is too broad and easy to misuse; exact-match behavior should be the default, with suffix matching requiring explicit syntax.

## missing-risk

- (major) §16.10 classifies direct private/internal destinations as `skipped:blocked-destination`, so the default `--fail-on=broken` will not fail CI when a repo introduces `http://127.0.0.1/...`, `169.254.169.254`, or RFC1918 links. Treating redirects-to-private as broken while treating direct private targets as skipped is the wrong asymmetry for a security-focused checker.

## unnecessary-scope

- (minor) Proxy support in §16.7 adds a large exception surface to a release whose main hardening story is transport-pinned SSRF defense. In proxy mode the core guarantee is explicitly weakened, CONNECT-only support is narrowly interoperable, and the implementation burden is high. This is better deferred until the non-proxy transport is proven.

## suggested-next-version

Resolve the binding contradictions before implementation, then narrow the release: make direct internal destinations fail by default, specify literal-IP handling and path containment precisely, remove sensitive-value dedup from default behavior, and defer proxy mode until the pinned-transport path is demonstrably solid.

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "weak-implementation",
      "text": "§§16.6, 16.8, and 18 make `--allow-internal-host` impossible as written. The declared evaluation order is `deny -> SSRF block -> scoped internal allow -> allow -> default`, but the scoped internal allow is supposed to bypass the SSRF block. An implementation that follows the binding order will never reach the scoped allow for blocked destinations.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "§§17.2, 17.3, and 21 contradict each other on sensitive-parameter deduplication. Default mode says URLs differing only in sensitive query values collapse to one probe, but the tests then say `--strict-redaction=off` keeps them distinct even though `off` is already the default. Beyond the contradiction, collapsing by redacted query value can hide a genuinely broken URL behind a healthy sibling.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "§16.10 classifies direct private/internal destinations as `skipped:blocked-destination`, so the default `--fail-on=broken` will not fail CI when a repo introduces `http://127.0.0.1/...`, `169.254.169.254`, or RFC1918 links. Treating redirects-to-private as broken while treating direct private targets as skipped is the wrong asymmetry for a security-focused checker.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "§22 says root confinement is enforced by checking whether a candidate real path has the root \"as a prefix\". If this is implemented as a raw string-prefix check, `/repo/docs-evil/file.md` passes for root `/repo/docs`. The spec needs component-aware containment semantics, not prefix wording.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "§16 defines SSRF protection entirely in terms of DNS A/AAAA resolution before connect, but it does not define the required behavior for IP-literal URLs such as `127.0.0.1`, `[::1]`, or `169.254.169.254`. Those are common SSRF inputs and must be classified directly without resolver-dependent behavior.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "§12 says a file read error aborts the run with exit 2 and no partial report, but §20 later says a fatal single-file failure, explicitly including I/O failure and decoder failure, is treated as a skip and the rest of the run continues. That is a control-plane contradiction with materially different CI behavior.",
      "severity": "major"
    },
    {
      "category": "unnecessary-scope",
      "text": "Proxy support in §16.7 adds a large exception surface to a release whose main hardening story is transport-pinned SSRF defense. In proxy mode the core guarantee is explicitly weakened, CONNECT-only support is narrowly interoperable, and the implementation burden is high. This is better deferred until the non-proxy transport is proven.",
      "severity": "minor"
    },
    {
      "category": "weak-implementation",
      "text": "§18 makes bare `example.com` match the apex and all subdomains. For safety-sensitive allow/deny flags, that default is too broad and easy to misuse; exact-match behavior should be the default, with suffix matching requiring explicit syntax.",
      "severity": "minor"
    }
  ],
  "summary": "v0.3 improved the SSRF story, but several safety-critical rules still contradict each other or leave common attacker inputs underspecified. The biggest problems are the unusable scoped-internal allow flow, redaction/dedup behavior that can suppress real failures, non-failing treatment of direct internal URLs, and path-confinement wording that is easy to implement unsafely.",
  "suggested_next_version": "Resolve the binding contradictions before implementation, then narrow the release: make direct internal destinations fail by default, specify literal-IP handling and path containment precisely, remove sensitive-value dedup from default behavior, and defer proxy mode until the pinned-transport path is demonstrably solid.",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
