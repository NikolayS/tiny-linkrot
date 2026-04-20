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
