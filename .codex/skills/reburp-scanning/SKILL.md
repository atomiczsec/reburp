---
name: reburp-scanning
description: Run Burp Scanner and the findings workflow over the reburp API - starting audits and crawls, polling tasks, reading and filtering issues, BChecks and Bambdas, Collaborator payloads and out-of-band interactions, raising custom issues, and generating reports. Use for vulnerability scanning and reporting rather than manual request replay.
---

# Scanning, Collaborator and findings

Read `reburp-api` first. Scanner, Collaborator and AI are **Burp Professional only**: on Community
they answer `403` with a message, so check `edition` in `GET /api/status` before planning a scan.
Full field lists are in `.codex/skills/reburp-api/references/endpoints.md`.

An active audit sends attack traffic. Confirm the target is in the user's engagement scope, and
prefer a passive audit or a scan limited to specific requests over pointing Burp at a whole host.

## Audits

- `POST /api/scanner/audit` - `configuration` is `ACTIVE` or `PASSIVE` (or the full
  `LEGACY_ACTIVE_AUDIT_CHECKS` / `LEGACY_PASSIVE_AUDIT_CHECKS`). Those are the only built-in
  configurations Montoya exposes; named configurations from Burp's UI cannot be selected. An
  unrecognised value is rejected rather than quietly running an active audit.
  Pass `requests` (raw HTTP strings) plus `host`/`port`/`use_https` to scan specific exchanges;
  omitting them starts an empty audit you feed later.
- `POST /api/scanner/audit/from-history` - audit one proxy history entry by `index`. The cheapest
  way to scan exactly what you just captured. Indexes shift as traffic arrives, so re-read history
  immediately before.
- `POST /api/scanner/tasks/{id}/add` - feed another request/response into a running audit.
- `POST /api/scanner/crawl` with `seed_urls` (validate them first with `/api/scanner/crawl/preview`).

Scans are asynchronous. The response gives a short task id; poll `GET /api/scanner/tasks/{id}` for
`status`, `request_count`, `error_count`, `issue_count`, and `/detail` for progress. Poll at a human
pace (every 10-30s) and report progress rather than blocking silently; audits run for minutes to
hours. `DELETE /api/scanner/tasks/{id}` cancels and removes a task - the only way to stop one.
Task ids live in extension memory and are lost if the extension reloads.

## Issues

`GET /api/scanner/issues` pages and filters by `severity`, `confidence`, `host`, `name_contains`
and `prefix`; keep `include_requests=false` until you are drilling into one finding, since the
evidence payloads are large. `GET /api/sitemap/issues` reads issues off the site map instead.
`POST /api/scanner/issues/consolidate` groups issues into a single audit result, and
`POST /api/scanner/issue-definition` shows how Burp will interpret a definition before you use it.

`POST /api/issues` raises your own issue (name, detail, remediation, `base_url`, `severity`,
`confidence`, optional request/response evidence) so a manual finding lands in Burp's issue list
and in reports with everything else - do this instead of only writing the finding into chat.

## Custom checks

`POST /api/scanner/bchecks/import` loads a BCheck script (Pro); `/api/scanner/bambda/import` and
`/api/bambda/import` load Bambdas; `/api/bambda/generate-chain` builds a request chain;
`/api/scanner/bchecks/config` and `/bambda/config` export what is currently loaded. Import is
additive and persists in the user's Burp - say what you loaded.

## Collaborator (out-of-band)

`POST /api/collaborator/generate` returns a `payload`, `interaction_id` and `secret_key`. **Keep
the secret key** - it is the only way to poll, and polling consumes interactions, so each hit comes
back once. Place the payload, then `GET /api/collaborator/interactions/{secretKey}` for hits with
protocol detail (the DNS query and type, the HTTP exchange, the SMTP conversation) - that detail is
the evidence. `GET /api/collaborator/poll/{secretKey}` is the thinner version.
`GET /api/collaborator/server` reports which server the project uses. Absence of a hit is not proof;
retry over a window before concluding anything.

## Reports

`POST /api/scanner/report` writes HTML or XML to a path on Burp's own filesystem;
`POST /api/scanner/report/download` returns the same file base64-encoded, which is what you want
when you are not on the same machine as Burp. Scope a report with `task_id`, or `all_issues: true`.
Reports embed request and response evidence, including any tokens in it - treat them as sensitive.
