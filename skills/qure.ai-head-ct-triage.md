---
name: qure-head-ct-triage
description: >-
  Upload a head CT series to the Qure.ai Platform API, invoke computation, and
  retrieve qER triage results for intracranial hemorrhage, midline shift, mass
  effect and cranial fracture.
api: Qure.ai Platform API
version: '3.1.33'
operations:
  - uploadDicoms
  - initiateComputation
  - fetchResultsqER
source: openapi/qure.ai-platform-api-xray-v2-er-openapi.yml
generated: '2026-08-26'
method: generated
---

# Head CT triage with qER

Three calls, not two. CT adds a **compute** step that X-rays do not have, and
skipping it is the single most common failure — it surfaces as a `409` on the
results call.

CT works at **series** grain: everything is keyed on `SeriesInstanceUID`, not
`SOPInstanceUID`.

## Before you start

`BASE_URL`, `TOKEN` and `SOURCE` are issued by `support@qure.ai` and differ per
environment. Every request carries `Authorization: Token <TOKEN>`,
`Source: <SOURCE>` and a `User-Agent`.

## Step 1 — upload the series (`uploadDicoms`)

```
POST {BASE_URL}/studies/
Content-Type: multipart/form-data
```

Attach every instance in the series under unique form keys. Read the
`SeriesInstanceUID` from any instance in the series.

## Step 2 — invoke computation (`initiateComputation`)

```
GET {BASE_URL}/compute/{SeriesInstanceUID}
```

| Status | Meaning |
|---|---|
| 200 | Queued for computation |
| 304 | Already pushed for computation — safe, treat as success |
| 401 | Bad/missing token |
| 403 | Bad/missing Source |
| 404 | Series not found — the upload has not landed yet |

The `304` is the closest thing this API has to an idempotency guarantee: calling
compute twice on the same series does not double-queue it. There is no
idempotency key on the upload path, however, so do not rely on the same behavior
for step 1.

## Step 3 — poll for results (`fetchResultsqER`)

```
GET {BASE_URL}/results/{SeriesInstanceUID}
```

| Status | Meaning | Do |
|---|---|---|
| 200 | Result ready | Read `triage_status` and `result.report` |
| 202 | Still processing | Back off and poll again |
| 400 | Unsupported modality | Stop |
| 401 | Bad/missing token | Stop |
| 403 | Bad/missing Source | Stop |
| 404 | Series not found | Retry a bounded number of times |
| 409 | **Series not yet invoked for computation** | Go back to step 2 |

A `409` almost always means step 2 was skipped or has not landed. Do not keep
polling through it — call compute.

For chest CT, the flow is identical but the results operation is
`fetchResultsqCT` on the same `GET /results/{SeriesInstanceUID}` path; see
`openapi/qure.ai-platform-api-xray-ct-openapi.yml`.

## Safety

qER is a triage and prioritization device. `triage_status: Critical` means the
model flagged a finding — it raises a study's place in the queue, it does not
close it. Escalate on Critical; never let an agent downgrade or suppress one.

**There is no undo.** No delete, cancel or purge operation is published for an
uploaded series.
