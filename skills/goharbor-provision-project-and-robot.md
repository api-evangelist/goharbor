---
name: goharbor-provision-project-and-robot
description: Create a Harbor project and issue a least-privilege robot account for a pipeline or an agent, then verify what that account can actually do.
api: goharbor:goharbor-harbor-v2-api
generated: '2026-09-12'
method: generated
source: openapi/_original/goharbor-harbor-api-v2.0-swagger.yml, https://goharbor.io/docs/2.15.0/administration/robot-accounts/
operations:
  - listProjects
  - createProject
  - getProject
  - getProjectSummary
  - CreateRobot
  - ListRobot
  - GetRobotByID
  - RefreshSec
  - DeleteRobot
  - getCurrentUserPermissions
  - listQuotas
---

# Provision a project and a robot account

Base URL: `https://{host}/api/v2.0`. Auth: HTTP Basic as an administrator for the provisioning
calls themselves.

## 1. Check the name is free

`listProjects` — `GET /projects?name=<name>` (or `q=name=<name>`). Harbor has no idempotency key,
so a blind `createProject` on an existing name returns `409 CONFLICT`. Check first; treat a
409 as "already provisioned" only after you have confirmed the existing project is the one you
meant.

## 2. Create the project

`createProject` — `POST /projects`

The body carries `project_name`, `public`, optional `storage_limit` (the project quota, in bytes;
`-1` for unlimited) and a `metadata` block where the policy switches live: `auto_scan`,
`prevent_vul`, `severity`, `enable_content_trust_cosign`, `auto_sbom_generation`. Set these at
creation — turning `auto_scan` on later does not scan what is already there.

Returns `201 Created` with a `Location` header naming the new resource.

## 3. Issue the robot account

`CreateRobot` — `POST /robots`

The body's `permissions[]` is the whole point. Each entry has a `kind` (`project` or `system`), a
`namespace` (the project name, or `/` for system) and an `access[]` list of
`{resource, action}` pairs. Choose them from the published permission reference reproduced in
`scopes/goharbor-scopes.yml` — that file lists, for every permission, the exact API operations it
unlocks, so you can grant precisely what the pipeline calls and nothing else.

Two rules Harbor enforces and one it does not:

- `push` on `repository` **must** be granted together with `pull`. Push alone is rejected.
- A robot account cannot log into the web UI. It is API and registry only.
- Nothing stops you granting `*` — which is why you should not.

Set `duration` (days) or mark it never-expiring. The response contains the **secret, once**.
Harbor does not store it. If it is lost, `RefreshSec` (`PATCH /robots/{robot_id}`) rotates to a new
one; there is no recovery.

## 4. Verify

Authenticate as the robot and call `getCurrentUserPermissions` —
`GET /users/current/permissions`. That returns the effective permission set, which is the only
honest confirmation that step 3 granted what you intended. Then exercise one real read
(`getProject`) before handing the credential to anything.

`getProjectSummary` (`GET /projects/{project_name_or_id}/summary`) and `listQuotas`
(`GET /quotas`) show what the project is actually consuming.

## Teardown

`DeleteRobot` — `DELETE /robots/{robot_id}`. Irreversible; the secret cannot be recovered and the
account cannot be restored. Deleting the project itself is `deleteProject`, and you should call
`getProjectDeletable` (`GET /projects/{project_name_or_id}/_deletable`) first — it tells you
whether the delete will succeed before you try it. There is no undo for either.
