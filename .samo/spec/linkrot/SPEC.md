# SPEC — tiny-linkrot v0.1

## 1. Persona

Veteran Bun/TypeScript CLI engineer. Ships small, well-tested, single-purpose tools to npm with minimal dependencies, fast startup, and POSIX-friendly ergonomics.

## 2. Idea

`tiny-linkrot` is a Bun + TypeScript CLI that scans a directory of Markdown files, extracts HTTP(S) links, probes each one over the network, and prints a human-readable report of broken or redirected links grouped by source file. It exits non-zero when rot is detected so it can slot into CI.

Design priorities (in order):
1. Correctness of the rot verdict on the golden-path inputs.
2. Predictable, bounded runtime (fixed concurrency, per-request timeout).
3. Minimal dependency footprint — prefer Bun/Node built-ins.
4. Readable output a human can scan in a terminal.

## 3. Goals / Non-goals

### Goals (v0.1)

- Recursively scan a directory for `*.md` files.
- Extract every unique HTTP(S) URL that appears as a Markdown link target or autolink.
- Probe each URL with `HEAD`, falling back to `GET` on 405/501 or when `HEAD` returns an obviously wrong body-requiring status.
- Follow redirects up to a bounded chain; report the final status and the redirect chain length.
- Print a grouped-by-file, human-readable report of rotted links.
- Exit `0` when no rot, `1` when rot is found, `2` on CLI/usage errors.
- Ship to npm as a Bun-runnable CLI with local fixture-backed tests.

### Non-goals (v0.1)

- No autofix or rewrite of rotted links in source files.
- No per-host rate limiting or politeness windows.
- No JSON/SARIF/JUnit output formats (text only).
- No authenticated probes (no cookies, headers, or auth flags).
- No scanning of non-Markdown sources (HTML, MDX, source comments).
- No crawling of link targets (depth = 1, no transitive discovery).
- No caching between runs.

## 4. Interview outcomes (frozen decisions)

| Question | Decision |
|---|---|
| link-sources | Markdown files in a given directory, scanned recursively. |
| check-semantics | A link is "rotted" when the final HTTP status (after following redirects) is `>= 400`, or when the probe errors out (DNS failure, TCP reset, TLS failure, timeout). |
| concurrency-and-politeness | Fixed global concurrency of 10 in-flight requests. No per-host throttling. |
| output-format | Human-readable plain text to stdout. Non-zero exit code when any link is rotted. |
| scope-cuts | No autofix / rewriting of source files. |

## 5. CLI surface

```
tiny-linkrot [options] <path>
```

- `<path>` — required. A directory (recursed) or a single `.md` file. Relative paths resolved against cwd.
- `--timeout <ms>` — per-request timeout in milliseconds. Default `10000`.
- `--concurrency <n>` — override the fixed global concurrency. Default `10`. Must be `>= 1`.
- `--max-redirects <n>` — cap redirect chain length. Default `5`. `0` disables following.
- `--user-agent <string>` — override the UA. Default `tiny-linkrot/<version> (+https://github.com/NikolayS/tiny-linkrot)`.
- `--include <glob>` / `--exclude <glob>` — repeatable. Default include: `**/*.md`. Defaults exclude: `**/node_modules/**`, `**/.git/**`.
- `--quiet` — suppress the "OK" summary line; still prints rot.
- `--version`, `--help` — standard.

Exit codes:
- `0` — scan completed, no rotted links.
- `1` — scan completed, at least one rotted link.
- `2` — usage error (bad flag, missing path, path not found).
- `3` — internal error (I/O failure outside the probe loop).

## 6. Link extraction

1. Enumerate files via `Bun.Glob` matching include/exclude patterns; read as UTF-8.
2. From each file, extract HTTP(S) URLs from:
   - Inline links: `[text](url)` and `[text](url "title")`.
   - Reference links: `[text][id]` resolved via `[id]: url` definitions within the same file.
   - Autolinks: `<https://example.com>`.
   - Bare URLs that appear inside paragraph text are NOT extracted in v0.1 (keeps the extractor a Markdown parser, not a URL sniffer). Documented explicitly.
3. Skip fenced code blocks and inline code spans — links inside code are not probed.
4. Strip trailing punctuation that GitHub-flavored Markdown commonly leaves attached (`.`, `,`, `)` when unbalanced).
5. Deduplicate URLs globally for probing, but retain every `(file, line, url)` occurrence for the report.

Use a small, well-tested Markdown parser (`remark` + `remark-parse` + `unist-util-visit`) rather than hand-rolled regex. This is the only non-trivial dependency; justified by the long tail of Markdown edge cases.

## 7. Probing

- Use `fetch` (Bun built-in). One pool, fixed concurrency = 10 (or `--concurrency`).
- Per-URL algorithm:
  1. Issue `HEAD` with `redirect: "manual"`, record redirect hops up to `--max-redirects`. Each hop has its own timeout of `--timeout` ms via `AbortController`.
  2. If any hop returns `405`, `501`, or `403` with `Allow:` not containing `HEAD`, retry that hop as `GET` with `redirect: "manual"` and immediately abort the response body after headers arrive.
  3. Stop when a non-3xx response is received, or when `--max-redirects` is exceeded (treated as rot with reason `too-many-redirects`).
  4. Network failures (DNS, TCP, TLS, timeout, abort) are rot with reason `network:<code>`.
- No retries on transient 5xx in v0.1. Documented as a likely v0.2 addition.
- Classification:
  - `ok` — final status `< 400` and redirect chain length `<= 1` (original or one permanent redirect resolved).
  - `redirect` — final status `< 400` but chain length `> 1`, OR any hop is a `301`/`308`. Reported as INFO, not rot. Does NOT affect exit code.
  - `rot` — final status `>= 400`, network error, or too-many-redirects.

## 8. Report format

Stdout, plain text, stable ordering (files sorted by path; within a file, by line number). Example:

```
tiny-linkrot scanned 12 file(s), 47 link(s), 42 ok, 3 redirect, 2 rot

docs/setup.md
  L14  rot      404  https://example.com/old-guide
  L52  redirect 301 -> 200  https://foo.test/a -> https://foo.test/a/

README.md
  L7   rot      network:ENOTFOUND  https://nope.invalid

2 rotted link(s) in 2 file(s). Exit 1.
```

Stderr is reserved for progress lines (only when stdout is a TTY) and internal warnings. Non-TTY runs (CI) emit zero progress noise.

## 9. Project layout

```
/                 package.json, tsconfig.json, README.md, LICENSE
/bin              tiny-linkrot  (bun shebang entry)
/src
  cli.ts          argv parsing, exit-code mapping
  scan.ts         file enumeration + markdown link extraction
  probe.ts        fetch-based prober with concurrency gate
  report.ts       grouping + formatting
  types.ts        shared types
/test
  fixtures/       .md files + a local http server used by tests
  *.test.ts       bun:test suites
```

## 10. Testing

- Runner: `bun test`.
- All network tests use a local HTTP server started in-process (`Bun.serve`) on an ephemeral port, returning scripted statuses/redirects/timeouts. No live internet access in the test suite.
- Required coverage:
  - Extractor: inline, reference, autolink, fenced-code exclusion, trailing-punct trimming, dedup with occurrence retention.
  - Probe: HEAD → GET fallback on 405, redirect chain capping, timeout via AbortController, network error mapping.
  - Report: grouping, ordering, exit-code mapping for all of {all-ok, mixed, all-rot, empty-scan}.
  - CLI: `--help`, `--version`, bad flag → exit 2, missing path → exit 2.

## 11. Dependencies

- Runtime: `remark`, `remark-parse`, `unist-util-visit`. Nothing else.
- Dev: `typescript`, `@types/bun`.
- No `node-fetch`, no `chalk` (use ANSI only when `process.stdout.isTTY`), no arg-parsing lib (hand-rolled parser for ~8 flags is smaller and has no surface).

## 12. Performance budget

- Cold start (`tiny-linkrot --help`) `< 150 ms` on a modern laptop.
- Scanning 500 markdown files with ~2000 unique links against the local test server: `< 5 s` wall clock with default concurrency.
- Memory: proportional to unique-URL count; no streaming required for v0.1.

## 13. Publishing

- `package.json` `bin` field points to `./bin/tiny-linkrot`.
- `engines`: `{ "bun": ">=1.1.0" }`. Node is not supported in v0.1.
- `files`: `bin/`, `dist/`, `README.md`, `LICENSE`.
- Build step: `bun build src/cli.ts --target=bun --outfile dist/cli.js` then the bin stub execs it.
- Tag-driven publish via GitHub Actions (out of scope for this spec beyond naming the workflow file `publish.yml`).

## 14. Open questions / deferred to v0.2

- Retry policy for transient 5xx and `429`.
- JSON/SARIF output.
- Per-host concurrency and `Retry-After` honoring.
- Optional on-disk cache keyed by `(url, etag/last-modified)`.
- Node compatibility.

## 15. Acceptance criteria for v0.1

1. `bun test` passes with zero network access.
2. Running the CLI against `test/fixtures/sample-docs` produces the documented report format and exit code `1`.
3. Running against a directory with only healthy links exits `0` and prints the one-line summary.
4. `tiny-linkrot --help` and `--version` work and exit `0`.
5. Package installs from a local `npm pack` tarball and the `bin` entry runs end-to-end.
