# Teammate 2 — Media Catalog, Private S3, Public APIs and Job Lifecycle

**Mission:** Own the NatGeo media lifecycle from authorized upload to retrievable asset: private S3 object, PostgreSQL catalog, indexing job, query-to-ML orchestration, and secure preview/download URLs. You do **not** build auth/RBAC internals or write embedding-model code.

## You own
- Backend packages `media/`, `storage/`, `jobs/` and associated tests.
- SQL migrations `V2xx__...sql`: `assets`, `index_jobs` (coordinate FK columns with teammates 1/3).
- All browser-facing asset APIs, plus the `/api/search` facade that calls teammate 3's Python retrieval endpoint.
- Presigned S3 PUT/GET URLs, object verification, metadata persistence and status transitions.

## Tasks in implementation order

- [ ] **T2.1 — Asset data model.** Stable UUID asset ID, `owner_id`, S3 key, `image|video`, title, description/shot notes, species, location, captured date, contributor, usage note, `UPLOADING|PROCESSING|READY|FAILED`.
- [ ] **T2.2 — Private S3 bucket integration.** Use IAM credentials only on backend, never browser. Dev bucket/prefix isolation; presigned PUT with short expiration and server-generated key. Limit MIME types and file size; object verification after upload.
- [ ] **T2.3 — Initiate/complete upload.** `POST /api/assets/initiate` checks `UPLOADER`, creates `UPLOADING`, returns assetId/PUT URL. Browser PUTs directly to S3; `POST /api/assets/{id}/complete` checks ownership, verifies object, records one `index_jobs` row, sets `PROCESSING` and returns 202. Complete should be idempotent.
- [ ] **T2.4 — Job persistence contract.** `index_jobs` status, attempts, error and lease; expose safe claim/update DB operations used by teammate 3's Python worker. Ensure crashed jobs can be retried. **You own job state/schema; teammate 3 owns the worker process.**
- [ ] **T2.5 — Catalog APIs.** `GET /api/assets/{id}` and minimal listing `GET /api/assets` for uploaders. Only return metadata to assigned roles consistent with agreed policy.
- [ ] **T2.6 — Search facade.** `POST /api/search` checks `RESEARCHER`, calls internal Python `/internal/search`, then loads `READY` asset records and applies species/location/date/type filtering. Return ranked asset IDs, scores and video timestamps (without S3 object URL). The ML service must not become a public browser API.
- [ ] **T2.7 — Preview/download.** `POST /api/assets/{id}/access-url` checks `RESEARCHER`, existence/status and generates an exact-key presigned GET URL (proposed TTL 60 sec). Invoke teammate 1's audit service for download events; define and test 401/403/404.
- [ ] **T2.8 — Tests.** S3 failures, wrong upload owner, duplicate complete, no `READY` results, type/metadata filters, user without role, invalid input, presigned URL scope, missing asset.

## Contract you publish at Checkpoint A

```http
POST /api/assets/initiate
Authorization: Bearer <uploader_jwt>
Content-Type: application/json
```

```json
{"filename":"DSC_0842.mp4","contentType":"video/mp4","title":"Elephants at watering hole","description":"Herd drinking at sunset","species":"African elephant","location":"Okavango Delta, Botswana","capturedAt":"2025-08-14","contributor":"Field crew","usageNote":"Internal review; verify rights before publication"}
```

```json
{"assetId":"uuid","uploadUrl":"https://presigned-S3-URL","status":"UPLOADING"}
```

```http
POST /api/assets/{id}/complete  -> 202 {"assetId":"uuid","status":"PROCESSING"}
GET  /api/assets/{id}           -> 200 {"assetId":"uuid","title":"...","processingStatus":"READY",...}
POST /api/assets/{id}/access-url -> 200 {"url":"https://...","expiresInSeconds":60}
```

```http
POST /api/search
Authorization: Bearer <researcher_jwt>
```

```json
{"query":"elephants drinking at sunset","mediaType":"video","location":"Botswana","species":null,"capturedAfter":null,"limit":10}
```

```json
{"results":[{"assetId":"uuid","title":"Elephants at watering hole","mediaType":"video","score":0.83,"frameTimestampSeconds":72,"species":"African elephant","location":"Okavango Delta, Botswana","processingStatus":"READY"}]}
```

## Dependencies / non-overlap
- **1** owns access-control implementation. You use their role-check service and audit interface; never edit their files.
- **3** owns CLIP/vector data, indexing worker and `/internal/search`. Before integration, stub `/internal/search` with fixture ranked IDs.
- **4** owns React calls and UI. Publish OpenAPI examples or a Postman collection and a mocked backend for UI progress.
- Agree asset UUID and `assets.id` column contract with **3**, and `users.id` with **1**. You do not write vector migrations.

## Independent demo
1. Authorized uploader creates an asset; browser/client PUTs sample elephant photo to private S3.
2. Upload-complete creates one retryable processing job and remains idempotent on retry.
3. Mock retrieval returns matching metadata through `/api/search` only for Researcher.
4. Authorized Researcher gets a short-lived download URL, anonymous requester gets 401.

## Done when
- [ ] Private S3 transfer, catalog persistence, index-job statuses and secure retrieval work.
- [ ] Search facade integrates with teammate 3 using the frozen contract.
- [ ] Tests validate authorization call sites, upload failures and retries.
- [ ] Uploaders cannot grant roles; researchers cannot upload without Uploader role.
