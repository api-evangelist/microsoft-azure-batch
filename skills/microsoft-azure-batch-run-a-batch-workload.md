---
name: microsoft-azure-batch-run-a-batch-workload
description: >-
  Stand up an Azure Batch pool, submit a job, add tasks, watch them to completion and
  collect their output — the end-to-end flow for running parallel work on Azure Batch.
  Use when asked to run a compute job on Azure Batch, submit work to a Batch account,
  or process a set of inputs in parallel on Azure.
api: Microsoft Azure Batch (data plane, api-version 2025-06-01)
base_url: https://{account}.{region}.batch.azure.com
generated: '2026-09-17'
method: generated
source: openapi/_original/microsoft-azure-batch-batch-service-openapi.json
operations:
  - Pools_CreatePool
  - Pools_GetPool
  - Pools_ListPoolNodeCounts
  - Jobs_CreateJob
  - Tasks_CreateTask
  - Tasks_CreateTaskCollection
  - Tasks_ListTasks
  - Jobs_GetJobTaskCounts
  - Tasks_ListTaskFiles
  - Tasks_GetTaskFile
  - Jobs_TerminateJob
  - Pools_DeletePool
---

# Run a workload on Azure Batch

Every operation below is in the provider's own 2025-06-01 contract
(`openapi/_original/microsoft-azure-batch-batch-service-openapi.json`).

## Before you start

- Base URL is per-account: `https://{account}.{region}.batch.azure.com`. There is no
  single global host.
- `api-version=2025-06-01` is a **required query parameter on every call**. Omit it and
  you get `MissingRequiredQueryParameter` (400); send an old one and you get
  `UnsupportedRequestVersion` (400).
- Auth: `Authorization: Bearer <Entra ID token>` for scope
  `https://batch.core.windows.net//.default`. Effective permission comes from your Azure
  RBAC role assignment on the Batch account, not from the scope.
- **You choose the ids.** `poolId`, `jobId` and `taskId` are yours. Pick them
  deterministically — that is the only create-side replay protection Batch gives you.

## 1. Create the pool

`Pools_CreatePool` — `POST /pools`

Supply `id`, `vmSize`, a `virtualMachineConfiguration` (image reference + node agent SKU),
and either `targetDedicatedNodes`/`targetLowPriorityNodes` or an autoscale formula.

Before committing an autoscale formula, rehearse it: `Pools_EvaluatePoolAutoScale`
(`POST /pools/{poolId}/evaluateautoscale`) returns what the formula *would* do without
applying it. It is the only dry-run in this API — use it.

Pool creation returns **201**, but the nodes are not ready. Poll `Pools_GetPool`
(`GET /pools/{poolId}`) until `allocationState` is `steady`, or watch
`Pools_ListPoolNodeCounts` (`GET /nodecounts`) for `idle` nodes.

**If it fails:** `PoolQuotaReached` or a core-quota error means the account's quota is the
wall, not your request. New accounts can have a dedicated-core quota of **zero** in some
regions — check `rate-limits/microsoft-azure-batch-rate-limits.yml` and request an increase
before assuming the call was wrong.

## 2. Create the job

`Jobs_CreateJob` — `POST /jobs`

Supply `id` and `poolInfo` (either `poolId` pointing at the pool from step 1, or an
`autoPoolSpecification` if the pool should live and die with the job).

Re-POSTing an existing `jobId` returns `JobExists` (409) — not a duplicate. Treat that 409
as success-on-retry, not as an error to escalate.

## 3. Add tasks

- One task: `Tasks_CreateTask` — `POST /jobs/{jobId}/tasks`
- Many tasks: `Tasks_CreateTaskCollection` — `POST /jobs/{jobId}/addtaskcollection`.
  **Prefer this.** It is the bulk path and it is what keeps a large submission from
  being thousands of round trips. Its response reports per-task status, so read every
  entry — a partial failure is reported inside a 200.

Each task carries a `commandLine`, optional `resourceFiles` (inputs pulled from Storage)
and `outputFiles` (results pushed back to Storage). Wiring `outputFiles` at submit time is
better than fetching files later, because task output lives only as long as the node does.

## 4. Watch for completion

`Jobs_GetJobTaskCounts` — `GET /jobs/{jobId}/taskcounts` — is the cheap poll. It returns
active/running/completed/succeeded/failed counts for the whole job in one call.

Only fall back to `Tasks_ListTasks` (`GET /jobs/{jobId}/tasks`) when you need per-task
detail, and when you do, trim it: `$select=id,state,executionInfo` and a `$filter` on
`state` keep the response small. `maxresults` sets the page size and `odata.nextLink`
carries you to the next page — follow the link, do not compute an offset.

There are **no webhooks and no events**. Polling is the only mechanism.

## 5. Collect output

`Tasks_ListTaskFiles` — `GET /jobs/{jobId}/tasks/{taskId}/files` — then
`Tasks_GetTaskFile` — `GET /jobs/{jobId}/tasks/{taskId}/files/{filePath}` for
`stdout.txt`, `stderr.txt` or your own outputs.

Completed task data is kept for **seven days by default**, and only while the compute node
it ran on still exists. Collect before you tear the pool down.

## 6. Tear down

- `Jobs_TerminateJob` — `POST /jobs/{jobId}/terminate`. **Not reversible** — a terminated
  job cannot be made active again.
- `Pools_DeletePool` — `DELETE /pools/{poolId}`. **Not reversible** — destroys every node
  and anything on it. Returns 202 and drains asynchronously; there is no documented cancel.

If you only want to stop paying while keeping the pool, resize it to zero
(`Pools_ResizePool`) instead of deleting it. That is reversible; delete is not.

## Error handling

Errors are **not** RFC 9457. The envelope is:

```json
{ "code": "InvalidQueryParameterValue",
  "message": { "lang": "en-us", "value": "..." },
  "values": [ { "key": "QueryParameterName", "value": "state" } ] }
```

Branch on `code`, not on the message. All 125 published codes are in
`errors/microsoft-azure-batch-error-codes.yml`.

Retry `ServerBusy` (503) and `OperationTimedOut` (500) with exponential backoff. There is
no `Retry-After` header and no 429 — do not wait for one.

Send a `client-request-id` GUID with `return-client-request-id: true` on every call so a
failure can be correlated in Batch service logs.
