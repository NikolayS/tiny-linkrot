# Reviewer B — Claude

## summary

v0.3 closes most r02 gaps, but several contradictions and unspecified seams remain that an implementer cannot resolve from the text alone: exit code 2/3 are overloaded by §§22–23 and §16.3 against §9; §22's 'silently skipped with a stderr warning' is self-contradictory; §11's per-host non-goal collides with §19's per-host concurrency + backpressure; §16.7's 'CONNECT only' leaves HTTP-through-proxy behavior undefined; §20's Retry-After cap-≤-0 clarification contradicts itself within one sentence; §17.2 dedup does not name which colliding URL is actually probed; §17.3 'no path-typical extension' is unbounded; §19's redirect-hop slot lifecycle on the source host is unspecified; and §16.3 does not name the resolver API whose semantics it relies on. Testing additions in §21 are good but miss the 'broader wins' rule, CIDR matching, --per-host-concurrency boundaries, the include/exclude merge rule, --max-redirect-headers, --proxy/--use-env-proxy precedence, and the DNS-rebinding test description does not exercise the property §16.3 actually guarantees.

## contradiction

- (major) §22 says off-root candidates are 'silently skipped with a stderr warning' — silent and warning-on-stderr are mutually exclusive. The same self-contradiction appears for cycle termination ('a cycle terminates the offending branch with a stderr warning, never the whole run' is fine; but the earlier 'silently skipped with a stderr warning' must pick one). Pick a single behavior (warn on stderr, or truly silent) and apply it consistently to off-root, special-file, and symlink-out-of-root cases.
- (major) §11 (frozen v0.1 non-goals) lists 'Per-host rate limiting … beyond a single user-agent string' as out of scope, but §19 introduces per-host concurrency caps and adds a 1 s synthetic backoff plus sticky cap-of-1 on 429/503 — that is rate limiting in everything but name. §20's 'Version labeling' clarification says v0.1 labels describe what is captured in this v0.3 doc, which makes the contradiction explicit rather than resolving it. Either amend §11 in §20's clarifications to carve out per-host concurrency/backpressure, or rename §19 to avoid the rate-limiting term — currently a reader cannot tell which statement governs.
- (major) Exit code 2 is overloaded. §9 fixes exit 2 as 'Usage error (unknown flag, bad value, path does not exist)', but §23 reuses exit 2 for runtime budget exhaustion (`--max-files`, `--max-urls`, `--max-occurrences`) discovered mid-extraction, which is not a usage error in any normal sense. CI scripts that branch on exit code cannot distinguish a malformed invocation from a successful-but-too-large scan. Either define a new exit code for budget abort or amend §9's definition of exit 2 to explicitly include budget exhaustion.
- (major) §16.3 says the implementation MUST 'fail closed: probing is disabled and the run exits 3' if no socket-level seam is available, but §9 reserves exit 3 for 'Internal error (unhandled exception). Stack trace on stderr.' A capability gap detected at startup is neither an unhandled exception nor warrants a stack trace. This will produce confusing CI output (a stack trace for a deterministic environment limitation) and overlaps semantically with exit 2. Define an explicit code (or extend §9 with a 'capability/environment error' meaning) and state whether a stack trace must be suppressed in this case.
- (minor) §16.10 maps `skipped:budget-exceeded` to the `skipped` summary bucket and says it does not trip `--fail-on=broken` but does trip `--fail-on=any`. §23 simultaneously says budget exhaustion aborts with exit 2 regardless. Per the §9 table exit 2 always wins, so the `--fail-on=any` mapping for `skipped:budget-exceeded` is dead text — it can never affect the exit code. Remove it from the table, or note explicitly that the row applies only to URLs marked `skipped:budget-exceeded` after a non-aborting budget cap (e.g., `--max-url-length`, which §23 describes as a per-URL skip, not an abort).

## ambiguity

- (major) §17.2 dedup-by-redacted-form leaves the actual probe target unspecified. When two source occurrences differ only in `?token=…` value, the spec says the redacted form is the dedup key, but does not say whose value is sent on the wire (first encountered? last? lexicographic minimum?). This is observable: the answer determines which bearer the upstream sees, whether logs/quotas are attributed to the right caller, and whether two genuinely different resources colliding on `key=` are probed against URL A or URL B. State the selection rule (e.g., 'the first occurrence in scan order') and add a unit test for it.
- (major) §19 redirect accounting specifies that the global slot is held during the wait for B's per-host bucket, but does not say what happens to host A's per-host slot during that wait. If A's slot is held, a long queue on B can starve dispatch to A; if A's slot is released, two probes to B that both originated from A could exceed accounting expectations. The spec also does not define what 'redirect from A to B' means when the chain is A→A (same-host hop) — does it consume a second A slot? Specify slot lifecycle on every hop transition and add a test that exercises an A→B→A chain under a saturated A.
- (major) §16.7 declares 'CONNECT is the only proxy mode supported in v0.3', which only covers HTTPS targets. The behavior for plain `http://` targets when a proxy is configured is undefined: are they probed direct (silently bypassing the proxy), refused at startup, classified as `skipped:*`, or treated as `broken:*`? Each choice has different security and CI implications. Pick one and document it; add a test for HTTP-target-through-proxy.
- (major) §20 'Retry-After' clarification is internally inconsistent: 'If the cap is ≤ 0, the link resolves on its current state with no further retry' followed immediately by 'the subsequent attempt still consumes one --retries slot' contradicts itself — there is no subsequent attempt to consume a slot. Re-state which sentence applies when (cap > 0 vs cap ≤ 0) and rewrite to remove the dangling clause.
- (major) §17.3 strict-redaction heuristic includes 'no path-typical extension' without enumerating the set or pointing to an authoritative list. Different implementers (or even one implementer across versions) will choose different extension sets, breaking the dedup-key invariant promised in §17.3 and producing flaky golden tests. Either inline an explicit extension list or define the rule purely on character composition without the extension carve-out.
- (major) §16.3 says the probe layer 'resolves the hostname once via the OS resolver' but does not name an API surface (Bun's `dns.lookup`, `dns.resolve4`/`resolve6`, `getaddrinfo`, c-ares, etc.). These differ in whether they consult `/etc/hosts`, honor `nsswitch.conf`, return CNAMEs, or merge A and AAAA atomically — all of which affect the §16.2 mixed-answer rule. Specify the resolver semantics (or at least the required guarantees: must return all A and AAAA, must not be cached by the runtime, must honor `/etc/hosts` yes/no).
- (minor) §19 'A 429 or 503 … reduces that host's effective cap to 1 for the remainder of the run' does not say whether intermediate-hop 429/503 responses (i.e., a redirect chain where hop 2 returns 503) trigger the reduction, and against which host (the redirecting host or the responder). Given §16.4's per-hop policy and §19's per-hop slot accounting, it is plausible to argue either reading. Specify the bound: e.g., 'only the host that emitted the 429/503 response, regardless of position in the chain'.
- (minor) §17.2 says parameter-name matching is 'case-insensitively' without specifying the case-folding algorithm (ASCII lower-case vs Unicode default-case-fold). A site that uses `Tökën` would match under Unicode folding but not ASCII. State 'ASCII case-insensitive' (likely intent) or 'Unicode default case fold' explicitly.
- (minor) §17.5 'Hostname rendering' is silent on what to do when the input host is already an A-label literal that the user typed (e.g., they wrote `xn--mnchen-3ya.de` directly in the doc, not the Unicode form). Render as Punycode-only? Or also expand to `(münchen.de)`? The latter aids spotting confusables; the former preserves user intent. Pick one and note it.
- (minor) §16.6 'broader wins for the matching link' is informal. Spell out the precedence: when `--allow-internal` is set, all internal destinations are allowed regardless of `--allow-internal-host`; when only scoped allowances are set, only matching destinations are allowed; combining them is equivalent to `--allow-internal` alone (since it already covers everything). State this directly so an implementer cannot read 'broader' as 'union of CIDRs', etc.
- (minor) §19 'Host identity uses the canonical form from §18, port-agnostic, for the accounting bucket. The host:port syntax in §18 only affects matching of allow/deny rules, not which bucket counts a request.' This means a single host serving on ports 80, 443, and 8080 shares one cap of 4. State whether that is intended (it likely is) and whether it interacts with `--per-host-concurrency` overrides — e.g., can a port-scoped allow entry implicitly bump the cap for that host? Currently no, but the spec does not say so.

## weak-testing

- (major) §21 'DNS-rebinding race' test description is incoherent with §16.3's design. §16.3 says connect uses the classifier-selected IP and never re-resolves, so 'inject a transport that returns one address to the SSRF-stage classifier and a different address at connect time' cannot meaningfully fail — there is no connect-time resolution to intercept. The test as written passes vacuously. The actual property to verify is: given a resolver fixture that returns address X then address Y on a second call, assert the TCP connect target equals X (the classifier's pick), not Y. Rewrite the test plan to match the seam being asserted.
- (major) §21 has no test for the §16.6 'broader wins' rule when both `--allow-internal` and `--allow-internal-host` are set, no test for CIDR matching in `--allow-internal-host` (despite §18 explicitly enabling CIDR only for that flag), no test for `--per-host-concurrency` boundary cases (N=1, N=--concurrency, N>--concurrency rejected with exit 2), no test for the §20 `--include`-replaces / `--exclude`-appends merge rule or `--no-default-excludes`, no test for `--max-redirect-headers`, and no test for `--proxy` + `--use-env-proxy` precedence. Each of these is an ambiguity-prone seam in §§16–23 that the testing section was added to lock down.
- (minor) §21 'Refused-redirect rendering' test pins the format `scheme://host[:port]/<elided>` but does not specify how the host itself is rendered when it is an IDN — §17.5 says non-ASCII hosts render as `<punycode> (<unicode>)`, which would break the verbatim golden if applied here. Either state explicitly that §17.5 rendering applies to refused-redirect targets too (and update the golden expectation), or carve out an exception. As written, the test will be flaky depending on which §17 rule the implementer threads through §16.4.
- (minor) §13 promises 'No live network in CI' and §21 adds many transport-seam tests, but neither section requires a test that asserts the bare CLI cannot accidentally bypass the seam (e.g., a test that runs the full binary with a misconfigured seam and verifies it fails closed per §16.3 rather than falling through to runtime `fetch`). Without that, a regression that bypasses the seam at the entry point would not be caught by per-unit tests of the seam itself.

## suggested-next-version

0.4

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "contradiction",
      "text": "§22 says off-root candidates are 'silently skipped with a stderr warning' — silent and warning-on-stderr are mutually exclusive. The same self-contradiction appears for cycle termination ('a cycle terminates the offending branch with a stderr warning, never the whole run' is fine; but the earlier 'silently skipped with a stderr warning' must pick one). Pick a single behavior (warn on stderr, or truly silent) and apply it consistently to off-root, special-file, and symlink-out-of-root cases.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "§11 (frozen v0.1 non-goals) lists 'Per-host rate limiting … beyond a single user-agent string' as out of scope, but §19 introduces per-host concurrency caps and adds a 1 s synthetic backoff plus sticky cap-of-1 on 429/503 — that is rate limiting in everything but name. §20's 'Version labeling' clarification says v0.1 labels describe what is captured in this v0.3 doc, which makes the contradiction explicit rather than resolving it. Either amend §11 in §20's clarifications to carve out per-host concurrency/backpressure, or rename §19 to avoid the rate-limiting term — currently a reader cannot tell which statement governs.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "Exit code 2 is overloaded. §9 fixes exit 2 as 'Usage error (unknown flag, bad value, path does not exist)', but §23 reuses exit 2 for runtime budget exhaustion (`--max-files`, `--max-urls`, `--max-occurrences`) discovered mid-extraction, which is not a usage error in any normal sense. CI scripts that branch on exit code cannot distinguish a malformed invocation from a successful-but-too-large scan. Either define a new exit code for budget abort or amend §9's definition of exit 2 to explicitly include budget exhaustion.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "§16.3 says the implementation MUST 'fail closed: probing is disabled and the run exits 3' if no socket-level seam is available, but §9 reserves exit 3 for 'Internal error (unhandled exception). Stack trace on stderr.' A capability gap detected at startup is neither an unhandled exception nor warrants a stack trace. This will produce confusing CI output (a stack trace for a deterministic environment limitation) and overlaps semantically with exit 2. Define an explicit code (or extend §9 with a 'capability/environment error' meaning) and state whether a stack trace must be suppressed in this case.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§17.2 dedup-by-redacted-form leaves the actual probe target unspecified. When two source occurrences differ only in `?token=…` value, the spec says the redacted form is the dedup key, but does not say whose value is sent on the wire (first encountered? last? lexicographic minimum?). This is observable: the answer determines which bearer the upstream sees, whether logs/quotas are attributed to the right caller, and whether two genuinely different resources colliding on `key=` are probed against URL A or URL B. State the selection rule (e.g., 'the first occurrence in scan order') and add a unit test for it.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§19 redirect accounting specifies that the global slot is held during the wait for B's per-host bucket, but does not say what happens to host A's per-host slot during that wait. If A's slot is held, a long queue on B can starve dispatch to A; if A's slot is released, two probes to B that both originated from A could exceed accounting expectations. The spec also does not define what 'redirect from A to B' means when the chain is A→A (same-host hop) — does it consume a second A slot? Specify slot lifecycle on every hop transition and add a test that exercises an A→B→A chain under a saturated A.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§16.7 declares 'CONNECT is the only proxy mode supported in v0.3', which only covers HTTPS targets. The behavior for plain `http://` targets when a proxy is configured is undefined: are they probed direct (silently bypassing the proxy), refused at startup, classified as `skipped:*`, or treated as `broken:*`? Each choice has different security and CI implications. Pick one and document it; add a test for HTTP-target-through-proxy.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§20 'Retry-After' clarification is internally inconsistent: 'If the cap is ≤ 0, the link resolves on its current state with no further retry' followed immediately by 'the subsequent attempt still consumes one --retries slot' contradicts itself — there is no subsequent attempt to consume a slot. Re-state which sentence applies when (cap > 0 vs cap ≤ 0) and rewrite to remove the dangling clause.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§17.3 strict-redaction heuristic includes 'no path-typical extension' without enumerating the set or pointing to an authoritative list. Different implementers (or even one implementer across versions) will choose different extension sets, breaking the dedup-key invariant promised in §17.3 and producing flaky golden tests. Either inline an explicit extension list or define the rule purely on character composition without the extension carve-out.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§16.3 says the probe layer 'resolves the hostname once via the OS resolver' but does not name an API surface (Bun's `dns.lookup`, `dns.resolve4`/`resolve6`, `getaddrinfo`, c-ares, etc.). These differ in whether they consult `/etc/hosts`, honor `nsswitch.conf`, return CNAMEs, or merge A and AAAA atomically — all of which affect the §16.2 mixed-answer rule. Specify the resolver semantics (or at least the required guarantees: must return all A and AAAA, must not be cached by the runtime, must honor `/etc/hosts` yes/no).",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§21 'DNS-rebinding race' test description is incoherent with §16.3's design. §16.3 says connect uses the classifier-selected IP and never re-resolves, so 'inject a transport that returns one address to the SSRF-stage classifier and a different address at connect time' cannot meaningfully fail — there is no connect-time resolution to intercept. The test as written passes vacuously. The actual property to verify is: given a resolver fixture that returns address X then address Y on a second call, assert the TCP connect target equals X (the classifier's pick), not Y. Rewrite the test plan to match the seam being asserted.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§21 has no test for the §16.6 'broader wins' rule when both `--allow-internal` and `--allow-internal-host` are set, no test for CIDR matching in `--allow-internal-host` (despite §18 explicitly enabling CIDR only for that flag), no test for `--per-host-concurrency` boundary cases (N=1, N=--concurrency, N>--concurrency rejected with exit 2), no test for the §20 `--include`-replaces / `--exclude`-appends merge rule or `--no-default-excludes`, no test for `--max-redirect-headers`, and no test for `--proxy` + `--use-env-proxy` precedence. Each of these is an ambiguity-prone seam in §§16–23 that the testing section was added to lock down.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§21 'Refused-redirect rendering' test pins the format `scheme://host[:port]/<elided>` but does not specify how the host itself is rendered when it is an IDN — §17.5 says non-ASCII hosts render as `<punycode> (<unicode>)`, which would break the verbatim golden if applied here. Either state explicitly that §17.5 rendering applies to refused-redirect targets too (and update the golden expectation), or carve out an exception. As written, the test will be flaky depending on which §17 rule the implementer threads through §16.4.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§19 'A 429 or 503 … reduces that host's effective cap to 1 for the remainder of the run' does not say whether intermediate-hop 429/503 responses (i.e., a redirect chain where hop 2 returns 503) trigger the reduction, and against which host (the redirecting host or the responder). Given §16.4's per-hop policy and §19's per-hop slot accounting, it is plausible to argue either reading. Specify the bound: e.g., 'only the host that emitted the 429/503 response, regardless of position in the chain'.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§17.2 says parameter-name matching is 'case-insensitively' without specifying the case-folding algorithm (ASCII lower-case vs Unicode default-case-fold). A site that uses `Tökën` would match under Unicode folding but not ASCII. State 'ASCII case-insensitive' (likely intent) or 'Unicode default case fold' explicitly.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§17.5 'Hostname rendering' is silent on what to do when the input host is already an A-label literal that the user typed (e.g., they wrote `xn--mnchen-3ya.de` directly in the doc, not the Unicode form). Render as Punycode-only? Or also expand to `(münchen.de)`? The latter aids spotting confusables; the former preserves user intent. Pick one and note it.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§16.6 'broader wins for the matching link' is informal. Spell out the precedence: when `--allow-internal` is set, all internal destinations are allowed regardless of `--allow-internal-host`; when only scoped allowances are set, only matching destinations are allowed; combining them is equivalent to `--allow-internal` alone (since it already covers everything). State this directly so an implementer cannot read 'broader' as 'union of CIDRs', etc.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "§13 promises 'No live network in CI' and §21 adds many transport-seam tests, but neither section requires a test that asserts the bare CLI cannot accidentally bypass the seam (e.g., a test that runs the full binary with a misconfigured seam and verifies it fails closed per §16.3 rather than falling through to runtime `fetch`). Without that, a regression that bypasses the seam at the entry point would not be caught by per-unit tests of the seam itself.",
      "severity": "minor"
    },
    {
      "category": "contradiction",
      "text": "§16.10 maps `skipped:budget-exceeded` to the `skipped` summary bucket and says it does not trip `--fail-on=broken` but does trip `--fail-on=any`. §23 simultaneously says budget exhaustion aborts with exit 2 regardless. Per the §9 table exit 2 always wins, so the `--fail-on=any` mapping for `skipped:budget-exceeded` is dead text — it can never affect the exit code. Remove it from the table, or note explicitly that the row applies only to URLs marked `skipped:budget-exceeded` after a non-aborting budget cap (e.g., `--max-url-length`, which §23 describes as a per-URL skip, not an abort).",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§19 'Host identity uses the canonical form from §18, port-agnostic, for the accounting bucket. The host:port syntax in §18 only affects matching of allow/deny rules, not which bucket counts a request.' This means a single host serving on ports 80, 443, and 8080 shares one cap of 4. State whether that is intended (it likely is) and whether it interacts with `--per-host-concurrency` overrides — e.g., can a port-scoped allow entry implicitly bump the cap for that host? Currently no, but the spec does not say so.",
      "severity": "minor"
    }
  ],
  "summary": "v0.3 closes most r02 gaps, but several contradictions and unspecified seams remain that an implementer cannot resolve from the text alone: exit code 2/3 are overloaded by §§22–23 and §16.3 against §9; §22's 'silently skipped with a stderr warning' is self-contradictory; §11's per-host non-goal collides with §19's per-host concurrency + backpressure; §16.7's 'CONNECT only' leaves HTTP-through-proxy behavior undefined; §20's Retry-After cap-≤-0 clarification contradicts itself within one sentence; §17.2 dedup does not name which colliding URL is actually probed; §17.3 'no path-typical extension' is unbounded; §19's redirect-hop slot lifecycle on the source host is unspecified; and §16.3 does not name the resolver API whose semantics it relies on. Testing additions in §21 are good but miss the 'broader wins' rule, CIDR matching, --per-host-concurrency boundaries, the include/exclude merge rule, --max-redirect-headers, --proxy/--use-env-proxy precedence, and the DNS-rebinding test description does not exercise the property §16.3 actually guarantees.",
  "suggested_next_version": "0.4",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
