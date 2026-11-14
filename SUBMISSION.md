# Submission

- **Name:**Deepak S
- **Date submitted:**[14/11/2026]
- **Repository link:**[https://github.com/Deadpool1827/pse-support-lab]
- **Total time spent (real time):**[3hrs]

> Replies below are drafts written on the **exercise clock** (Monday 16 November 2026, Asia/Dubai, UTC+4). They were not sent to real customers. Real work time must be recorded separately at the end.

> **Important scope note:** I successfully built and started the lab initially (`make up` reported all nine application/observability containers healthy). Subsequent checks exposed a Grafana frontend packaging issue and Docker Desktop/WSL failures that prevented reliable portal, Grafana dashboard, PostgreSQL and container-log investigation. I therefore distinguish workbook/source-code findings from production-runtime facts. No production import IDs, invoice IDs, ASP responses or database results have been invented. The submission is a working evidence-based draft and must be completed with runtime evidence if the environment can be recovered before the deadline.

- **Items selected for deeper investigation:**
  - **Incident 03 – Al Noor Fuel:** workbook arithmetic reproduced the ASP BR-CO-15 mismatch and the PINT mapping source explains the inconsistent calculation path.
  - **Incident 04 – Sahara Medical:** workbook confirms 40 invoices; source code confirms an unhandled HTTP 429 path and a retry query that excludes `Submitted` invoices. Customer-specific receipt of 429 still requires runtime proof.
  - **Incident 05 – Ghaf Tree Café:** workbook values, the template field guide, the validator (`validate.go:113–119`) and the parser (`sheet.go:167–185`) confirm that slash-formatted date text is not accepted. Runtime portal/database confirmation remains outstanding.
  - Incidents 01 and 02 were also investigated at source/workbook level, but their specific production events could not be confirmed without logs and database/container state.

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Proposed send time (exercise clock):** 09:10, 16 Nov 2026
- **Received:** 09:05
- **First-response deadline:** 10:05 (P2, one hour)
- **Reply (draft only):**

> Hello Rashid,
>
> Thank you for explaining what happened. I understand that repeated uploads have left you unsure which November invoices have actually been sent, and that avoiding duplicates is essential for your month-end close.
>
> I am checking the import records, invoice statuses, status history and ASP references so I can reconcile what was created and what, if anything, was accepted by the ASP. Please do not upload the same workbook again or retry individual invoices until I confirm their current status.
>
> I will update you by 09:40 UAE time with the verified position or, if the records are not yet available, the next action and update time.
>
> Regards,  
> Deepak – Product Support

### Priority and justification

- **Priority:** P2.
- **Why:** Repeated upload failures affect a monthly consolidated file, duplicate-number warnings now appear, and the customer has month-end and downstream customer pressure. The report does not yet prove that all invoices failed to reach the ASP, so P1 is not established. Raise to P1 if reconciliation shows that invoices cannot reach the ASP for this customer.
- **SLA deadlines I am working to:** First response by 10:05; next update promised 09:40; P2 resolution target is eight business hours from receipt (17:05 on 16 Nov, within the stated 08:00–18:00 Dubai business window). This is a target, not a promise that a code fix will be delivered by then.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:15 | Read the customer report and the invoice lifecycle / template guidance. | Customer retried the same workbook three times; the portal later reported existing document numbers. The customer was correctly advised not to retry again until reconciliation. |
| 09:25 | Inspected `samples/oasis-builders-nov-2026.xlsx` with `openpyxl`. | Workbook has 600 data rows grouped into 8 invoice numbers, 75 lines per invoice. Issue dates are `2026-11-15`; the audit found no line-net calculation mismatches. |
| 09:35 | Inspected import orchestration and duplicate validation code. | Import flow sets `Processing`, parses and inserts invoices, validates them, marks the import completed, then queues valid invoices. The duplicate check looks for the same seller/invoice number in another import, excluding invoices whose status is `Rejected`. |
| 09:45 | Inspected HTTP import handler and import status persistence. | The HTTP handler returns an internal error if `Importer.Import` returns an error. Several import processing error paths return without an explicit `FinishImport(..., Failed, ...)`, so an import could remain `Processing`. |
| 10:00 | Attempted to obtain runtime container/database/log evidence. | Docker CLI later crashed with a segmentation fault in Ubuntu; Windows Docker client requests returned HTTP 500. No reliable production-like import records or status history could be retrieved. |

**Dead ends / ruled out:** The initial read-only workbook inspection reported only one row because the sheet dimension metadata was not reliable in read-only mode. Re-running the audit with normal workbook loading exposed the actual 601 × 32 worksheet. The workbook itself does not show invalid dates or line-net mismatches. This does not rule out an importer timeout, infrastructure error or repeated-upload side effect.

### Root cause

- **What happened:** The precise cause of Rashid's first and second stuck uploads is **not confirmed**. Source inspection identifies two mechanisms that are consistent with the symptoms: (1) some importer errors can return before the import is explicitly marked `Failed`, leaving its status as `Processing`; and (2) a subsequent upload with the same invoice numbers can be rejected by AE-BR-09 if earlier non-rejected invoice records already exist under another import.
- **Where (component, `file:line`):** `services/api/internal/importer/importer.go` (processing timeout, transition to `Processing`, early returns, completion path); `services/api/internal/httpapi/httpapi.go` (`createImport` error handling); `services/api/internal/store/store.go` (`ExistingStatus` duplicate query); `services/api/internal/validate/validate.go` (AE-BR-09 check). Verify exact line numbers with `nl -ba` before final submission.
- **Why it happens:** Duplicate validation intentionally checks invoice numbers across imports for the same seller and excludes only `Rejected` invoices. The importer sets `Processing` before multiple later operations; not every error return is followed by `FinishImport(..., Failed, ...)`. These code paths make the reported symptoms possible, but do not prove which path occurred for Oasis.
- **Evidence:** `evidence/workbook-audit.txt` (600 rows / 8 invoice groups / no line-net mismatch); `evidence/inspect_workbooks.py`; `evidence/oasis-investigation.txt` (import flow, HTTP handler, duplicate query and status transaction excerpts); `evidence/source-code-references.txt` (duplicate-detection excerpt). Source paths listed above. Runtime dashboard, application logs, import IDs, database rows and invoice events remain **not collected** because the Docker environment became unstable.
- **Confidence and open questions:** High confidence in the duplicate-check behaviour and the importer error-handling gap from source. Low confidence about the actual first-upload failure and how many of the eight invoices reached the ASP. Need actual import IDs, status counts, `invoice_events`, `asp_message_id`, ASP sender document list and API logs before stating which invoices were sent.

### Customer resolution reply OR escalation note

- **Which one and why:** Customer update / investigation remains with L1. Engineering escalation is appropriate only after runtime evidence identifies a product defect or data repair requirement. Do not claim a confirmed incident cause yet.

> Hello Rashid,
>
> I have confirmed that the workbook contains eight invoice numbers across 600 invoice lines, and I am reconciling each invoice against its import and ASP acknowledgement. The duplicate message means an invoice number already exists in an earlier import; it does not, by itself, tell us whether that invoice was accepted by the ASP.
>
> Please continue to hold off on re-uploading the workbook. I will only recommend a retry after checking the invoice status and ASP reference for each number, so we do not create duplicate submissions. I will update you by 10:30 UAE time with the reconciliation status and any safe next step.
>
> Regards,  
> Deepak – Product Support

### Workaround offered

- No batch-level workaround is safe until the original imports and ASP acknowledgements are reconciled.
- Once runtime access is restored, classify each of the eight invoice numbers as accepted (ASP reference present), rejected by portal/ASP, failed, validated or still submitted. Retry only invoices confirmed eligible by the documented retry path and after checking ASP receipt. Do not resend the entire workbook blindly.

### Prevention ideas

- Ensure every importer error path finalizes the import as `Failed` and records an actionable error.
- Make import processing resilient to client disconnects/timeouts and expose a durable processing state.
- Add idempotency protection for repeated uploads and a clear “existing import / resume reconciliation” workflow.
- Provide customers with per-invoice ASP status and references before enabling another upload.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Proposed send time (exercise clock):** 09:52, 16 Nov 2026 (after the 09:42 alert and Priya's 09:50 forward)
- **Sent to:** Support on-call queue and Priya Nair
- **First-response deadline:** 10:42 (P2, one hour from the alert time)
- **Reply (draft only):**

> Hi Priya,
>
> I have taken ownership of the API restart alert and the reports of intermittent 502 errors and invoices remaining in `Validated`.
>
> I am checking the restart history, API and database logs, submission queue and ASP response metrics to establish the customer impact. Please share any affected customer/import IDs available to your team. In the meantime, please ask affected customers not to resend batches or retry invoices until we verify whether they have already reached the ASP.
>
> I will update you by 10:15 UAE time. If evidence shows that invoices cannot reach the ASP for an affected customer, I will raise the incident to P1 and page engineering on-call.
>
> Regards,  
> Deepak – Product Support

### Priority and justification

- **Priority:** P2, provisional; raise to P1 if evidence confirms that no invoices are reaching the ASP for one or more customers.
- **Why:** The alert reports three API restarts in 15 minutes, and Priya reports multiple customers receiving intermittent 502 responses and invoices stalled in `Validated`. This is major degradation across customers. The alert's `critical` label is not, on its own, enough to prove the P1 condition in the SLA matrix.
- **SLA deadlines I am working to:** First response by 10:42; next update promised 10:15; P2 resolution target eight business hours from 09:42 (17:42 on 16 Nov). P2 updates every 2–4 hours thereafter. Escalation does not pause the customer update clock.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:52 | Correlated the alert with Priya's customer-impact report. | Three restarts in 15 minutes; intermittent 502s; multiple customers say invoices have remained `Validated` since about 09:30. Marina Bay Hospitality has a go-live on 18 Nov. |
| 10:00 | Inspected `services/api/main.go`. | API exits with status 1 if database configuration fails, if PostgreSQL cannot be reached after 30 attempts, or if the HTTP server stops unexpectedly. Four submission workers are started. |
| 10:05 | Inspected `docker-compose.yml`. | API container has `mem_limit: 128m`, `memswap_limit: 128m`, a health check and `restart: unless-stopped`. This makes memory pressure a hypothesis, not a proven cause. |
| 10:10 | Inspected the database pool initialization. | PostgreSQL pool maximum is 10 connections. This does not show that the pool was exhausted during the incident. |
| 10:20 | Attempted `docker inspect` and `docker compose logs` for runtime state and timestamps. | Docker CLI crashed with segmentation fault in Ubuntu. In Windows PowerShell, Docker Engine calls returned HTTP 500. Therefore OOMKilled, exit code, restart timestamps and API/PostgreSQL log evidence could not be retrieved. |

**Dead ends / ruled out:** Local Docker Desktop/containerd failures are separate from the fictional production API incident and are not evidence of its root cause. A 128 MB memory limit is only a lead; no OOM event was observed. A failed health check alone would not establish why the process restarted.

### Root cause

- **What happened:** Three API restarts and customer-facing 502 errors are reported, but the production root cause is **unknown** from the evidence collected so far.
- **Where (component, `file:line`):** API startup/exit paths in `services/api/main.go`; API container memory and restart policy in `docker-compose.yml` (API service section, approximately lines 43–63); database pool in `services/api/internal/store/store.go` (`Open`).
- **Why it happens:** Source shows explicit process-exit paths on database startup failure and unexpected HTTP server termination. The Compose configuration has a 128 MB hard memory limit and automatic restart policy. Any of these could contribute, but no runtime evidence differentiates them.
- **Evidence:** `evidence/api-restart-investigation.txt` and `evidence/api-exit-paths.txt`. Inbox alert and Priya report are in `inbox/02-alert-api-restarted-marina-bay.md`. Missing: `docker inspect` state (`OOMKilled`, exit code, restart count), timestamped API/PostgreSQL logs, Grafana panels, CPU/memory metrics and submission queue time series.
- **Confidence and open questions:** High confidence in the configuration and exit paths; low confidence in the incident cause. Need restart timestamps and exit reason, memory working set/OOM counter, PostgreSQL availability, API error logs and whether any customers had zero ASP deliveries. Do not state “OOM” as root cause without this proof.

### Customer resolution reply OR escalation note

- **Which one and why:** Escalate the production investigation to L3/on-call because multiple customers are reporting 502s and the API has restarted repeatedly. L1 retains ownership of Priya's and affected customers' updates. P1 paging is conditional on confirming the P1 impact threshold; do not delay escalation of the P2 production defect investigation.

> Hi Priya,
>
> The alert confirms three API restarts in the 15-minute window, and your report indicates intermittent upload failures and invoices waiting in `Validated`. I have raised this for engineering investigation and am checking the customer and ASP impact. I cannot yet confirm whether the restarts were caused by memory pressure or database connectivity.
>
> Please keep affected customers from resending batches until their invoice/ASP status is reconciled. I will update you by 11:00 UAE time, including whether the incident needs to be raised to P1. I will also provide a separate go-live readiness update for Marina Bay once service stability and submission recovery are verified.
>
> Regards,  
> Deepak – Product Support

**Engineering escalation summary:**

- **Priority:** P2 provisional; reclassify P1 if any affected customer has no invoices reaching the ASP.
- **Customer / seller:** Multiple customers reported by Customer Success; exact seller IDs and affected imports not yet retrieved.
- **Reported by / at:** Grafana alert at 09:42; Priya Nair forward at 09:50, Monday 16 Nov 2026, Dubai time.
- **Affected:** Multiple customers; specific import IDs and invoice IDs unknown pending runtime access.
- **Symptoms:** Intermittent portal 502 responses; invoices staying `Validated`; API restarted three times in 15 minutes.
- **Evidence:** Alert payload in `inbox/02-alert-api-restarted-marina-bay.md`; API exit paths and Compose resource limits in `evidence/api-restart-investigation.txt` and `evidence/api-exit-paths.txt`. Runtime logs, OOM state, Grafana metrics and database query results are pending because the Docker engine is returning errors.
- **Steps to reproduce:** Not safely reproducible with the local Docker engine currently unstable. Obtain affected import IDs and correlate API restart times with logs and queue metrics.
- **Tried / ruled out:** Inspected startup paths, DB pool and Compose resource settings. Did not treat local Docker crash as production root cause.
- **Suspected cause:** Unknown. Check OOMKilled / exit code, DB connectivity and server termination first.
- **Workaround:** Do not re-upload or retry until ASP delivery status is checked. Keep customers updated.
- **Ask:** Identify restart exit reason, restore API stability, inspect any `Validated` backlog and confirm safe retry/reconciliation procedure. Provide an engineering update before 11:00 Dubai time.
- **Next customer update due:** 11:00 Dubai time (or earlier if impact changes).

### Workaround offered

- Ask customers to avoid duplicate upload/retry actions while the API and submission state are being reconciled.
- Once runtime access is restored, identify invoices in `Validated` for more than 15 minutes and use the documented retry path only after confirming they were not already accepted by the ASP.
- Confirm Marina Bay readiness only after stable API health and successful end-to-end submission checks.

### Prevention ideas

- Alert on restart count, OOM kills, API 5xx rate, queue depth and age of oldest `Validated` invoice.
- Review API memory limit using measured working-set data; do not raise it without profiling.
- Add graceful handling and observability for DB startup failures and server shutdown causes.
- Add load tests for large imports and queue bursts; provide a go-live readiness checklist and health evidence for new customers.

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Proposed send time (exercise clock):** 10:20, 16 Nov 2026
- **Received:** 10:15
- **First-response deadline:** 11:15 (P2, one hour)
- **Reply (draft only):**

> Dear Fatima,
>
> Thank you for sharing the workbook and the BR-CO-15 examples. I understand that your audit team needs a clear explanation of why only some invoices were rejected.
>
> I am comparing the workbook values with the electronic invoice totals generated by the portal and the ASP's validation rule. Please retain the original file and avoid changing correct quantities, prices or line amounts while I complete the reconciliation.
>
> I will provide a written update by 11:15 UAE time.
>
> Kind regards,  
> Deepak – Product Support

### Priority and justification

- **Priority:** P2.
- **Why:** Several invoices in one customer batch are rejected by the ASP, and the customer is responding to an external VAT audit. This is a subset of invoices and not evidence that all invoice submissions are unavailable.
- **SLA deadlines I am working to:** First response by 11:15; next update promised 11:15; P2 resolution target is eight business hours from 10:15, which crosses the 18:00 close. With the clock paused outside business hours, the target falls at 08:15 on 17 Nov. Updates every 2–4 business hours. This is an SLA target, not a guaranteed engineering fix time.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 10:25 | Read the template field guide and ASP BR-CO-15 definition. | ASP requires invoice total with VAT = invoice total without VAT + total VAT. The portal calculates totals; totals are not supplied directly in the workbook. |
| 10:35 | Audited `samples/al-noor-fuel-nov-2026.xlsx`. | 13 data rows grouped into 6 invoices. All line-net amounts match quantity × unit price within the stated tolerance. Three invoices show a reproduced inclusive-total mismatch. |
| 10:45 | Inspected `services/api/internal/pint/pint.go`. | VAT is rounded per line from the workbook line-net amount; the total gross is independently accumulated from quantity × unit price × (1 + VAT rate), then rounded. Net and VAT are therefore not necessarily derived from the same monetary base as gross. |
| 10:55 | Inspected `services/asp-mock/main.go` validation. | ASP compares `cents(TaxInclusiveAmount)` with `cents(TaxExclusiveAmount) + cents(TaxAmount)` and emits BR-CO-15 when they differ. |
| 11:05 | Reproduced the calculation for all six invoice groups using the workbook values and the source algorithm. | ANF-2611-0042 fails by AED 0.01; ANF-2611-0044 fails by AED 0.02; ANF-2611-0046 fails by AED 0.01. The other three pass. |

**Dead ends / ruled out:** The workbook's line-net amount calculations were consistent with quantity × net price after two-decimal rounding. The finding is not that the customer typed an incorrect invoice total: the portal computes invoice totals itself. The first rough worksheet analysis compared category aggregate VAT and was superseded by the line-level implementation reproduction, which matches the Go code's `VATAmount` calculation.

### Root cause

- **What happened:** The PINT AE mapping calculates invoice net and VAT from the rounded workbook line-net amounts / rounded line VAT, but calculates invoice gross separately from raw quantity × unit price and VAT rate. These separate bases can produce a one- or two-cent difference. The ASP then correctly rejects those payloads under BR-CO-15.
- **Where (component, `file:line`):** `services/api/internal/pint/pint.go`: `round2` (around line 81), line `VATAmount` mapping (around line 147), and `computeTotals` (around lines 156–185). ASP check is in `services/asp-mock/main.go` around line 227.
- **Why it happens:** The code independently accumulates `net`, rounded per-line `vat`, and unrounded `gross`, then rounds each aggregate. The gross is not calculated from the final rounded exclusive amount plus final rounded VAT amount. This makes the PINT totals internally inconsistent at cent precision.
- **Evidence:** `evidence/workbook-audit.txt`; `evidence/al-noor-line-details.txt`; `evidence/inspect_workbooks.py`; `evidence/source-code-references.txt` (PINT calculation and ASP validation excerpts); source files `services/api/internal/pint/pint.go` and `services/asp-mock/main.go`. Reproduced values:

  | Invoice | Exclusive (AED) | VAT (AED) | Generated inclusive (AED) | Expected exclusive + VAT (AED) | Difference |
  |---|---:|---:|---:|---:|---:|
  | ANF-2611-0042 | 14,307.08 | 715.35 | 15,022.42 | 15,022.43 | -0.01 |
  | ANF-2611-0044 | 11,254.84 | 562.75 | 11,817.57 | 11,817.59 | -0.02 |
  | ANF-2611-0046 | 5,343.80 | 267.20 | 5,610.99 | 5,611.00 | -0.01 |

  Runtime payload capture, production import IDs and DB rows were not available because Docker runtime access failed. The reproduction applies the source calculation to the supplied workbook; it is not represented as a live portal transaction.
- **Confidence and open questions:** High confidence that the mapping logic can produce BR-CO-15 failures and that these three supplied invoices reproduce the defect. Confirm against a live payload once the lab runs again. Engineering should check all VAT categories / multi-line cases and use decimal or integer-cents arithmetic consistently.

### Customer resolution reply OR escalation note

- **Which one and why:** Escalate to L3 Engineering because the defect is in product payload calculation and the customer should not alter correct financial values to work around it. L1 remains responsible for Fatima's audit-facing updates.

**Engineering escalation – BR-CO-15 totals inconsistent after PINT mapping**

- **Priority:** P2 – three of six invoices in the supplied workbook reproduce an ASP rejection, with external audit impact.
- **Customer / seller:** Al Noor Fuel & Lubricants LLC, Abu Dhabi; retrieve seller database ID/TRN from the live lab before final escalation.
- **Reported by / at:** Fatima Khoury, customer email, 10:15 Dubai time, 16 Nov 2026.
- **Affected:** Sample workbook `al-noor-fuel-nov-2026.xlsx`; invoices ANF-2611-0042, ANF-2611-0044 and ANF-2611-0046 in the reproducible calculation. Production import/invoice UUIDs not retrieved.
- **Symptoms:** ASP rejects invoices with BR-CO-15 while other invoices in the same workbook are accepted.
- **Evidence:** Workbook and reproducible arithmetic in `evidence/al-noor-line-details.txt`; mapping in `services/api/internal/pint/pint.go`; ASP comparison in `services/asp-mock/main.go`. Live ASP response bodies, logs, dashboard and DB query results remain to be collected.
- **Steps to reproduce:** Use the supplied Al Noor workbook; group rows by invoice number; map the invoice using the current PINT `Map` / `computeTotals`; compare inclusive cents with exclusive cents plus VAT cents as the ASP does. The three invoice numbers above fail.
- **Tried / ruled out:** Checked line-net arithmetic; it is consistent with quantity × unit price rounded to two decimals. Customer-supplied totals are not the issue because portal computes document totals.
- **Suspected cause (high confidence):** `computeTotals` calculates gross independently from raw quantity × unit price while net and VAT use rounded line amounts. Totals can differ by one or two cents.
- **Workaround in place:** No safe customer-side correction. Do not advise changing correct quantities/prices or manually adjusting invoice totals.
- **Timeline:** 10:15 received; 10:20 acknowledgement draft; 10:35 workbook audit; 10:45 code review; 11:05 reproduction.
- **Ask:** Correct total calculation using one consistent monetary basis (prefer decimal/integer cents); add regression tests for fractional unit prices and multi-line invoices; confirm corrected PINT payload passes BR-CO-15 before reprocessing affected records.
- **Next customer update due:** 11:15 UAE time (then every 2–4 business hours).

**Customer update draft:**

> Dear Fatima,
>
> I reproduced the issue using your workbook and our invoice-calculation code. Three of the six invoice numbers produce a one- or two-fils-equivalent AED rounding difference between the generated inclusive total and the exclusive total plus VAT. The line quantities, prices and line amounts in the workbook are consistent; the inconsistency is introduced when our system calculates the electronic invoice totals.
>
> I have escalated the calculation defect to engineering. Please do not alter correct source values or re-upload these invoices yet. I will confirm the correction and a safe reprocessing plan by 13:15 UAE time.
>
> Kind regards,  
> Deepak – Product Support

### Workaround offered

- No safe customer-side workaround. Preserve the original workbook and do not manually manipulate correct figures to satisfy the validation rule.
- After engineering supplies a corrected calculation, regenerate/verify the payload and confirm the ASP outcome before asking the customer to reissue or retry anything.

### Prevention ideas

- Calculate all monetary totals using decimal or integer-cents arithmetic with a documented rounding policy.
- Derive the inclusive total from the same final exclusive and VAT totals that are serialized into the payload.
- Add unit/integration tests for fractional unit prices, multiple invoice lines, category subtotals and one-cent boundary cases.
- Add a pre-submission invariant check that blocks a PINT payload if inclusive cents do not equal exclusive cents plus VAT cents.

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Proposed send time (exercise clock):** 11:35, 16 Nov 2026
- **Received:** 11:30
- **First-response deadline:** 12:30 (P2, one hour)
- **Reply (draft only):**

> Hi Daniel,
>
> Thank you for the batch details. I understand that the connector received HTTP 201, the portal shows the invoices as `Submitted`, and Emirates Palm Hospital has received only some of the 40 invoices.
>
> I am checking the invoice status history and ASP acknowledgements to separate invoices accepted by the ASP from invoices that are still awaiting confirmation. Please do not resend the batch from your connector while we reconcile it; that could create duplicate submissions.
>
> I will update you by 12:30 UAE time with the verified position and a safe next step.
>
> Regards,  
> Deepak – Product Support

### Priority and justification

- **Priority:** P2.
- **Why:** The batch contains 40 invoices, the buyer received only some, the ASP reference is missing for many, and the hospital payment run is approaching. This is a subset/major degradation and duplicate delivery risk, not yet proof that all customer invoice traffic is down.
- **SLA deadlines I am working to:** First response by 12:30; next update promised 12:30; P2 eight business-hour resolution clock from 11:30 has 6.5 business hours remaining until 18:00 and resumes with 1.5 hours at 08:00 on 17 Nov, giving a target of 09:30 on 17 Nov. Updates every 2–4 business hours.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 11:40 | Audited `samples/sahara-medical-batch-1116.xlsx`. | Workbook contains 60 data rows grouped into exactly 40 invoice numbers; 20 invoices have two lines and 20 have one. This matches the partner's 40-invoice batch report. |
| 11:50 | Inspected ASP client `Send` implementation. | It reads the ASP response and returns the HTTP status/body to the submission worker; HTTP 429 is returned as a response, not a transport error. |
| 12:00 | Inspected submission worker and ASP mock rate limiter. | Worker changes invoice to `Submitted` before sending. ASP rate limiter can return 429 with `Retry-After` and returns before storing a received document. |
| 12:10 | Inspected worker response switch and `Retryable` query. | Switch handles 200, 422 and selected 5xx codes, but not 429 and has no default branch. `Retryable` selects only `Validated` and `Failed`, not `Submitted`. |
| 12:20 | Attempted runtime verification using container logs/database. | Docker CLI crashes in Ubuntu and Windows Engine API returns HTTP 500. Cannot confirm that Sahara specifically received 429 or count affected invoices. |

**Dead ends / ruled out:** Partner's HTTP 201 confirms acceptance of the import request by the Invoicely API; it does not prove the ASP accepted all 40 invoices. Missing ASP references alone do not prove 429, but the source-code handling gap is consistent with this symptom. No connector resend was performed.

### Root cause

- **What happened:** Source code confirms a defect path: if the ASP responds HTTP 429, the submission worker has no case for that status and no default case, so it leaves the invoice in `Submitted`. The ASP rate limiter returns before it stores the document, so no ASP message ID is created. The normal import retry query only selects `Validated` and `Failed`, so these stuck `Submitted` invoices are not included in that retry operation.
- **Where (component, `file:line`):** `services/api/internal/submit/submit.go` (`submit` transitions to `Submitted` before request; response switch around lines 128–180); `services/asp-mock/main.go` (sender burst/rate constants around lines 36–37; limiter around lines 122–150; HTTP 429 response around lines 268–281); `services/api/internal/store/store.go` (`Retryable` around lines 368–380).
- **Why it happens:** HTTP 429 is a valid rate-limit response, not a network error. The ASP includes `Retry-After`, but the API worker does not process that header, transition the invoice to a retryable state, schedule a retry or record a distinct throttled outcome. It exits the switch after the unhandled status and the invoice remains `Submitted`.
- **Evidence:** `evidence/workbook-audit.txt` confirms 40 invoices / 60 rows; `services/api/internal/submit/submit.go` client and worker code; `services/asp-mock/main.go` rate limit and 429 response; `services/api/internal/store/store.go` retryable query. `evidence/sahara-rate-limit-code.txt`, `evidence/sahara-root-cause.txt` and `evidence/source-code-references.txt` (submission response handling and retry eligibility excerpts). Runtime API/ASP logs, 429 count, invoice status/ASP IDs and ASP received-document list are still missing.
- **Confidence and open questions:** High confidence that the code has an unhandled 429 path which can leave invoices `Submitted` without ASP references and outside normal retry selection. Medium confidence that this explains Sahara's particular symptom; actual 429 responses and affected count must be confirmed from runtime evidence.

### Customer resolution reply OR escalation note

- **Which one and why:** Escalate the code-handling defect to L3 because it requires a code change and controlled data/status recovery. L1 continues owning Daniel and Sahara Medical's communications.

**Engineering escalation – Unhandled ASP HTTP 429 leaves invoices Submitted**

- **Priority:** P2 – potentially affects a subset of a 40-invoice batch and a hospital payment run.
- **Customer / seller:** Sahara Medical Supplies FZE; retrieve the seller database ID and Peppol ID from the live portal/database before final escalation.
- **Reported by / at:** Daniel Okafor, Bluepeak ERP Integrations; partner email at 11:30 UAE time, 16 Nov 2026.
- **Affected:** Batch file `sahara-medical-batch-1116.xlsx`, 40 invoice numbers / 60 lines. Production import UUID and affected invoice UUIDs not yet retrieved.
- **Symptoms:** Portal shows `Submitted`; ASP reference blank for many invoices; buyer received only some. Connector got HTTP 201 from import endpoint.
- **Evidence:** Workbook audit in `evidence/workbook-audit.txt`; worker and ASP source code described above. Need live ASP status/list, 429 response logs, DB rows and status history.
- **Steps to reproduce:** Submit more documents than the per-sender burst/rate allows; ASP mock returns HTTP 429 plus `Retry-After`. Observe worker has no 429/default case after setting `Submitted`; verify invoice remains `Submitted`, with no `asp_message_id`; inspect `Retryable` query and confirm it omits `Submitted`.
- **What already tried / ruled out:** Confirmed batch has 40 invoices. Did not resend. Import HTTP 201 is not equivalent to per-invoice ASP acceptance.
- **Suspected cause (high code confidence; customer runtime confirmation pending):** Unhandled HTTP 429 in submission worker; retry selection excludes the stuck state.
- **Workaround in place:** No resend. Reconcile each invoice against ASP `GET /v1/documents?sender=<seller-peppol-id>` and the invoice database before any recovery action.
- **Timeline:** 11:30 received; 11:35 acknowledgement; 11:40 workbook audit; 11:50–12:10 source inspection.
- **Ask:** Confirm whether these invoices got HTTP 429; implement handling that records a distinct throttled/retryable state, honours `Retry-After` with backoff/jitter, and makes retries durable/idempotent. Provide an approved method to reconcile and safely re-submit invoices that the ASP did not receive. Do not blindly reset all `Submitted` records.
- **Next customer update due:** 12:30 UAE time, then every 2–4 business hours.

**Customer update draft:**

> Hi Daniel,
>
> I found a submission-handling gap in our code for the ASP's HTTP 429 rate-limit response. That response means the ASP has not received the document, and our current worker does not move that invoice into the normal retryable state. This is consistent with a `Submitted` invoice having no ASP reference. I am still checking whether this exact response occurred for Sahara's batch and how many invoices are affected.
>
> Please continue to hold the connector batch. We will reconcile the sender's ASP document list and our invoice records before authorising any resend. I will update you by 14:30 UAE time.
>
> Regards,  
> Deepak – Product Support

### Workaround offered

- Do not resend the 40-invoice batch.
- Once runtime is available, reconcile the seller's ASP document list with all 40 invoice numbers and the local invoice table / `invoice_events` / `asp_message_id`.
- Only invoices positively confirmed as not received by the ASP may be considered for controlled recovery. A `Submitted` status by itself is not sufficient proof of receipt or non-receipt.

### Prevention ideas

- Explicitly handle HTTP 429 and parse `Retry-After`; schedule backoff with jitter.
- Use a durable, idempotent submission queue and a distinct `Throttled` / retry-wait state or a safe retry transition.
- Add a default response branch so every unrecognised HTTP status results in an explicit state and event.
- Add metrics for 429/throttled submissions and alerts on age/count of `Submitted` invoices without an ASP message ID.
- Add tests for burst submissions, partial batch success and safe retries.

---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Proposed send time (exercise clock):** 12:15, 16 Nov 2026
- **Received:** 12:10
- **First-response deadline:** 16:10 (P3, four hours)
- **Reply (draft only):**

> Hello Hessa,
>
> Thank you for contacting us. I understand that the invoices are needed by your customers and that `13/11/2026` is a valid date in the day/month/year format you use.
>
> I checked the workbook format against our UAE invoice template. The issue-date column must be supplied as `YYYY-MM-DD` text or as a genuine Excel date cell; text such as `13/11/2026` is not accepted because the system must not guess which date convention was intended.
>
> I will send you a simple correction example by 13:00 UAE time. You do not need to change the quantities, prices or other invoice information for this date-format error.
>
> Regards,  
> Deepak – Product Support

### Priority and justification

- **Priority:** P3.
- **Why:** One customer is affected, and the error is a specific documented date-format validation rule. The file can be corrected and re-uploaded; there is no evidence of a platform-wide outage.
- **SLA deadlines I am working to:** First response by 16:10; next update promised 13:00; P3 resolution target is three days under the SLA matrix (business-hour clock applies; confirm internal interpretation of the “days” unit). P3 updates daily after the immediate correction guidance.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 12:20 | Inspected `samples/ghaf-cafe-nov-2026.xlsx`. | Eight data rows form four invoice numbers. Issue-date cells are text values: `13/11/2026` (four rows), `14/11/2026` (three rows), and `15/11/2026` (one row). |
| 12:25 | Compared values with `docs/template-field-guide.md`. | Dates must be `YYYY-MM-DD` text or genuine Excel date cells. Text `DD/MM/YYYY` / `MM/DD/YYYY` forms are explicitly rejected with AE-BR-04 to avoid ambiguity. |
| 12:30 | Checked invoice lifecycle documentation. | Portal-rejected invoices were not sent to the ASP. Rejected invoice numbers do not become consumed; corrected invoices may be uploaded with the same invoice number, provided no other non-rejected invoice with that number exists. |
| 12:35 | Attempted portal/database confirmation. | Runtime verification could not be completed because the Docker engine/CLI was unstable. The workbook and documented validation rule independently explain the reported error text. |

**Dead ends / ruled out:** The date `13/11/2026` is a valid calendar date when interpreted as day/month/year, but the issue is its text representation, not the calendar validity. Do not infer that every locale-specific slash date is unambiguous. The correct approach is to convert the source text explicitly rather than ask the portal to guess.

### Root cause

- **What happened:** The workbook stores issue dates as text in `DD/MM/YYYY` format. The portal template requires `YYYY-MM-DD` text or genuine Excel date cells. The portal's AE-BR-04 rule rejects this format by design; this is not evidence that the date itself is invalid.
- **Where (component, `file:line`):** `docs/template-field-guide.md` → Dates section and AE-BR-04 validation rule; `services/api/internal/validate/validate.go:113–119` calls `ParseDate`; `services/api/internal/sheet/sheet.go:167–185` implements `ParseDate` and accepts ISO `YYYY-MM-DD` text or Excel serial numbers, not slash-formatted text.
- **Why it happens:** The uploaded values are text strings such as `13/11/2026`. The application deliberately accepts ISO `YYYY-MM-DD` text or proper Excel date cells and rejects ambiguous slash-separated text instead of guessing regional date order.
- **Evidence:** `evidence/workbook-audit.txt` (eight date values, all non-standard); `evidence/inspect_workbooks.py`; `evidence/source-code-references.txt` (date validator and `ParseDate` excerpts); `docs/template-field-guide.md` Dates / AE-BR-04; `docs/status-lifecycle.md` rejected invoice and invoice-number behaviour. Portal validation result and DB invoice status are not available due Docker instability.
- **Confidence and open questions:** High confidence that the supplied workbook format conflicts with the documented template and matches the customer's reported error. Need portal/database confirmation that all four invoice groups were rejected by portal and none were submitted to ASP.

### Customer resolution reply OR escalation note

- **Which one and why:** Customer resolution guidance; no engineering escalation is required because the product is behaving according to the published template rule.

> Hello Hessa,
>
> I checked the file and found that the issue dates are stored as text such as `13/11/2026`. Our invoice template accepts dates in `YYYY-MM-DD` format (for example, `2026-11-13`) or genuine Excel date cells. The portal rejects slash-separated text because it cannot safely assume whether the first number means a day or a month.
>
> In Excel, please convert the issue-date and payment-due-date columns to real date cells and format them as `yyyy-mm-dd`. If the values were entered as text, use a helper column to convert day/month/year explicitly, then paste the results as values. Please check a few dates before uploading the corrected workbook.
>
> The portal-rejected invoices were not sent to the ASP. A corrected upload can use the same invoice numbers once the previous records are confirmed as portal-rejected. I will update you by 13:00 UAE time if you need help checking the corrected file.
>
> Regards,  
> Deepak – Product Support

### Workaround offered

- In Excel, convert the issue-date and payment-due-date cells to genuine dates and apply custom display format `yyyy-mm-dd`.
- If the entries are text, use a helper formula that explicitly interprets the first two characters as day, the next two as month and the last four as year, following the field guide. For example, with the source text in `A2`: `=TEXT(DATE(RIGHT(A2,4),MID(A2,4,2),LEFT(A2,2)),"yyyy-mm-dd")`. Paste the converted results as values and verify before upload.
- Confirm prior invoice statuses are portal `Rejected` before reusing the same numbers. Do not reuse numbers that exist in another non-rejected status.

### Prevention ideas

- Provide a date-format example directly beside the upload field and in the workbook template.
- Add a pre-upload date-format check with a friendly example and a link to conversion instructions.
- Consider a safe, explicit locale selector or date conversion preview in the portal, while never silently guessing date order for tax invoices.

---

## Overall

### Prioritisation

I acknowledged all five items in timestamp order and assigned priorities based on impact, not solely on the alert severity label:

1. **Ticket 01 – Oasis Builders (P2):** month-end consolidated upload, repeated attempts and duplicate-number warnings.
2. **Ticket 02 – API restarts (P2 provisional):** multiple customers, intermittent 502 errors and `Validated` backlog; promote to P1 if no invoices can reach the ASP for an affected customer.
3. **Ticket 03 – Al Noor Fuel (P2):** several invoice rejections and external audit impact.
4. **Ticket 04 – Sahara Medical (P2):** partial delivery of a 40-invoice batch and payment-run urgency.
5. **Ticket 05 – Ghaf Tree Café (P3):** one customer with a documented, correctable date-format issue.

After acknowledging the queue, I investigated the workbook and source-code paths with reproducible evidence. I would work the production API degradation and Oasis month-end impact first operationally, while progressing the reproducible Al Noor and Sahara code findings. The Docker/Grafana failure prevented a complete live-system comparison, so I did not claim that the production causes for Oasis or the API alert were proven.

### What I would do differently

- Establish and save a baseline (clean sample upload, portal status, dashboard and DB counts) before replaying customer traffic.
- Capture import IDs and invoice IDs immediately for every reproduction, then correlate portal status, `invoice_events`, API logs, ASP responses and the ASP document list.
- Use a durable retry/idempotency check before recommending any re-upload or retry.
- Verify workbook inspection tooling against a known row count. My first `openpyxl` pass used read-only mode and returned a misleading 1 × 1 dimension; I changed the inspection to normal workbook loading and independently checked the worksheet dimensions.
- Avoid inferring an incident cause from configuration alone. A 128 MB memory limit is a hypothesis until supported by OOMKilled state, exit code and timestamps.
- If a local dependency fails, record the exact error and continue safely with offline source/workbook analysis rather than changing data or resetting the environment.

### What I learned / what I would need to learn

- **Invoice status lifecycle:** distinguished portal rejection, ASP rejection, failed delivery and `Submitted` without an ASP reference; reviewed `docs/status-lifecycle.md` and the `invoice_events` schema.
- **PINT AE total validation:** followed the portal mapping and ASP BR-CO-15 check; reproduced the inconsistent rounding path using the supplied workbook and source calculation.
- **HTTP rate limiting:** learned that HTTP 429 is a response, not a transport error, and that `Retry-After` must be honoured. The current worker's unhandled status can leave an invoice in a misleading state.
- **PostgreSQL evidence:** identified how `imports`, `invoices` and `invoice_events` support reconciliation; further practice needed with joins, status counts and transaction inspection against a running database.
- **Observability:** learned where Prometheus metrics and Loki logs are exposed and which signals matter (restart counts, OOM kills, request 5xx, queue depth, status age and ASP response codes). I could not complete live dashboard queries because the Grafana HTML templates in the supplied image were zero bytes and Docker later became unstable.
- **Tools:** used Git, WSL 2, Docker Compose, curl, `openpyxl`, Go source inspection and shell tools. I would study Docker/containerd diagnostics, Grafana/Loki queries, PromQL and Go error/state handling further.

### Time log (real time)

> Fill in actual local time (Asia/Kolkata) from your own work sessions. Do not use the exercise clock here and do not estimate time that you did not work.

| Date | From–to (Asia/Kolkata) | What I worked on |
|---|---|---|
| 14 Nov 2026 | [12:00 PM – 3:00 PM] | WSL 2 and Docker Desktop setup; repository clone; initial `make up`; Grafana/Docker troubleshooting. |

### AI usage disclosure

I used ChatGPT (GPT-6) as an assistance tool to explain the assignment workflow, suggest shell inspection commands, help create an `openpyxl` workbook-audit script, interpret Go source excerpts and help draft customer acknowledgements/escalation notes. I ran the commands locally and checked the suggestions against the repository's actual workbook contents, Go code, template documentation and outputs. The first workbook inspection produced a false 1 × 1 result due to read-only worksheet dimensions; I corrected the method and independently verified the worksheet dimensions and sample rows. AI-generated hypotheses were not treated as runtime proof. I have explicitly marked missing dashboard, log and database evidence rather than inventing it.

---
