---
name: qure-chest-xray-triage
description: >-
  Upload a chest or musculoskeletal X-ray DICOM to the Qure.ai Platform API and
  retrieve the qXR/qMSK AI triage result, including the Critical/Routine triage
  status, the narrative report and the generated DICOM SC/SR assets.
api: Qure.ai Platform API
version: '3.1.33'
operations:
  - uploadDicoms
  - fetchResultsv2XRay
source: openapi/qure.ai-platform-api-xray-v2-er-openapi.yml
generated: '2026-08-26'
method: generated
---

# Chest X-ray triage with qXR

Two calls: upload, then poll. X-rays do **not** need a separate compute step —
that is CT only.

## Before you start

You need three values, all issued by `support@qure.ai` and all different per
environment (test / pre-production / production):

- `BASE_URL` — prepended to every path
- `TOKEN` — sent as `Authorization: Token <TOKEN>`
- `SOURCE` — sent as `Source: <SOURCE>`

Every request must also carry a `User-Agent`. The provider documents that
omitting it can cause errors.

Read the `SOPInstanceUID` out of the DICOM file itself before you call anything
(the docs use `pydicom`). Qure issues no ids of its own — DICOM UIDs are the
only identity in this API.

## Step 1 — upload (`uploadDicoms`)

```
POST {BASE_URL}/studies/
Authorization: Token <TOKEN>
Source: <SOURCE>
Content-Type: multipart/form-data
User-Agent: <YOUR_UA>
```

Attach each file under a unique form key (`file_0`, `file_1`, …). `POST /upload`
is an equivalent alias with the same schema, request and response.

Failure modes: `400` means the use case is not implemented or no DICOMs were
sent; `401` means the token is missing or wrong.

## Step 2 — poll for results (`fetchResultsv2XRay`)

```
GET {BASE_URL}/results/v2/{SOPInstanceUID}
Authorization: Token <TOKEN>
Source: <SOURCE>
User-Agent: <YOUR_UA>
```

Poll until you stop getting `202`. Treat the HTTP status as the control flow —
do **not** branch on the `success` field, which the provider's own spec marks as
deprecated and always true when present.

| Status | Meaning | Do |
|---|---|---|
| 200 | Result ready | Read `triage_status` and `result.report` |
| 202 | Still processing | Back off and poll again |
| 206 | Valid X-ray modality, invalid scan | Stop. SC/SR assets are attached and say so |
| 400 | Unsupported modality or invalid scan | Stop. Check the DICOM Specifications page |
| 401 | Bad/missing token | Stop |
| 403 | Bad/missing Source, or the DICOM is outside your workspace | Stop |
| 404 | Not found — may not be uploaded yet | Retry a bounded number of times, then stop |
| 500 / 502 | Qure system failure / upstream down | Back off and retry |

There is no `Retry-After` and no rate-limit header on this API, so choose your
own backoff — start around 5s and grow it.

## Step 3 — read the result

- `triage_status` — `Critical` or `Routine`. `Routine` means the AI detected no
  abnormality; `Critical` means it did. This is the field a triage workflow acts on.
- `result.report.impression` — one-line summary, e.g. "Abnormal study".
- `result.report.findings` — the free-text narrative.
- `result.report.findings_list` — pairs of `report_key` and `findings_text`; use
  the key when you need a stable handle on a specific finding.
- `result.files` — generated DICOM Secondary Capture and Structured Report assets.

## Safety

`triage_status` is a prioritization signal from a regulated medical device, not
a diagnosis. Do not let an agent present it as one, suppress a Critical result,
or auto-close a case on `Routine`. A clinician reads the study.

**There is no undo.** The API publishes no delete, cancel or purge operation, so
an uploaded study cannot be withdrawn through the API. Confirm you have the right
file and the right patient before `POST /studies`.
