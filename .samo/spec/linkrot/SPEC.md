# tiny-linkrot — SPEC v0.2

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

New in v0.2. Added to harden default behavior when `linkrot` runs in CI against untrusted Markdown.

- **Default-deny private destinations.** Every probe performs its own DNS resolution (A and AAAA) and inspects the resolved addresses *before* opening a TCP connection. If any resolved address falls in a reserved range, the probe is not issued and the link is classified as `skipped:blocked-destination`. Reserved ranges:
  - IPv4 loopback `127.0.0.0/8`, private `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, link-local `169.254.0.0/16` (covers cloud metadata `169.254.169.254`), CGNAT `100.64.0.0/10`, multicast `224.0.0.0/4`, broadcast `255.255.255.255`, reserved `0.0.0.0/8`, `192.0.0.0/24`, `198.18.0.0/15`, `240.0.0.0/4`.
  - IPv6 loopback `::1/128`, unique-local `fc00::/7`, link-local `fe80::/10`, multicast `ff00::/8`, documentation `2001:db8::/32`, IPv4-mapped (`::ffff:0:0/96`) that map into any blocked IPv4 range.
- **Mixed-answer handling.** If a host resolves to both blocked and non-blocked addresses, the host is treated as blocked. This defeats DNS-rebinding bypasses.
- **Per-hop re-check.** The policy is applied again before each redirect hop. A public URL that 3xx-redirects to a blocked destination produces `broken:blocked-destination-redirect`; the final URL is shown truncated at the hop that was refused.
- **Opt-in `--allow-internal`.** Disables the SSRF block for the whole run. Must be set explicitly on the CLI; there is no env-var counterpart. When set, the report prefixes and suffixes output with a `⚠ internal destinations enabled` banner on both stdout and stderr.
- **Interaction with `--allow-host` / `--deny-host`.** Deny matches still short-circuit to `skipped:host`. Allow entries do **not** bypass the SSRF block unless `--allow-internal` is also set. Evaluation order: deny → SSRF block → allow → default.
- **Userinfo handling.** `user:pass@` in URLs is never sent in `Authorization` headers; it is stripped from the request and from the displayed form (see §17).

## 17. Output Redaction & Control-Character Sanitization

New in v0.2. All URLs, file paths, and free-form messages that originate from user-controlled input pass through a render filter before they are written to stdout, stderr, or the progress line.

- **Userinfo.** The `user:pass@` component is removed from URLs in the rendered form and replaced with `<redacted>@`. Dedup normalization in §5 also drops userinfo so credential variants collapse into one probe.
- **Sensitive query parameters.** Parameter values are replaced with `<redacted>` in the rendered form (the original value is still used for the probe) when the parameter name, case-insensitively, matches any of: `token`, `access_token`, `id_token`, `refresh_token`, `api_key`, `apikey`, `key`, `secret`, `signature`, `sig`, `password`, `auth`, `x-amz-signature`, `x-amz-credential`, `x-amz-security-token`.
- **Control and format characters.** Before emission, the following are replaced by their `\uXXXX` escape: ASCII C0 controls except `\t`, DEL (`\x7f`), C1 controls (`\x80`–`\x9f`), ANSI/OSC introducers (`\x1b`, `\x9b`), bidi overrides (U+202A–U+202E, U+2066–U+2069), zero-width and BOM (U+200B–U+200F, U+FEFF). Line numbers, counts, and literal report chrome are safe and are not rewritten.
- **Scope.** The filter applies uniformly to the TTY report, the stderr progress line, warnings, error messages, and any future machine-readable output.

## 18. Host Matching Canonicalization

New in v0.2. Applies to `--allow-host`, `--deny-host`, and any future host-scoped flag or policy.

- Both the flag value and the URL host are converted to their IDNA A-label (Punycode) form and case-folded to lower case before comparison.
- A trailing dot on either side is stripped.
- IPv6 literals are provided on the CLI without surrounding brackets (e.g. `--deny-host ::1`) and are compared after RFC 5952 canonicalization.
- Match semantics by prefix:
  - `example.com` — matches the apex **and** all subdomains.
  - `.example.com` — matches subdomains only, not the apex.
  - `=example.com` — matches the apex only.
- Ports are not part of the host value. A port-scoped entry uses `host:port` syntax and matches only that exact port.
- Glob characters are not supported in v0.2.
- Evaluation order (see also §16): deny → SSRF block → allow → default. If `--allow-host` is set and no allow entry matches, the link is `skipped:host`, even for public hosts.

## 19. Per-Host Concurrency (Baseline)

New in v0.2. Adds a conservative per-host cap so a single domain cannot monopolize the worker pool.

- Default per-host cap: **4** concurrent in-flight requests.
- Override via `--per-host-concurrency N` (`1 ≤ N ≤ --concurrency`).
- Host identity uses the canonical form from §18, port-agnostic.
- The global `--concurrency` pool remains the primary limit; per-host cap only gates dispatch.
- A `429` or `503` with `Retry-After` from a host reduces that host's effective cap to 1 for the remainder of the run.

## 20. Clarifications

New in v0.2. The user-frozen sections above carry some ambiguities flagged in the last review round. These clarifications are binding where they narrow language elsewhere; they do not replace frozen text.

- **Version labeling.** "v0.1" labels in §§2, 3, 4, 8, 11, 13, 14 describe the surface captured in this v0.2 document. They are not deferrals past v0.2.
- **Timeout scope.** `--timeout` is a **whole-request** budget covering DNS, connect, TLS, write, read, and all redirect hops. §6.1 step 4 is the authoritative reading; §4's phrase "connect + read" is a description of what the budget covers, not a per-hop timer.
- **HEAD → GET fallback.** The method switch in §6.1 step 2 is **not** a retry and does not consume a `--retries` slot. Retry policy in §6.3 then applies to the GET in the usual way.
- **Invalid URL class.** A URL string that fails `new URL(...)` parsing is classified as `broken:invalid-url`, counted under `broken` in the summary, honored by `--fail-on=broken`, and never retried. §12's reference to `broken:network` with reason `invalid-url` is superseded.
- **Manual redirect following.** The implementation must use `redirect: "manual"` (or an equivalent manual-follow seam) so that hop count, cycle detection, and the SSRF per-hop re-check (§16) are deterministic regardless of Bun's internal redirect cap.
- **Retry-After.** Parsed as both delta-seconds and HTTP-date. Wait is capped to `min(Retry-After, --timeout − elapsed)`. If the cap is ≤ 0, the link resolves on its current state with no further retry. Wait time counts against the URL's wall-clock `--timeout` budget, but the subsequent attempt still consumes one `--retries` slot.
- **Backoff.** Retry n (1-indexed) waits `200ms × 2^(n-1)` with full jitter applied uniformly in `[0, base]`. The wait is truncated so total URL wall-clock does not exceed `--timeout`.
- **`--include`/`--exclude` merge.** User `--include` values **replace** the built-in includes. User `--exclude` values **append** to the built-in excludes. `--no-default-excludes` disables the built-in excludes.
- **Scheme-skip accounting.** URLs filtered for non-HTTP(S) schemes (including `data:` image URLs) are excluded from both "links checked" and "skipped" in the summary and appear nowhere in the output. Reference-style image links `![alt][ref]` are handled identically to non-image reference links.
- **Image probing.** On by default in v0.2 per §5. `--no-images` disables it without affecting other behavior.
- **Fan-out rendering.** A URL present in multiple files is probed once. In the report it appears once under **each** file where it occurs; each occurrence shows that file's own line numbers. In the summary, `links checked: N unique (M occurrences)` reports N as unique URLs probed and M as the total source-location count; `ok`, `redirects`, `broken`, `skipped` count unique URLs.
- **Extraction failure on one file.** A fatal parser error on a single file is treated as a skip (stderr warning, file excluded, other files still scanned). The summary's `files scanned` counts only fully-extracted files, and a `files skipped` line appears when non-zero. Total URL count for progress reflects the successfully extracted set.
- **Env-var scope.** `--include`, `--exclude`, `--allow-host`, `--deny-host`, `--fail-on`, `--accept-redirects`, `--allow-internal`, `--per-host-concurrency`, and `--no-images` are intentionally CLI-only. They materially affect safety and classification and must be explicit at invocation.

## 21. Supplemental Testing Requirements

New in v0.2. In addition to §13, the following test cases are required.

- **Signals.** SIGINT and SIGTERM during probing → exit 130, partial report emitted, in-flight requests aborted.
- **Redirect limits.** Chain of exactly 10 hops (pass), 11 hops (→ `broken:redirect-loop`), 3-node cycle (→ `broken:redirect-loop`).
- **Retry-After.** Delta-seconds, HTTP-date, value greater than `--timeout`, value of `0`, malformed value.
- **Method switch.** HEAD → GET on 400, 403, 405; verify that the switch does not consume a retry slot.
- **Precedence.** Flag > env > default for every flag that has an env-var counterpart in §10.
- **Host canonicalization.** IDNA/Punycode, case folding, trailing dot, IPv6 literal, apex-only (`=`) and subdomain-only (`.`) prefix forms.
- **SSRF policy.** Host resolves to loopback / RFC1918 / link-local / metadata / mixed public+private; public → private redirect; `--allow-internal` opt-in flips behavior and surfaces the banner.
- **Per-host cap.** A burst of requests to a single host respects the default and the `--per-host-concurrency` override; dispatch to other hosts is not blocked.
- **Global cap.** A burst spread across many hosts never exceeds `--concurrency` in-flight requests.
- **Redaction and sanitization.** URLs with userinfo, sensitive query parameters, and embedded control / ANSI / bidi / zero-width characters render safely in the report and on stderr; file paths with the same classes of characters likewise.
- **Determinism for golden tests.** A test mode (e.g., `LINKROT_DETERMINISTIC=1`) must render `duration` as a fixed placeholder and suppress the transient progress counter so golden snapshots are stable.
- **Transport seam.** Unit tests can inject DNS NXDOMAIN, TCP RST after headers, TLS handshake failure, and header-phase timeout through an injectable fetch/transport seam, without live network.

<!-- samospec:lead-directive -->
The user has manually edited sections 1. Overview, 1. Purpose, 10. Configuration Precedence, 10. Security and Privacy Considerations, 11. Non-Goals (v0.1), 11. Open Questions, 12. Acceptance Criteria, 12. Error Handling & Robustness, 13. Testing Strategy, 14. Future Work (post-v0.1), 15. Open Questions, 2. Goals and Non-Goals, 2. Persona & Scope, 3. Runtime & Distribution, 3. User Stories, 4. CLI Surface, 5. Input Handling, 5. Link Extraction, 6. HTTP Probing, 6. Network Probing, 7. Concurrency Model, 7. Output, 8. Implementation Notes, 8. Output, 9. Exit Codes, 9. Testing of the spec since the last round. Treat their exact wording as final for those sections; do not rewrite them.
<!-- samospec:lead-directive end -->
