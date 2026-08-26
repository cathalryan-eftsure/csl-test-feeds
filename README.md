# CSL test feeds

Synthetic Consolidated Screening List feeds for dev ingestion testing (BR-2671).
Real public trade.gov CSL data plus clearly-marked synthetic test entities.

## consolidated-cathal-test-service-700000.json

trade.gov CSL snapshot taken 2026-07-27 (25,994 real records), plus one synthetic
entity: `entity_number`/`id` `700000`, name `Cathal's Sanctions Test Service`.
`total` bumped to 25,995.

## Failure-path fixtures (`cases/`)

Cases for exercising `sanctions-data-ingestion`'s download/parse failure paths
(BR-2801), built against what `SanctionsDownloader`, `CslParser` and
`ZipEntryExtractor` actually throw — not just "broken" files.

Point a source's URL config at the raw file to use one, e.g.:

```
CSL_URL=https://raw.githubusercontent.com/cathalryan-eftsure/csl-test-feeds/main/cases/csl-feed-truncated.json
```

| File | Simulates | Exception | Retried? |
|---|---|---|---|
| `csl-feed-truncated.json` | Download cut off mid-stream (invalid JSON syntax) | `SanctionsParseException` | No |
| `csl-feed-error-page.html` | WAF/CDN block page or outage response instead of the feed | `SanctionsParseException` | No |
| `csl-feed-empty-body.json` (0 bytes) | Upstream returns 200 with an empty body | `SanctionsDownloadException` | No |
| `ofac-sdn-corrupt.zip` | Archive found and readable at the entry-listing level, but the entry's deflate payload is corrupted (verified against a real `ZipInputStream` — this is not merely a broken zip) | `SanctionsDownloadException` | No |

`ofac-sdn-corrupt.zip` goes with `OFAC_SDN_URL` or `OFAC_CONS_URL`, not `CSL_URL` —
it's only usable by the two OFAC sources, since they're the only ones that download
a ZIP archive.

Two failure types can't be represented as static files here and need a different
approach:
- **4xx (not retried)**: point a URL at a path that doesn't exist in this repo —
  GitHub raw genuinely returns 404, no fixture file needed.
- **5xx / 429 / timeout (retried)**: needs a live endpoint that returns a specific
  status on demand (e.g. `httpbin.org/status/503`), or the app's own WireMock-based
  `SanctionsDownloaderIT`. GitHub raw only ever serves 200 or 404.
