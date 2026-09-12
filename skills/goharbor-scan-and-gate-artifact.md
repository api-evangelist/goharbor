---
name: goharbor-scan-and-gate-artifact
description: Scan a container artifact in Harbor and decide whether to promote it, using the vulnerability report Harbor attaches to the artifact.
api: goharbor:goharbor-harbor-v2-api
generated: '2026-09-12'
method: generated
source: openapi/_original/goharbor-harbor-api-v2.0-swagger.yml, https://goharbor.io/docs/2.15.0/administration/vulnerability-scanning/
operations:
  - listArtifacts
  - getArtifact
  - scanArtifact
  - stopScanArtifact
  - getReportLog
  - getVulnerabilitiesAddition
  - getSecuritySummary
---

# Scan an artifact and gate promotion

Base URL: `https://{host}/api/v2.0` — `{host}` is the Harbor instance you were given. Harbor is
self-hosted; there is no shared endpoint.

Auth: HTTP Basic. Use a robot account, not a human's password. The permissions this flow needs are
`list:artifact`, `read:artifact`, `read:artifact-addition` and `create:scan` at project level
(see `scopes/goharbor-scopes.yml`). Send `X-Request-Id` on every call and keep the value — Harbor
echoes it on success and on failure, and it is what you quote when something goes wrong.

## 1. Find the artifact

`listArtifacts` — `GET /projects/{project_name}/repositories/{repository_name}/artifacts`

Repository names containing `/` must be URL-encoded (`library/nginx` → `library%2Fnginx`).
Paginate with `page` and `page_size`; read `X-Total-Count` and the `Link` header rather than
guessing when you are done. Narrow with `q` — Harbor's query syntax is `k=v` exact, `k=~v` fuzzy,
`k=[min~max]` range.

Ask for the scan data up front: `with_scan_overview=true`. If the artifact already carries a
recent report you may not need to scan at all.

## 2. Trigger the scan

`scanArtifact` — `POST /projects/{project_name}/repositories/{repository_name}/artifacts/{reference}/scan`

`{reference}` is a tag or a digest; prefer the digest, since tags move. This returns `202 Accepted`
and enqueues a job — it is **not** idempotent. Harbor publishes no idempotency key, so calling it
twice queues two scans. If you need to abandon one, call `stopScanArtifact`
(`POST .../scan/stop`) while it is still queued or running.

## 3. Wait, then read the report

Poll `getArtifact` (`GET .../artifacts/{reference}?with_scan_overview=true`) until the scan
overview reports a finished status. For the detail, call `getVulnerabilitiesAddition` —
`GET .../artifacts/{reference}/additions/vulnerabilities` — sending the
`X-Accept-Vulnerabilities` header for the report MIME type you want.

If the scan failed, `getReportLog` (`GET .../scan/{report_id}/log`) returns the scanner's log.

## 4. Gate

Decide from the severity counts in the report, not from the HTTP status. Harbor can also enforce
this for you server-side: a project can be configured to prevent pulls of artifacts above a CVE
severity threshold, in which case the *pull* fails with
`{"errors":[{"code":"PROJECTPOLICYVIOLATION", ...}]}` rather than the API call you just made.

For an instance-wide view instead of one artifact, call `getSecuritySummary`
(`GET /security/summary`) or `ListVulnerabilities` (`GET /security/vul`).

## Errors

Every failure is `{"errors":[{"code":"...","message":"..."}]}`. The ones you will actually hit:
`404 NOT_FOUND` (wrong project/repository/reference, or an unencoded `/`), `403 FORBIDDEN` (the
robot lacks `create:scan`), `401 UNAUTHORIZED` (bad or expired robot secret — robot accounts
expire, and Harbor never re-issues a secret, only rotates it), `412 PRECONDITION` (a policy check
blocked the operation). Full catalog: `errors/goharbor-problem-types.yml`.
