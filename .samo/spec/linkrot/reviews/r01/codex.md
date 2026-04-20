# Reviewer A — Codex

## summary

The main gaps are safe-by-default outbound networking and safe logging. In its current form, this spec is not robust enough for CI against untrusted repositories because it can probe internal destinations, follow redirects across trust boundaries, leak secrets to logs, and over-concentrate traffic on single hosts.

## missing-risk

- (major) The spec has no SSRF guardrail. In CI, untrusted Markdown can force probes to `localhost`, RFC1918 space, link-local ranges, IPv6 ULA/link-local, or cloud metadata endpoints. Safe behavior needs to default-deny private/reserved destinations after DNS resolution, with an explicit opt-in for internal hosts.
- (major) Host allow/deny policy is only described for the original URL, but redirects are followed automatically. An allowed public URL can redirect into a denied or private host unless policy is re-applied on every hop before each connection.
- (major) The report prints full source URLs and final redirect targets. That will leak embedded credentials, signed query strings, and other secrets into local terminals and CI logs. Redaction rules for userinfo and sensitive query material need to be in the spec.
- (major) Output sanitization is unspecified. File paths and URLs from untrusted Markdown can contain control characters or ANSI/OSC escape sequences, which can poison terminals and CI logs if printed verbatim.

## weak-implementation

- (major) `--allow-host` / `--deny-host` matching is under-specified for subdomains, trailing dots, punycode/IDNA, IPv6 literals, and ports. For a control that gates outbound network access, those normalization rules must be explicit or different implementations will create bypasses.
- (major) Treating per-host throttling as a post-v0.1 feature is too risky for an ops-facing network checker. A repo with many links to one domain can concentrate the full worker pool on a single host, trip WAF/rate limits, or look like abuse. A conservative per-host cap belongs in the baseline.
- (minor) `invalid-url` is currently folded into `broken:network`. That is the wrong failure domain and will produce misleading retries, summaries, and remediation. Malformed author input should be its own class.

## unnecessary-scope

- (minor) Checking image URLs in v0.1 expands outbound traffic, rate-limit exposure, and false-positive surface without being essential to the core 'docs hyperlink rot' use case. Make image probing opt-in or defer it until the network-safety baseline is in place.

## suggested-next-version

Tighten the next version around a safe baseline: default-deny private/reserved targets after DNS resolution, re-check policy on every redirect hop, fully specify canonical host matching, redact and sanitize all printed output, add per-host concurrency caps, and move image checking behind an explicit flag.

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "missing-risk",
      "text": "The spec has no SSRF guardrail. In CI, untrusted Markdown can force probes to `localhost`, RFC1918 space, link-local ranges, IPv6 ULA/link-local, or cloud metadata endpoints. Safe behavior needs to default-deny private/reserved destinations after DNS resolution, with an explicit opt-in for internal hosts.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "Host allow/deny policy is only described for the original URL, but redirects are followed automatically. An allowed public URL can redirect into a denied or private host unless policy is re-applied on every hop before each connection.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "`--allow-host` / `--deny-host` matching is under-specified for subdomains, trailing dots, punycode/IDNA, IPv6 literals, and ports. For a control that gates outbound network access, those normalization rules must be explicit or different implementations will create bypasses.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "The report prints full source URLs and final redirect targets. That will leak embedded credentials, signed query strings, and other secrets into local terminals and CI logs. Redaction rules for userinfo and sensitive query material need to be in the spec.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "Output sanitization is unspecified. File paths and URLs from untrusted Markdown can contain control characters or ANSI/OSC escape sequences, which can poison terminals and CI logs if printed verbatim.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "Treating per-host throttling as a post-v0.1 feature is too risky for an ops-facing network checker. A repo with many links to one domain can concentrate the full worker pool on a single host, trip WAF/rate limits, or look like abuse. A conservative per-host cap belongs in the baseline.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "`invalid-url` is currently folded into `broken:network`. That is the wrong failure domain and will produce misleading retries, summaries, and remediation. Malformed author input should be its own class.",
      "severity": "minor"
    },
    {
      "category": "unnecessary-scope",
      "text": "Checking image URLs in v0.1 expands outbound traffic, rate-limit exposure, and false-positive surface without being essential to the core 'docs hyperlink rot' use case. Make image probing opt-in or defer it until the network-safety baseline is in place.",
      "severity": "minor"
    }
  ],
  "summary": "The main gaps are safe-by-default outbound networking and safe logging. In its current form, this spec is not robust enough for CI against untrusted repositories because it can probe internal destinations, follow redirects across trust boundaries, leak secrets to logs, and over-concentrate traffic on single hosts.",
  "suggested_next_version": "Tighten the next version around a safe baseline: default-deny private/reserved targets after DNS resolution, re-check policy on every redirect hop, fully specify canonical host matching, redact and sanitize all printed output, add per-host concurrency caps, and move image checking behind an explicit flag.",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
