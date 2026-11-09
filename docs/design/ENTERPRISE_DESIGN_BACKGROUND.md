# MediaMesh — Enterprise Design Background and Decisions

**Companion reference | Version 0.3 | 8 November 2026**  
**Non-normative:** This document explains the reasoning behind the NatGeo MVP; it is **not** a second requirements or API specification. The authoritative source is [NATGEO_STAKEHOLDER_DESIGN.md](NATGEO_STAKEHOLDER_DESIGN.md). Where documents differ, follow that file and the approved shared team API contract.

## 1. Business framing

MediaMesh is designed as a hypothetical enterprise media discovery pilot for a National Geographic-style editorial organization. Field photographers and archive staff upload wildlife photographs and documentary clips. Footage researchers search visually relevant media with natural language and can retrieve assets they are authorized to access. A media administrator assigns and revokes upload and search permissions.

Example workflow: an uploader submits elephant photographs and a short watering-hole clip, records species/location/date/contributor and usage notes, and the system indexes them. A researcher searches for **"elephants drinking at a watering hole at sunset"** and receives ranked photos or approximate video frame timestamps.

This is an educational demonstration. It does not claim access to National Geographic's actual archives, sign-in systems, or licensing workflows.

## 2. Why these architectural choices?

| Decision | MVP choice | Rationale | Not included |
|---|---|---|---|
| Enterprise login | Single OIDC provider with Authorization Code + PKCE | Leverage existing identity standards rather than build passwords | Custom identity platform, multiple SSO protocols |
| Permissions | DB-backed `ADMIN`, `UPLOADER`, `RESEARCHER` | Admin controls access and role revocation takes effect on subsequent API calls | Complex asset ACLs, multi-tenancy |
| API/backend | Spring Boot + Spring Security | One external API boundary and centralized authorization | API gateway, service mesh |
| Media storage | Private Amazon S3 | Managed object durability and presigned data transfer | Custom replication engine |
| Metadata + vectors | PostgreSQL with pgvector | One transactional database for business records and similarity search | Separate production vector cluster |
| ML | Standalone Python CLIP service | Shared image/text embedding space for media search | LLM agent, training foundation models |
| Video retrieval | FFmpeg sampled keyframes | Demonstrates useful approximate video search at low cost | Temporal video model, speech transcription |
| Indexing | Durable database job table and Python worker | Retry/recovery without a broker | Kafka, SQS, distributed workflow engine |
| Delivery | Docker Compose and Jenkins | Small repeatable deployment and build/test pipeline | Kubernetes, multi-region platform |

## 3. Enterprise security rationale

**Authentication is not authorization.** SSO confirms employee identity; MediaMesh's `users` and `user_roles` records determine capabilities. On first sign-in, a user has no business role. The first administrator is established through a controlled bootstrap process. An admin can grant multiple roles to an employee but does not inherently gain upload or search access.

The public Spring Boot API checks access JWT signature, issuer, audience and expiry, then reads current application roles. Expired or invalid JWT returns `401`; a signed-in user lacking permission receives `403`. The browser cannot call the internal ML service directly.

**Proposed session strategy:** use a supported OIDC SPA library with Authorization Code + PKCE, tokens kept in memory instead of localStorage, and provider-managed secure renewal or reauthentication. Candidate limits are 15-minute access tokens, 30-minute inactivity and an 8-hour maximum login session; the team must verify actual enforcement with the chosen provider. Do not promise instant revocation of previously issued S3 presigned URLs. Keep media URLs short-lived.

## 4. Search and media design rationale

Media bytes belong in S3, not the database. After a verified upload, a durable indexing job is created in the same database used for asset status. The Python worker claims the job, reads private media, creates CLIP embeddings for photographs or sampled video keyframes, stores versioned vectors, and marks the asset READY. Bounded retries and idempotent upserts prevent permanent loss or duplicate indexing.

The Python search service exposes `POST /internal/search` to Spring Boot. A researcher's browser calls **`POST /api/search`** only. Spring Boot checks the Researcher role, invokes retrieval, applies READY and requested metadata conditions, and checks permissions before releasing results. For the single-library MVP, every Researcher can search all READY pilot assets. Candidate over-fetching mitigates loss of results after metadata filtering; strict filtered top-K is an optional follow-up.

The ML service is a standalone MediaMesh component with its own Python implementation, model selection, vectors and worker. Its origin or internal implementation history is irrelevant to the public project specification.

## 5. Team interface boundaries

| Owner | Owns | Integrates through |
|---|---|---|
| Teammate 1 | OIDC/JWT validation, roles, admin APIs, audit | Shared Spring security interface and `users` / `user_roles` |
| Teammate 2 | Asset catalog, S3 signed URLs, indexing-job schema, public search endpoint | Spring APIs and database asset/job contract |
| Teammate 3 | Python model, embedding/indexing worker, vectors, internal search | `/internal/search`, `asset_vectors`, agreed `index_jobs` contract |
| Teammate 4 | React portal, Docker Compose, Jenkins, demo integration | Stable public `/api/...` contract and mock fixtures |

Do not let multiple teammates independently edit the same schema migration or implementation package. Freeze request/response JSON, migration namespaces, image/model dimensions, and the job-lease handshake at the first integration checkpoint.

## 6. Deferred capabilities

- Native audio search or transcription.
- Embargoed/private collection ACLs and rights clearance enforcement.
- User-behavior-based personalized recommendations.
- Custom distributed storage, streaming architecture and multi-region replication.
- Kubernetes, gateway/service mesh, complex observability stack, and LLM agent orchestration.

These are not required to demonstrate the enterprise MVP and should not block the four agreed workstreams.

## 7. Stakeholder approvals needed

1. Confirm the single identity provider for the pilot (Keycloak or accessible OIDC provider).
2. Approve and test access-token lifetime and absolute/idle session behavior.
3. Approve the single internal media library accessible to all `RESEARCHER` users; decide whether restricted collections are genuinely required.
4. Agree supported sample image/video formats, upload size caps, S3 budget and AWS region.
5. Freeze `POST /api/search`, indexing job states, and CLIP model/vector dimensions in the team contract.

After those approvals, the [stakeholder design](NATGEO_STAKEHOLDER_DESIGN.md) and [shared team overview](../team/00_TEAM_OVERVIEW.md) define the implementation baseline.
