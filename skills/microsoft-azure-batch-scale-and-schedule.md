---
name: microsoft-azure-batch-scale-and-schedule
description: >-
  Autoscale an Azure Batch pool safely and run recurring work with job schedules —
  rehearse the formula, apply it, observe it, and roll it back. Use when asked to make a
  Batch pool scale with demand, cut Batch compute cost, or run a Batch job on a recurring
  schedule.
api: Microsoft Azure Batch (data plane, api-version 2025-06-01)
base_url: https://{account}.{region}.batch.azure.com
generated: '2026-09-17'
method: generated
source: openapi/_original/microsoft-azure-batch-batch-service-openapi.json
operations:
  - Pools_EvaluatePoolAutoScale
  - Pools_EnablePoolAutoScale
  - Pools_DisablePoolAutoScale
  - Pools_ResizePool
  - Pools_StopPoolResize
  - Pools_GetPool
  - Pools_ListPoolUsageMetrics
  - JobSchedules_CreateJobSchedule
  - JobSchedules_ListJobSchedules
  - JobSchedules_DisableJobSchedule
  - JobSchedules_EnableJobSchedule
  - JobSchedules_TerminateJobSchedule
  - Jobs_ListJobsFromSchedule
---

# Scale and schedule on Azure Batch

## Autoscale, in the order that cannot hurt you

1. **Rehearse.** `Pools_EvaluatePoolAutoScale` — `POST /pools/{poolId}/evaluateautoscale`.
   Send the formula; get back the values it would produce and any evaluation error,
   **without applying it**. This is the only operation in the whole Batch contract that
   lets you see the consequence before you cause it. Always do this first — an autoscale
   formula is the single parameter in Batch that can spend real money by accident.
2. **Apply.** `Pools_EnablePoolAutoScale` — `POST /pools/{poolId}/enableautoscale`. Takes
   the formula and an `autoScaleEvaluationInterval` (minimum 5 minutes).
3. **Observe.** `Pools_GetPool` — read `autoScaleRun` for the last evaluation's results,
   timestamp and error. `Pools_ListPoolUsageMetrics` (`GET /poolusagemetrics`) gives the
   aggregated core-hours the pool actually consumed.
4. **Roll back.** `Pools_DisablePoolAutoScale` —
   `POST /pools/{poolId}/disableautoscale` — returns the pool to manual target counts.
   Fully reversible; re-enable at any time.

`AutoScalingFormulaSyntaxError` and `InvalidAutoScalingSettings` (both 400) are what a bad
formula returns at step 1. Fix and re-evaluate — never enable a formula you have not
evaluated.

### Manual resize

`Pools_ResizePool` — `POST /pools/{poolId}/resize` — with new target node counts. While
`allocationState` is `resizing`, `Pools_StopPoolResize`
(`POST /pools/{poolId}/stopresize`) cancels it. **That window closes when the resize
completes.** After that, reversing a shrink means a fresh resize that allocates *new*
nodes — the old ones and anything on them are gone.

`Pools_RemoveNodes` is never reversible. Prefer a resize to a targeted node removal
unless you specifically need to evict named nodes.

## Recurring work

`JobSchedules_CreateJobSchedule` — `POST /jobschedules` — takes an `id`, a `schedule`
(`doNotRunUntil`, `doNotRunAfter`, `startWindow`, `recurrenceInterval`) and a
`jobSpecification` that is instantiated as a job on each occurrence.

- `Jobs_ListJobsFromSchedule` — `GET /jobschedules/{jobScheduleId}/jobs` — the jobs the
  schedule has produced.
- `JobSchedules_DisableJobSchedule` / `JobSchedules_EnableJobSchedule` — pause and resume.
  **Reversible**, and the right way to stop a schedule you might want back.
- `JobSchedules_TerminateJobSchedule` / `JobSchedules_DeleteJobSchedule` — **not**
  reversible. Reach for disable first.

## Quotas bound all of this

Scaling stops at the account's quota, not at your formula: dedicated cores, Spot cores,
pools per account, and active jobs + job schedules per account are all capped, and the
defaults for a new account can be as low as zero cores in some regions. The full published
set is in `rate-limits/microsoft-azure-batch-rate-limits.yml`; increases are free but go
through an Azure support request and can take up to two business days.

## Conditional writes

Pool and job-schedule updates accept `If-Match` with the resource's ETag. Use it whenever
you are changing something you read earlier — a mismatch returns `ConditionNotMet` (412)
instead of silently overwriting a change someone else made. There is no `Idempotency-Key`
in this API; conditional requests are the concurrency control it does provide.
