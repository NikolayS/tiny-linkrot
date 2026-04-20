# Reviewer B — Claude

## summary

v0.2 adds substantial new safety surface (SSRF, redaction, host canonicalization, per-host caps) but introduces several ambiguities and at least one direct contradiction with the frozen text. The most load-bearing issues: (1) §4 vs §20 backoff formulas disagree; (2) the new SSRF classes are not wired into the §6.2 classification table, summary accounting, or `--fail-on` semantics; (3) §18's `host:port` syntax collides with bracketless IPv6 literals; (4) §17 leaves dedup behavior for sensitive query params undefined; (5) the SSRF mechanism rests on an unspecified DNS-resolution seam separate from Bun's fetch. §21 also misses tests for the new failure classes and for `--allow-host`/SSRF interaction. Resolving these in v0.3 should not require touching frozen sections except via a clarifications-style override.

## contradiction

- (major) §4 specifies retry backoff as `200ms × 2^n, jittered ±25%`, while §20 redefines it as `200ms × 2^(n-1)` with **full** jitter in `[0, base]`. These yield different first-retry distributions (e.g., n=1 → 400ms ±25% vs uniform `[0,200ms]`). §20 is presented as a clarification but actually changes both the base curve and the jitter shape, so implementers will not know which to honor.
- (major) §16 introduces two new classes (`skipped:blocked-destination` and `broken:blocked-destination-redirect`) but they are not added to the §6.2 classification table, not mapped into the summary buckets in §8.1 (`ok / redirects / broken / skipped`), and their `--fail-on` semantics are unstated. Result: it is unclear whether a blocked-destination-redirect counts toward `broken` for exit-code purposes, and whether `skipped:blocked-destination` increments the `skipped` summary line.

## ambiguity

- (major) §18's `host:port` port-scoped syntax collides with IPv6 literals (which contain colons and are passed without brackets per the same section). E.g., `--deny-host ::1:443` is ambiguous: host `::1` port `443`, or host `::1:443`? No parsing rule is given, and brackets are explicitly forbidden on the CLI.
- (major) §17 says the original (non-redacted) URL is used for probing while sensitive query values are replaced for display, but it does not state whether dedup/normalization in §5 collapses URLs that differ only in sensitive query values. Userinfo is explicitly dropped for dedup; sensitive params are not addressed. This determines whether `?token=A` and `?token=B` to the same host/path are one probe or two.
- (major) §16 mandates a pre-connect DNS resolution by linkrot itself, but Bun's `fetch` performs its own resolution internally. The spec does not define the implementation seam (custom dispatcher, undici-style connect hook, low-level socket?) nor what happens if linkrot's resolution and the resolution used by the actual TCP connect diverge (TTL expiry, round-robin reorder, NAT64). This is the core SSRF guarantee and currently rests on an unspecified mechanism.
- (major) §16 does not specify the classification when the SSRF-stage DNS resolution itself fails (NXDOMAIN, SERVFAIL, timeout). Is it `broken:network`, `skipped:blocked-destination`, or something else? §6.2 lists DNS failure under `broken:network`, but §16's pre-connect resolution is a distinct step.
- (major) §19 declares per-host accounting `port-agnostic`, but §18 supports `host:port` entries for `--deny-host`/`--allow-host`. The interaction is unstated: does a `host:443` deny share any state with the per-host cap accounting for `host:8080`? Also, what host identity is used when the request is redirected from host A to host B mid-chain — does the cap of A, B, or both gate the next hop?
- (minor) §16 says `the final URL is shown truncated at the hop that was refused`, but `truncated` is not defined. Is the displayed URL the last public URL in the chain, the refused (private) target with its path elided, or only the scheme+host? Golden tests cannot be written without this.
- (minor) §20's `files scanned counts only fully-extracted files` interacts unclearly with §12's recovery mode for malformed Markdown. A file partially recovered (some regions skipped, others extracted) is presumably `scanned`, while a fatal parser failure is `skipped`, but the threshold between recoverable warning and fatal failure is left to the parser. State the rule explicitly.
- (minor) §19 says a `429`/`503` with `Retry-After` reduces a host's effective cap to 1 for the remainder of the run, but does not specify behavior for `429`/`503` *without* `Retry-After`, nor whether the cap reduction is reset on subsequent successful responses, nor whether it applies when the existing per-host cap is already 1.
- (minor) §17's control-character list omits U+2028 (LINE SEPARATOR) and U+2029 (PARAGRAPH SEPARATOR), both of which can disrupt terminal/CI log rendering and JSON serialization. Either include them or state explicitly that they are intentionally allowed through.
- (minor) §16's per-hop SSRF re-check does not state DNS-cache semantics: must each hop perform a fresh resolution, or may a TTL-bound cache shared across the run be used? This affects both performance and the rebinding threat model the section is designed to defeat.
- (minor) §17 says redaction applies uniformly to all rendered URLs, but does not clarify whether the rendered host (post-IDNA, possibly Punycode) is what users see, or whether Unicode is restored for display. §18 mandates Punycode for *matching*; the *rendering* form is unspecified, making spoofing-via-confusables possible in the report.
- (minor) §16 strips userinfo from the request, but the default `fetch` will encode `user:pass@` as a Basic `Authorization` header. The spec needs to state that the implementation must not just rely on calling `fetch` with the original URL — userinfo must be removed from the URL passed to the transport, otherwise the `never sent in Authorization headers` guarantee is violated by the runtime's default behavior.
- (minor) §18's apex/subdomain prefixes (`=`, `.`) are described for DNS names; their behavior on IP-literal entries (especially IPv6) is unstated. Presumably `=` and `.` are nonsensical for IPs, but this should be rejected explicitly rather than left to implementation.

## weak-testing

- (major) §21 lists SSRF cases but omits two important ones: (a) interaction with `--allow-host` (a host explicitly allowed but resolving to a private address must still be blocked unless `--allow-internal`); (b) DNS-rebinding race where the SSRF-stage resolution and the actual connect-time resolution disagree. Without these, the §16 guarantees are not exercised end-to-end.
- (major) §21 has no test for `broken:invalid-url` (per §20's reclassification), no test asserting that invalid-URL items are counted in the `broken` summary bucket and trip `--fail-on=broken`, and no test for the `--allow-internal` banner appearing on **both** stdout and stderr (§16). The new v0.2 classes (`skipped:blocked-destination`, `broken:blocked-destination-redirect`) likewise have no summary-accounting test.
- (minor) §8.1's example shows no fan-out case (a single URL appearing in N>1 files) even though §20 makes fan-out a binding rendering rule. A worked example/golden fixture is needed so implementers can verify summary arithmetic (`N unique (M occurrences)`) and per-file repetition.

## suggested-next-version

0.3

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "contradiction",
      "text": "§4 specifies retry backoff as `200ms × 2^n, jittered ±25%`, while §20 redefines it as `200ms × 2^(n-1)` with **full** jitter in `[0, base]`. These yield different first-retry distributions (e.g., n=1 → 400ms ±25% vs uniform `[0,200ms]`). §20 is presented as a clarification but actually changes both the base curve and the jitter shape, so implementers will not know which to honor.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "§16 introduces two new classes (`skipped:blocked-destination` and `broken:blocked-destination-redirect`) but they are not added to the §6.2 classification table, not mapped into the summary buckets in §8.1 (`ok / redirects / broken / skipped`), and their `--fail-on` semantics are unstated. Result: it is unclear whether a blocked-destination-redirect counts toward `broken` for exit-code purposes, and whether `skipped:blocked-destination` increments the `skipped` summary line.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§18's `host:port` port-scoped syntax collides with IPv6 literals (which contain colons and are passed without brackets per the same section). E.g., `--deny-host ::1:443` is ambiguous: host `::1` port `443`, or host `::1:443`? No parsing rule is given, and brackets are explicitly forbidden on the CLI.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§17 says the original (non-redacted) URL is used for probing while sensitive query values are replaced for display, but it does not state whether dedup/normalization in §5 collapses URLs that differ only in sensitive query values. Userinfo is explicitly dropped for dedup; sensitive params are not addressed. This determines whether `?token=A` and `?token=B` to the same host/path are one probe or two.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§16 mandates a pre-connect DNS resolution by linkrot itself, but Bun's `fetch` performs its own resolution internally. The spec does not define the implementation seam (custom dispatcher, undici-style connect hook, low-level socket?) nor what happens if linkrot's resolution and the resolution used by the actual TCP connect diverge (TTL expiry, round-robin reorder, NAT64). This is the core SSRF guarantee and currently rests on an unspecified mechanism.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§16 does not specify the classification when the SSRF-stage DNS resolution itself fails (NXDOMAIN, SERVFAIL, timeout). Is it `broken:network`, `skipped:blocked-destination`, or something else? §6.2 lists DNS failure under `broken:network`, but §16's pre-connect resolution is a distinct step.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§19 declares per-host accounting `port-agnostic`, but §18 supports `host:port` entries for `--deny-host`/`--allow-host`. The interaction is unstated: does a `host:443` deny share any state with the per-host cap accounting for `host:8080`? Also, what host identity is used when the request is redirected from host A to host B mid-chain — does the cap of A, B, or both gate the next hop?",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§21 lists SSRF cases but omits two important ones: (a) interaction with `--allow-host` (a host explicitly allowed but resolving to a private address must still be blocked unless `--allow-internal`); (b) DNS-rebinding race where the SSRF-stage resolution and the actual connect-time resolution disagree. Without these, the §16 guarantees are not exercised end-to-end.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§21 has no test for `broken:invalid-url` (per §20's reclassification), no test asserting that invalid-URL items are counted in the `broken` summary bucket and trip `--fail-on=broken`, and no test for the `--allow-internal` banner appearing on **both** stdout and stderr (§16). The new v0.2 classes (`skipped:blocked-destination`, `broken:blocked-destination-redirect`) likewise have no summary-accounting test.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§16 says `the final URL is shown truncated at the hop that was refused`, but `truncated` is not defined. Is the displayed URL the last public URL in the chain, the refused (private) target with its path elided, or only the scheme+host? Golden tests cannot be written without this.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§20's `files scanned counts only fully-extracted files` interacts unclearly with §12's recovery mode for malformed Markdown. A file partially recovered (some regions skipped, others extracted) is presumably `scanned`, while a fatal parser failure is `skipped`, but the threshold between recoverable warning and fatal failure is left to the parser. State the rule explicitly.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§19 says a `429`/`503` with `Retry-After` reduces a host's effective cap to 1 for the remainder of the run, but does not specify behavior for `429`/`503` *without* `Retry-After`, nor whether the cap reduction is reset on subsequent successful responses, nor whether it applies when the existing per-host cap is already 1.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§17's control-character list omits U+2028 (LINE SEPARATOR) and U+2029 (PARAGRAPH SEPARATOR), both of which can disrupt terminal/CI log rendering and JSON serialization. Either include them or state explicitly that they are intentionally allowed through.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§16's per-hop SSRF re-check does not state DNS-cache semantics: must each hop perform a fresh resolution, or may a TTL-bound cache shared across the run be used? This affects both performance and the rebinding threat model the section is designed to defeat.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§17 says redaction applies uniformly to all rendered URLs, but does not clarify whether the rendered host (post-IDNA, possibly Punycode) is what users see, or whether Unicode is restored for display. §18 mandates Punycode for *matching*; the *rendering* form is unspecified, making spoofing-via-confusables possible in the report.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "§8.1's example shows no fan-out case (a single URL appearing in N>1 files) even though §20 makes fan-out a binding rendering rule. A worked example/golden fixture is needed so implementers can verify summary arithmetic (`N unique (M occurrences)`) and per-file repetition.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§16 strips userinfo from the request, but the default `fetch` will encode `user:pass@` as a Basic `Authorization` header. The spec needs to state that the implementation must not just rely on calling `fetch` with the original URL — userinfo must be removed from the URL passed to the transport, otherwise the `never sent in Authorization headers` guarantee is violated by the runtime's default behavior.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§18's apex/subdomain prefixes (`=`, `.`) are described for DNS names; their behavior on IP-literal entries (especially IPv6) is unstated. Presumably `=` and `.` are nonsensical for IPs, but this should be rejected explicitly rather than left to implementation.",
      "severity": "minor"
    }
  ],
  "summary": "v0.2 adds substantial new safety surface (SSRF, redaction, host canonicalization, per-host caps) but introduces several ambiguities and at least one direct contradiction with the frozen text. The most load-bearing issues: (1) §4 vs §20 backoff formulas disagree; (2) the new SSRF classes are not wired into the §6.2 classification table, summary accounting, or `--fail-on` semantics; (3) §18's `host:port` syntax collides with bracketless IPv6 literals; (4) §17 leaves dedup behavior for sensitive query params undefined; (5) the SSRF mechanism rests on an unspecified DNS-resolution seam separate from Bun's fetch. §21 also misses tests for the new failure classes and for `--allow-host`/SSRF interaction. Resolving these in v0.3 should not require touching frozen sections except via a clarifications-style override.",
  "suggested_next_version": "0.3",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
