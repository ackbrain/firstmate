# pi-ollama-cloud-usage footer patch

## Problem

The Ollama Cloud usage API at `https://ollama.com/api/usage` changed its response shape around 2026-10-08.
The old shape returned `limits.session.usage`, `limits.weekly.usage`, and `activity.cost`.
The new shape returns a 7-day window with day-granularity buckets:

```json
{
  "range": "7d",
  "scope": "self",
  "granularity": "day",
  "from": "...",
  "until": "...",
  "totals": { "request_count": 1988 },
  "buckets": [
    { "from": "...", "until": "...", "request_count": 2 },
    ...
  ]
}
```

The published `pi-ollama-cloud-usage` extension (v1.0.1) still parses the old shape.
When `usage.session` and `usage.weekly` are undefined and `cost` is missing, the footer falls through to the literal string `Ollama ok`.

## Fix

Forked the upstream repository to `github.com/ackbrain/pi-ollama-cloud-usage`.
Patched `extensions/usage.ts` so it handles both the old limits shape and the new bucket shape:

- `parseUsage` now derives `weeklyRequestCount` from `totals.request_count`.
- It derives `sessionRequestCount` by summing buckets that intersect the last 5 hours.
- Because the API currently offers only day granularity, the current bucket is pro-rated by the fraction of its elapsed time that falls inside the last 5 hours.
- `renderUsageStatus` shows percentages when the old shape is present, otherwise falls back to `Ollama ~N req 5h / M wk` instead of `Ollama ok`.
- The existing `/ollama-usage` report was updated to show the approximated request counts.

Fork commit: `8e7c787` on `main`.
Tests pass: 34/34 across `tests/status.test.ts`, `tests/auth.test.ts`, and `tests/parser.test.ts`.

## Durability

The extension is installed from the fork instead of the npm registry.
`~/.pi/agent/settings.json` now lists:

```text
git:github.com/ackbrain/pi-ollama-cloud-usage
```

instead of `npm:pi-ollama-cloud-usage`.
`pi update --extensions` will refresh from that git source, so the patch survives extension updates.

## Verification

Live probe against `https://ollama.com/api/usage` using the key in `~/.pi/agent/auth.json` returned the new bucket shape.
The patched footer rendered:

```text
Ollama ~134 req 5h / 2125 wk
```

not `Ollama ok`.
