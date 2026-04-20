# Reviewer B — Claude

## summary

v0.2 has solid bones but ships several material contradictions (timeout scope, HEAD→GET vs retry policy, v0.1/v0.2 labeling) and ambiguities that will produce divergent implementations (--include/--exclude merge semantics, --allow-host matching, Retry-After interaction with --timeout, fan-out rendering, redirect-loop enforcement under Bun's fetch). The testing strategy is too thin: golden snapshots will be non-deterministic due to embedded durations, several spec'd behaviors (SIGINT, retry/backoff, HEAD→GET, env precedence, network-error classes) lack required test coverage, and the 'no live network' constraint is incompatible with simulating DNS/TLS/RST failures without an injectable transport. Resolving these before v0.3 will substantially reduce implementation drift and CI flakiness.

## contradiction

- (major) Title says SPEC v0.2 but §2 and §3 still scope features as 'Out of scope for v0.1' / 'No runtime deps... for v0.1' and §4 labels 'Options (v0.1)'. Either bump all internal references to v0.2 or restate explicitly which v0.1 constraints carry forward and which were relaxed in v0.2.
- (major) §6.1 step 2 prescribes a HEAD→GET fallback when the final status is 405/403/400, but §6.3 says 'Retries never apply to 4xx other than 408, 425, 429.' Whether the GET fallback is a 'retry' (and counts against --retries) is unspecified and the two clauses read as mutually exclusive for 400/403/405. Clarify that the GET fallback is a method-switch, not a retry, and whether it consumes a retry budget slot.
- (major) §4 defines --timeout as 'Per-request timeout (connect + read) in ms', but §6.1 step 4 says 'Apply --timeout to the whole request including all redirect hops.' Per-hop and whole-chain budgets behave very differently under slow redirects. Pick one and align both sections.
- (minor) §7 states 'File scanning and URL extraction run to completion before probing starts, so the total URL count is known and the progress display is accurate', but §12 says malformed Markdown emits a stderr warning and continues. If extraction encounters a fatal parser error mid-scan, does the run abort (exit 2/3) or proceed with a partially-extracted URL set whose 'total' is therefore inaccurate? Reconcile.

## ambiguity

- (major) §6.2 references reporting skipped:scheme 'only in --help-level verbose mode; otherwise filtered silently', but no --verbose flag is defined in §4 and --help merely prints help and exits. Either add the flag or remove the dangling reference and state how scheme skips are surfaced (if at all).
- (major) §4 does not specify whether repeated --include/--exclude flags replace the built-in defaults or append to them. With defaults of **/*.md, **/*.markdown and **/node_modules/**, a user passing --include docs/**/*.md will get surprising behavior either way. Define merge semantics explicitly.
- (major) §4 --allow-host/--deny-host take a HOST value but do not say whether matching is exact, case-insensitive, or includes subdomains (e.g., does --deny-host example.com match api.example.com?). Also unspecified: whether the value may be a glob or include a port. This directly affects classification, so it must be pinned down.
- (major) §6.3 'Retry-After header respect that header up to --timeout, counted against --retries' is unclear on three points: (a) is --timeout the per-request budget or a wall-clock budget for the URL, (b) what happens if Retry-After exceeds --timeout (skip the wait, fail immediately, or cap the wait), (c) does waiting for Retry-After itself consume part of the --timeout budget for the next attempt.
- (major) §12 classifies new URL() parse failures as 'broken:network with reason invalid-url'. This conflates a static parse error with network failure, will pollute network-error metrics, and contradicts §6.2 which defines broken:network as 'DNS failure, connection refused, TLS error, reset, timeout after retries.' Introduce a dedicated broken:invalid-url class or otherwise reconcile.
- (major) §8.1 example shows a per-file collapsed line 'README.md  ✓ 14 ok, 0 redirects, 0 broken', but §5 says a URL appearing in multiple files is probed once and 'fanned out to all source locations.' How fan-out is rendered (each file's expanded section repeats the finding? a single section with all locations? counted once or N times in the summary's 'links checked / occurrences' totals?) is not shown by the example.
- (minor) §6.3 specifies 'Exponential backoff: 200ms × 2^n, jittered ±25%' but does not say whether n starts at 0 (first retry waits 200ms) or 1 (first retry waits 400ms), nor whether jitter is uniform or full-jitter, nor whether the cap is --timeout. These are observable behaviors that tests will need to assert.
- (major) §6.1 step 1 uses fetch with redirect: 'follow', but step 3 caps redirects at 10. Bun's fetch enforces its own internal redirect cap and does not surface the chain. The spec should state that the implementation must use redirect: 'manual' (or equivalent) to count hops, detect cycles, and enforce the 10-hop limit deterministically — otherwise broken:redirect-loop cannot be reliably triggered.
- (minor) §8.1 summary line 'skipped: 0' aggregates skipped:host and skipped:scheme, but §6.2 says scheme skips are 'filtered silently.' Whether silently-filtered URLs are excluded from 'links checked', counted in 'skipped', or invisible everywhere needs to be explicit so the totals reconcile.
- (minor) §10 lists five env vars but omits obvious analogues for flags that materially affect results: LINKROT_INCLUDE, LINKROT_EXCLUDE, LINKROT_ALLOW_HOST, LINKROT_DENY_HOST, LINKROT_FAIL_ON, LINKROT_ACCEPT_REDIRECTS. Either state that these are intentionally CLI-only or extend the list; otherwise CI users will hit a surprising gap.
- (minor) §5 says image links '![alt](url)' are checked, but does not specify behavior for image URLs using data: scheme (silently ignored as non-HTTP, presumably) nor whether referenced image definitions '![alt][ref]' are also handled. Spell this out alongside the four bullet points.

## weak-testing

- (major) §13 'Golden report tests: snapshot the TTY report against fixture directories' will be flaky because the report includes 'duration: 4.1s' (§8.1) and the progress counter is time-dependent. The spec should require the implementation to expose a deterministic mode (e.g., LINKROT_FREEZE_TIME, redacted duration in golden output, or an explicit --no-timing flag for tests).
- (major) §13 lists unit, integration, and golden tests but does not require coverage for several behaviors that the spec calls out as critical: SIGINT/SIGTERM cancellation and exit code 130 (§7), redirect-loop and >10-hop detection (§6.1), Retry-After parsing including delta-seconds vs HTTP-date and oversize values (§6.3), HEAD→GET method switch for 400/403/405 (§6.1), env-var precedence (§10), --allow-host/--deny-host filtering, and concurrency cap enforcement under load. Each should be an explicit required test case.
- (major) §13 says 'No live network in CI' and uses a local Bun HTTP fixture, but several broken:network triggers (DNS NXDOMAIN, TCP RST mid-headers, TLS handshake error) cannot be produced by an in-process HTTP server. Spec should either require an injectable transport/fetch seam so these failure modes can be unit-tested, or accept that broken:network reasons go untested and document that gap.

## suggested-next-version

v0.3

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "contradiction",
      "text": "Title says SPEC v0.2 but §2 and §3 still scope features as 'Out of scope for v0.1' / 'No runtime deps... for v0.1' and §4 labels 'Options (v0.1)'. Either bump all internal references to v0.2 or restate explicitly which v0.1 constraints carry forward and which were relaxed in v0.2.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "§6.1 step 2 prescribes a HEAD→GET fallback when the final status is 405/403/400, but §6.3 says 'Retries never apply to 4xx other than 408, 425, 429.' Whether the GET fallback is a 'retry' (and counts against --retries) is unspecified and the two clauses read as mutually exclusive for 400/403/405. Clarify that the GET fallback is a method-switch, not a retry, and whether it consumes a retry budget slot.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "§4 defines --timeout as 'Per-request timeout (connect + read) in ms', but §6.1 step 4 says 'Apply --timeout to the whole request including all redirect hops.' Per-hop and whole-chain budgets behave very differently under slow redirects. Pick one and align both sections.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§6.2 references reporting skipped:scheme 'only in --help-level verbose mode; otherwise filtered silently', but no --verbose flag is defined in §4 and --help merely prints help and exits. Either add the flag or remove the dangling reference and state how scheme skips are surfaced (if at all).",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§4 does not specify whether repeated --include/--exclude flags replace the built-in defaults or append to them. With defaults of **/*.md, **/*.markdown and **/node_modules/**, a user passing --include docs/**/*.md will get surprising behavior either way. Define merge semantics explicitly.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§4 --allow-host/--deny-host take a HOST value but do not say whether matching is exact, case-insensitive, or includes subdomains (e.g., does --deny-host example.com match api.example.com?). Also unspecified: whether the value may be a glob or include a port. This directly affects classification, so it must be pinned down.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§6.3 'Retry-After header respect that header up to --timeout, counted against --retries' is unclear on three points: (a) is --timeout the per-request budget or a wall-clock budget for the URL, (b) what happens if Retry-After exceeds --timeout (skip the wait, fail immediately, or cap the wait), (c) does waiting for Retry-After itself consume part of the --timeout budget for the next attempt.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§12 classifies new URL() parse failures as 'broken:network with reason invalid-url'. This conflates a static parse error with network failure, will pollute network-error metrics, and contradicts §6.2 which defines broken:network as 'DNS failure, connection refused, TLS error, reset, timeout after retries.' Introduce a dedicated broken:invalid-url class or otherwise reconcile.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§8.1 example shows a per-file collapsed line 'README.md  ✓ 14 ok, 0 redirects, 0 broken', but §5 says a URL appearing in multiple files is probed once and 'fanned out to all source locations.' How fan-out is rendered (each file's expanded section repeats the finding? a single section with all locations? counted once or N times in the summary's 'links checked / occurrences' totals?) is not shown by the example.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§6.3 specifies 'Exponential backoff: 200ms × 2^n, jittered ±25%' but does not say whether n starts at 0 (first retry waits 200ms) or 1 (first retry waits 400ms), nor whether jitter is uniform or full-jitter, nor whether the cap is --timeout. These are observable behaviors that tests will need to assert.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§6.1 step 1 uses fetch with redirect: 'follow', but step 3 caps redirects at 10. Bun's fetch enforces its own internal redirect cap and does not surface the chain. The spec should state that the implementation must use redirect: 'manual' (or equivalent) to count hops, detect cycles, and enforce the 10-hop limit deterministically — otherwise broken:redirect-loop cannot be reliably triggered.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§8.1 summary line 'skipped: 0' aggregates skipped:host and skipped:scheme, but §6.2 says scheme skips are 'filtered silently.' Whether silently-filtered URLs are excluded from 'links checked', counted in 'skipped', or invisible everywhere needs to be explicit so the totals reconcile.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "§13 'Golden report tests: snapshot the TTY report against fixture directories' will be flaky because the report includes 'duration: 4.1s' (§8.1) and the progress counter is time-dependent. The spec should require the implementation to expose a deterministic mode (e.g., LINKROT_FREEZE_TIME, redacted duration in golden output, or an explicit --no-timing flag for tests).",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§13 lists unit, integration, and golden tests but does not require coverage for several behaviors that the spec calls out as critical: SIGINT/SIGTERM cancellation and exit code 130 (§7), redirect-loop and >10-hop detection (§6.1), Retry-After parsing including delta-seconds vs HTTP-date and oversize values (§6.3), HEAD→GET method switch for 400/403/405 (§6.1), env-var precedence (§10), --allow-host/--deny-host filtering, and concurrency cap enforcement under load. Each should be an explicit required test case.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§13 says 'No live network in CI' and uses a local Bun HTTP fixture, but several broken:network triggers (DNS NXDOMAIN, TCP RST mid-headers, TLS handshake error) cannot be produced by an in-process HTTP server. Spec should either require an injectable transport/fetch seam so these failure modes can be unit-tested, or accept that broken:network reasons go untested and document that gap.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§10 lists five env vars but omits obvious analogues for flags that materially affect results: LINKROT_INCLUDE, LINKROT_EXCLUDE, LINKROT_ALLOW_HOST, LINKROT_DENY_HOST, LINKROT_FAIL_ON, LINKROT_ACCEPT_REDIRECTS. Either state that these are intentionally CLI-only or extend the list; otherwise CI users will hit a surprising gap.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§5 says image links '![alt](url)' are checked, but does not specify behavior for image URLs using data: scheme (silently ignored as non-HTTP, presumably) nor whether referenced image definitions '![alt][ref]' are also handled. Spell this out alongside the four bullet points.",
      "severity": "minor"
    },
    {
      "category": "contradiction",
      "text": "§7 states 'File scanning and URL extraction run to completion before probing starts, so the total URL count is known and the progress display is accurate', but §12 says malformed Markdown emits a stderr warning and continues. If extraction encounters a fatal parser error mid-scan, does the run abort (exit 2/3) or proceed with a partially-extracted URL set whose 'total' is therefore inaccurate? Reconcile.",
      "severity": "minor"
    }
  ],
  "summary": "v0.2 has solid bones but ships several material contradictions (timeout scope, HEAD→GET vs retry policy, v0.1/v0.2 labeling) and ambiguities that will produce divergent implementations (--include/--exclude merge semantics, --allow-host matching, Retry-After interaction with --timeout, fan-out rendering, redirect-loop enforcement under Bun's fetch). The testing strategy is too thin: golden snapshots will be non-deterministic due to embedded durations, several spec'd behaviors (SIGINT, retry/backoff, HEAD→GET, env precedence, network-error classes) lack required test coverage, and the 'no live network' constraint is incompatible with simulating DNS/TLS/RST failures without an injectable transport. Resolving these before v0.3 will substantially reduce implementation drift and CI flakiness.",
  "suggested_next_version": "v0.3",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
