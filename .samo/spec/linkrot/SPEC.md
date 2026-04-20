# tiny-linkrot — SPEC v0.3

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

## 15. Open Questions

1. Should reference-style link labels that are defined but never used be reported as warnings? (Leaning **no** for v0.1 — out of scope for link rot.)
2. Should we treat 2xx responses with suspicious bodies (e.g., soft-404s like `<title>Not Found</title>` returning 200) as broken? (Leaning **no** — false-positive risk too high without per-site heuristics.)
3. Is `User-Agent` customization enough, or do we also need `--header K:V` for niche sites that demand `Accept: text/html`? (Defer to v0.2.)

## 16. Security — SSRF & Outbound Destination Policy

Revised in v0.3 to close transport-seam, proxy, redirect-rendering, and userinfo gaps flagged in review r02.

### 16.1 Default-deny private destinations

Every probe performs its own DNS resolution (A and AAAA) and inspects the resolved addresses *before* opening a TCP connection. If any resolved address falls in a reserved range, the probe is not issued and the link is classified as `skipped:blocked-destination`. Reserved ranges:

- IPv4 loopback `127.0.0.0/8`, private `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, link-local `169.254.0.0/16` (covers cloud metadata `169.254.169.254`), CGNAT `100.64.0.0/10`, multicast `224.0.0.0/4`, broadcast `255.255.255.255`, reserved `0.0.0.0/8`, `192.0.0.0/24`, `198.18.0.0/15`, `240.0.0.0/4`.
- IPv6 loopback `::1/128`, unique-local `fc00::/7`, link-local `fe80::/10`, multicast `ff00::/8`, documentation `2001:db8::/32`, IPv4-mapped (`::ffff:0:0/96`) that map into any blocked IPv4 range.

### 16.2 Mixed-answer handling

If a host resolves to both blocked and non-blocked addresses, the host is treated as blocked. This defeats DNS-rebinding answers that mix public and private addresses.

### 16.3 Transport pinning (defeats DNS rebinding / TOCTOU)

The implementation MUST connect to a specific vetted IP address rather than re-resolving the hostname inside the runtime's `fetch`. Concretely:

- The probe layer resolves the hostname once via the OS resolver, classifies every returned address per §16.1, and selects a single non-blocked address.
- The TCP connection is opened to that exact IP. The HTTP `Host` header and TLS SNI carry the original hostname for correct virtual-hosting and certificate validation.
- TLS certificate validation MUST be performed against the original hostname; the connection-time IP is not used for certificate identity.
- Implementation seam: a custom `dispatch` / `connect` hook on the HTTP client (e.g., undici-style, or a Bun socket-level transport). Calling Bun's high-level `fetch` with the original URL is **not** sufficient on its own because Bun may re-resolve internally.
- A short-TTL DNS cache (≤ 30 s, per-run, in-process) is permitted for the *classification* step; the *connect* step always uses the address selected by classification, never a cache-bypassing re-resolution.
- If the underlying runtime cannot guarantee that `connect` uses the supplied address (no socket-level seam available), the implementation MUST fail closed: probing is disabled and the run exits 3 with a clear error. Silent fallback to runtime-default resolution is forbidden.

### 16.4 Per-hop re-check

The policy is applied again before each redirect hop. A redirect target whose host resolves to a blocked address (or mixed) produces `broken:blocked-destination-redirect` and the chain is terminated. The displayed final URL is the refused target rendered as `scheme://host[:port]/<elided>` — path, query, and fragment are replaced by the literal `<elided>` to avoid leaking attacker-controlled data. The number of public hops traversed before the refusal is shown next to the arrow.

### 16.5 SSRF-stage DNS failure

If the SSRF-stage resolution fails (NXDOMAIN, SERVFAIL, timeout, no addresses), the link is classified as `broken:network` with reason `dns:<code>` (e.g., `dns:nxdomain`). This is the same bucket §6.2 uses; SSRF block is only triggered by *successful* resolution to blocked space.

### 16.6 Opt-in escape hatches

- `--allow-internal` disables the SSRF block for the whole run. When set, the report is bracketed by a `⚠ internal destinations enabled` banner on **both** stdout and stderr, and the per-link rendering of any internal destination is also marked with `⚠`.
- `--allow-internal-host HOST_OR_CIDR` (repeatable) is the **scoped** alternative: only matching destinations bypass the block; everything else still defaults to deny. Accepted forms: bare hostname (canonicalized per §18), IPv4/IPv6 literal, or CIDR. The summary lists `internal allowances: <count>` instead of the full-run banner. A scoped allowance applies to the redirect-hop check too.
- `--allow-internal` and `--allow-internal-host` may be combined (the broader wins for the matching link). Both must be set on the CLI and have no env-var counterpart.

### 16.7 Proxy policy

- By default, `linkrot` ignores `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, and `NO_PROXY` environment variables. CI environments that inject these for other tools do not silently route linkrot through them, and the SSRF guarantees of §16.3 remain meaningful.
- `--proxy URL` enables an explicit proxy for all probes. `--use-env-proxy` opts into honoring the environment proxy variables.
- When a proxy is in use:
  - The proxy URL itself is subject to the §16.1 destination policy. A proxy resolving to private space is rejected at startup with exit 2 unless `--allow-internal` (or a matching `--allow-internal-host`) is also set.
  - IP pinning (§16.3) cannot apply to the eventual target because resolution happens on the proxy. The spec acknowledges that proxy mode reduces the SSRF guarantee: hostname-based denylisting (§18) still runs on the URL, but a malicious proxy or proxy-side rebind cannot be defeated.
  - The summary line `proxy: <url> (reduced ssrf guarantees)` is printed whenever a proxy is in use.
- The `CONNECT` method is the only proxy mode supported in v0.3. Forward HTTP proxies are not supported.

### 16.8 Interaction with `--allow-host` / `--deny-host`

Deny matches still short-circuit to `skipped:host`. Allow entries do **not** bypass the SSRF block unless `--allow-internal` (or a matching `--allow-internal-host`) is also set. Evaluation order: deny → SSRF block → scoped internal allow → allow → default.

### 16.9 Userinfo handling

`user:pass@` in URLs is never passed to the transport. The probe layer strips userinfo from the URL string *before* handing it to `fetch` / the dispatcher, so the runtime cannot synthesize an `Authorization: Basic …` header. The displayed form replaces it with `<redacted>@` (see §17). Dedup normalization in §5 also drops userinfo so credential variants collapse into one probe.

### 16.10 Classification & summary mapping (binding for v0.3)

§6.2 is frozen text from v0.1; the new v0.2/v0.3 classes are mapped here:

| Class | Summary bucket | `--fail-on=broken`? | `--fail-on=any`? | Retried? |
| --- | --- | --- | --- | --- |
| `skipped:blocked-destination` | `skipped` | no | yes | no |
| `broken:blocked-destination-redirect` | `broken` | yes | yes | no |
| `broken:invalid-url` (per §20) | `broken` | yes | yes | no |
| `skipped:url-too-long` (per §23) | `skipped` | no | yes | no |
| `skipped:budget-exceeded` (per §23) | `skipped` | no | yes | no |

`--fail-on=redirect` covers `redirect` only and does not flip on `skipped:*` or on the new `broken:*` classes (which are already covered by `--fail-on=broken`).

## 17. Output Redaction & Control-Character Sanitization

Revised in v0.3.

All URLs, file paths, and free-form messages that originate from user-controlled input pass through a render filter before they are written to stdout, stderr, or the progress line.

### 17.1 Userinfo

The `user:pass@` component is removed from URLs in the rendered form and replaced with `<redacted>@`. The same removal happens at the transport boundary (§16.9). Dedup normalization (§5) drops userinfo so credential variants collapse into one probe.

### 17.2 Sensitive query parameters (default mode)

Parameter values are replaced with `<redacted>` in the rendered form (the original value is still used for the probe) when the parameter name, case-insensitively, matches any of: `token`, `access_token`, `id_token`, `refresh_token`, `api_key`, `apikey`, `key`, `secret`, `signature`, `sig`, `password`, `auth`, `x-amz-signature`, `x-amz-credential`, `x-amz-security-token`. The list is a baseline and not authoritative for every site.

Deduplication treats two URLs that differ only in sensitive-parameter *values* as the **same** probe target (the redacted form is the dedup key). This avoids amplifying probe count when a single endpoint appears with rotating bearer values across many files. The documented trade-off is that two genuinely different resources sharing a parameter name like `key=` will collapse; operators who need per-value probes should rename the parameter or run with `--strict-redaction=off`.

### 17.3 Strict redaction mode

`--strict-redaction` (default off; recommended on for CI logs that may be archived or shared) replaces **every** query-parameter value and every URL path segment that looks like opaque bearer material (length ≥ 24, ≥ 60% urlsafe-base64 or hex characters, no path-typical extension) with `<redacted>`. In strict mode the dedup key is the post-redaction URL, so distinct opaque tokens collapse to one probe.

### 17.4 Control and format characters

Before emission, the following are replaced by their `\uXXXX` escape:

- ASCII C0 controls except `\t`, DEL (`\x7f`), C1 controls (`\x80`–`\x9f`).
- ANSI/OSC introducers `\x1b`, `\x9b`.
- Bidi overrides U+202A–U+202E, U+2066–U+2069.
- Line and paragraph separators U+2028, U+2029.
- Zero-width and BOM U+200B–U+200F, U+FEFF.

Line numbers, counts, and literal report chrome are safe and are not rewritten.

### 17.5 Hostname rendering

Hosts that contain non-ASCII labels are rendered as `<punycode> (<unicode>)`, e.g., `xn--mnchen-3ya.de (münchen.de)`. The Punycode form is what §18 matches against; showing both forms makes spoofing-via-confusables visible in the report without breaking copy-paste of the canonical form. Pure-ASCII hosts are rendered as-is.

### 17.6 Scope

The filter applies uniformly to the TTY report, the stderr progress line, warnings, error messages, and any future machine-readable output.

## 18. Host Matching Canonicalization

Revised in v0.3.

Applies to `--allow-host`, `--deny-host`, `--allow-internal-host`, and any future host-scoped flag or policy.

- Both the flag value and the URL host are converted to their IDNA A-label (Punycode) form and case-folded to lower case before comparison.
- A trailing dot on either side is stripped.
- IPv6 literals on the CLI are written **without** surrounding brackets when no port is attached (e.g. `--deny-host ::1`) and **with** brackets when a port is attached (e.g. `--deny-host [::1]:8443`). Brackets are the disambiguator that resolves the colon collision noted in review.
- IPv4 and IPv6 literals are compared after dotted-quad (IPv4) or RFC 5952 (IPv6) canonicalization.
- Match semantics by prefix (DNS names only):
  - `example.com` — matches the apex **and** all subdomains.
  - `.example.com` — matches subdomains only, not the apex.
  - `=example.com` — matches the apex only.
- The `=` and `.` prefixes are **rejected with exit 2** when applied to an IP-literal entry; they are nonsensical for IPs.
- Ports are not part of the host *identity*. A port-scoped entry uses `host:port` syntax for DNS hosts and `[ipv6]:port` for IPv6 literals, and matches only that exact port. Port-scoped entries are matching rules; per-host *accounting* (§19) remains port-agnostic.
- CIDR is supported for `--allow-internal-host` only (not for `--allow-host` / `--deny-host`).
- Glob characters are not supported in v0.3.
- Evaluation order (see also §16.8): deny → SSRF block → scoped internal allow → allow → default. If `--allow-host` is set and no allow entry matches, the link is `skipped:host`, even for public hosts.

## 19. Per-Host Concurrency & Backpressure

Revised in v0.3.

- Default per-host cap: **4** concurrent in-flight requests.
- Override via `--per-host-concurrency N` (`1 ≤ N ≤ --concurrency`).
- Host identity uses the canonical form from §18, **port-agnostic**, for the *accounting bucket*. The `host:port` syntax in §18 only affects matching of allow/deny rules, not which bucket counts a request.
- The global `--concurrency` pool remains the primary limit; per-host cap only gates dispatch.
- Backpressure on signals from a host:
  - A `429` or `503` *with* `Retry-After` reduces that host's effective cap to 1 for the remainder of the run.
  - A `429` or `503` *without* `Retry-After` likewise reduces the cap to 1 and additionally inserts a 1 s synthetic backoff before the next dispatch to that host.
  - The reduced cap is sticky for the run; it is not restored by subsequent successful responses.
  - When the existing cap is already 1, the rule is a no-op (no further reduction).
- Redirect accounting: when a probe redirects from host A to host B, the next hop's dispatch consumes a slot on B's bucket. If B is at its cap, the hop waits in B's per-host queue; the global slot held by the probe is **not** released during the wait, to preserve the global cap invariant. The wait is bounded by `--timeout`.

## 20. Clarifications

The user-frozen sections above carry some ambiguities; these clarifications are binding where they narrow language elsewhere.

- **Version labeling.** "v0.1" labels in §§2, 3, 4, 8, 11, 13, 14 describe the surface captured in this v0.3 document. They are not deferrals past v0.3.
- **Timeout scope.** `--timeout` is a **whole-request** budget covering DNS, connect, TLS, write, read, and all redirect hops. §6.1 step 4 is the authoritative reading; §4's phrase "connect + read" is a description of what the budget covers, not a per-hop timer.
- **HEAD → GET fallback.** The method switch in §6.1 step 2 is **not** a retry and does not consume a `--retries` slot. Retry policy in §6.3 then applies to the GET in the usual way.
- **Invalid URL class.** A URL string that fails `new URL(...)` parsing is classified as `broken:invalid-url`, counted under `broken` in the summary (per §16.10), honored by `--fail-on=broken`, and never retried. §12's reference to `broken:network` with reason `invalid-url` is superseded.
- **Manual redirect following.** The implementation must use `redirect: "manual"` (or an equivalent manual-follow seam) so that hop count, cycle detection, and the SSRF per-hop re-check (§16.4) are deterministic regardless of Bun's internal redirect cap.
- **Retry-After.** Parsed as both delta-seconds and HTTP-date. Wait is capped to `min(Retry-After, --timeout − elapsed)`. If the cap is ≤ 0, the link resolves on its current state with no further retry. Wait time counts against the URL's wall-clock `--timeout` budget, but the subsequent attempt still consumes one `--retries` slot.
- **Backoff (binding reading of §4).** §4 specifies `200ms × 2^n, jittered ±25%`. The binding reading is: for retry attempt `n` (1-indexed, so the first retry has `n=1`), the base wait is `200ms × 2^n` (so 400 ms at `n=1`, 800 ms at `n=2`, …), and the actual wait is sampled uniformly from `[base × 0.75, base × 1.25]` (multiplicative ±25%). The wait is truncated so total URL wall-clock does not exceed `--timeout`. (This supersedes the conflicting `2^(n-1)` / full-jitter wording that appeared in v0.2.)
- **`--include`/`--exclude` merge.** User `--include` values **replace** the built-in includes. User `--exclude` values **append** to the built-in excludes. `--no-default-excludes` disables the built-in excludes.
- **Scheme-skip accounting.** URLs filtered for non-HTTP(S) schemes (including `data:` image URLs) are excluded from both "links checked" and "skipped" in the summary and appear nowhere in the output. Reference-style image links `![alt][ref]` are handled identically to non-image reference links.
- **Image probing.** On by default in v0.3 per §5. `--no-images` disables it without affecting other behavior.
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

- **Extraction failure on one file.** A *fatal* parser error on a single file is treated as a skip (stderr warning, file excluded, other files still scanned). "Fatal" means the parser cannot produce a token stream for the file at all (I/O failure, decoder failure, or parser exception). A file where the parser recovers and emits a partial token stream — with localized warnings for the unparseable regions — counts as **scanned**, not skipped. The summary's `files scanned` counts fully- and partially-extracted files; a `files skipped` line appears when non-zero. Total URL count for progress reflects the successfully extracted set.
- **Env-var scope.** `--include`, `--exclude`, `--allow-host`, `--deny-host`, `--fail-on`, `--accept-redirects`, `--allow-internal`, `--allow-internal-host`, `--per-host-concurrency`, `--no-images`, `--proxy`, `--use-env-proxy`, `--strict-redaction`, `--follow-symlinks`, `--max-files`, `--max-urls`, `--max-occurrences`, `--max-url-length`, `--max-redirect-headers`, and `--no-default-excludes` are intentionally CLI-only. They materially affect safety, classification, or work bounds and must be explicit at invocation.

## 21. Supplemental Testing Requirements

Revised in v0.3. In addition to §13, the following test cases are required.

- **Signals.** SIGINT and SIGTERM during probing → exit 130, partial report emitted, in-flight requests aborted.
- **Redirect limits.** Chain of exactly 10 hops (pass), 11 hops (→ `broken:redirect-loop`), 3-node cycle (→ `broken:redirect-loop`).
- **Retry-After.** Delta-seconds, HTTP-date, value greater than `--timeout`, value of `0`, malformed value, and `429`/`503` *without* `Retry-After` (cap reduction + 1 s synthetic backoff per §19).
- **Method switch.** HEAD → GET on 400, 403, 405; verify the switch does not consume a retry slot.
- **Precedence.** Flag > env > default for every flag that has an env-var counterpart in §10.
- **Host canonicalization.** IDNA/Punycode, case folding, trailing dot, IPv6 literal (with and without bracketed `[ipv6]:port` form), apex-only (`=`) and subdomain-only (`.`) prefix forms, rejection of `=`/`.` on IP-literal entries.
- **SSRF policy.** Host resolves to loopback / RFC1918 / link-local / metadata / mixed public+private; public → private redirect (verify `broken:blocked-destination-redirect` is counted under `broken` and trips `--fail-on=broken`); `--allow-internal` opt-in flips behavior and surfaces the banner on **both** stdout and stderr; `--allow-internal-host` scoped allow lets only matching destinations through and emits the `internal allowances: <count>` summary line instead of the full-run banner.
- **SSRF + `--allow-host`.** A host explicitly on `--allow-host` that resolves to a private address is still blocked unless `--allow-internal` / `--allow-internal-host` covers it.
- **DNS-rebinding race.** Inject a transport that returns one address to the SSRF-stage classifier and a different address at connect time; verify the connect uses the classifier-selected IP (per §16.3) and the rebinding answer is never reached.
- **SSRF DNS failure.** NXDOMAIN at the SSRF-stage classification → `broken:network` with reason `dns:nxdomain`.
- **Refused-redirect rendering.** A public URL that redirects to a private destination renders the refused target as `scheme://host[:port]/<elided>` per §16.4; golden test covers the rendering verbatim.
- **Invalid URL.** A malformed URL → `broken:invalid-url`, counted under `broken`, trips `--fail-on=broken`, never retried.
- **Per-host cap.** A burst of requests to a single host respects the default and the `--per-host-concurrency` override; dispatch to other hosts is not blocked.
- **Per-host cap on redirect.** A redirect from host A to host B counts against B's bucket; a saturated B causes A's probe to wait without releasing its global slot (bounded by `--timeout`).
- **Global cap.** A burst spread across many hosts never exceeds `--concurrency` in-flight requests.
- **Redaction and sanitization.** URLs with userinfo, sensitive query parameters, embedded U+2028 / U+2029, ANSI / bidi / zero-width characters render safely in the report and on stderr; file paths with the same classes of characters likewise. `--strict-redaction` collapses opaque-token variants in dedup.
- **Sensitive-param dedup.** Two URLs differing only in `?token=` value collapse to one probe in default mode (per §17.2); `--strict-redaction=off` keeps them distinct.
- **IDN rendering.** A URL on `münchen.de` renders as `xn--mnchen-3ya.de (münchen.de)` in the report.
- **Proxy.** Env proxy is ignored unless `--use-env-proxy` is set; `--proxy` to a private destination is rejected at startup unless `--allow-internal` also covers it; the `proxy: <url> (reduced ssrf guarantees)` summary line is present whenever a proxy is in use.
- **Symlinks.** A symlink inside the scan root that points outside the root is not followed by default; `--follow-symlinks` allows following but still confines to the scan root; symlink loops do not hang the scanner.
- **Work budgets.** Exceeding `--max-files`, `--max-urls`, or `--max-occurrences` aborts with exit 2 and a budget-named stderr message; URLs longer than `--max-url-length` are classified `skipped:url-too-long` and counted under `skipped`.
- **Determinism for golden tests.** A test mode (e.g., `LINKROT_DETERMINISTIC=1`) renders `duration` as a fixed placeholder and suppresses the transient progress counter so golden snapshots are stable.
- **Transport seam.** Unit tests can inject DNS NXDOMAIN, TCP RST after headers, TLS handshake failure, header-phase timeout, and IP-pinned connect through an injectable transport seam, without live network. The fail-closed behavior of §16.3 is exercised by a fixture that disables the seam.
- **Fan-out golden.** A fixture with one URL referenced from two files produces the §20 worked-example output exactly.

## 22. Filesystem Traversal & Symlinks

New in v0.3. Closes the untrusted-repo traversal risk flagged in review r02.

- The scan **root** is the resolved real path of the `path` argument (or `.`). All traversal is confined to that root: any candidate file whose resolved real path does not have the root as a prefix is silently skipped with a stderr warning.
- Symbolic links (POSIX) and reparse points (Windows) are **not** followed by default. The bare symlink itself is also not read.
- `--follow-symlinks` enables following, but root confinement still applies. A symlink that resolves outside the root is skipped with a stderr warning even with `--follow-symlinks`.
- Symlink and directory cycles are detected via real-path bookkeeping; a cycle terminates the offending branch with a stderr warning, never the whole run.
- Special files (sockets, devices, FIFOs, block/char devices) are skipped with a stderr warning, regardless of glob match.

## 23. Work Budgets

New in v0.3. The pre-probe extraction phase can otherwise grow without bound on a malicious docs change.

- `--max-files N` (default `10000`): scanning aborts with exit 2 once N matching files have been enumerated.
- `--max-urls N` (default `50000`): unique-URL count is capped after dedup; reaching the cap aborts extraction with exit 2 before any probe is issued.
- `--max-occurrences N` (default `200000`): total source-location count cap; same fail-fast behavior.
- `--max-url-length BYTES` (default `8192`): URLs longer than the cap are classified `skipped:url-too-long` and never probed (counted under `skipped`).
- `--max-redirect-headers N` (default `64`): retained per-hop header metadata is bounded; excess hops still count for the redirect limit but are not stored for the report.
- The fail-fast `exit 2` cases use the `skipped:budget-exceeded` class internally for any URLs not yet probed at the time of the abort, so the partial summary remains coherent. The stderr message names the budget and the observed value so CI logs explain the abort.

## 24. Document Status

This document is SPEC v0.3. The frozen sections (1–15) carry user-final wording from prior rounds; the v0.2/v0.3 additions (16–24) remain subject to revision in subsequent rounds.

<!-- samospec:lead-directive -->
The user has manually edited sections 1. Overview, 1. Purpose, 10. Configuration Precedence, 10. Security and Privacy Considerations, 11. Non-Goals (v0.1), 11. Open Questions, 12. Acceptance Criteria, 12. Error Handling & Robustness, 13. Testing Strategy, 14. Future Work (post-v0.1), 15. Open Questions, 2. Goals and Non-Goals, 2. Persona & Scope, 3. Runtime & Distribution, 3. User Stories, 4. CLI Surface, 5. Input Handling, 5. Link Extraction, 6. HTTP Probing, 6. Network Probing, 7. Concurrency Model, 7. Output, 8. Implementation Notes, 8. Output, 9. Exit Codes, 9. Testing of the spec since the last round. Treat their exact wording as final for those sections; do not rewrite them.
<!-- samospec:lead-directive end -->
