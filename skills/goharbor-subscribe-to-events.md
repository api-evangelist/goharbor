---
name: goharbor-subscribe-to-events
description: Subscribe to Harbor project events over webhooks, choosing the CloudEvents payload format, and debug a delivery that stopped arriving.
api: goharbor:goharbor-harbor-v2-api
generated: '2026-09-12'
method: generated
source: https://goharbor.io/docs/2.15.0/working-with-projects/project-configuration/configure-webhooks/, openapi/_original/goharbor-harbor-api-v2.0-swagger.yml
operations:
  - GetSupportedEventTypes
  - CreateWebhookPolicyOfProject
  - ListWebhookPoliciesOfProject
  - GetWebhookPolicyOfProject
  - UpdateWebhookPolicyOfProject
  - DeleteWebhookPolicyOfProject
  - ListExecutionsOfWebhookPolicy
  - ListTasksOfWebhookExecution
  - GetLogsOfWebhookTask
---

# Subscribe to Harbor events

Harbor has no streaming API and no AsyncAPI document. Its event surface is a per-project webhook
policy: you register an endpoint, Harbor POSTs to it. Everything below is scoped to one project.

Permissions needed: `create:notification-policy` / `read:notification-policy` /
`list:notification-policy` at project level.

## 1. Ask the instance what it supports

`GetSupportedEventTypes` — `GET /projects/{project_name_or_id}/webhook/events`

Do this rather than hard-coding the list: event types have been added across releases, and this
operation is the running instance's own answer. As of Harbor 2.15 the documented set is
`PUSH_ARTIFACT`, `PULL_ARTIFACT`, `DELETE_ARTIFACT`, `SCANNING_COMPLETED`, `SCANNING_STOPPED`,
`SCANNING_FAILED`, `QUOTA_EXCEED`, `QUOTA_WARNING`, `REPLICATION`, `TAG_RETENTION`.

## 2. Register the policy

`CreateWebhookPolicyOfProject` — `POST /projects/{project_name_or_id}/webhook/policies`

The body carries `name`, `event_types[]`, `enabled`, and `targets[]` where each target has
`type` (`http` or `slack`), `address`, an optional `auth_header`, `skip_cert_verify`, and
`payload_format`.

Set `payload_format: CloudEvents` unless you need the legacy shape. The CloudEvents envelope is
CloudEvents 1.0 — `specversion`, `id`, `source` (`/projects/{id}/webhook/policies/{policy_id}`),
`type` (for example `harbor.artifact.pushed`), `datacontenttype`, `time`, `data` — which means a
generic CloudEvents consumer can read it without a Harbor-specific parser. `Default` is Harbor's
original shape: `type`, `occur_at`, `operator`, `event_data.resources[]`,
`event_data.repository{}`.

Harbor publishes **no signature scheme** for webhook payloads. If you need authenticity, set
`auth_header` on the target and verify it at your endpoint, and do not trust the body alone.
Harbor also publishes no retry or backoff policy — assume at-most-once until you have measured
otherwise.

## 3. Debug a delivery

Three operations, in order of narrowing:

- `ListExecutionsOfWebhookPolicy` — `GET .../webhook/policies/{webhook_policy_id}/executions`
- `ListTasksOfWebhookExecution` — `GET .../executions/{execution_id}/tasks`
- `GetLogsOfWebhookTask` — `GET .../executions/{execution_id}/tasks/{task_id}/log`

The task log is where a failed POST shows its status code and response. Do not use
`LastTrigger` (`GET .../webhook/lasttrigger`) or `ListWebhookJobs` (`GET .../webhook/jobs`) —
both are marked deprecated in Harbor's own contract.

## 4. Change or remove

`UpdateWebhookPolicyOfProject` (PUT) replaces the policy; `DeleteWebhookPolicyOfProject`
(DELETE) removes it. Neither is reversible and neither replays missed events — there is no
event backfill in Harbor. If your consumer was down, reconcile by listing artifacts
(`listArtifacts`) rather than expecting redelivery.
