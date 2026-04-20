# tiny-linkrot — SPEC v0.1

## 1. Overview

`tiny-linkrot` is a Bun + TypeScript command-line tool that scans Markdown files, extracts HTTP(S) links, probes each link over the network, and prints a human-readable (or JSON) report of broken and redirected links grouped by source file. It is the reference demo for the samospec workflow.

This specification covers **v0.1**, the first shippable release. Anything not explicitly in scope below is deferred.

## 2. Goals and Non-Goals

### 2.1 Goals

- Detect dead HTTP(S) links in a single Markdown file with acceptable speed on inputs up to ~500 links.
- Ship as a single self-contained executable produced by `bun build --compile`.
- Provide both a human-oriented terminal report and a stable `--json` output suitable for CI.
- Exit non-zero when any link is classified as broken, so CI pipelines can gate on it.

### 2.2 Non-goals (explicitly deferred)

- Directory/recursive crawl (multiple files, globs, `.gitignore` awareness).
- Non-Markdown inputs (HTML, MDX, AsciiDoc, plain text).
- Anchor / fragment validation (`#section`) — fragments are stripped before probing.
- Relative-path link resolution to the local filesystem.
- Persistent cache between runs.
- Per-host rate limiting, robots.txt, or politeness beyond a global concurrency cap.
- Authentication, cookies, proxy configuration.
- Retries or exponential backoff beyond a single HEAD→GET fallback.
- Npm distribution, Homebrew formula, Docker image (a single compiled binary is the only artifact).
- Windows support is best-effort; first-class targets are macOS and Linux.

## 3. User Stories

1. **Doc author, local check.** As a docs maintainer, I run `tiny-linkrot README.md` and see a grouped report of any broken or redirected links so I can fix them before pushing.
2. **CI gate.** As a CI pipeline, I run `tiny-linkrot --json docs/GUIDE.md > report.json`; the exit code tells me pass/fail and the JSON is archived as a build artifact.
3. **Spot check a redirect chain.** As a reviewer, I see in the report that a link returned `301 → 200` with the final URL, so I know whether to update the source.

## 4. CLI Surface

### 4.1 Invocation

```
tiny-linkrot [options] <file.md>
```

Exactly one positional argument: a path to a Markdown file. If zero or more than one positional is supplied, print usage to stderr and exit `2`.

### 4.2 Options (v0.1)

| Flag            | Type      | Default | Meaning                                                                 |
|-----------------|-----------|---------|-------------------------------------------------------------------------|
| `--json`        | boolean   | false   | Emit machine-readable JSON to stdout instead of the human report.       |
| `--timeout <ms>`| integer   | 10000   | Per-request timeout in milliseconds (applies to each HEAD and each GET).|
| `--concurrency <n>` | integer | 10    | Max in-flight requests globally. Must be ≥1.                           |
| `--user-agent <s>` | string | `tiny-linkrot/0.1 (+https://github.com/NikolayS/tiny-linkrot)` | Value of `User-Agent` header. |
| `--version`     | boolean   | —       | Print version and exit `0`.                                             |
| `--help` / `-h` | boolean   | —       | Print usage and exit `0`.                                               |

Unknown flags are a fatal error (exit `2`, usage to stderr).

### 4.3 Exit codes

- `0` — all links OK (no broken links; redirects are not broken).
- `1` — at least one link classified as **broken** (see §6.3).
- `2` — usage / argument error (bad flag, missing file, file not readable, not a file).
- `3` — internal error (uncaught exception, runtime panic). Always accompanied by a stack trace on stderr.

Redirects alone never cause a non-zero exit; they are reported as warnings.

## 5. Input Handling

### 5.1 File read

The input path is resolved against the current working directory, read with `Bun.file(path).text()`, and treated as UTF-8. Files larger than 10 MiB are rejected with exit `2` (defensive guard; v0.1 is not designed for huge inputs).

### 5.2 Link extraction

Links are extracted from the raw Markdown source using a **GitHub-Flavored Markdown-aware parser**, not a regex, to avoid false positives inside fenced code blocks. The implementation SHOULD use a well-known parser (`marked`, `markdown-it`, or the equivalent). Extracted link sources:

- Inline links: `[text](https://example.com)`
- Reference links: `[text][ref]` + `[ref]: https://example.com`
- Autolinks: `<https://example.com>`
- Bare URLs in prose (GFM autolink extension) when the parser surfaces them.
- Image links: `![alt](https://example.com/a.png)` — treated identically to regular links.

**Excluded** from extraction:

- Any URL appearing inside a fenced or indented code block.
- Any URL inside an inline code span (`` `like this` ``).
- Non-HTTP schemes: `mailto:`, `ftp:`, `tel:`, `javascript:`, relative paths, `#fragment` — all ignored silently.

### 5.3 URL normalization

Before probing, each URL is:

1. Parsed with the WHATWG URL API. Unparseable URLs are reported as broken with reason `invalid_url` and do not produce a network request.
2. Fragment (`#...`) stripped (fragments are not validated in v0.1).
3. Deduplicated: identical post-normalization URLs are probed once, but every source occurrence (file + line) is recorded so the report still shows each mention.

### 5.4 Scale assumption

The tool is designed and tested for **a single Markdown file with up to ~500 links**. Larger inputs may work but are not a v0.1 target. The 10 MiB file cap and the 500-link soft assumption together set the expected working envelope.

## 6. Network Probing

### 6.1 Request flow per unique URL

1. Issue `HEAD` with `redirect: 'manual'` so we can observe redirect status codes directly.
2. If the response status is a redirect (`301`, `302`, `303`, `307`, `308`):
   - Follow the `Location` header, resolving relatively against the current URL.
   - Repeat step 1 on the new URL, incrementing a hop counter.
   - Hard cap: **5 redirect hops**. On the 6th, classify as broken with reason `too_many_redirects`.
   - Detect loops (any URL repeated in the chain) → broken with reason `redirect_loop`.
3. If the `HEAD` response is `405 Method Not Allowed`, `501 Not Implemented`, or the server returned a network-level error that could plausibly be HEAD-specific (e.g. connection reset before headers), retry **once** with `GET` using `redirect: 'manual'` and the same redirect handling as above. The GET body is discarded without being read into memory (`response.body?.cancel()`).
4. Any other response status is the final status for that URL.

Only a single HEAD→GET fallback is attempted per hop; there is no retry loop, no backoff, no jitter.

### 6.2 Timeout

Each individual HTTP request (each HEAD, each GET) uses an `AbortController` with the `--timeout` value (default 10 000 ms). A timeout is classified as broken with reason `timeout`. The timeout applies per request, not per URL — a URL that redirects five times may spend up to `6 × timeout` wall-clock time in the worst case.

### 6.3 Broken vs. OK vs. Redirected

| Final outcome                              | Classification | Exit contribution |
|--------------------------------------------|----------------|-------------------|
| Final status `2xx`, no redirects           | `ok`           | none              |
| Final status `2xx`, via 1–5 redirects      | `redirected`   | none (warning)    |
| Final status `3xx` that terminates the chain without a usable `Location` | `broken` (`bad_redirect`) | exit 1 |
| Final status `4xx` or `5xx`                | `broken`       | exit 1            |
| Network error (DNS, TCP, TLS, reset)       | `broken` (`network_error`) | exit 1   |
| Timeout                                    | `broken` (`timeout`) | exit 1     |
| `too_many_redirects` / `redirect_loop`     | `broken`       | exit 1            |
| `invalid_url`                              | `broken`       | exit 1            |

Note: `3xx` appearing inside a chain is normal and not itself broken.

### 6.4 Concurrency

A simple global semaphore caps in-flight requests at `--concurrency` (default **10**). There is no per-host cap in v0.1 — if a Markdown file points every link at one host, that host will see up to 10 concurrent requests. The README MUST document this limitation so users probing sensitive or rate-limited hosts can lower `--concurrency` manually.

### 6.5 Headers

Outbound headers on every request:

- `User-Agent`: value of `--user-agent`.
- `Accept: */*`
- `Accept-Encoding: identity` on HEAD (avoid servers that mishandle compressed HEAD responses); default on GET.

No cookies, no auth, no custom headers configurable in v0.1.

## 7. Output

### 7.1 Human-readable report (default)

Written to **stdout**. Progress indicators, if any, go to **stderr** so stdout stays parseable when redirected. Minimum content:

- A header line with the file path and totals: `tiny-linkrot: 3 broken, 2 redirected, 12 ok (17 links, 15 unique)`.
- For each source file (v0.1 always one), a grouped list ordered by line number. Each entry shows:
  - Line number (1-based, from the source Markdown).
  - Status label, colorized when stdout is a TTY: `BROKEN` (red), `REDIRECT` (yellow), `OK` suppressed by default (see below).
  - Original URL.
  - For `BROKEN`: reason code and, if available, numeric status (e.g. `404 Not Found`, `timeout after 10000ms`, `DNS NXDOMAIN`).
  - For `REDIRECT`: final URL and final status (e.g. `→ https://example.com/new (200)`).
- OK links are **not** printed in human mode (noise reduction). The summary counts them.
- Color uses ANSI escapes only when `process.stdout.isTTY` is true **and** `NO_COLOR` is not set in the environment.

### 7.2 JSON output (`--json`)

When `--json` is set, stdout is a **single JSON document** (not NDJSON), terminated by a newline, with this shape:

```json
{
  "tool": "tiny-linkrot",
  "version": "0.1.0",
  "started_at": "2026-04-20T16:03:55.016Z",
  "finished_at": "2026-04-20T16:04:01.220Z",
  "input": { "path": "README.md", "bytes": 2048 },
  "summary": { "total": 17, "unique": 15, "ok": 12, "redirected": 2, "broken": 3 },
  "links": [
    {
      "url": "https://example.com/dead",
      "occurrences": [{ "file": "README.md", "line": 42 }],
      "status": "broken",
      "reason": "http_status",
      "http_status": 404,
      "final_url": "https://example.com/dead",
      "redirect_chain": [],
      "duration_ms": 312
    }
  ]
}
```

Field contract (stable across v0.1 patches; additive changes allowed, renames/removals are breaking):

- `status`: one of `"ok"`, `"redirected"`, `"broken"`.
- `reason`: present only when `status="broken"`. Enum: `http_status`, `network_error`, `timeout`, `too_many_redirects`, `redirect_loop`, `bad_redirect`, `invalid_url`.
- `http_status`: present when a final HTTP response was received; otherwise omitted.
- `redirect_chain`: ordered list of `{ url, status }` for each hop before the final response. Empty when there were no redirects.
- `duration_ms`: total wall-clock time spent probing this URL, summed across hops.
- `occurrences[].line` is 1-based.

Progress and warnings must not appear on stdout in `--json` mode.

### 7.3 Exit behavior with output

The process prints the full report first (human or JSON), flushes stdout, then exits with the code defined in §4.3.

## 8. Implementation Notes

### 8.1 Runtime and distribution

- Target runtime: **Bun** (latest stable at implementation time). No Node.js compatibility guarantee.
- Language: TypeScript, strict mode.
- Entry point: `src/cli.ts`. Built with `bun build --compile --outfile=dist/tiny-linkrot src/cli.ts`.
- The compiled binary is the only artifact for v0.1. It is checked in to GitHub Releases on tagged versions; it is **not** published to npm.
- `package.json` has no `bin` entry in v0.1 (no npm distribution).

### 8.2 Dependencies

Minimize dependencies. Acceptable third-party modules in v0.1:

- One Markdown parser (`marked` or `markdown-it`).
- Optionally one tiny ANSI color module (or inline the handful of escapes).

No HTTP client library — use the built-in `fetch`. No CLI framework — hand-parse flags (there are only a few).

### 8.3 Concurrency primitive

A minimal in-file semaphore is sufficient (an array of pending resolvers plus a counter). No dependency on `p-limit` etc.

### 8.4 Errors during scan

Network errors on individual URLs are **not** propagated as process errors; they become `status: "broken"` entries with `reason: "network_error"` and a short human-readable `message` field in JSON output. Only programmer errors (unexpected exceptions) should surface as exit `3`.

## 9. Testing

v0.1 test plan (minimum bar for "done"):

1. **Unit tests (Bun test)**
   - Markdown extraction: inline, reference, autolink, image, GFM bare URL, code-block exclusion, inline-code exclusion, mailto/relative ignored.
   - URL normalization: fragment stripping, dedup by normalized URL, invalid-URL handling.
   - Classifier: mapping of HTTP status / network error / timeout / redirect count to `status` + `reason`.
2. **Integration tests against a local HTTP server** (spun up in the test, no network): a fixture server that can return 200/301/302/404/500/405-then-200-on-GET, a redirect loop, a 6-hop chain, a slow endpoint (for timeout), and a connection-reset endpoint.
3. **Golden output tests**: snapshot the JSON output for a small fixture Markdown against the fixture server; assert structural equality modulo `started_at`/`finished_at`/`duration_ms`.
4. **Exit-code matrix**: one test per row of the §4.3 table.

No live-internet tests in CI.

## 10. Security and Privacy Considerations

- The tool issues outbound HTTP(S) requests to every URL it finds in the input file. Users should only point it at Markdown they trust or are comfortable exposing to the network.
- Request bodies are never read into memory on GET fallback; the body is cancelled immediately after headers arrive.
- No data from responses is persisted to disk.
- The `User-Agent` identifies the tool so server operators can block it if desired.
- TLS certificate validation uses Bun/`fetch` defaults; there is no flag to disable it in v0.1.

## 11. Open Questions

None blocking v0.1. Candidates for v0.2+:

- Per-host concurrency / politeness.
- Directory and glob input.
- A local cache keyed by URL + ETag/Last-Modified.
- Configurable retry policy.
- Anchor (fragment) validation for same-origin HTML targets.

## 12. Acceptance Criteria

v0.1 is considered complete when **all** of the following hold:

- `tiny-linkrot README.md` on a file with a mix of good, 404, and redirected links produces the grouped human report described in §7.1 and exits `1`.
- `tiny-linkrot --json README.md` produces a JSON document matching §7.2, validated against the documented schema.
- `bun build --compile` produces a working single-file binary on macOS (arm64 + x64) and Linux (x64) that runs on a machine without Bun installed.
- The test plan in §9 passes in CI with no network egress.
- The README documents: install, basic usage, `--json`, the concurrency/politeness caveat (§6.4), and exit-code semantics.
