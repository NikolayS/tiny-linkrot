# tiny-linkrot — SPEC v0.5

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

## 4a. Additional Options (v0.2–v0.5)

§4 is user-frozen wording from v0.1 and does not enumerate the safety-, classification-, and budget-control flags introduced in v0.2 through v0.5. The authoritative CLI surface for v0.5 is the union of §4 and this section. `linkrot --help` MUST emit both tables; conflict between §4 and §4a is resolved in favor of §4 (no v0.2+ change has weakened or relabeled a v0.1 flag).

| Flag | Default | Introduced | Meaning |
| --- | --- | --- | --- |
| `--allow-internal` | off | v0.2 | Disable the SSRF block for the whole run (§16.7). Banner on stdout **and** stderr. |
| `--allow-internal-host CIDR_OR_IP` (repeatable) | *(none)* | v0.2, narrowed in v0.5 | Scoped SSRF bypass. **IP literal or CIDR only** in v0.5; bare hostnames are rejected with exit 2 (see §16.7, §18). |
| `--per-host-concurrency N` | `4` | v0.3 | Per-host in-flight cap (§19). Valid range `[1, --concurrency]`; out-of-range → exit 2. |
| `--no-images` | off (i.e., images probed) | v0.3 | Skip image links (§5, §20). |
| `--strict-redaction` | off | v0.3, scope cut in v0.5 | **Render-only** in v0.5. Aggressively redacts query-parameter values and bearer-shaped path segments in output (§17.3). Does not change dedup or what is sent on the wire. |
| `--follow-symlinks` | off | v0.3 | Follow symlinks within the scan root (§22). |
| `--max-files N` | `10000` | v0.3 | Abort with exit 2 once N matching files have been enumerated (§23). |
| `--max-urls N` | `50000` | v0.3 | Abort with exit 2 once N unique URLs have been enumerated (§23). |
| `--max-occurrences N` | `200000` | v0.3 | Abort with exit 2 once N source occurrences have been enumerated (§23). |
| `--max-url-length BYTES` | `8192` | v0.3 | Per-URL cap; longer URLs → `skipped:url-too-long` (§23). |
| `--max-redirect-headers N` | `64` | v0.3, clarified in v0.5 | Per-hop count cap on retained response-header lines for the report. Excess lines are dropped from the report only; redirect-loop detection is unaffected (§23). |
| `--max-response-header-bytes BYTES` | `65536` | v0.5 | Per-hop hard cap on total response-header bytes received. Exceeding → `broken:network` reason `header-bytes-exceeded`; the connection is torn down (§16.12). |
| `--max-location-length BYTES` | `4096` | v0.5 | Per-hop cap on the byte length of `Location:`. Exceeding → `broken:network` reason `location-too-long`; chain terminates (§16.12). |
| `--no-default-excludes` | off | v0.3 | Disable the built-in excludes from §4 (§20). |

## 5. Link Extraction

- Markdown is parsed with a real parser (v0.1: `marked` or equivalent) — **not** regex — so that links inside code fences and inline code spans are ignored.
- Extracted link kinds:
  - `[text](url)` inline links
  - `[text][ref]` + `[ref]: url` reference links
  - Bare autolinks `<https://example.com>`
  - Image links `![alt](url)` are also checked.
- Only `http:` and `https:` URLs are probed. `mailto:`, `tel:`, relative links, and fragment-only (`#foo`) links are ignored silently.
- URLs are normalized before deduplication: lowercase scheme+host, default-port stripping, empty path → `/`, percent-encoding normalized **per RFC 3986 §6.2.2.2 (uppercase hex digits in percent-encoded triplets, percent-decode unreserved characters per §2.3)**, fragment stripped. Query strings are preserved as-is.
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
- Lossy redaction-aware dedup. v0.3/v0.4 had a mode that collapsed URLs differing only in a redacted parameter to a single probe; review r04 flagged it as suppressing real findings, and v0.5 removed it. A future spec round may reintroduce it as an explicit accuracy-reducing flag (working name `--collapse-redacted-dedup`) once the trade-off is acceptable to operators.
- Hostname-scoped internal allowances. v0.4 accepted bare hostnames in `--allow-internal-host`; review r04 flagged the DNS-drift / rebinding risk and v0.5 narrowed it to IP/CIDR only. Hostname allowances may return alongside an explicit private-CIDR constraint set.

## 15. Open Questions

1. Should reference-style link labels that are defined but never used be reported as warnings? (Leaning **no** for v0.1 — out of scope for link rot.)
2. Should we treat 2xx responses with suspicious bodies (e.g., soft-404s like `<title>Not Found</title>` returning 200) as broken? (Leaning **no** — false-positive risk too high without per-site heuristics.)
3. Is `User-Agent` customization enough, or do we also need `--header K:V` for niche sites that demand `Accept: text/html`? (Defer to v0.2.)

## 16. Security — SSRF & Outbound Destination Policy

Revised in v0.5 to (a) re-apply the full host policy on every redirect hop, not just the SSRF check; (b) narrow `--allow-internal-host` to IP/CIDR only; (c) tighten transport pinning to fresh-socket-per-hop with no cross-host reuse or HTTP/2 origin coalescing; (d) name the Bun socket-level seam; (e) specify post-connect (happy-eyeballs) fallback; and (f) add explicit response-byte caps (§16.12).

### 16.1 Default-deny private destinations

Every probe performs its own DNS resolution (A and AAAA) and inspects the resolved addresses *before* opening a TCP connection. If any resolved address falls in a reserved range, the probe is not issued and the link is classified as `broken:blocked-destination` (see §16.10 for the change from v0.3's `skipped:*` bucket). Reserved ranges:

- IPv4 loopback `127.0.0.0/8`, private `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, link-local `169.254.0.0/16` (covers cloud metadata `169.254.169.254`), CGNAT `100.64.0.0/10`, multicast `224.0.0.0/4`, broadcast `255.255.255.255`, reserved `0.0.0.0/8`, `192.0.0.0/24`, `198.18.0.0/15`, `240.0.0.0/4`.
- IPv6 loopback `::1/128`, unique-local `fc00::/7`, link-local `fe80::/10`, multicast `ff00::/8`, documentation `2001:db8::/32`, IPv4-mapped (`::ffff:0:0/96`) that map into any blocked IPv4 range.

### 16.2 Mixed-answer handling

If a host resolves to both blocked and non-blocked addresses, the host is treated as blocked. This defeats DNS-rebinding answers that mix public and private addresses.

### 16.3 IP-literal URLs

If the URL host is an IP literal (dotted-quad IPv4, bracketed IPv6, or an `::ffff:a.b.c.d` mapped literal), **no DNS resolution is performed**. The literal is classified directly per §16.1 and, if not blocked, is pinned to itself for connect. Behavioral notes:

- IPv6 zone-identifier suffixes (`fe80::1%eth0`) are rejected as `broken:invalid-url`. Because WHATWG `new URL()` percent-encodes the `%` rather than rejecting the input, the implementation MUST run a pre-parse predicate that rejects any URL whose host component contains a literal `%` character (or, equivalently, post-parse: rejects any host whose decoded form contains `%`). The policy intentionally does not probe scoped addresses.
- Non-canonical IPv4 literal forms that are legal per browsers but ambiguous in tooling (octal `0177.0.0.1`, decimal `2130706433`, hex `0x7f000001`, short `127.1`) are normalized to their canonical dotted-quad before classification. An implementation that cannot normalize one of these forms MUST refuse to probe the URL as `broken:invalid-url` rather than passing it through.
- IPv4-mapped IPv6 literals are classified both against the IPv6 ranges in §16.1 and, after unmapping, against the IPv4 ranges.

### 16.4 Transport pinning (defeats DNS rebinding / TOCTOU)

The implementation MUST connect to a specific vetted IP address rather than re-resolving the hostname inside the runtime's `fetch`. Concretely:

- The probe layer resolves the hostname once using Node's `node:dns/promises` `resolve4` and `resolve6` (Bun re-exports `node:dns/promises`; equivalently, c-ares-style direct DNS queries). It MUST NOT use `dns.lookup` / `getaddrinfo`, because `/etc/hosts` and `nsswitch.conf` can introduce platform-dependent results and hide rebinding vectors. The classifier inspects the full union of A and AAAA answers atomically.
- Every returned address is classified per §16.1. Address-selection order: IPv6 preferred when both families are present and non-blocked, falling back to IPv4 (`--prefer-ipv4` may be added later).
- The TCP connection is opened to that exact IP via Bun's socket-level API (`Bun.connect` for plaintext, `Bun.connect({ tls: { serverName: <originalHost> } })` for HTTPS). The HTTP request is then framed on top of the resulting socket; the runtime's high-level `fetch` is **not** sufficient because Bun may re-resolve internally.
- TLS SNI and certificate validation MUST use the original hostname; the connection-time IP is not used for certificate identity.
- **Capability requirement.** Bun ≥ 1.1 exposes `Bun.connect` with TLS support; this is the named runtime dependency for v0.5. If `Bun.connect` is unavailable or cannot be configured to connect to a supplied IP while overriding SNI, the implementation MUST fail closed: probing is disabled and the run exits 3 with a single-line capability error and **no stack trace** (see §20). Silent fallback to runtime-default resolution is forbidden.
- **Fresh socket per hop.** Each redirect hop opens a new TCP/TLS connection. The implementation MUST disable connection pooling, keep-alive reuse across hops, and HTTP/2 origin coalescing for probe traffic. Rationale: a pooled connection or coalesced HTTP/2 origin can carry a later request to a host whose IP was never re-classified, defeating §16.4. A v0.5 implementation that uses HTTP/1.1 and closes the socket after each response trivially satisfies this; an HTTP/2-capable implementation MUST set ALPN to advertise `http/1.1` only for probe traffic, or assert that the negotiated session is restricted to the exact `(host, ip)` pair selected for the current hop.
- **Connection-pool isolation.** Even within one probe, no socket is reused for a different `(host, ip)` pair. Cross-probe socket reuse is forbidden.
- **Classification cache.** A short-TTL DNS cache (≤ 30 s, per-run, in-process) is permitted, but it MUST cache the **classification decision** (`{ chosenIP, allowed | blocked-with-reason }`) and not the raw address list. This means a host that classified as blocked stays blocked for the cache window even if a later resolution would return a different mix; rebinding cannot get a second shot inside the window. Negative classifications expire on the same TTL. The connect step always uses the cached `chosenIP`, never a cache-bypassing re-resolution.
- **Happy-eyeballs / post-connect fallback.** If the chosen IPv6 address fails the TCP/TLS handshake (timeout, RST, TLS error), the probe MAY fall back to **another address from the same `(resolve4, resolve6)` answer set that was classified non-blocked in the same call**. It MUST NOT re-resolve the hostname for fallback. If no other classified-non-blocked address is available, the probe is `broken:network`. This fallback consumes wall-clock against the probe's `--timeout` but does not consume `--retries` slots (it is part of establishing the first attempt); subsequent retries (§6.3) use the same address-selection rules against the cached classification.

### 16.5 Per-hop re-check and full host-policy re-application

The full host policy is re-evaluated before each redirect hop, not only the SSRF check. Specifically, for every hop the implementation runs the §16.8 evaluation order against the redirect target's host:

1. `--deny-host` match → chain terminates as `broken:blocked-redirect-host` (see §16.10).
2. `--allow-host` gate (when set and unmatched) → chain terminates as `broken:blocked-redirect-host`.
3. SSRF check (§16.1, §16.2, with §16.7 allowances applied) → blocked → `broken:blocked-destination-redirect`.
4. Otherwise the hop proceeds, opening a fresh socket per §16.4.

This means an attacker-controlled redirect cannot bounce the checker onto a host the operator excluded with `--deny-host`, nor onto a public host outside an `--allow-host` gate.

**Refused-redirect rendering.** The displayed final URL is the refused target rendered as `<host-rendered>[:port]/<elided>` where `<host-rendered>` is produced by §17.5 (so IDN hosts show `<punycode> (<unicode>)`). Path, query, and fragment are replaced by the literal `<elided>` to avoid leaking attacker-controlled data. The number of public hops traversed before the refusal is shown in parentheses after the arrow. **Literal format:**

```
  ✗ https://good.example/start          → xn--mnchen-3ya.de (münchen.de)/<elided>  (refused after 2 hops)
    line 7
```

The trailing parenthetical is exactly `(refused after N hops)` where N counts hops successfully traversed before the refusal (so N=0 if the refusal happens on the very first redirect target). Two spaces precede the parenthetical.

### 16.6 SSRF-stage DNS failure

If the SSRF-stage resolution fails (NXDOMAIN, SERVFAIL, timeout, no addresses), the link is classified as `broken:network` with reason `dns:<code>` (e.g., `dns:nxdomain`). SSRF block is only triggered by *successful* resolution to blocked space.

### 16.7 Opt-in escape hatches

- `--allow-internal` disables the SSRF block for the whole run. When set, the report is bracketed by a `⚠ internal destinations enabled` banner on **both** stdout and stderr, and the per-link rendering of any internal destination is also marked with `⚠`.
- `--allow-internal-host CIDR_OR_IP` (repeatable) is the **scoped** alternative: only matching destinations bypass the SSRF block; everything else still defaults to deny. **In v0.5 this flag accepts only IP literals (IPv4 or IPv6) and CIDR ranges.** Bare hostnames are rejected at startup with exit 2 and a stderr message naming the offending value. Rationale (review r04): hostname allowances bless whatever addresses the name resolves to now or later, turning a scoped exception into a moving target under DNS drift or rebinding. The summary lists `internal allowances: <count>` instead of the full-run banner; this line is written to **stdout** (as part of the §8.2 summary), not mirrored to stderr.
- `--allow-internal` and `--allow-internal-host` may be combined. Precedence when both are set: `--allow-internal` fully supersedes scoped allowances (every internal destination is allowed; the full-run banner is shown). When only `--allow-internal-host` is set, only matching destinations are allowed.
- Both flags are CLI-only and have no env-var counterpart (§20).
- Scoped allowances apply to the redirect-hop check too.

### 16.8 Interaction with `--allow-host` / `--deny-host` (and the redirect-hop binding)

The evaluation order below is applied to **the initial URL and to every redirect-target host**, per §16.5.

**Evaluation order, per host (initial or redirect target):**

1. **Deny match (`--deny-host`)** → `skipped:host` on the initial URL; `broken:blocked-redirect-host` on a redirect target. Terminal.
2. **Allowlist gate (`--allow-host`)**. If `--allow-host` is set and the host does not match → `skipped:host` (initial) / `broken:blocked-redirect-host` (redirect). Terminal.
3. **Scoped internal allow (`--allow-internal-host` or `--allow-internal`)**. If the destination would otherwise be blocked by §16.1 and a matching allowance covers it, the SSRF block is bypassed for this hop. Continue.
4. **SSRF block (§16.1 + §16.2)**. Blocked destinations not covered by step 3 → `broken:blocked-destination` (initial) or `broken:blocked-destination-redirect` (redirect). Terminal.
5. **Probe (open a fresh socket per §16.4).**

`--allow-host` membership does **not** bypass the SSRF block on its own; only an `--allow-internal*` flag can do that.

### 16.9 Userinfo handling

`user:pass@` in URLs is never passed to the transport. The probe layer strips userinfo from the URL string *before* handing it to the dispatcher, so the runtime cannot synthesize an `Authorization: Basic …` header. The displayed form replaces it with `<redacted>@` (see §17). Dedup normalization in §5 also drops userinfo so credential variants collapse into one probe.

### 16.10 Classification & summary mapping (binding for v0.5)

§6.2 is frozen text from v0.1; the v0.2–v0.5 additions are mapped here.

| Class | Summary bucket | `--fail-on=broken`? | `--fail-on=any`? | Retried? |
| --- | --- | --- | --- | --- |
| `broken:blocked-destination` | `broken` | yes | yes | no |
| `broken:blocked-destination-redirect` | `broken` | yes | yes | no |
| `broken:blocked-redirect-host` (new in v0.5) | `broken` | yes | yes | no |
| `broken:invalid-url` (per §20) | `broken` | yes | yes | no |
| `skipped:url-too-long` (per §23) | `skipped` | no | yes | no |
| `skipped:budget-exceeded` (per §23) | `skipped` | no | yes | no |

Notes:

- `broken:blocked-redirect-host` is new in v0.5 and covers a redirect target rejected by the host-policy re-check (§16.5/§16.8 steps 1–2). It is distinct from `broken:blocked-destination-redirect` (SSRF) so operators can tell apart a deny-listed bounce from an internal-network bounce.
- `skipped:budget-exceeded` only appears on per-URL non-aborting budgets. Aborting budgets exit 2 before probing; their rows are not in this table because exit 2 always wins over `--fail-on` (see §20).
- `--fail-on=redirect` covers `redirect` only; it does not flip on `skipped:*` or on `broken:*` classes (which are already covered by `--fail-on=broken`).

### 16.11 Proxy support

Deferred out of v0.5 (in §14 future work). By default, `linkrot` **ignores** `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, and `NO_PROXY` environment variables — CI environments that inject these for other tools do not silently route linkrot through them.

### 16.12 Response-byte and rendered-field caps

A hostile endpoint can flood logs and CI artifacts within a normal `--timeout`. v0.5 adds explicit caps:

- **Total response-header bytes per hop.** Bounded by `--max-response-header-bytes` (default 65536, see §4a). The probe layer reads bytes off the socket through a counting buffer; once the cap is reached before headers are fully parsed, the probe is classified `broken:network` reason `header-bytes-exceeded`, the socket is closed, and no retry slot is consumed.
- **`Location:` header byte length.** Bounded by `--max-location-length` (default 4096). Exceeding → `broken:network` reason `location-too-long`; the chain terminates and the displayed final URL renders the over-long location truncated to `<first 64 bytes>…<elided, N bytes>` where N is the original byte length. Path/query/fragment of the truncated portion are not rendered.
- **Status reason-phrase.** Truncated to 200 bytes for rendering; the wire value is unaffected for classification. Excess is rendered as `…`.
- **Rendered URL length.** Any URL rendered in the report (initial, final, refused) is capped at 512 bytes; excess is rendered as the first 256 bytes, the literal `…`, and the last 64 bytes. The wire URL is unaffected.
- **Network-error message.** Server- or runtime-supplied error text is truncated to 256 bytes and passed through §17.4 sanitization.
- All caps are pre-sanitization; the caps bound bytes received or rendered, and §17.4 then escapes control characters in the (already bounded) rendered text.

These caps interact with §16.4: because each hop opens a fresh socket, the per-hop cap straightforwardly bounds memory growth across a redirect chain at `(hops × max-response-header-bytes)` worst case.

## 17. Output Redaction & Control-Character Sanitization

Revised in v0.5. **`--strict-redaction` is now render-only** (review r04 — the v0.4 lossy-dedup behavior could suppress real findings behind a healthy sibling).

All URLs, file paths, and free-form messages that originate from user-controlled input pass through a render filter before they are written to stdout, stderr, or the progress line.

### 17.1 Userinfo

The `user:pass@` component is removed from URLs in the rendered form and replaced with `<redacted>@`. The same removal happens at the transport boundary (§16.9). Dedup normalization (§5) drops userinfo so credential variants collapse into one probe.

### 17.2 Sensitive query parameters (default mode)

Parameter values are replaced with `<redacted>` **in the rendered form only** (the original value is still used for the probe; dedup is unaffected) when the parameter name matches, **ASCII-case-insensitively**, any of: `token`, `access_token`, `id_token`, `refresh_token`, `api_key`, `apikey`, `key`, `secret`, `signature`, `sig`, `password`, `auth`, `x-amz-signature`, `x-amz-credential`, `x-amz-security-token`. The list is a baseline and is not authoritative for every site.

Deduplication uses the **un-redacted** normalized URL (per §5). Two URLs that differ only in a sensitive-parameter value are probed separately.

### 17.3 Strict redaction mode (render-only in v0.5)

`--strict-redaction` (default off; recommended on for CI logs that may be archived or shared) replaces **every** query-parameter value and every URL path segment that looks like opaque bearer material with `<redacted>` **in the rendered form only**. It does not change deduplication, the on-the-wire URL, or which probes are issued.

The opaque-segment test is:

- Length ≥ 24 characters, **and**
- ≥ 60% of characters are members of urlsafe-base64 or hex alphabet (`A-Z`, `a-z`, `0-9`, `-`, `_`).

Rationale for the v0.5 narrowing: the v0.4 strict mode collapsed distinct probes that shared a parameter name, hiding broken URLs behind healthy siblings; correctness should not depend on `--strict-redaction` being off, nor on file ordering. A future flag (working name `--collapse-redacted-dedup`) may reintroduce the lossy collapse explicitly (§14 future work). The summary counts (`links checked: N unique (M occurrences)`) are unaffected by `--strict-redaction` in v0.5 — they reflect the un-redacted dedup keys.

### 17.4 Control and format characters

Before emission, the following are replaced by their `\uXXXX` escape:

- ASCII C0 controls (`\x00`–`\x1f`) **except** `\t`.
- DEL (`\x7f`).
- C1 controls (`\x80`–`\x9f`).
- ANSI/OSC introducers `\x1b`, `\x9b` (these are already covered by C0/C1; listed again for clarity).
- Bidi overrides U+202A–U+202E, U+2066–U+2069.
- Line and paragraph separators U+2028, U+2029.
- Zero-width and BOM U+200B–U+200F, U+FEFF.

(v0.4 wording "ASCII C0 controls except \t, DEL" was ambiguous about whether DEL was excepted from escaping; v0.5 clarifies that DEL **is** escaped.)

Line numbers, counts, and literal report chrome are safe and are not rewritten.

### 17.5 Hostname rendering

Hosts that contain non-ASCII labels, **or** hosts that the user typed as an A-label literal (`xn--...`) whose Unicode form differs from the ASCII form, are rendered as `<punycode> (<unicode>)`, e.g., `xn--mnchen-3ya.de (münchen.de)`. The Punycode form is what §18 matches against; showing both forms makes spoofing-via-confusables visible and applies uniformly whether the input was typed in Unicode or already as an A-label. Pure-ASCII hosts whose A-label form equals the user input are rendered as-is. This rendering applies to all URLs in the report, including refused-redirect targets rendered per §16.5.

### 17.6 Scope

The filter applies uniformly to the TTY report, the stderr progress line, warnings, error messages, and any future machine-readable output.

## 18. Host Matching Canonicalization

Applies to `--allow-host`, `--deny-host`, and any future host-scoped flag or policy. **`--allow-internal-host` is governed separately by §16.7** (IP/CIDR only in v0.5).

- Both the flag value and the URL host are converted to their IDNA A-label (Punycode) form and case-folded to lower case before comparison.
- A trailing dot on either side is stripped.
- IPv6 literals on the CLI are written **without** surrounding brackets when no port is attached (e.g. `--deny-host ::1`) and **with** brackets when a port is attached (e.g. `--deny-host [::1]:8443`). Brackets are the disambiguator that resolves the colon collision noted in review r02.
- IPv4 and IPv6 literals are compared after dotted-quad (IPv4) or RFC 5952 (IPv6) canonicalization.
- Match semantics by prefix (DNS names only):
  - `example.com` — matches the **apex only** (changed from v0.3).
  - `.example.com` — matches subdomains only, not the apex.
  - `*.example.com` — matches apex **and** all subdomains (explicit opt-in; the only glob character accepted, and only in this exact position).
  - `=example.com` is still accepted as a synonym of the bare form for backwards compatibility with v0.3 allowlists, and matches the apex only.
- The `=`, `.`, and `*.` prefixes are **rejected with exit 2** when applied to an IP-literal entry.
- Ports are not part of the host *identity*. A port-scoped entry uses `host:port` for DNS hosts and `[ipv6]:port` for IPv6 literals, and matches only that exact port. Port-scoped entries are matching rules; per-host *accounting* (§19) remains port-agnostic.
- Glob characters other than the exact `*.` apex-plus-subdomains prefix are not supported in v0.5.
- Evaluation order is given in §16.8 and is re-applied at every redirect hop per §16.5.

### 18.1 v0.3 → v0.5 migration notice

In v0.3, `--allow-host example.com` admitted both `example.com` and `www.example.com`. From v0.4 onward, the bare form matches the **apex only**; `www.example.com` requires the explicit `*.example.com` form.

For the v0.4 → v0.5 transition window, when the runtime sees a URL whose host would have matched a bare `--allow-host` / `--deny-host` entry under v0.3 semantics (i.e., is a strict subdomain of an entry), but does **not** match under v0.5 semantics, it MUST emit a single stderr deprecation warning per such entry on the first encounter:

```
warning: --allow-host example.com no longer matches subdomains in v0.4+; saw www.example.com — change to *.example.com to restore v0.3 behavior
```

The warning is suppressed once per `(entry, classification-decision)` pair to avoid flooding logs. The warning is informational only; it does not change classification. It is removed in v0.6.

## 19. Per-Host Concurrency & Backpressure

§11 non-goal reconciliation: §11's non-goal on "per-host rate limiting" refers to crawl politeness / global rate-limits per host as a courtesy toward upstream. §19 is a defensive concurrency bound and a short reactive backoff on explicit upstream signals (429 / 503). §20 binds this reading — §19 supersedes §11 for this aspect in v0.3+.

- Default per-host cap: **4** concurrent in-flight requests.
- Override via `--per-host-concurrency N`. Valid range: `1 ≤ N ≤ --concurrency`. Out of range → exit 2.
- Host identity uses the canonical form from §18, **port-agnostic**, for the *accounting bucket*.
- The global `--concurrency` pool remains the primary limit; per-host cap only gates dispatch.
- **Backpressure on signals from a host:**
  - A `429` or `503` *with* `Retry-After` reduces **the emitting host's** effective cap to 1 for the remainder of the run. When the response was emitted mid-chain, the reduction applies to the responder, not to the originating host or any intermediate host.
  - A `429` or `503` *without* `Retry-After` likewise reduces the cap to 1 and additionally inserts a 1 s synthetic backoff. **Binding reading of "synthetic backoff":** the 1 s wait is applied before **every subsequent dispatch** to that host for the remainder of the run (combined with cap=1, this means each request to the host is preceded by a 1 s delay). It is not applied once-and-done.
  - The cap reduction and synthetic backoff apply even when the 429/503 is observed on a chain that the probe will ultimately fail on (e.g., final 503, retries exhausted) — the responder's behavior is the signal, not the probe outcome.
  - The reduced cap is sticky for the run; it is not restored by subsequent successful responses.
  - A wait against the per-host bucket (slot acquisition or synthetic backoff) counts against the waiting probe's `--timeout` wall-clock. Exhaustion classifies the probe as `broken:network` reason `timeout-per-host-queue`.
  - When the existing cap is already 1, the cap-reduction rule is a no-op, but the synthetic-backoff insertion (without `Retry-After`) still applies.
- **Redirect hop slot lifecycle.** A probe holds one global slot for the duration of the probe (across all hops). Per-host slots behave as follows:
  1. When dispatch begins to host A, the probe acquires a per-host slot on A.
  2. When A returns a redirect response, **A's per-host slot is released immediately** after the response is consumed, before the probe waits for B's bucket.
  3. The probe then waits for a per-host slot on B (the redirect target's host). The global slot is **held** throughout this wait to preserve the global cap invariant.
  4. If B == A (same-host hop), the slot on A is released in step 2 and re-acquired in step 3; the probe rejoins A's queue **at the tail** (no priority for in-flight chains). Starvation is bounded by the probe's `--timeout`; exhaustion → `broken:network` reason `timeout-per-host-queue`. This is intentional in v0.5: granting in-flight chains priority would let a single slow chain monopolize the host bucket against fresh probes that may complete quickly.
  5. The wait in step 3 is bounded by the remaining `--timeout` budget for the probe.

## 20. Clarifications

The user-frozen sections above carry some ambiguities; these clarifications are binding where they narrow language elsewhere.

- **Version labeling.** "v0.1" labels in §§2, 3, 4, 8, 11, 13, 14 describe the surface captured in this v0.5 document. They are not deferrals past v0.5.
- **§4 vs §4a.** §4 enumerates the v0.1 CLI surface; §4a enumerates v0.2+ additions. The authoritative full surface is the union; conflicts resolve in favor of §4 (no v0.2+ change has weakened or relabeled a v0.1 flag). `linkrot --help` must enumerate both.
- **§11 vs §19 reconciliation.** §11 parks crawl politeness; §19 implements defensive per-host concurrency and reactive backoff. §19 supersedes §11 for this aspect starting in v0.3.
- **Timeout scope.** `--timeout` is a **whole-request** budget covering DNS, connect, TLS, write, read, all redirect hops, per-host-bucket waits, and synthetic backoffs. §6.1 step 4 is the authoritative reading; §4's phrase "connect + read" is a description, not a per-hop timer.
- **HEAD → GET fallback.** The method switch in §6.1 step 2 is **not** a retry and does not consume a `--retries` slot.
- **Invalid URL class.** A URL string that fails `new URL(...)` parsing — or fails the §16.3 zone-id pre-parse predicate, or cannot be normalized per §16.3 — is classified as `broken:invalid-url`, counted under `broken` (per §16.10), honored by `--fail-on=broken`, and never retried. §12's reference to `broken:network` with reason `invalid-url` is superseded.
- **§12 vs §20 file-failure reconciliation.** §12 (frozen) covers the initial scan-root argument: ENOENT/EACCES on the root or read failure on a single-file argument → exit 2, no partial report. §20's "Extraction failure on one file" rule (below) covers per-file errors discovered after the scan root has been enumerated.
- **Manual redirect following.** The implementation MUST use `redirect: "manual"` (or an equivalent manual-follow seam) so that hop count, cycle detection, the SSRF per-hop re-check (§16.5), the host-policy re-check (§16.8), and fresh-socket-per-hop (§16.4) are deterministic regardless of Bun's internal redirect cap.
- **Retry-After.** Parsed as both delta-seconds and HTTP-date. The wait is capped to `cap = min(Retry-After, --timeout − elapsed)` where `elapsed` is wall-clock since probe start. **If `cap > 0`**: the probe waits `cap`, the subsequent attempt consumes one `--retries` slot, and the wait counts against the URL's `--timeout` budget. **If `cap ≤ 0`** (i.e., the per-URL timeout budget is exhausted): no further retry is issued; the link resolves on its current state; **no `--retries` slot is consumed**.
- **Backoff (binding reading of §4).** For retry attempt `n` (1-indexed), base wait is `200ms × 2^n` (400 ms at `n=1`, 800 ms at `n=2`, …); the actual wait is sampled uniformly from `[base × 0.75, base × 1.25]`. Truncated so total URL wall-clock does not exceed `--timeout`.
- **`--include`/`--exclude` merge.** User `--include` values **replace** the built-in includes. User `--exclude` values **append** to the built-in excludes. `--no-default-excludes` disables the built-in excludes.
- **Scheme-skip accounting.** URLs filtered for non-HTTP(S) schemes are excluded from both "links checked" and "skipped" in the summary and appear nowhere in the output. Reference-style image links `![alt][ref]` are handled identically to non-image reference links.
- **Image probing.** On by default in v0.5 per §5. `--no-images` disables it without affecting other behavior.
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

- **Extraction failure on one file.** A *fatal* parser error on a single file is treated as a skip (stderr warning, file excluded, other files still scanned). "Fatal" means the parser cannot produce a token stream for the file at all (I/O failure during read of a non-root file, decoder failure, or parser exception). A file where the parser recovers and emits a partial token stream counts as **scanned**, not skipped. The summary's `files scanned` counts fully- and partially-extracted files; a `files skipped` line appears when non-zero.
- **Exit code 2 scope.** §9 defines exit 2 as "Usage error." v0.4 extended the *scope* of exit 2 to also cover work-budget exhaustion under `--max-files`, `--max-urls`, `--max-occurrences` (§23); v0.5 retains this. Stderr message names the budget. v0.5 also adds: a bare-hostname value to `--allow-internal-host` exits 2 with a stderr message naming the offending value.
- **Exit code 3 scope.** §9 defines exit 3 as "Internal error (unhandled exception). Stack trace on stderr." v0.4 extended the scope to capability errors detected at startup (§16.4 fail-closed); v0.5 retains this. Capability errors MUST NOT print a stack trace — only a single-line message naming the missing capability and the flag or runtime condition that triggered the check.
- **Env-var scope (CLI-only flags).** The following flags have no env-var counterpart and MUST be ignored if present in the environment with a `LINKROT_*` prefix: `--include`, `--exclude`, `--allow-host`, `--deny-host`, `--fail-on`, `--accept-redirects`, `--allow-internal`, `--allow-internal-host`, `--per-host-concurrency`, `--no-images`, `--strict-redaction`, `--follow-symlinks`, `--max-files`, `--max-urls`, `--max-occurrences`, `--max-url-length`, `--max-redirect-headers`, `--max-response-header-bytes`, `--max-location-length`, and `--no-default-excludes`. They materially affect safety, classification, or work bounds and must be explicit at invocation. A `LINKROT_*` env var that maps to one of these names is silently ignored (no startup warning, no classification effect); regression tests assert this (§21).
- **DNS-classification cache scope.** Per §16.4, the in-process per-run cache stores the **classification decision** including the chosen IP. A blocked classification stays blocked for the cache window; a non-blocked classification reuses the same chosen IP for connect within the window.

## 21. Supplemental Testing Requirements

In addition to §13, the following test cases are required.

- **Signals.** SIGINT and SIGTERM during probing → exit 130, partial report emitted, in-flight requests aborted.
- **Redirect limits.** Chain of exactly 10 hops (pass), 11 hops (→ `broken:redirect-loop`), 3-node cycle (→ `broken:redirect-loop`).
- **Manual-redirect enforcement.** Test that the transport seam observes each hop separately and that `fetch`-style automatic following is never invoked. A regression test substitutes a transport that fails if the runtime tries to follow a redirect internally (i.e., asserts the SSRF classifier and host-policy re-check are called once per hop).
- **Retry-After.** Delta-seconds, HTTP-date, value greater than the **remaining `--timeout` budget** at the moment of evaluation (→ `cap ≤ 0` → final on current state, no retry slot consumed), value of `0`, malformed value, and `429`/`503` *without* `Retry-After` (cap reduction + 1 s synthetic backoff per §19, applied to every subsequent dispatch to that host).
- **Synthetic backoff & cap-stickiness.** A `429` without `Retry-After` from host H reduces H's cap to 1 and inserts a 1 s wait before each subsequent dispatch to H. A subsequent `200` from H does **not** restore the cap; tested by issuing a third request to H and observing it still waits and runs at cap=1.
- **Method switch.** HEAD → GET on 400, 403, 405; verify the switch does not consume a retry slot.
- **Precedence.** Flag > env > default for every flag with an env-var counterpart in §10 (`LINKROT_CONCURRENCY`, `_TIMEOUT`, `_RETRIES`, `_USER_AGENT`, `_NO_COLOR`).
- **CLI-only env-var rejection.** `LINKROT_ALLOW_INTERNAL=1`, `LINKROT_PER_HOST_CONCURRENCY=1`, `LINKROT_STRICT_REDACTION=1`, `LINKROT_MAX_RESPONSE_HEADER_BYTES=…`, etc., are present in the environment but MUST NOT alter behavior; the corresponding flag values come from the CLI or default. Verified per flag listed in §20's CLI-only set.
- **Host canonicalization.** IDNA/Punycode, ASCII case folding, trailing dot, IPv6 literal (with and without bracketed `[ipv6]:port` form), apex-only (bare and `=`) matching, subdomain-only (`.`) prefix, apex-plus-subdomains (`*.`) prefix, rejection of `=`/`.`/`*.` on IP-literal entries, rejection of `--per-host-concurrency` outside `[1, --concurrency]` with exit 2.
- **v0.3→v0.5 migration warning.** A bare `--allow-host example.com` invocation that encounters `www.example.com` in the input emits the §18.1 stderr deprecation warning exactly once, does not change classification (`www.example.com` is `skipped:host`).
- **SSRF policy — direct.** Direct `http://127.0.0.1`, `http://[::1]`, `http://169.254.169.254`, and a hostname resolving only to RFC1918 → `broken:blocked-destination`, trips `--fail-on=broken` (default), counted under `broken`, never retried.
- **SSRF policy — IP literals.** Non-canonical forms (`0177.0.0.1`, `2130706433`, `0x7f000001`, `127.1`, `::ffff:127.0.0.1`) are either normalized and blocked, or refused as `broken:invalid-url`. Never probed through.
- **SSRF policy — zone id.** `fe80::1%eth0` → `broken:invalid-url` via the §16.3 pre-parse predicate.
- **SSRF policy — redirect.** Public host redirects to a private destination → `broken:blocked-destination-redirect`, trips `--fail-on=broken`.
- **Host-policy on redirect.** A public host redirects to another public host that matches `--deny-host` → `broken:blocked-redirect-host`, trips `--fail-on=broken`. With `--allow-host` set so that the redirect target is outside the allowlist → also `broken:blocked-redirect-host`. Distinct from the SSRF redirect case.
- **SSRF policy — mixed.** Host resolves to one public and one private address → blocked.
- **SSRF policy — opt-in.** `--allow-internal` flips behavior for all internal destinations and surfaces the banner on **both** stdout and stderr; `--allow-internal-host` (CIDR/IP) lets only matching destinations through and emits the `internal allowances: <count>` summary line **on stdout** (not mirrored to stderr); when both flags are set, `--allow-internal` wins.
- **SSRF policy — bare-hostname rejection.** `--allow-internal-host internal.example` exits 2 with a stderr message naming the value (per §16.7 v0.5 narrowing).
- **SSRF policy — IPv4 CIDR.** `--allow-internal-host 10.0.0.0/8` admits a destination in `10.x`, still blocks `192.168.x` and link-local.
- **SSRF policy — IPv6 CIDR.** `--allow-internal-host fd00::/8` admits a ULA destination in `fd12:…`, still blocks link-local `fe80::/10` and loopback `::1`.
- **SSRF policy — evaluation order.** A destination that matches `--deny-host` is `skipped:host` (initial) / `broken:blocked-redirect-host` (redirect) even if `--allow-internal-host` covers it. A destination that passes `--allow-host` and matches `--allow-internal-host` and resolves to private space is probed. A destination that passes `--allow-host` but does not match any `--allow-internal*` and resolves to private space is `broken:blocked-destination`.
- **DNS-rebinding race.** Inject a resolver fixture whose `resolve4` returns address X and whose subsequent calls would return address Y; assert the TCP connect target equals X, that no second resolution occurs at connect time, and that within the §16.4 cache window a second probe of the same host reuses X (cached classification, not raw addresses).
- **SSRF DNS failure.** NXDOMAIN → `broken:network` with reason `dns:nxdomain`.
- **Refused-redirect rendering.** A public URL that redirects to a private destination renders the refused target per §16.5's literal format `<host-rendered>[:port]/<elided>  (refused after N hops)`; the IDN case renders `<punycode> (<unicode>)`. Golden tests cover both ASCII-host and IDN-host cases, and the host-policy bounce case (`broken:blocked-redirect-host`).
- **Invalid URL.** A malformed URL → `broken:invalid-url`, counted under `broken`, trips `--fail-on=broken`, never retried.
- **Per-host cap.** A burst of requests to a single host respects the default and the `--per-host-concurrency` override; dispatch to other hosts is not blocked. Boundary cases: `N=1`, `N=--concurrency`, `N=--concurrency + 1` → exit 2.
- **Per-host cap on redirect.** A→B hop releases A's per-host slot immediately and waits on B's bucket while holding the global slot. A saturated B causes the probe to wait bounded by `--timeout`; timeout in that wait classifies as `broken:network` with reason `timeout-per-host-queue`. A→A hop releases and re-acquires A's slot at the tail.
- **Per-host cap on intermediate 429/503.** A chain where an intermediate hop on host M emits 429/503 reduces **M's** cap, not A's nor B's.
- **Global cap.** A burst spread across many hosts never exceeds `--concurrency` in-flight requests.
- **Fresh socket per hop.** A regression test asserts that each hop in a redirect chain opens a new TCP connection; cross-hop socket reuse and HTTP/2 origin coalescing are absent. Tested via a fixture that counts TCP accepts and asserts `accepts == hops` for an HTTPS chain on the same origin.
- **Response-byte caps.** A server that returns 70 KB of headers under default settings → `broken:network` reason `header-bytes-exceeded`; raising `--max-response-header-bytes` to 131072 lets the same probe succeed. A `Location:` header of 5000 bytes → `broken:network` reason `location-too-long`. A 1000-character status reason-phrase is truncated in the report to 200 bytes + `…`. A URL longer than 512 bytes is rendered with the head-tail-elision pattern.
- **Redaction and sanitization.** URLs with userinfo, sensitive query parameters, embedded U+2028/U+2029, ANSI/bidi/zero-width characters render safely; DEL (`\x7f`) is escaped as `\u007f` per §17.4.
- **Sensitive-param dedup (default).** Two URLs differing only in `?token=` value are probed **separately** (per §17.2). Both appear in the report; both count toward `links checked` unique.
- **Strict redaction is render-only.** Two URLs differing only in `?token=` value are still probed **separately** under `--strict-redaction`; both count toward `links checked` unique. Only their rendered form is `<redacted>`. (Changed from v0.4.)
- **Strict opaque-segment heuristic.** The `length ≥ 24 and ≥ 60% urlsafe-base64/hex` rule is tested against representative positives (JWT path segments, S3 bearer path components, GitHub tokens) and negatives (human-readable slugs of length 24+, path segments with `.html`/`.pdf` extensions).
- **IDN rendering.** A URL on `münchen.de` renders as `xn--mnchen-3ya.de (münchen.de)`; a URL the author typed as `xn--mnchen-3ya.de` renders the same way.
- **Symlinks.** A symlink inside the scan root that points outside the root is not followed by default; `--follow-symlinks` allows following but still confines to the scan root; symlink loops do not hang the scanner.
- **Path containment.** A scan root of `/repo/docs` does not admit a candidate resolving to `/repo/docs-evil/file.md` (component-aware containment).
- **Path containment — root-is-separator edge case.** A scan root that is the filesystem root (`/` on POSIX, `C:\` on Windows) does not double-append a separator; admission still works for legitimate paths under the root.
- **Work budgets.** Exceeding `--max-files`, `--max-urls`, or `--max-occurrences` aborts with exit 2 and a budget-named stderr message; URLs longer than `--max-url-length` are classified `skipped:url-too-long`; `--max-redirect-headers` cap drops report-only header lines without affecting `broken:redirect-loop` detection.
- **Include/exclude merge.** User `--include` replaces built-in includes; user `--exclude` appends; `--no-default-excludes` disables built-ins.
- **Determinism for golden tests.** A test mode (e.g., `LINKROT_DETERMINISTIC=1`) renders `duration` as a fixed placeholder and suppresses the transient progress counter so golden snapshots are stable.
- **Transport seam.** Unit tests can inject DNS NXDOMAIN, TCP RST after headers, TLS handshake failure, header-phase timeout, IP-pinned connect, and IPv6→IPv4 happy-eyeballs fallback through an injectable transport seam, without live network. The fail-closed behavior of §16.4 is exercised by a fixture that disables the seam — the full CLI binary must exit 3 with a one-line capability error and **no stack trace**.
- **Happy-eyeballs fallback.** A host classified with both IPv6 and IPv4 non-blocked addresses; the IPv6 connect attempt fails (RST). The probe falls back to the same call's IPv4 address (no second resolution). If both fail, the probe is `broken:network`. If a second resolution would have returned a different mix, the test asserts that mix is **not** consulted.
- **End-to-end seam confinement.** A regression test runs the real CLI with a sentinel transport that refuses connection to any address other than a test-selected IP; any SSRF classification or probe that bypasses the seam fails the test.
- **Fan-out golden.** A fixture with one URL referenced from two files produces the §20 worked-example output exactly.

## 22. Filesystem Traversal & Symlinks

- The scan **root** is the resolved real path of the `path` argument (or `.`). The root value retained internally for containment checks is `realRoot + separator` (e.g., `/repo/docs/`), **except** when `realRoot` already ends in the platform path separator (filesystem root `/` on POSIX, drive root `C:\` on Windows), in which case the separator is not double-appended.
- All traversal is confined to that root. A candidate file is admitted only if its resolved real path is either equal to `realRoot` or begins with `realRoot + separator` (computed per the rule above). A raw string-prefix check against `realRoot` alone is **not** sufficient: `/repo/docs` must not admit `/repo/docs-evil/file.md`. Implementations that use `path.relative(realRoot, candidate)` MUST verify the result is neither absolute nor begins with `..` before admitting the candidate. The component-aware test applies to the scan root itself.
- Any candidate that fails the containment check is **skipped with a stderr warning** naming the candidate.
- Symbolic links (POSIX) and reparse points (Windows) are **not** followed by default. The bare symlink itself is also not read.
- `--follow-symlinks` enables following, but root containment still applies. A symlink that resolves outside the root is skipped with a stderr warning even with `--follow-symlinks`.
- Symlink and directory cycles are detected via real-path bookkeeping; a cycle terminates the offending branch with a stderr warning, never the whole run.
- Special files (sockets, devices, FIFOs, block/char devices) are skipped with a stderr warning, regardless of glob match.

## 23. Work Budgets

- `--max-files N` (default `10000`): scanning aborts with exit 2 once N matching files have been enumerated.
- `--max-urls N` (default `50000`): unique-URL count is capped after dedup; reaching the cap aborts extraction with exit 2 before any probe is issued.
- `--max-occurrences N` (default `200000`): total source-location count cap; same fail-fast behavior.
- `--max-url-length BYTES` (default `8192`): URLs longer than the cap are classified `skipped:url-too-long`, counted under `skipped`, never probed. Per-URL cap.
- `--max-redirect-headers N` (default `64`): **per-hop count of response-header lines retained for the report**. Excess lines on a given hop are dropped from the report only; redirect-loop detection (§6.1 step 3) and SSRF/host-policy re-checks are unaffected because they use the wire response, not the retained metadata. The unit is header lines, not bytes; the byte cap is the separate `--max-response-header-bytes` flag (§16.12).
- `--max-response-header-bytes BYTES` (default `65536`): per-hop hard cap on total response-header bytes received from the wire. Exceeding → `broken:network` reason `header-bytes-exceeded` (§16.12).
- `--max-location-length BYTES` (default `4096`): per-hop cap on `Location:` byte length. Exceeding → `broken:network` reason `location-too-long` (§16.12).
- The aborting cases (`--max-files`, `--max-urls`, `--max-occurrences`) exit 2 per §20; the stderr message names the budget and observed value. `skipped:budget-exceeded` is reserved for future non-aborting per-URL budgets.

## 24. Document Status

This document is SPEC v0.5. Frozen sections (1–15) carry user-final wording from prior rounds; v0.2/v0.3/v0.4/v0.5 additions (4a, 16–23) remain subject to revision in subsequent rounds.

**v0.5 changes (against v0.4):**

- **§4a (new).** Enumerates v0.2+ CLI flags so the authoritative CLI surface is no longer split between §4 and prose — closes review r04's "§4 missing v0.2+ flags" finding.
- **§5.** Cites RFC 3986 §6.2.2.2 for percent-encoding normalization so dedup keys are stable across implementations.
- **§16.4.** Names `Bun.connect` as the v0.5 capability; specifies fresh socket per hop, no cross-host socket reuse, no HTTP/2 origin coalescing for probe traffic; specifies happy-eyeballs fallback semantics (no second resolution); specifies that the in-process cache stores the classification decision (not raw addresses) so a blocked classification cannot be re-resolved within the window.
- **§16.5.** Re-evaluates the **full host policy** at every redirect hop, not just the SSRF check; gives the literal refused-redirect render format.
- **§16.7.** Narrows `--allow-internal-host` to **IP/CIDR only**; bare hostnames exit 2.
- **§16.10.** Adds `broken:blocked-redirect-host` for redirect targets refused by `--deny-host` / `--allow-host`.
- **§16.12 (new).** Hard caps for response-header bytes, `Location:` length, status reason-phrase, rendered URL length, and network-error message text.
- **§17.3.** `--strict-redaction` is now **render-only**; the v0.4 lossy dedup is removed and reserved as a possible future explicit flag (§14).
- **§17.4.** Disambiguates DEL — DEL is escaped.
- **§18.1 (new).** Migration warning for the v0.3 → v0.4/v0.5 bare-host narrowing.
- **§19.** Pins synthetic-backoff scope (every subsequent dispatch), confirms cap reduction applies regardless of probe outcome, confirms per-host wait counts against `--timeout`, retains tail-of-queue rejoin for in-flight A→A.
- **§20.** Retry-After test wording reconciled with the cap formula by saying "remaining `--timeout` budget at evaluation"; CLI-only env-var set must be silently ignored, with regression coverage in §21.
- **§21.** Adds tests for IPv6 CIDR allowance, manual-redirect enforcement, synthetic backoff & cap-stickiness, CLI-only env-var rejection, fresh-socket-per-hop, response-byte caps, host-policy bounce on redirect, happy-eyeballs fallback, root-as-separator edge case, and v0.3→v0.5 migration warning.
- **§22.** Handles the edge case where `realRoot` already ends in the platform separator.
- **§23.** Clarifies `--max-redirect-headers` as **per-hop count of header lines retained for the report**; byte caps live in §16.12.

Not addressed in v0.5 (deferred):

- A dedicated exit code for budget-exhaustion remains in v0.6+ scope (currently exit 2 with a budget-named stderr message).
- A `--prefer-ipv4` flag for §16.4 address selection is deferred.
- An optional explicit lossy-dedup flag for redacted parameters (`--collapse-redacted-dedup`) is in §14 future work.
- Hostname-scoped `--allow-internal-host` allowances may return alongside an explicit private-CIDR constraint set (§14).
- Proxy support remains in §14 future work.
