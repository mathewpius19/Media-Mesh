# MediaMesh for NatGeo — Team Work Allocation (MVP)

**Purpose:** Four independently owned workstreams for a NatGeo-style editorial media library. This is a proposal to approve before coding, not a claim about National Geographic's actual internal systems.

## Product scope
A NatGeo photographer/archive assistant uploads wildlife photos and short documentary videos with species, location, capture date, photographer, and usage-rights notes. A NatGeo footage researcher searches with phrases like **"elephants drinking at a watering hole at sunset"**, sees matching photos/video keyframes, and securely previews or downloads an authorized item. An administrator grants/revokes the Uploader and Researcher roles. Staff sign in through enterprise SSO; APIs validate JWTs and respect session expiry.

**MVP yes:** OIDC SSO, short-lived access JWT, DB-backed admin-managed RBAC, private S3 media, PostgreSQL/pgvector, CLIP image/video-keyframe retrieval, basic similar-media suggestions, React UI, Docker Compose, CI, basic retryable indexing and audit records.

**MVP no:** custom distributed storage, Kafka/SQS, Kubernetes, LLM/Ollama/MCP agent, native audio embeddings, full rights clearance, complex collection-level ACLs, multiple organizations or production-grade disaster recovery. Audio is out of semantic search scope in this MVP.

## Ownership — no overlapping source-code areas

| Teammate | Workstream | Exclusive source ownership | Primary demo |
|---|---|---|---|
| **1** | Identity, RBAC and audit | `backend/src/.../security/`, `.../identity/`, `.../admin/`, security migrations | SSO login, admin grants/revokes role; 401/403 |
| **2** | Media API, S3 and indexing job lifecycle | `backend/src/.../media/`, `.../storage/`, `.../jobs/`, catalog/job migrations | Upload and secure download of NatGeo wildlife photo/video |
| **3** | ML embeddings, indexing and retrieval | `ml-service/`, vector migrations | Natural-language image/video-frame search, similar assets |
| **4** | React portal, DevOps and demo automation | `frontend/`, `infra/`, `.github/` or `Jenkinsfile`, demo fixtures | End-to-end role-aware NatGeo portal and automated CI |

> Backend packages above are conceptual: choose a package prefix once (for example `com.mediamesh`). Each teammate owns their package, but the backend team jointly agrees on API DTOs early. **Only one person edits a given shared file.**

## Suggested repository layout

```text
mediamesh/
  backend/
    src/main/java/.../security/     # teammate 1
    src/main/java/.../identity/     # teammate 1
    src/main/java/.../admin/        # teammate 1
    src/main/java/.../media/        # teammate 2
    src/main/java/.../storage/      # teammate 2
    src/main/java/.../jobs/         # teammate 2
    src/main/resources/db/migration/ # teammate-specific V###_ prefixes (see below)
  ml-service/                       # teammate 3
  frontend/                         # teammate 4
  infra/                            # teammate 4
  docs/contracts/                   # owner: teammate 4, change by reviewed PR
  Jenkinsfile                        # teammate 4
```

## Shared API and integration contract — agree before coding

The public browser API is **only Spring Boot** (`/api/...`). The ML service is called by the backend or the indexing worker; the browser never calls Python directly. API bearer JWTs are validated at Spring Boot. The backend uses **current DB roles**, not just stale token claims.

| Owner | HTTP route | Behavior |
|---|---|---|
| 1 | `GET /api/me` | JWT authenticated -> `{id,email,displayName,roles:[...]}`; first login provisions unassigned user |
| 1 | `GET /api/admin/users` | ADMIN only; list provisioned users and assigned roles |
| 1 | `PUT /api/admin/users/{id}/roles` | ADMIN only; request `{"roles":["UPLOADER"]}`; replace assigned roles, audit change |
| 2 | `POST /api/assets/initiate` | UPLOADER only; create asset and give scoped presigned PUT URL |
| 2 | `POST /api/assets/{id}/complete` | UPLOADER/owner; check object exists; queue DB job; return 202 |
| 2 | `GET /api/assets/{id}` | Authorized role; metadata and `UPLOADING/PROCESSING/READY/FAILED` |
| 2 | `POST /api/assets/{id}/access-url` | RESEARCHER only; short-lived presigned GET URL |
| 2 | `POST /api/search` | RESEARCHER only; calls Python retrieval, filters asset metadata, returns ranked results |
| 3 | `POST /internal/embed-media` | Internal-only, worker uses to obtain image/frame embeddings |
| 3 | `POST /internal/search` | Internal-only, backend uses to obtain ranked asset IDs / frame offsets |
| 3 | `GET /health` | Python liveness |

**Shared error conventions:** unauthenticated/expired/invalid token = `401`; authenticated lacking role = `403`; absent asset = `404`; invalid input = `400`; processing search returns no `READY` candidate. Pagination can be omitted for small demo dataset.

**Minimal asset schema** (owned by teammate 2): `id`, `owner_id`, `s3_key`, `media_type` (`image|video`), `title`, `description`, `species`, `location`, `captured_at`, `contributor`, `usage_note`, `processing_status`. Frontend uses these exact JSON keys unless contract PR changes them.

**Search request:** `{ "query":"elephants at a watering hole", "mediaType":"video", "location":"Botswana", "species":null, "capturedAfter":null, "limit":10 }`. Filters are optional; model produces semantic candidate IDs, backend ensures role and READY/metadata filtering. Search response: `{ "results": [{"assetId":"...","title":"...","mediaType":"video","score":0.83,"frameTimestampSeconds":72,"species":"African elephant","location":"Botswana","processingStatus":"READY"}] }`. Scores are ranking signals, not probabilities.

**Internal ML interface:** `/internal/search` accepts `{query,limit}` and returns `{matches:[{assetId,score,frameTimestampSeconds}]}`; `/internal/embed-media` accepts a private-accessible object reference or locally mounted asset path and returns embedding records. Decide early whether worker and ML live in one Python process: recommended **yes**. Do not expose internal endpoints publicly. The precise worker-to-ML handoff can be a Python function if co-located.

## Database ownership / migration convention

- **1:** `users`, `user_roles`, `audit_events` and their migrations (reserve `V1xx__...sql`). Identity subject is unique; permissions apply on next protected request.
- **2:** `assets`, `index_jobs` and their migrations (reserve `V2xx__...sql`). Asset status changes are idempotent.
- **3:** `asset_vectors` and pgvector extension/index migrations (reserve `V3xx__...sql`). Stable FK to `assets.id`, compatible CLIP dimension and model version stored. **Do not alter teammate 2's tables** without a reviewed contract change.
- **4:** no database schema changes; use APIs and mocks.

## Security defaults to approve

- OIDC Authorization Code + PKCE against **one configured identity provider** (dev/test can use Keycloak; a real external enterprise federation is optional until configured). Do not implement custom passwords.
- Suggested **15-minute access JWT**, **30-minute idle session**, **8-hour absolute session** (configurable examples, not automatic OIDC behavior). Renewal/re-auth must actually enforce chosen limits. Avoid refresh tokens in browser localStorage.
- An employee may be **ADMIN**, **UPLOADER**, **RESEARCHER**, or a combination. Newly logged-in users have no roles. Admin needs an explicit first-admin bootstrap procedure.
- Spring Boot verifies JWT signature, issuer, audience and expiry, then loads current roles from DB per protected action. Revocation must work with an unexpired JWT.
- Private S3 bucket. Only backend-generated, narrowly scoped presigned URLs; propose 60-second GET expiry. Treat issued URLs as usable until expiry even after role revoke.
- For MVP, researchers with the RESEARCHER role see all READY assets in the single internal pilot library; no per-asset embargo permissions. Usage notes are informative, not automated licensing decisions.

## Parallel work schedule (relative, not calendar dates)

| Checkpoint | Person 1 | Person 2 | Person 3 | Person 4 |
|---|---|---|---|---|
| **A — Contracts** | Mock `/api/me` and admin JSON | Mock asset/upload/search JSON | Sample ranked matches and vector format | Screens/wireframes + mock API fixture |
| **B — Standalone** | OIDC/JWT and RBAC tests | S3 upload + PostgreSQL catalog tests | CLIP local image search on demo photos | React screens using fixtures; Docker baseline |
| **C — Integration** | Role provisioning and revocation | Complete job + retrieval endpoints | Video frames + pgvector + worker | Connect UI to API; Jenkins and seeded demo |
| **D — Acceptance** | Expiry and 401/403 demonstration | Signed download and retry demonstration | Query quality and matching timestamps | Full walkthrough, runbook, CI proof |

## Mandatory collaborative tests (each has a single lead)

1. **Login + permissions** (lead 1; support 4): new employee signs in and cannot search; Admin grants Researcher; permitted; Admin revokes; access denied with same unexpired JWT.
2. **Upload + indexing** (lead 2; support 3): Uploader uploads a photo and 1 short video; status moves from UPLOADING to PROCESSING to READY; duplicate `complete` does not create duplicate vectors.
3. **Search quality** (lead 3; support 2): natural-language query retrieves relevant NatGeo sample photos and a video timestamp; species/location filter works on submitted metadata.
4. **End-to-end + CI** (lead 4; all help): browser SSO -> role assignment -> upload -> search -> preview; GitHub change triggers automated tests.

## Team process to avoid collisions

- One branch per feature, PR into `main`; do not commit directly to `main`.
- One code owner per package. If a change crosses boundaries, request an interface change and have **the owner** implement their part.
- Freeze public JSON field names, role names, error codes and asset statuses at Checkpoint A. Mock unavailable services instead of waiting for them.
- Shared config (`application.yml`, root build files, Docker Compose) has an assigned owner: **1** owns Spring application security/config; **4** owns container orchestration and CI. Other contributors provide config requirements as PR notes, not simultaneous edits.
- Each teammate supplies a 2–3 minute isolated demo and README within their owned directory.

## Decisions still open — approve as a team

1. Identity provider: Keycloak (fast local OIDC demo) versus hosted IdP; whether to demonstrate actual external federation.
2. Cloud account access and S3 budget; use mock/local S3 for CI, real private S3 for final demonstration.
3. CLIP model/version and vector dimension (freeze before schema migration).
4. Maximum upload size and allowed formats; restrict demo video duration to bound indexing costs.
5. Whether user roles are exactly three + combinations, and which seeded admin starts the system.

**Definition of done:** A NatGeo-style editor can sign in, receive an Admin-assigned role, upload wildlife media, search via natural language, and securely access permitted results in the browser; CI passes and denial/expiry behavior is demonstrated.
