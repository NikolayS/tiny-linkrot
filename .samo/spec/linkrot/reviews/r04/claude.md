# Reviewer B — Claude

## summary

v0.4 closes most r03 findings cleanly (evaluation order, IP-literal handling, sensitive-param dedup, exit-code scope, fan-out) and the new pinned-transport story in §16.4 is substantively correct. Remaining issues cluster in four areas: (1) §4's frozen CLI table never absorbed any v0.2+ flag so the document's authoritative CLI surface is missing ~12 flags the rest of the spec depends on; (2) a contradiction in §21's Retry-After test versus §20's cap formula; (3) two grammar/rendering ambiguities (§17.4's 'except \t, DEL', §16.5's refused-redirect hop-count format) that will cause implementation divergence; and (4) gaps around §16.4's Bun runtime feasibility (Node APIs prescribed) and IPv6 connect-fallback semantics. Testing additions for IPv6 CIDR, manual-redirect enforcement, §19 synthetic backoff, and CLI-only env-var negative cases are needed before v0.5 is implementation-ready. §18's silent bare-host narrowing from v0.3 is a real user-facing breakage that should land with a migration note or transitional warning.

## contradiction

- (major) §21 Retry-After test case asserts that a value 'greater than --timeout' yields 'cap ≤ 0 → final on current state, no retry slot consumed', but §20's binding formula is `cap = min(Retry-After, --timeout − elapsed)`. When elapsed is small (e.g., first probe), `--timeout − elapsed` is positive, so cap is positive even though Retry-After > --timeout. The test premise therefore contradicts the formula; only when `--timeout − elapsed ≤ 0` does cap ≤ 0. Either the test must be rephrased to pin elapsed (e.g., 'Retry-After larger than remaining budget'), or §20's formula must be revised.
- (major) §4's CLI surface table is still the v0.1 table and does not list any of the v0.2/v0.3/v0.4 flags the rest of the spec (and §20 env-var scope, §21 tests) depends on: `--allow-internal`, `--allow-internal-host`, `--strict-redaction`, `--per-host-concurrency`, `--no-images`, `--follow-symlinks`, `--max-files`, `--max-urls`, `--max-occurrences`, `--max-url-length`, `--max-redirect-headers`, `--no-default-excludes`. §4 is nominally frozen but is the authoritative CLI surface; the omission makes `linkrot --help` undefined in v0.4 and lets §4 and §§16–23 contradict each other. v0.4 should either extend §4 (with a 'v0.2+ additions' subsection) or introduce §4a to enumerate the full flag set and their defaults.

## ambiguity

- (major) §17.4 lists the set of characters to escape as 'ASCII C0 controls except \t, DEL (\x7f), C1 controls (\x80–\x9f).' The grammar is genuinely ambiguous: it parses as either '(C0 except \t and DEL), plus C1' (DEL NOT escaped) or '(C0 except \t), plus DEL, plus C1' (DEL IS escaped). Given the security intent, DEL presumably should be escaped, but two compliant implementations can diverge. Rewrite as e.g., 'C0 controls except \t; DEL (\x7f); C1 controls (\x80–\x9f)' or with a bulleted list.
- (major) §23 specifies `--max-redirect-headers N (default 64)` while §6.1 caps redirect hops at 10. The relationship is undefined: 64 headers per hop? 64 headers total across the chain? 64 bytes? 64 individual header lines to retain for the report? The sentence 'retained per-hop header metadata is bounded' is also self-contradictory with a single scalar N if it's per-hop-but-total. Specify the unit (count vs bytes), the scope (per hop vs per chain), and what is dropped first when the cap is hit.
- (major) §19's reactive backoff rule is under-specified in three ways: (1) 'inserts a 1 s synthetic backoff before the next dispatch to that host' — is the 1 s applied once (before only the immediately-next dispatch) or before every subsequent dispatch to the host for the remainder of the run? (2) Does the 1 s wait count against the waiting probe's `--timeout` wall-clock, and if exhausted there, is the classification `broken:network` / `timeout-per-host-queue` (§19) or some other reason? (3) If a `429`/`503` is observed on a chain that the probe is already going to fail on (e.g., final 503, no Retry-After, retries exhausted), does the cap reduction still apply to the responder? Pin all three.
- (major) §16.4 prescribes `node:dns/promises` `resolve4`/`resolve6` and 'undici-style' or 'Bun socket-level' dispatch hooks, but tiny-linkrot is explicitly a Bun project with no runtime deps (§3). Whether Bun 1.1+ exposes a socket-level seam that lets `fetch` connect to a pinned IP while sending the original hostname in TLS SNI and Host is load-bearing for v0.4 feasibility, and the spec does not cite it. The fail-closed escape in §16.4 also creates a scenario where a v0.4 build fails to probe anything on some Bun versions. Either cite the specific Bun API the reference implementation will use (e.g., `Bun.connect`, a named undici polyfill), or add a §16.4 bullet calling out the capability as a known runtime dependency.
- (major) §18's bare-hostname semantics change silently breaks prior allowlists: in v0.3, `--allow-host example.com` admitted `www.example.com`; in v0.4 it no longer does, and `=example.com` (the v0.3 synonym) also narrows to apex-only. Operators upgrading from v0.3 will see sites previously probed newly classified as `skipped:host` with no signal the semantics changed. Spec should either (a) emit a stderr deprecation warning when bare-host forms match allowlists under the old-but-not-new semantics during a transition window, or (b) explicitly declare the v0.3→v0.4 migration story and call out this break in §24.
- (major) §16.4 says 'IPv6 preferred when both families are present and non-blocked, falling back to IPv4' for the pre-connect selection, but is silent on post-connect fallback: if the chosen IPv6 address times out or returns TCP RST during the initial handshake, does the probe (a) retry on an IPv4 address from the same classification result, (b) treat the whole probe as `broken:network`, or (c) invoke `--retries` against the same pinned address? Happy-eyeballs-style fallback has SSRF implications (the fallback IP must also have been classified non-blocked in the original resolution) and belongs in §16.4.
- (major) §16.5 states 'The number of public hops traversed before the refusal is shown next to the arrow' but gives no example. §8's TTY sample does not include a blocked-destination-redirect row, and §21's refused-redirect rendering test only anchors the `<host-rendered>[:port]/<elided>` part. Implementations will diverge on format (e.g., `↪ https://good.example/redir → xn--mnchen-3ya.de (münchen.de)/<elided>  (2 public hops)` vs `[2 hops] → …` vs inline with the arrow). Give the literal format.
- (minor) §22 defines the containment key as `realRoot + separator` to defeat the `/repo/docs` vs `/repo/docs-evil` ambiguity. Edge case: when `realRoot` already ends in `separator` (filesystem root `/` on POSIX, `C:\` on Windows), `realRoot + separator` double-appends (`//`, `C:\\`), and string-prefix checks will admit nothing. Spec should state the separator is appended iff `realRoot` does not already end with one, or use an explicit component-aware test (which §22's last bullet already recommends for consumers but does not self-apply to the root value).
- (minor) §16.3 classifies `fe80::1%eth0` as `broken:invalid-url` 'at URL parse time', but WHATWG `new URL()` does not reject percent-embedded zone identifiers (it percent-encodes `%`), so this requires an explicit pre-parse validation step. Spec should name that step (e.g., 'Before `new URL(...)` the implementation rejects any host matching `/%/` as `broken:invalid-url`') or place the rejection after parse with a defined predicate.
- (minor) §5 says 'percent-encoding normalized' for the dedup key with no reference standard. RFC 3986 §6.2.2.2 defines the normalization but implementations differ (uppercase vs lowercase hex, which reserved characters to unreserve). Since dedup keys must be stable across implementations for the §21 'first occurrence in scan order' test to be meaningful, either cite RFC 3986 §6.2.2.2 explicitly or define the canonical form inline.
- (minor) §17.3's strict-mode interaction with the summary line is unclear: if two distinct source URLs collapse to one dedup key under strict redaction, does `links checked: N unique (M occurrences)` report N=1 (one probe) and M = sum of occurrences of both URLs, or N=2 because the scan enumerated two distinct strings before redaction? §21 pins selection but not counting. Given §8.1 and §21's 'Fan-out golden' hinge on these counts, pin the counting rule too.
- (minor) §16.7 escape hatches apply the `⚠ internal destinations enabled` banner on both stdout and stderr under `--allow-internal`, but the summary line under `--allow-internal-host` is specified only on one stream ('The summary lists `internal allowances: <count>`'). Is that line on stdout (summary is on stdout per §8.2) or mirrored to stderr? Pin the stream.
- (minor) §19's redirect-hop slot lifecycle says the probe 'rejoins A's queue at the tail (no priority for in-flight chains)' for A→A hops. This interacts with §19's cap-reduction rule: if host A's cap was reduced to 1 mid-chain by a 429 at hop 1, the chain now waits for that single slot it just released. With other probes queued on A ahead, the chain may starve until the global `--timeout` fires. Is this the intended behavior, or should in-flight chains keep priority on same-host continuations to bound worst-case latency? Pin it.
- (minor) §16.4 permits a 'short-TTL DNS cache (≤ 30 s, per-run, in-process)' for the classification step but is silent on negative-result caching. If a prior resolution of `internal.example` returned only RFC1918 addresses and was cached, must a subsequent probe re-resolve (giving a DNS-rebinding attacker a second shot), or may the cached negative-classification result be reused? The safer default is to cache the classification result (not just the addresses); state this.

## weak-testing

- (major) §21's SSRF CIDR coverage is IPv4-only ('`--allow-internal-host 10.0.0.0/8` admits a destination in 10.x'). IPv6 CIDR matching (e.g., `fd00::/8` admitting a ULA destination while still blocking link-local `fe80::/10`) is neither asserted nor exercised. Given §16.1 enumerates several IPv6 blocked ranges and §16.4 prefers IPv6, an IPv6 CIDR test belongs alongside the IPv4 case.
- (major) §20 mandates 'Manual redirect following' (`redirect: 'manual'` or equivalent) so that hop counting, cycle detection, and per-hop SSRF re-check are deterministic. §21 has no test that asserts `fetch` is never called with `redirect: 'follow'` in v0.4, nor that each hop is separately observed by the SSRF classifier. Without that test, a regression where the runtime silently collapses hops would pass the suite while defeating §16.5.
- (major) §21 has no test for §19's 1 s synthetic backoff (cap-reduction-without-Retry-After path), and no test for §19's cap-sticky property under a subsequent `200` success. A follow-up 200 must NOT restore the cap per §19, but that invariant is not pinned in tests.
- (minor) §21's 'Precedence' test says 'Flag > env > default for every flag that has an env-var counterpart in §10' but §10 only enumerates five env vars (`LINKROT_CONCURRENCY`, `TIMEOUT`, `RETRIES`, `USER_AGENT`, `NO_COLOR`). §20's 'Env-var scope' clarification declares many v0.2+ flags CLI-only. There's no test that asserts the CLI-only set is in fact rejected from the environment (e.g., `LINKROT_ALLOW_INTERNAL=1` must NOT enable the escape hatch); without it, a future contributor could quietly plumb an env var and bypass the safety-sensitive opt-in.

## suggested-next-version

v0.5

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "contradiction",
      "text": "§21 Retry-After test case asserts that a value 'greater than --timeout' yields 'cap ≤ 0 → final on current state, no retry slot consumed', but §20's binding formula is `cap = min(Retry-After, --timeout − elapsed)`. When elapsed is small (e.g., first probe), `--timeout − elapsed` is positive, so cap is positive even though Retry-After > --timeout. The test premise therefore contradicts the formula; only when `--timeout − elapsed ≤ 0` does cap ≤ 0. Either the test must be rephrased to pin elapsed (e.g., 'Retry-After larger than remaining budget'), or §20's formula must be revised.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "§4's CLI surface table is still the v0.1 table and does not list any of the v0.2/v0.3/v0.4 flags the rest of the spec (and §20 env-var scope, §21 tests) depends on: `--allow-internal`, `--allow-internal-host`, `--strict-redaction`, `--per-host-concurrency`, `--no-images`, `--follow-symlinks`, `--max-files`, `--max-urls`, `--max-occurrences`, `--max-url-length`, `--max-redirect-headers`, `--no-default-excludes`. §4 is nominally frozen but is the authoritative CLI surface; the omission makes `linkrot --help` undefined in v0.4 and lets §4 and §§16–23 contradict each other. v0.4 should either extend §4 (with a 'v0.2+ additions' subsection) or introduce §4a to enumerate the full flag set and their defaults.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§17.4 lists the set of characters to escape as 'ASCII C0 controls except \\t, DEL (\\x7f), C1 controls (\\x80–\\x9f).' The grammar is genuinely ambiguous: it parses as either '(C0 except \\t and DEL), plus C1' (DEL NOT escaped) or '(C0 except \\t), plus DEL, plus C1' (DEL IS escaped). Given the security intent, DEL presumably should be escaped, but two compliant implementations can diverge. Rewrite as e.g., 'C0 controls except \\t; DEL (\\x7f); C1 controls (\\x80–\\x9f)' or with a bulleted list.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§23 specifies `--max-redirect-headers N (default 64)` while §6.1 caps redirect hops at 10. The relationship is undefined: 64 headers per hop? 64 headers total across the chain? 64 bytes? 64 individual header lines to retain for the report? The sentence 'retained per-hop header metadata is bounded' is also self-contradictory with a single scalar N if it's per-hop-but-total. Specify the unit (count vs bytes), the scope (per hop vs per chain), and what is dropped first when the cap is hit.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§19's reactive backoff rule is under-specified in three ways: (1) 'inserts a 1 s synthetic backoff before the next dispatch to that host' — is the 1 s applied once (before only the immediately-next dispatch) or before every subsequent dispatch to the host for the remainder of the run? (2) Does the 1 s wait count against the waiting probe's `--timeout` wall-clock, and if exhausted there, is the classification `broken:network` / `timeout-per-host-queue` (§19) or some other reason? (3) If a `429`/`503` is observed on a chain that the probe is already going to fail on (e.g., final 503, no Retry-After, retries exhausted), does the cap reduction still apply to the responder? Pin all three.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§16.4 prescribes `node:dns/promises` `resolve4`/`resolve6` and 'undici-style' or 'Bun socket-level' dispatch hooks, but tiny-linkrot is explicitly a Bun project with no runtime deps (§3). Whether Bun 1.1+ exposes a socket-level seam that lets `fetch` connect to a pinned IP while sending the original hostname in TLS SNI and Host is load-bearing for v0.4 feasibility, and the spec does not cite it. The fail-closed escape in §16.4 also creates a scenario where a v0.4 build fails to probe anything on some Bun versions. Either cite the specific Bun API the reference implementation will use (e.g., `Bun.connect`, a named undici polyfill), or add a §16.4 bullet calling out the capability as a known runtime dependency.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§18's bare-hostname semantics change silently breaks prior allowlists: in v0.3, `--allow-host example.com` admitted `www.example.com`; in v0.4 it no longer does, and `=example.com` (the v0.3 synonym) also narrows to apex-only. Operators upgrading from v0.3 will see sites previously probed newly classified as `skipped:host` with no signal the semantics changed. Spec should either (a) emit a stderr deprecation warning when bare-host forms match allowlists under the old-but-not-new semantics during a transition window, or (b) explicitly declare the v0.3→v0.4 migration story and call out this break in §24.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§16.4 says 'IPv6 preferred when both families are present and non-blocked, falling back to IPv4' for the pre-connect selection, but is silent on post-connect fallback: if the chosen IPv6 address times out or returns TCP RST during the initial handshake, does the probe (a) retry on an IPv4 address from the same classification result, (b) treat the whole probe as `broken:network`, or (c) invoke `--retries` against the same pinned address? Happy-eyeballs-style fallback has SSRF implications (the fallback IP must also have been classified non-blocked in the original resolution) and belongs in §16.4.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§16.5 states 'The number of public hops traversed before the refusal is shown next to the arrow' but gives no example. §8's TTY sample does not include a blocked-destination-redirect row, and §21's refused-redirect rendering test only anchors the `<host-rendered>[:port]/<elided>` part. Implementations will diverge on format (e.g., `↪ https://good.example/redir → xn--mnchen-3ya.de (münchen.de)/<elided>  (2 public hops)` vs `[2 hops] → …` vs inline with the arrow). Give the literal format.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§21's SSRF CIDR coverage is IPv4-only ('`--allow-internal-host 10.0.0.0/8` admits a destination in 10.x'). IPv6 CIDR matching (e.g., `fd00::/8` admitting a ULA destination while still blocking link-local `fe80::/10`) is neither asserted nor exercised. Given §16.1 enumerates several IPv6 blocked ranges and §16.4 prefers IPv6, an IPv6 CIDR test belongs alongside the IPv4 case.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§20 mandates 'Manual redirect following' (`redirect: 'manual'` or equivalent) so that hop counting, cycle detection, and per-hop SSRF re-check are deterministic. §21 has no test that asserts `fetch` is never called with `redirect: 'follow'` in v0.4, nor that each hop is separately observed by the SSRF classifier. Without that test, a regression where the runtime silently collapses hops would pass the suite while defeating §16.5.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§21 has no test for §19's 1 s synthetic backoff (cap-reduction-without-Retry-After path), and no test for §19's cap-sticky property under a subsequent `200` success. A follow-up 200 must NOT restore the cap per §19, but that invariant is not pinned in tests.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§22 defines the containment key as `realRoot + separator` to defeat the `/repo/docs` vs `/repo/docs-evil` ambiguity. Edge case: when `realRoot` already ends in `separator` (filesystem root `/` on POSIX, `C:\\` on Windows), `realRoot + separator` double-appends (`//`, `C:\\\\`), and string-prefix checks will admit nothing. Spec should state the separator is appended iff `realRoot` does not already end with one, or use an explicit component-aware test (which §22's last bullet already recommends for consumers but does not self-apply to the root value).",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§16.3 classifies `fe80::1%eth0` as `broken:invalid-url` 'at URL parse time', but WHATWG `new URL()` does not reject percent-embedded zone identifiers (it percent-encodes `%`), so this requires an explicit pre-parse validation step. Spec should name that step (e.g., 'Before `new URL(...)` the implementation rejects any host matching `/%/` as `broken:invalid-url`') or place the rejection after parse with a defined predicate.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§5 says 'percent-encoding normalized' for the dedup key with no reference standard. RFC 3986 §6.2.2.2 defines the normalization but implementations differ (uppercase vs lowercase hex, which reserved characters to unreserve). Since dedup keys must be stable across implementations for the §21 'first occurrence in scan order' test to be meaningful, either cite RFC 3986 §6.2.2.2 explicitly or define the canonical form inline.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§17.3's strict-mode interaction with the summary line is unclear: if two distinct source URLs collapse to one dedup key under strict redaction, does `links checked: N unique (M occurrences)` report N=1 (one probe) and M = sum of occurrences of both URLs, or N=2 because the scan enumerated two distinct strings before redaction? §21 pins selection but not counting. Given §8.1 and §21's 'Fan-out golden' hinge on these counts, pin the counting rule too.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§16.7 escape hatches apply the `⚠ internal destinations enabled` banner on both stdout and stderr under `--allow-internal`, but the summary line under `--allow-internal-host` is specified only on one stream ('The summary lists `internal allowances: <count>`'). Is that line on stdout (summary is on stdout per §8.2) or mirrored to stderr? Pin the stream.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "§21's 'Precedence' test says 'Flag > env > default for every flag that has an env-var counterpart in §10' but §10 only enumerates five env vars (`LINKROT_CONCURRENCY`, `TIMEOUT`, `RETRIES`, `USER_AGENT`, `NO_COLOR`). §20's 'Env-var scope' clarification declares many v0.2+ flags CLI-only. There's no test that asserts the CLI-only set is in fact rejected from the environment (e.g., `LINKROT_ALLOW_INTERNAL=1` must NOT enable the escape hatch); without it, a future contributor could quietly plumb an env var and bypass the safety-sensitive opt-in.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§19's redirect-hop slot lifecycle says the probe 'rejoins A's queue at the tail (no priority for in-flight chains)' for A→A hops. This interacts with §19's cap-reduction rule: if host A's cap was reduced to 1 mid-chain by a 429 at hop 1, the chain now waits for that single slot it just released. With other probes queued on A ahead, the chain may starve until the global `--timeout` fires. Is this the intended behavior, or should in-flight chains keep priority on same-host continuations to bound worst-case latency? Pin it.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§16.4 permits a 'short-TTL DNS cache (≤ 30 s, per-run, in-process)' for the classification step but is silent on negative-result caching. If a prior resolution of `internal.example` returned only RFC1918 addresses and was cached, must a subsequent probe re-resolve (giving a DNS-rebinding attacker a second shot), or may the cached negative-classification result be reused? The safer default is to cache the classification result (not just the addresses); state this.",
      "severity": "minor"
    }
  ],
  "summary": "v0.4 closes most r03 findings cleanly (evaluation order, IP-literal handling, sensitive-param dedup, exit-code scope, fan-out) and the new pinned-transport story in §16.4 is substantively correct. Remaining issues cluster in four areas: (1) §4's frozen CLI table never absorbed any v0.2+ flag so the document's authoritative CLI surface is missing ~12 flags the rest of the spec depends on; (2) a contradiction in §21's Retry-After test versus §20's cap formula; (3) two grammar/rendering ambiguities (§17.4's 'except \\t, DEL', §16.5's refused-redirect hop-count format) that will cause implementation divergence; and (4) gaps around §16.4's Bun runtime feasibility (Node APIs prescribed) and IPv6 connect-fallback semantics. Testing additions for IPv6 CIDR, manual-redirect enforcement, §19 synthetic backoff, and CLI-only env-var negative cases are needed before v0.5 is implementation-ready. §18's silent bare-host narrowing from v0.3 is a real user-facing breakage that should land with a migration note or transitional warning.",
  "suggested_next_version": "v0.5",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
