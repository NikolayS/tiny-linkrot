# Reviewer B — Claude

## summary

The spec is in good shape for a v0.1, but several rules at the network layer are under-specified or self-contradictory in ways that will produce inconsistent implementations and untestable acceptance criteria. The most pressing issues are: (1) the worst-case timeout bound in §6.2 contradicts the per-hop HEAD→GET fallback in §6.1; (2) §8.4 introduces a JSON `message` field that §7.2's stable schema omits; (3) the 'plausibly HEAD-specific' criterion in §6.1 step 3 is not implementable; (4) §5.3 normalization/dedup leaves out host case, default ports, trailing slashes, and query ordering; and (5) §6.3's classification table doesn't cover 3xx statuses outside the followed set. The test plan in §9 is the weakest section: it does not exercise human-mode output, color/TTY gating, stdout/stderr separation, the 10 MiB cap, unknown-flag handling, concurrency-cap enforcement, dedup occurrences, or the redirect_chain contents — all of which §12 implicitly requires for 'done'. Several minor ambiguities (version source, line-tiebreak ordering, timeout/concurrency bounds, exit-3 behavior in --json) should be tightened before implementation.

## contradiction

- (major) §6.2 states the worst-case wall-clock for a URL is `6 × timeout` ('a URL that redirects five times may spend up to 6 × timeout'), but §6.1 step 3 permits a HEAD→GET fallback at each hop. With the 5-redirect cap (so up to 6 attempts in the chain) and a HEAD+GET on each, the real worst case is `12 × timeout`. Either the bound or the per-hop fallback rule must be corrected.
- (major) §8.4 says network errors 'become `status: "broken"` entries with `reason: "network_error"` and a short human-readable `message` field in JSON output', but the §7.2 'Field contract' enumerates the JSON fields and never lists `message`. The example object also omits it. Implementers cannot tell whether `message` is part of the stable schema or not.
- (major) §5.2 lists GFM bare URLs as an extraction source 'when the parser surfaces them' — i.e., behavior is conditional on the chosen parser. §9 then mandates a unit test for 'GFM bare URL' extraction. If the implementer picks a parser without GFM autolink (e.g. stock `markdown-it` without the `linkify` option enabled), the spec simultaneously permits and forbids the behavior. Either require GFM autolink unconditionally or drop the test.
- (major) §5.1 routes 'files larger than 10 MiB' to exit code 2, but §4.3 defines exit 2 as 'usage / argument error (bad flag, missing file, file not readable, not a file)'. A 10 MiB Markdown file is none of those. Either broaden the §4.3 definition to cover 'input rejected by precheck' or assign a different exit code.

## ambiguity

- (major) §6.1 step 3 retries with GET on '405 Method Not Allowed', '501 Not Implemented', or 'a network-level error that could plausibly be HEAD-specific (e.g. connection reset before headers)'. 'Plausibly HEAD-specific' is not implementable: two engineers will disagree on whether DNS NXDOMAIN, TLS handshake failure, ECONNREFUSED, mid-body reset, or a HEAD timeout qualifies. Provide an explicit allowlist of error classes that trigger GET retry.
- (major) §5.3 'URL normalization' only specifies WHATWG parsing, fragment stripping, and dedup by 'identical post-normalization URLs'. It does not say whether dedup is sensitive to host case, percent-encoding case, default-port presence (`:80`/`:443`), trailing slash on the path-empty case, query-parameter order, or duplicate query keys. These choices materially change the unique-URL count surfaced in the report and the §9 dedup tests.
- (major) §6.1 step 2 enumerates the redirect statuses it follows as 301/302/303/307/308. The §6.3 classification table has no row for 'final status 3xx that is not in that set' (e.g. 300 Multiple Choices, 304 Not Modified, 305, 306, 310 — and 308 is sometimes returned without a Location). It is unclear whether such responses are 'ok' (final 3xx that's not 4xx/5xx), 'broken/bad_redirect', or 'broken/http_status'.
- (minor) §7.2 example sets `version: "0.1.0"` while §4.2's `--user-agent` default contains `tiny-linkrot/0.1`. The spec does not say where the version string is sourced (package.json? a constant in `cli.ts`? `bun build` injection?), nor whether the user-agent and JSON `version` must agree. This makes both fields prone to drift.
- (minor) §3 user story 3 advertises a report that surfaces a redirect as '301 → 200', implying the intermediate status is visible. §7.1's human report only specifies 'final URL and final status' on the REDIRECT line and does not mandate showing the chain. The user story and the output contract disagree on whether the intermediate status is visible by default.
- (minor) §7.1 says 'OK links are not printed in human mode' and qualifies it with 'suppressed by default', implying a flag exists to surface them. No such flag is listed in §4.2. Either drop 'by default' or define the override flag.
- (minor) §4.2 declares `--timeout` and `--concurrency` as integers but does not bound them. Behavior for `--timeout 0`, negative values, non-integer strings, or `--concurrency 100000` is undefined. At minimum, define the validation/exit-2 cases.
- (minor) §4.3 mandates 'a stack trace on stderr' for exit 3, but §7.2 says 'Progress and warnings must not appear on stdout in `--json` mode' and is silent on stderr in error scenarios. It is unclear whether a partial JSON document may have been written to stdout before the panic, and whether the consumer should expect valid JSON, partial JSON, or nothing at all on stdout when exit 3 fires mid-run.
- (minor) §7.1 orders entries 'by line number' but does not define a tiebreaker when multiple URLs appear on the same line (column order? extraction order? URL alphabetical?). Golden tests in §9 will be non-deterministic without a rule.
- (minor) §6.3 table classifies a generic 'Network error (DNS, TCP, TLS, reset)' as `network_error`, but §6.1 step 3 says some network errors trigger a GET retry. The interaction is not crisply stated: does `network_error` only apply after the GET fallback also fails, or after HEAD alone fails for non-retriable errors? A note that 'reason is assigned only after all permitted retries are exhausted' would close the gap.
- (minor) §5.3 step 3 says occurrences are recorded per `(file + line)`, but does not say whether the same URL appearing twice on the same line yields one occurrence or two. The JSON `occurrences` array shape and any dedup-within-line rule are unspecified.

## weak-testing

- (major) §9 has no test coverage for human-readable output (§7.1): grouping by line, color gating on `process.stdout.isTTY` AND absence of `NO_COLOR`, suppression of OK lines, the exact summary header, or the requirement that progress/warnings go to stderr while the report goes to stdout. The whole human-mode contract is untested.
- (major) §9 does not require tests for: the 10 MiB file rejection, unknown-flag handling (exit 2 + usage to stderr), zero/multiple positional args, `--concurrency` actually capping in-flight requests (e.g. observing peak concurrency against the fixture server), `--json` mode silencing stderr progress, dedup of repeat occurrences (file + line list), or that `redirect_chain` matches the actual hop sequence and statuses. The §12 acceptance criteria therefore cannot be mechanically verified.
- (major) §9 item 4 says 'one test per row of the §4.3 table', but §4.3 lists exit codes (0, 1, 2, 3), not the broken-reason matrix in §6.3. It is unclear whether 'one test per row' means four exit-code tests (insufficient — exit 2 alone has at least four distinct triggers) or one test per failure mode in §6.3. As written, the matrix is under-covered.
- (minor) §9 integration tests describe a fixture server returning specific statuses but do not require coverage of: HEAD returning a redirect that itself uses a 405-on-HEAD intermediate, a redirect with a relative `Location`, a redirect with no `Location`, an HTTPS endpoint with a bad TLS certificate, or IPv6/IDN hosts. These are exactly the cases where the §6.1 algorithm has the most subtle behavior.

## suggested-next-version

0.2

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "contradiction",
      "text": "§6.2 states the worst-case wall-clock for a URL is `6 × timeout` ('a URL that redirects five times may spend up to 6 × timeout'), but §6.1 step 3 permits a HEAD→GET fallback at each hop. With the 5-redirect cap (so up to 6 attempts in the chain) and a HEAD+GET on each, the real worst case is `12 × timeout`. Either the bound or the per-hop fallback rule must be corrected.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "§8.4 says network errors 'become `status: \"broken\"` entries with `reason: \"network_error\"` and a short human-readable `message` field in JSON output', but the §7.2 'Field contract' enumerates the JSON fields and never lists `message`. The example object also omits it. Implementers cannot tell whether `message` is part of the stable schema or not.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§6.1 step 3 retries with GET on '405 Method Not Allowed', '501 Not Implemented', or 'a network-level error that could plausibly be HEAD-specific (e.g. connection reset before headers)'. 'Plausibly HEAD-specific' is not implementable: two engineers will disagree on whether DNS NXDOMAIN, TLS handshake failure, ECONNREFUSED, mid-body reset, or a HEAD timeout qualifies. Provide an explicit allowlist of error classes that trigger GET retry.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§5.3 'URL normalization' only specifies WHATWG parsing, fragment stripping, and dedup by 'identical post-normalization URLs'. It does not say whether dedup is sensitive to host case, percent-encoding case, default-port presence (`:80`/`:443`), trailing slash on the path-empty case, query-parameter order, or duplicate query keys. These choices materially change the unique-URL count surfaced in the report and the §9 dedup tests.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§6.1 step 2 enumerates the redirect statuses it follows as 301/302/303/307/308. The §6.3 classification table has no row for 'final status 3xx that is not in that set' (e.g. 300 Multiple Choices, 304 Not Modified, 305, 306, 310 — and 308 is sometimes returned without a Location). It is unclear whether such responses are 'ok' (final 3xx that's not 4xx/5xx), 'broken/bad_redirect', or 'broken/http_status'.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "§5.2 lists GFM bare URLs as an extraction source 'when the parser surfaces them' — i.e., behavior is conditional on the chosen parser. §9 then mandates a unit test for 'GFM bare URL' extraction. If the implementer picks a parser without GFM autolink (e.g. stock `markdown-it` without the `linkify` option enabled), the spec simultaneously permits and forbids the behavior. Either require GFM autolink unconditionally or drop the test.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "§5.1 routes 'files larger than 10 MiB' to exit code 2, but §4.3 defines exit 2 as 'usage / argument error (bad flag, missing file, file not readable, not a file)'. A 10 MiB Markdown file is none of those. Either broaden the §4.3 definition to cover 'input rejected by precheck' or assign a different exit code.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§9 has no test coverage for human-readable output (§7.1): grouping by line, color gating on `process.stdout.isTTY` AND absence of `NO_COLOR`, suppression of OK lines, the exact summary header, or the requirement that progress/warnings go to stderr while the report goes to stdout. The whole human-mode contract is untested.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§9 does not require tests for: the 10 MiB file rejection, unknown-flag handling (exit 2 + usage to stderr), zero/multiple positional args, `--concurrency` actually capping in-flight requests (e.g. observing peak concurrency against the fixture server), `--json` mode silencing stderr progress, dedup of repeat occurrences (file + line list), or that `redirect_chain` matches the actual hop sequence and statuses. The §12 acceptance criteria therefore cannot be mechanically verified.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§9 item 4 says 'one test per row of the §4.3 table', but §4.3 lists exit codes (0, 1, 2, 3), not the broken-reason matrix in §6.3. It is unclear whether 'one test per row' means four exit-code tests (insufficient — exit 2 alone has at least four distinct triggers) or one test per failure mode in §6.3. As written, the matrix is under-covered.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§7.2 example sets `version: \"0.1.0\"` while §4.2's `--user-agent` default contains `tiny-linkrot/0.1`. The spec does not say where the version string is sourced (package.json? a constant in `cli.ts`? `bun build` injection?), nor whether the user-agent and JSON `version` must agree. This makes both fields prone to drift.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§3 user story 3 advertises a report that surfaces a redirect as '301 → 200', implying the intermediate status is visible. §7.1's human report only specifies 'final URL and final status' on the REDIRECT line and does not mandate showing the chain. The user story and the output contract disagree on whether the intermediate status is visible by default.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§7.1 says 'OK links are not printed in human mode' and qualifies it with 'suppressed by default', implying a flag exists to surface them. No such flag is listed in §4.2. Either drop 'by default' or define the override flag.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§4.2 declares `--timeout` and `--concurrency` as integers but does not bound them. Behavior for `--timeout 0`, negative values, non-integer strings, or `--concurrency 100000` is undefined. At minimum, define the validation/exit-2 cases.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§4.3 mandates 'a stack trace on stderr' for exit 3, but §7.2 says 'Progress and warnings must not appear on stdout in `--json` mode' and is silent on stderr in error scenarios. It is unclear whether a partial JSON document may have been written to stdout before the panic, and whether the consumer should expect valid JSON, partial JSON, or nothing at all on stdout when exit 3 fires mid-run.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§7.1 orders entries 'by line number' but does not define a tiebreaker when multiple URLs appear on the same line (column order? extraction order? URL alphabetical?). Golden tests in §9 will be non-deterministic without a rule.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§6.3 table classifies a generic 'Network error (DNS, TCP, TLS, reset)' as `network_error`, but §6.1 step 3 says some network errors trigger a GET retry. The interaction is not crisply stated: does `network_error` only apply after the GET fallback also fails, or after HEAD alone fails for non-retriable errors? A note that 'reason is assigned only after all permitted retries are exhausted' would close the gap.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§5.3 step 3 says occurrences are recorded per `(file + line)`, but does not say whether the same URL appearing twice on the same line yields one occurrence or two. The JSON `occurrences` array shape and any dedup-within-line rule are unspecified.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "§9 integration tests describe a fixture server returning specific statuses but do not require coverage of: HEAD returning a redirect that itself uses a 405-on-HEAD intermediate, a redirect with a relative `Location`, a redirect with no `Location`, an HTTPS endpoint with a bad TLS certificate, or IPv6/IDN hosts. These are exactly the cases where the §6.1 algorithm has the most subtle behavior.",
      "severity": "minor"
    }
  ],
  "summary": "The spec is in good shape for a v0.1, but several rules at the network layer are under-specified or self-contradictory in ways that will produce inconsistent implementations and untestable acceptance criteria. The most pressing issues are: (1) the worst-case timeout bound in §6.2 contradicts the per-hop HEAD→GET fallback in §6.1; (2) §8.4 introduces a JSON `message` field that §7.2's stable schema omits; (3) the 'plausibly HEAD-specific' criterion in §6.1 step 3 is not implementable; (4) §5.3 normalization/dedup leaves out host case, default ports, trailing slashes, and query ordering; and (5) §6.3's classification table doesn't cover 3xx statuses outside the followed set. The test plan in §9 is the weakest section: it does not exercise human-mode output, color/TTY gating, stdout/stderr separation, the 10 MiB cap, unknown-flag handling, concurrency-cap enforcement, dedup occurrences, or the redirect_chain contents — all of which §12 implicitly requires for 'done'. Several minor ambiguities (version source, line-tiebreak ordering, timeout/concurrency bounds, exit-3 behavior in --json) should be tightened before implementation.",
  "suggested_next_version": "0.2",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
