# tiny-linkrot — SPEC v0.4

## 1. Purpose

`tiny-linkrot` is a small Bun + TypeScript CLI that scans a directory of Markdown files, extracts every HTTP(S) link, probes each link over the network, and prints a human-readable report of broken and redirected links grouped by source file. It is designed to be run locally by authors and in CI to prevent link rot in documentation.

## 2. Persona & Scope

This spec is authored for a veteran CLI software engineer. It assumes familiarity with Bun, TypeScript, `fetch`, Markdown tooling, POSIX exit codes, and CI conventions. Out of scope for v0.1: JSON/SARIF output, auth-gated links, JavaScript-rendered pages, non-HTTP schemes (mailto:, ftp:, ...), caching across runs, and fixing links.

## 3. Runtime & Distribution

- **Runtime:** Bun ≥ 1.1 (uses built-in `fetch`, `Bun.Glob`, `Bun.file`).
- **Language:** TypeScript, `"type": "module"`.
- **Entry point:** `bin/linkrot.ts` exposed as `linkrot` via `package.json#bin`.
- **Install:** `bun install` + `bun link`, or `bunx tiny-linkrot`.
- **No runtime deps** beyond Bun built-ins for v0.1. Dev deps: `typescript`, `@types/bun`, a Markdown parser (see §5).

## 4. CLI Surface

```
linkrot [path] [options]
```

`path` defaults to `.` (current working directory) and may be a file or a directory. Directories are scanned recursively.

### Options (v0.1)

| Flag | Default | Meaning |
| --- | --- | --- |
| `--concurrency N` | `8` | Size of the global worker pool. |
| `--timeout MS` | `10000` | Per-request timeout (connect + read) in ms. |
| `--retries N` | `1` | Retries on network errors and 5xx (excluding 501, 505). Exponential backoff: 200ms × 2^n, jittered ±25%. |
| `--user-agent STRING` | `tiny-linkrot/<version> (+https://github.com/NikolayS/tiny-linkrot)` | Value of the `User-Agent` header. |
| `--include GLOB` (repeatable) | `**/*.md`, `**/*.markdown` | File globs to scan. |
| `--exclude GLOB` (repeatable) | `**/node_modules/**`, `**/.git/**` | File globs to skip. |
| `--allow-host HOST` (repeatable) | *(none)* | Host allowlist. If set, only these hosts are checked; all other links are reported as `skipped:host`. |
| `--deny-host HOST` (repeatable) | *(none)* | Host denylist. Matching links are reported as `skipped:host`. |
| `--accept-redirects` | off | Treat 3xx → 2xx chains as `ok` instead of `redirect`. |
| `--fail-on LEVEL` | `broken` | One of `broken`, `redirect`, `any`. Controls which findings flip the exit code. |
| `--no-color` | off | Disable ANSI color in the report. Auto-disabled when stdout is not a TTY. |
| `--version`, `-v` | — | Print version and exit 0. |
| `--help`, `-h` | — | Print help and exit 0. |

Unknown flags cause exit code 2 (usage error) with a one-line error on stderr.

## 5. Link Extraction

- Markdown is parsed with a real parser (v0.1: `marked` or equivalent) — **not** regex — so that links inside code fences and inline code spans are ignored.
- Extracted link kinds:
  - `[text](url)` inline links
  - `[text][ref]` + `[ref]: url` reference links
  - Bare autolinks `<https://example.com>`
  - Image links `![alt](url)` are also checked.
- Only `http:` and `https:` URLs are probed. `mailto:`, `tel:`, relative links, and fragment-only (`#foo`) links are ignored silently.
- URLs are normalized before deduplication: lowercase scheme+host, default-port stripping, empty path → `/`, percent-encoding normalized, fragment stripped. Query strings are preserved as-is.
- A URL that appears in multiple files is probed **once** and its status fanned out to all source locations in the report.

## 6. HTTP Probing

### 6.1 Request shape

1. Send `HEAD` with `redirect: "follow"`, configured timeout, and the CLI's `User-Agent`.
2. If the final status is `405 Method Not Allowed`, `403`, `400`, or the server returns a network-level failure that could plausibly be a HEAD-averse server (e.g., RST, EOF), retry once with `GET` using a streaming body that is aborted as soon as headers arrive.
3. Honor up to 10 redirects. More than 10 → `broken:redirect-loop`.
4. Apply `--timeout` to the **whole** request including all redirect hops.

### 6.2 Classification (per unique URL)

| Class | Trigger |
| --- | --- |
| `ok` | Final response 2xx. |
| `redirect` | At least one 3xx in the chain, final response 2xx, and `--accept-redirects` not set. The report shows the final URL. |
| `broken:http` | Final response 4xx or 5xx after following redirects. |
| `broken:network` | DNS failure, connection refused, TLS error, reset, timeout after retries. |
| `broken:redirect-loop` | > 10 hops or a cycle detected. |
| `skipped:host` | Host excluded via `--allow-host`/`--deny-host`. |
| `skipped:scheme` | Non-HTTP(S) scheme (reported only in `--help`-level verbose mode; otherwise filtered silently). |

"Broken" per the interview contract = `broken:*`. `redirect` is informational by default and only fails CI when `--fail-on=redirect` or `--fail-on=any`.

### 6.3 Retries & backoff

- Retries apply to `broken:network` and to final 5xx **except** 501 and 505.
- 429 and 503 with a `Retry-After` header respect that header up to `--timeout`, counted against `--retries`.
- Retries never apply to 4xx other than 408, 425, 429.

## 7. Concurrency Model

- A single global fixed-size worker pool of `--concurrency` workers consumes a queue of unique URLs.
- No per-host throttling in v0.1 (explicit non-goal, documented in §11).
- File scanning and URL extraction run to completion **before** probing starts, so the total URL count is known and the progress display is accurate.
- Cancellation: `SIGINT`/`SIGTERM` aborts in-flight requests, drains the queue, and prints the partial report with an `interrupted` banner. Exit code in that case is 130.

## 8. Output

### 8.1 Human-readable TTY report (only format in v0.1)

The report is grouped by source file, in the order the files were discovered (stable, sorted lexicographically by path relative to CWD). Example:

```
docs/intro.md
  ✗ https://example.com/gone               404 Not Found
    line 12
  ↪ http://example.com/moved               → https://example.com/moved  (301)
    line 27, line 41
  ✗ https://unreachable.invalid            network: getaddrinfo ENOTFOUND
    line 33

README.md
  ✓ 14 ok, 0 redirects, 0 broken

Summary
  files scanned : 23
  links checked : 118 unique  (142 occurrences)
  ok            : 110
  redirects     :   5
  broken        :   3
  skipped       :   0
  duration      : 4.1s
```

- Only files with at least one non-`ok` finding are expanded; files that are fully clean are collapsed into a single `✓ N ok` summary line.
- Colors: green `✓`, yellow `↪`, red `✗`, dim for line numbers and final URLs. Respect `NO_COLOR` and `--no-color`.
- Progress while running: a single-line updating counter on stderr (`[ 42 / 118 ] checking…`), suppressed when stderr is not a TTY.

### 8.2 Stderr vs stdout

- Report goes to **stdout**.
- Progress, warnings, and errors go to **stderr**.
- This keeps `linkrot > report.txt` clean in CI.

## 9. Exit Codes

| Code | Meaning |
| --- | --- |
| 0 | Scan completed; nothing at or above `--fail-on` was found. |
| 1 | Scan completed; at least one finding at or above `--fail-on`. Default (`--fail-on=broken`) ⇒ non-zero exit on any broken link, matching the interview answer. |
| 2 | Usage error (unknown flag, bad value, path does not exist). |
| 3 | Internal error (unhandled exception). Stack trace on stderr. |
| 130 | Interrupted by `SIGINT`/`SIGTERM`. Partial report still printed. |

## 10. Configuration Precedence

CLI flags > environment variables > built-in defaults. Environment variables, all optional:

- `LINKROT_CONCURRENCY`, `LINKROT_TIMEOUT`, `LINKROT_RETRIES`, `LINKROT_USER_AGENT`, `LINKROT_NO_COLOR`.

No config file in v0.1.

## 11. Non-Goals (v0.1)

- JSON, SARIF, JUnit, or GitHub Actions annotation output.
- Per-host rate limiting, robots.txt, or crawl politeness beyond a single user-agent string.
- Authenticated requests, cookies, or proxies.
- Link checking in non-Markdown formats (HTML, MDX, rST, AsciiDoc).
- Cross-run caching or incremental mode.
- Auto-fixing or suggesting replacements.

These are deliberately parked to keep v0.1 tight; §14 tracks them.

## 12. Error Handling & Robustness

- File read errors abort the run with exit 2 and a clear path in the error message; partial reports are not produced for read failures (distinct from network failures).
- Malformed Markdown does not abort: the parser's recovery mode is used and any unparseable region is skipped with a stderr warning.
- Extremely large files (> 5 MB) emit a stderr warning but are still scanned.
- A URL string that fails `new URL(...)` parsing is reported as `broken:network` with reason `invalid-url`.

## 13. Testing Strategy

- **Unit tests** (`bun test`): Markdown extraction (fences, reference links, autolinks, images), URL normalization, classification table, exit-code mapping.
- **Integration tests**: a local Bun HTTP server fixture that returns curated 2xx/3xx/4xx/5xx/timeout/redirect-loop responses, driven by the real CLI binary.
- **Golden report tests**: snapshot the TTY report against fixture directories, with color forced off.
- **No live network** in CI. A separate opt-in `bun run test:live` hits a small set of pinned URLs for smoke coverage.

## 14. Future Work (post-v0.1)

- `--format json|sarif|junit` for machine-readable output.
- Per-host concurrency caps and `Retry-After`-aware global backoff.
- On-disk cache keyed by URL + ETag/Last-Modified.
- HTML / MDX / AsciiDoc extractors behind a pluggable extractor interface.
- `--fix` suggestions using redirect targets.
- GitHub Actions wrapper that turns `redirect`/`broken` findings into PR annotations.
- Proxy support (`--proxy`, `--use-env-proxy`). Deferred out of v0.4 per review r03; to return once the pinned-transport path is demonstrably solid and the HTTP-target-through-proxy, proxy-SSRF, and forward-proxy questions have been specified end-to-end.

## 15. Open Questions

1. Should reference-style link labels that are defined but never used be reported as warnings? (Leaning **no** for v0.1 — out of scope for link rot.)
2. Should we treat 2xx responses with suspicious bodies (e.g., soft-404s like `<title>Not Found</title>` returning 200) as broken? (Leaning **no** — false-positive risk too high without per-site heuristics.)
3. Is `User-Agent` customization enough, or do we also need `--header K:V` for niche sites that demand `Accept: text/html`? (Defer to v0.2.)

## 16. Security — SSRF & Outbound Destination Policy

Revised in v0.4 to close the evaluation-order, IP-literal, and resolver-API gaps flagged in review r03.

### 16.1 Default-deny private destinations

Every probe performs its own DNS resolution (A and AAAA) and inspects the resolved addresses *before* opening a TCP connection. If any resolved address falls in a reserved range, the probe is not issued and the link is classified as `broken:blocked-destination` (see §16.10 for the change from v0.3's `skipped:*` bucket). Reserved ranges:

- IPv4 loopback `127.0.0.0/8`, private `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, link-local `169.254.0.0/16` (covers cloud metadata `169.254.169.254`), CGNAT `100.64.0.0/10`, multicast `224.0.0.0/4`, broadcast `255.255.255.255`, reserved `0.0.0.0/8`, `192.0.0.0/24`, `198.18.0.0/15`, `240.0.0.0/4`.
- IPv6 loopback `::1/128`, unique-local `fc00::/7`, link-local `fe80::/10`, multicast `ff00::/8`, documentation `2001:db8::/32`, IPv4-mapped (`::ffff:0:0/96`) that map into any blocked IPv4 range.

### 16.2 Mixed-answer handling

If a host resolves to both blocked and non-blocked addresses, the host is treated as blocked. This defeats DNS-rebinding answers that mix public and private addresses.

### 16.3 IP-literal URLs

If the URL host is an IP literal (dotted-quad IPv4, bracketed IPv6, or an `::ffff:a.b.c.d` mapped literal), **no DNS resolution is performed**. The literal is classified directly per §16.1 and, if not blocked, is pinned to itself for connect. Behavioral notes:

- IPv6 zone-identifier suffixes (`fe80::1%eth0`) are rejected at URL parse time as `broken:invalid-url`; the policy intentionally does not probe scoped addresses.
- Non-canonical IPv4 literal forms that are legal per browsers but ambiguous in tooling (octal `0177.0.0.1`, decimal `2130706433`, hex `0x7f000001`, short `127.1`) are normalized to their canonical dotted-quad before classification. An implementation that cannot normalize one of these forms MUST refuse to probe the URL as `broken:invalid-url` rather than passing it through.
- IPv4-mapped IPv6 literals are classified both against the IPv6 ranges in §16.1 and, after unmapping, against the IPv4 ranges.

### 16.4 Transport pinning (defeats DNS rebinding / TOCTOU)

The implementation MUST connect to a specific vetted IP address rather than re-resolving the hostname inside the runtime's `fetch`. Concretely:

- The probe layer resolves the hostname once using Node's `node:dns/promises` `resolve4` and `resolve6` (equivalently, c-ares-style direct DNS queries). It MUST NOT use `dns.lookup` / `getaddrinfo`, because `/etc/hosts` and `nsswitch.conf` can introduce platform-dependent results and hide rebinding vectors. The classifier inspects the full union of A and AAAA answers atomically.
- Every returned address is classified per §16.1. A single non-blocked address is selected for connect (IPv6 preferred when both families are present and non-blocked, falling back to IPv4; a `--prefer-ipv4` flag may be added later but is not in v0.4).
- The TCP connection is opened to that exact IP. The HTTP `Host` header and TLS SNI carry the original hostname for correct virtual-hosting and certificate validation.
- TLS certificate validation MUST be performed against the original hostname; the connection-time IP is not used for certificate identity.
- Implementation seam: a custom `dispatch` / `connect` hook on the HTTP client (e.g., undici-style, or a Bun socket-level transport). Calling Bun's high-level `fetch` with the original URL is **not** sufficient on its own because Bun may re-resolve internally.
- A short-TTL DNS cache (≤ 30 s, per-run, in-process) is permitted for the *classification* step; the *connect* step always uses the address selected by classification, never a cache-bypassing re-resolution.
- If the underlying runtime cannot guarantee that `connect` uses the supplied address (no socket-level seam available), the implementation MUST fail closed: probing is disabled and the run exits 3 with a clear error and **no stack trace** (see §20 for the §9-exit-3 clarification distinguishing capability errors from unhandled exceptions). Silent fallback to runtime-default resolution is forbidden.

### 16.5 Per-hop re-check

The policy is applied again before each redirect hop. A redirect target whose host resolves to a blocked address (or mixed) produces `broken:blocked-destination-redirect` and the chain is terminated. The displayed final URL is the refused target rendered as `<host-rendered>[:port]/<elided>` where `<host-rendered>` is produced by §17.5 (so IDN hosts show `<punycode> (<unicode>)`). Path, query, and fragment are replaced by the literal `<elided>` to avoid leaking attacker-controlled data. The number of public hops traversed before the refusal is shown next to the arrow.

### 16.6 SSRF-stage DNS failure

If the SSRF-stage resolution fails (NXDOMAIN, SERVFAIL, timeout, no addresses), the link is classified as `broken:network` with reason `dns:<code>` (e.g., `dns:nxdomain`). SSRF block is only triggered by *successful* resolution to blocked space.

### 16.7 Opt-in escape hatches

- `--allow-internal` disables the SSRF block for the whole run. When set, the report is bracketed by a `⚠ internal destinations enabled` banner on **both** stdout and stderr, and the per-link rendering of any internal destination is also marked with `⚠`.
- `--allow-internal-host HOST_OR_CIDR` (repeatable) is the **scoped** alternative: only matching destinations bypass the block; everything else still defaults to deny. Accepted forms: bare hostname (canonicalized per §18), IPv4/IPv6 literal, or CIDR. The summary lists `internal allowances: <count>` instead of the full-run banner. A scoped allowance applies to the redirect-hop check too.
- `--allow-internal` and `--allow-internal-host` may be combined. Precedence when both are set: `--allow-internal` fully supersedes scoped allowances (every internal destination is allowed; the full-run banner is shown). When only `--allow-internal-host` is set, only matching destinations are allowed. "Broader wins" in §16.7 means exactly this — it does **not** mean union-of-CIDRs or any other set operation.
- Both flags are CLI-only and have no env-var counterpart.

### 16.8 Interaction with `--allow-host` / `--deny-host`

The evaluation order below is corrected from v0.3 so that the scoped internal allow can actually bypass the SSRF block (the r03 reviewer noted v0.3's order made `--allow-internal-host` impossible).

**Evaluation order, per link:**

1. **Deny match (`--deny-host`)** → `skipped:host`. Terminal.
2. **Allowlist gate (`--allow-host`)**. If `--allow-host` is set and the URL host does not match, → `skipped:host`. Terminal. Otherwise continue.
3. **Scoped internal allow (`--allow-internal-host` or `--allow-internal`)**. If the destination would otherwise be blocked by §16.1 and a matching allowance covers it, the SSRF block is bypassed for this URL. Continue to probe.
4. **SSRF block (§16.1 + §16.2)**. Blocked destinations not covered by step 3 → `broken:blocked-destination` (direct) or `broken:blocked-destination-redirect` (redirect hop). Terminal.
5. **Probe.**

`--allow-host` membership does **not** bypass the SSRF block on its own; only an `--allow-internal*` flag can do that.

### 16.9 Userinfo handling

`user:pass@` in URLs is never passed to the transport. The probe layer strips userinfo from the URL string *before* handing it to `fetch` / the dispatcher, so the runtime cannot synthesize an `Authorization: Basic …` header. The displayed form replaces it with `<redacted>@` (see §17). Dedup normalization in §5 also drops userinfo so credential variants collapse into one probe.

### 16.10 Classification & summary mapping (binding for v0.4)

§6.2 is frozen text from v0.1; the v0.2–v0.4 additions are mapped here. **Changed from v0.3:** direct blocked destinations now land in `broken:blocked-destination` (not `skipped:*`) so the default `--fail-on=broken` fails CI on a newly introduced `http://127.0.0.1/...` or `http://169.254.169.254/...` link, as review r03 required.

| Class | Summary bucket | `--fail-on=broken`? | `--fail-on=any`? | Retried? |
| --- | --- | --- | --- | --- |
| `broken:blocked-destination` | `broken` | yes | yes | no |
| `broken:blocked-destination-redirect` | `broken` | yes | yes | no |
| `broken:invalid-url` (per §20) | `broken` | yes | yes | no |
| `skipped:url-too-long` (per §23) | `skipped` | no | yes | no |
| `skipped:budget-exceeded` (per §23) | `skipped` | no | yes | no |

Notes:

- `skipped:budget-exceeded` only appears on per-URL non-aborting budgets (e.g., a URL that exceeds `--max-url-length`). The aborting budgets (`--max-files`, `--max-urls`, `--max-occurrences`) exit 2 before probing; their rows are not in this table because exit 2 always wins over `--fail-on` (see §20).
- `--fail-on=redirect` covers `redirect` only; it does not flip on `skipped:*` or on the new `broken:*` classes (which are already covered by `--fail-on=broken`).

### 16.11 Proxy support

Deferred out of v0.4 (moved to §14 future work). v0.3 sketched `--proxy` / `--use-env-proxy` with CONNECT-only semantics, but the HTTP-target-through-proxy behavior, proxy-side SSRF guarantees, and forward-proxy handling were under-specified. The hardening story in v0.4 is the pinned non-proxy transport; a proxy feature will return in a later spec round once those questions have explicit answers.

By default, `linkrot` **ignores** `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, and `NO_PROXY` environment variables — CI environments that inject these for other tools do not silently route linkrot through them.

## 17. Output Redaction & Control-Character Sanitization

Revised in v0.4. The sensitive-parameter dedup of v0.3 is removed (per review r03 — it could suppress genuinely broken URLs behind healthy siblings sharing a parameter name).

All URLs, file paths, and free-form messages that originate from user-controlled input pass through a render filter before they are written to stdout, stderr, or the progress line.

### 17.1 Userinfo

The `user:pass@` component is removed from URLs in the rendered form and replaced with `<redacted>@`. The same removal happens at the transport boundary (§16.9). Dedup normalization (§5) drops userinfo so credential variants collapse into one probe.

### 17.2 Sensitive query parameters (default mode)

Parameter values are replaced with `<redacted>` **in the rendered form only** (the original value is still used for the probe; dedup is unaffected) when the parameter name matches, **ASCII-case-insensitively**, any of: `token`, `access_token`, `id_token`, `refresh_token`, `api_key`, `apikey`, `key`, `secret`, `signature`, `sig`, `password`, `auth`, `x-amz-signature`, `x-amz-credential`, `x-amz-security-token`. The list is a baseline and is not authoritative for every site.

Deduplication uses the **un-redacted** normalized URL (per §5). Two URLs that differ only in a sensitive-parameter value are probed separately — the v0.3 behavior of collapsing them is removed because it could hide a broken URL behind a healthy sibling that shares the same parameter name. Operators who want to reduce probe count on high-cardinality bearer values should use `--strict-redaction` (§17.3), which is opt-in and intentionally collapses.

### 17.3 Strict redaction mode

`--strict-redaction` (default off; recommended on for CI logs that may be archived or shared) replaces **every** query-parameter value and every URL path segment that looks like opaque bearer material with `<redacted>`. The opaque-segment test is:

- Length ≥ 24 characters, **and**
- ≥ 60% of characters are members of urlsafe-base64 or hex alphabet (`A-Z`, `a-z`, `0-9`, `-`, `_`).

The v0.3 "no path-typical extension" carve-out is removed; the rule depends purely on length and character composition so that dedup keys are stable across implementations.

In strict mode, and **only in strict mode**, the dedup key is the post-redaction URL, so distinct opaque tokens collapse to one probe. When two source occurrences collapse to one probe, the **first occurrence in scan order** (file order per §8.1, then ascending line number within the file) selects the URL that is actually sent on the wire. This selection is deterministic and covered by a unit test (§21).

### 17.4 Control and format characters

Before emission, the following are replaced by their `\uXXXX` escape:

- ASCII C0 controls except `\t`, DEL (`\x7f`), C1 controls (`\x80`–`\x9f`).
- ANSI/OSC introducers `\x1b`, `\x9b`.
- Bidi overrides U+202A–U+202E, U+2066–U+2069.
- Line and paragraph separators U+2028, U+2029.
- Zero-width and BOM U+200B–U+200F, U+FEFF.

Line numbers, counts, and literal report chrome are safe and are not rewritten.

### 17.5 Hostname rendering

Hosts that contain non-ASCII labels, **or** hosts that the user typed as an A-label literal (`xn--...`) whose Unicode form differs from the ASCII form, are rendered as `<punycode> (<unicode>)`, e.g., `xn--mnchen-3ya.de (münchen.de)`. The Punycode form is what §18 matches against; showing both forms makes spoofing-via-confusables visible and applies uniformly whether the input was typed in Unicode or already as an A-label. Pure-ASCII hosts whose A-label form equals the user input are rendered as-is. This rendering applies to all URLs in the report, including refused-redirect targets rendered per §16.5.

### 17.6 Scope

The filter applies uniformly to the TTY report, the stderr progress line, warnings, error messages, and any future machine-readable output.

## 18. Host Matching Canonicalization

Revised in v0.4. Bare-hostname entries now match the **apex only** (v0.3 matched apex + all subdomains, which review r03 flagged as too broad a default for a safety-sensitive flag).

Applies to `--allow-host`, `--deny-host`, `--allow-internal-host`, and any future host-scoped flag or policy.

- Both the flag value and the URL host are converted to their IDNA A-label (Punycode) form and case-folded to lower case before comparison.
- A trailing dot on either side is stripped.
- IPv6 literals on the CLI are written **without** surrounding brackets when no port is attached (e.g. `--deny-host ::1`) and **with** brackets when a port is attached (e.g. `--deny-host [::1]:8443`). Brackets are the disambiguator that resolves the colon collision noted in review r02.
- IPv4 and IPv6 literals are compared after dotted-quad (IPv4) or RFC 5952 (IPv6) canonicalization.
- Match semantics by prefix (DNS names only):
  - `example.com` — matches the **apex only** (changed from v0.3).
  - `.example.com` — matches subdomains only, not the apex.
  - `*.example.com` — matches apex **and** all subdomains (explicit opt-in; the only glob character accepted, and only in this exact position).
  - `=example.com` is still accepted as a synonym of the bare form for backwards compatibility with v0.3 allowlists, and matches the apex only.
- The `=`, `.`, and `*.` prefixes are **rejected with exit 2** when applied to an IP-literal entry; they are nonsensical for IPs.
- Ports are not part of the host *identity*. A port-scoped entry uses `host:port` for DNS hosts and `[ipv6]:port` for IPv6 literals, and matches only that exact port. Port-scoped entries are matching rules; per-host *accounting* (§19) remains port-agnostic.
- CIDR is supported for `--allow-internal-host` only (not for `--allow-host` / `--deny-host`).
- Glob characters other than the exact `*.` apex-plus-subdomains prefix are not supported in v0.4.
- Evaluation order is given in §16.8.

## 19. Per-Host Concurrency & Backpressure

Revised in v0.4 to specify slot lifecycle on redirect hops and to name the host that triggers backpressure.

§11 non-goal reconciliation: §11's non-goal on "per-host rate limiting" refers to crawl politeness / global rate-limits per host as a courtesy toward upstream. §19 is a defensive concurrency bound and a short reactive backoff on explicit upstream signals (429 / 503). §20 binds this reading — §19 supersedes §11 for this aspect in v0.3+.

- Default per-host cap: **4** concurrent in-flight requests.
- Override via `--per-host-concurrency N`. Valid range: `1 ≤ N ≤ --concurrency`. `N < 1` or `N > --concurrency` is rejected at startup with exit 2.
- Host identity uses the canonical form from §18, **port-agnostic**, for the *accounting bucket*. A single host serving on ports 80, 443, and 8080 therefore shares one cap. The `host:port` syntax in §18 only affects matching of allow/deny rules, not which bucket counts a request. Port-scoped allow entries do not bump the cap for the matching host.
- The global `--concurrency` pool remains the primary limit; per-host cap only gates dispatch.
- Backpressure on signals from a host:
  - A `429` or `503` *with* `Retry-After` reduces **the emitting host's** effective cap to 1 for the remainder of the run. When the response was emitted mid-chain (e.g., a 503 at hop 2 of a 3-hop chain), the reduction applies to the responder, not to the originating host or any intermediate host.
  - A `429` or `503` *without* `Retry-After` likewise reduces the cap to 1 and additionally inserts a 1 s synthetic backoff before the next dispatch to that host.
  - The reduced cap is sticky for the run; it is not restored by subsequent successful responses.
  - When the existing cap is already 1, the rule is a no-op (no further reduction).
- **Redirect hop slot lifecycle.** A probe holds one global slot for the duration of the probe (across all hops). Per-host slots behave as follows:
  1. When dispatch begins to host A, the probe acquires a per-host slot on A.
  2. When A returns a redirect response, **A's per-host slot is released immediately** after the response is consumed, before the probe waits for B's bucket.
  3. The probe then waits for a per-host slot on B (the redirect target's host). The global slot is **held** throughout this wait to preserve the global cap invariant.
  4. If B == A (same-host hop), the slot on A is released in step 2 and re-acquired in step 3; the probe rejoins A's queue at the tail (no priority for in-flight chains).
  5. The wait in step 3 is bounded by the remaining `--timeout` budget for the probe. Timeout during the wait classifies the probe as `broken:network` with reason `timeout-per-host-queue`.

## 20. Clarifications

The user-frozen sections above carry some ambiguities; these clarifications are binding where they narrow language elsewhere.

- **Version labeling.** "v0.1" labels in §§2, 3, 4, 8, 11, 13, 14 describe the surface captured in this v0.4 document. They are not deferrals past v0.4.
- **§11 vs §19 reconciliation.** §11 parks "per-host rate limiting, robots.txt, or crawl politeness beyond a single user-agent string" as out of scope. §19 implements a defensive per-host concurrency cap and a short reactive backoff on upstream 429/503. These are not the same concept: §11's non-goal is outbound politeness / global rate-limits as a courtesy; §19 is a defensive cap against pool-exhaustion and a short reaction to explicit upstream signals. §19 supersedes §11 for this aspect starting in v0.3.
- **Timeout scope.** `--timeout` is a **whole-request** budget covering DNS, connect, TLS, write, read, and all redirect hops. §6.1 step 4 is the authoritative reading; §4's phrase "connect + read" is a description of what the budget covers, not a per-hop timer.
- **HEAD → GET fallback.** The method switch in §6.1 step 2 is **not** a retry and does not consume a `--retries` slot. Retry policy in §6.3 then applies to the GET in the usual way.
- **Invalid URL class.** A URL string that fails `new URL(...)` parsing is classified as `broken:invalid-url`, counted under `broken` in the summary (per §16.10), honored by `--fail-on=broken`, and never retried. §12's reference to `broken:network` with reason `invalid-url` is superseded.
- **§12 vs §20 file-failure reconciliation.** §12 (frozen) says file read errors abort the run with exit 2 and no partial report. §20's "Extraction failure on one file" rule applies to fatal *parser* errors during the extraction phase (after the scan root has been enumerated successfully) — I/O errors on individual files encountered during extraction count as per-file skips, not as whole-run aborts. §12's rule governs the initial scan root: if the argument `path` cannot be read at all (ENOENT, EACCES on the root, read error on a single-file argument), the run aborts with exit 2.
- **Manual redirect following.** The implementation must use `redirect: "manual"` (or an equivalent manual-follow seam) so that hop count, cycle detection, and the SSRF per-hop re-check (§16.5) are deterministic regardless of Bun's internal redirect cap.
- **Retry-After.** Parsed as both delta-seconds and HTTP-date. The wait is capped to `cap = min(Retry-After, --timeout − elapsed)`. **If `cap > 0`**: the probe waits `cap`, the subsequent attempt consumes one `--retries` slot, and the wait counts against the URL's wall-clock `--timeout` budget. **If `cap ≤ 0`**: no further retry is issued; the link resolves on its current state (e.g., the 429/503 is final); **no `--retries` slot is consumed**. This supersedes the conflicting wording that appeared in v0.3.
- **Backoff (binding reading of §4).** §4 specifies `200ms × 2^n, jittered ±25%`. The binding reading is: for retry attempt `n` (1-indexed, so the first retry has `n=1`), the base wait is `200ms × 2^n` (so 400 ms at `n=1`, 800 ms at `n=2`, …), and the actual wait is sampled uniformly from `[base × 0.75, base × 1.25]` (multiplicative ±25%). The wait is truncated so total URL wall-clock does not exceed `--timeout`.
- **`--include`/`--exclude` merge.** User `--include` values **replace** the built-in includes. User `--exclude` values **append** to the built-in excludes. `--no-default-excludes` disables the built-in excludes.
- **Scheme-skip accounting.** URLs filtered for non-HTTP(S) schemes (including `data:` image URLs) are excluded from both "links checked" and "skipped" in the summary and appear nowhere in the output. Reference-style image links `![alt][ref]` are handled identically to non-image reference links.
- **Image probing.** On by default in v0.4 per §5. `--no-images` disables it without affecting other behavior.
- **Fan-out rendering.** A URL present in multiple files is probed once. In the report it appears once under **each** file where it occurs; each occurrence shows that file's own line numbers. In the summary, `links checked: N unique (M occurrences)` reports N as unique URLs probed and M as the total source-location count; `ok`, `redirects`, `broken`, `skipped` count unique URLs. Worked example:

  ```
  docs/a.md
    ✗ https://example.com/gone               404 Not Found
      line 4
  docs/b.md
    ✗ https://example.com/gone               404 Not Found
      line 9, line 12

  Summary
    files scanned : 2
    links checked : 1 unique (3 occurrences)
    ok            : 0
    redirects     : 0
    broken        : 1
    skipped       : 0
  ```

- **Extraction failure on one file.** A *fatal* parser error on a single file is treated as a skip (stderr warning, file excluded, other files still scanned). "Fatal" means the parser cannot produce a token stream for the file at all (I/O failure during read, decoder failure, or parser exception). A file where the parser recovers and emits a partial token stream — with localized warnings for the unparseable regions — counts as **scanned**, not skipped. The summary's `files scanned` counts fully- and partially-extracted files; a `files skipped` line appears when non-zero. Total URL count for progress reflects the successfully extracted set.
- **Exit code 2 scope.** §9 defines exit 2 as "Usage error (unknown flag, bad value, path does not exist)." v0.4 extends the *scope* (not the name) of exit 2 to also cover work-budget exhaustion under `--max-files`, `--max-urls`, and `--max-occurrences` (§23). Rationale: these are pre-probe resource guards whose exhaustion signals a misconfigured invocation relative to the input, which is substantively a usage error. The stderr message always names the budget (e.g., `error: --max-urls=50000 exceeded after enumerating 50001 unique URLs`) so a CI script that needs to branch on "bad-flag vs budget" can do so by parsing the stderr prefix. A dedicated exit code for budget-exhaustion remains deferred to a future major version.
- **Exit code 3 scope.** §9 defines exit 3 as "Internal error (unhandled exception). Stack trace on stderr." v0.4 extends the scope to also cover deterministic capability errors detected at startup, specifically the §16.4 fail-closed case when the runtime lacks a socket-level seam. Capability errors MUST NOT print a stack trace — only a single-line message naming the missing capability and the flag or runtime condition that triggered the check. Unhandled exceptions continue to print a stack trace.
- **Env-var scope.** `--include`, `--exclude`, `--allow-host`, `--deny-host`, `--fail-on`, `--accept-redirects`, `--allow-internal`, `--allow-internal-host`, `--per-host-concurrency`, `--no-images`, `--strict-redaction`, `--follow-symlinks`, `--max-files`, `--max-urls`, `--max-occurrences`, `--max-url-length`, `--max-redirect-headers`, and `--no-default-excludes` are intentionally CLI-only. They materially affect safety, classification, or work bounds and must be explicit at invocation.

## 21. Supplemental Testing Requirements

Revised in v0.4. In addition to §13, the following test cases are required.

- **Signals.** SIGINT and SIGTERM during probing → exit 130, partial report emitted, in-flight requests aborted.
- **Redirect limits.** Chain of exactly 10 hops (pass), 11 hops (→ `broken:redirect-loop`), 3-node cycle (→ `broken:redirect-loop`).
- **Retry-After.** Delta-seconds, HTTP-date, value greater than `--timeout` (→ `cap ≤ 0` → final on current state, no retry slot consumed), value of `0` (→ `cap = 0` → final, no retry slot consumed), malformed value, and `429`/`503` *without* `Retry-After` (cap reduction + 1 s synthetic backoff per §19).
- **Method switch.** HEAD → GET on 400, 403, 405; verify the switch does not consume a retry slot.
- **Precedence.** Flag > env > default for every flag that has an env-var counterpart in §10.
- **Host canonicalization.** IDNA/Punycode, ASCII case folding, trailing dot, IPv6 literal (with and without bracketed `[ipv6]:port` form), apex-only (bare and `=`) matching, subdomain-only (`.`) prefix, apex-plus-subdomains (`*.`) prefix, rejection of `=`/`.`/`*.` on IP-literal entries, rejection of `--per-host-concurrency` outside `[1, --concurrency]` with exit 2.
- **SSRF policy — direct.** Direct `http://127.0.0.1`, `http://[::1]`, `http://169.254.169.254`, and a hostname resolving only to RFC1918 → `broken:blocked-destination`, trips `--fail-on=broken` (default), counted under `broken`, never retried.
- **SSRF policy — IP literals.** Non-canonical forms (`0177.0.0.1`, `2130706433`, `0x7f000001`, `127.1`, `::ffff:127.0.0.1`) are either normalized and blocked, or refused as `broken:invalid-url`. Never probed through.
- **SSRF policy — zone id.** `fe80::1%eth0` → `broken:invalid-url`.
- **SSRF policy — redirect.** Public host redirects to a private destination → `broken:blocked-destination-redirect`, trips `--fail-on=broken`, counted under `broken`.
- **SSRF policy — mixed.** Host resolves to one public and one private address → blocked.
- **SSRF policy — opt-in.** `--allow-internal` flips behavior for all internal destinations and surfaces the banner on **both** stdout and stderr; `--allow-internal-host` scoped allow lets only matching destinations through and emits the `internal allowances: <count>` summary line instead of the full-run banner; when both flags are set, `--allow-internal` wins (broader wins → full-run banner, scoped rules become redundant) per §16.7.
- **SSRF policy — CIDR.** `--allow-internal-host 10.0.0.0/8` admits a destination in `10.x`, still blocks `192.168.x` and link-local.
- **SSRF policy — evaluation order.** A destination that matches `--deny-host` is `skipped:host` even if `--allow-internal-host` covers it. A destination that passes `--allow-host` and matches `--allow-internal-host` and resolves to private space is probed. A destination that passes `--allow-host` but does not match any `--allow-internal*` and resolves to private space is `broken:blocked-destination` (per §16.8: `--allow-host` does **not** bypass SSRF).
- **DNS-rebinding race.** Inject a resolver fixture whose `resolve4` returns address X and whose subsequent calls would return address Y; assert the TCP connect target **equals X** (the classifier's pick), and that no second resolution occurs at connect time. The test asserts the §16.4 pinning property directly: the seam observes exactly one DNS call and one connect to that same address.
- **SSRF DNS failure.** NXDOMAIN → `broken:network` with reason `dns:nxdomain`.
- **Refused-redirect rendering.** A public URL that redirects to a private destination renders the refused target as `<host-rendered>[:port]/<elided>` per §16.5; when the refused host is an IDN, `<host-rendered>` is the `<punycode> (<unicode>)` form per §17.5. Golden tests cover both the ASCII-host and IDN-host cases.
- **Invalid URL.** A malformed URL → `broken:invalid-url`, counted under `broken`, trips `--fail-on=broken`, never retried.
- **Per-host cap.** A burst of requests to a single host respects the default and the `--per-host-concurrency` override; dispatch to other hosts is not blocked. Boundary cases: `N=1`, `N=--concurrency`, `N=--concurrency + 1` → exit 2.
- **Per-host cap on redirect.** A→B hop releases A's per-host slot immediately and waits on B's bucket while holding the global slot. A saturated B causes the probe to wait bounded by `--timeout`; timeout in that wait classifies as `broken:network` with reason `timeout-per-host-queue`. A→A hop releases and re-acquires A's slot (no priority).
- **Per-host cap on intermediate 429/503.** A chain where an intermediate hop on host M emits 429/503 reduces **M's** cap, not A's (the originator) nor B's (the final target).
- **Global cap.** A burst spread across many hosts never exceeds `--concurrency` in-flight requests.
- **Redaction and sanitization.** URLs with userinfo, sensitive query parameters, embedded U+2028/U+2029, ANSI/bidi/zero-width characters render safely in the report and on stderr; file paths with the same classes of characters likewise.
- **Sensitive-param dedup (default).** Two URLs differing only in `?token=` value are probed **separately** in default mode (per §17.2). Both appear in the report; both count toward `links checked` unique.
- **Sensitive-param dedup (strict).** `--strict-redaction` collapses the two `?token=` variants to one probe; the on-the-wire URL is deterministically the **first occurrence in scan order** (file order per §8.1, ascending line within file). A unit test pins the selection.
- **Strict opaque-segment heuristic.** The `length ≥ 24 and ≥ 60% urlsafe-base64/hex` rule is tested against representative positives (JWT path segments, S3 bearer path components, GitHub tokens) and negatives (human-readable slugs of length 24+, path segments with `.html`/`.pdf` extensions — which are no longer a carve-out but remain negative because the alphabet composition falls below 60%).
- **IDN rendering.** A URL on `münchen.de` renders as `xn--mnchen-3ya.de (münchen.de)`; a URL the author typed as `xn--mnchen-3ya.de` renders the same way (A-label input still expanded).
- **Symlinks.** A symlink inside the scan root that points outside the root is not followed by default; `--follow-symlinks` allows following but still confines to the scan root; symlink loops do not hang the scanner.
- **Path containment.** A scan root of `/repo/docs` does **not** admit a candidate resolving to `/repo/docs-evil/file.md` (component-aware containment, not raw string prefix). Covered by a unit test that exercises exactly that pair.
- **Work budgets.** Exceeding `--max-files`, `--max-urls`, or `--max-occurrences` aborts with exit 2 and a budget-named stderr message; URLs longer than `--max-url-length` are classified `skipped:url-too-long` and counted under `skipped`; `--max-redirect-headers` cap is enforced without affecting `broken:redirect-loop` detection.
- **Include/exclude merge.** User `--include` replaces built-in includes; user `--exclude` appends; `--no-default-excludes` disables built-ins. Verified by fixture directory with files that fall on each boundary.
- **Determinism for golden tests.** A test mode (e.g., `LINKROT_DETERMINISTIC=1`) renders `duration` as a fixed placeholder and suppresses the transient progress counter so golden snapshots are stable.
- **Transport seam.** Unit tests can inject DNS NXDOMAIN, TCP RST after headers, TLS handshake failure, header-phase timeout, and IP-pinned connect through an injectable transport seam, without live network. The fail-closed behavior of §16.4 is exercised by a fixture that disables the seam — the full CLI binary must exit 3 with a one-line capability error and **no stack trace** (distinct from the §9 exit-3 unhandled-exception case).
- **End-to-end seam confinement.** A regression test runs the real CLI with a sentinel transport that refuses connection to any address other than a test-selected IP; any SSRF classification or probe that bypasses the seam (e.g., accidental `fetch` fallthrough) fails the test. This guards against a regression that subverts pinning at the entry point even if per-unit seam tests pass.
- **Fan-out golden.** A fixture with one URL referenced from two files produces the §20 worked-example output exactly.

## 22. Filesystem Traversal & Symlinks

Revised in v0.4 to tighten containment semantics and remove the self-contradictory "silently skipped with a stderr warning" phrasing.

- The scan **root** is the resolved real path of the `path` argument (or `.`). The root value retained internally is `realRoot + separator` for containment checks (e.g., `/repo/docs/`); the final separator disambiguates sibling directories whose names begin with the root's basename.
- All traversal is confined to that root. A candidate file is admitted only if its resolved real path is either equal to `realRoot` or begins with `realRoot + separator`. A raw string-prefix check against `realRoot` alone is **not** sufficient: `/repo/docs` must not admit `/repo/docs-evil/file.md`. Implementations that use `path.relative(realRoot, candidate)` MUST verify the result is neither absolute nor begins with `..` before admitting the candidate.
- Any candidate that fails the containment check is **skipped with a stderr warning** naming the candidate. (The v0.3 phrase "silently skipped with a stderr warning" is replaced; skips are always warned on stderr, never silent.)
- Symbolic links (POSIX) and reparse points (Windows) are **not** followed by default. The bare symlink itself is also not read.
- `--follow-symlinks` enables following, but root containment still applies. A symlink that resolves outside the root is skipped with a stderr warning even with `--follow-symlinks`.
- Symlink and directory cycles are detected via real-path bookkeeping; a cycle terminates the offending branch with a stderr warning, never the whole run.
- Special files (sockets, devices, FIFOs, block/char devices) are skipped with a stderr warning, regardless of glob match.

## 23. Work Budgets

Revised in v0.4 for classification clarity.

- `--max-files N` (default `10000`): scanning aborts with exit 2 once N matching files have been enumerated.
- `--max-urls N` (default `50000`): unique-URL count is capped after dedup; reaching the cap aborts extraction with exit 2 before any probe is issued.
- `--max-occurrences N` (default `200000`): total source-location count cap; same fail-fast behavior.
- `--max-url-length BYTES` (default `8192`): URLs longer than the cap are classified `skipped:url-too-long`, counted under `skipped`, and never probed. Per-URL cap, not an abort.
- `--max-redirect-headers N` (default `64`): retained per-hop header metadata is bounded; excess hops still count for the redirect limit but are not stored for the report.
- The aborting cases (`--max-files`, `--max-urls`, `--max-occurrences`) exit 2. Per §20's exit-code-2 clarification, this is substantively a usage error against the input; the stderr message always names the budget and the observed value so CI logs can distinguish bad-flag from budget exhaustion. `skipped:budget-exceeded` is reserved for URLs that were enumerated but not probed at the time of a non-aborting cap (today only `--max-url-length` populates `skipped:url-too-long`; `skipped:budget-exceeded` remains a forward-compatible class for future non-aborting budgets).

## 24. Document Status

This document is SPEC v0.4. The frozen sections (1–15) carry user-final wording from prior rounds; the v0.2/v0.3/v0.4 additions (16–24) remain subject to revision in subsequent rounds. v0.4 addresses the r03 findings on evaluation-order infeasibility, direct-internal-destination severity, sensitive-param dedup suppression risk, path-containment safety, IP-literal handling, file-failure control-plane contradictions, Retry-After cap wording, strict-redaction heuristic stability, resolver-API ambiguity, redirect-hop slot lifecycle, exit-code overloading, IDN refused-redirect rendering, and the `--allow-host` / SSRF interaction; proxy support is deferred to future work.
