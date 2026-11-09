# MediaMesh — Enterprise Media Discovery Platform

**Solution design proposal | Stakeholder review | Version 0.2 | 8 November 2026**  
**Illustrative organization:** National Geographic (NatGeo-style use case; not an actual NatGeo integration)  
**Status:** Proposed architecture — decisions pending

## 1. Executive summary

MediaMesh is a secure internal media library where authorized employees upload photographs and videos, while authorized researchers use natural-language queries to locate relevant assets. An administrator assigns application roles. Employees authenticate through enterprise single sign-on (SSO); APIs validate JWT access tokens and enforce role-based access control (RBAC). Original files are held privately in Amazon S3. Metadata and vector embeddings are stored in PostgreSQL with pgvector. A standalone Python ML service provides CLIP-based multimodal embeddings, video-keyframe indexing, and semantic similarity retrieval.

**MVP objective:** Demonstrate one complete, secure enterprise workflow: **SSO → admin role assignment → authorized media upload → automatic indexing → authorized semantic search → secure preview/download**.

The goal is **minimum implementation effort**, not a custom distributed storage system or a broad AI platform. S3 provides managed object storage; no custom replication, Kafka, Kubernetes, service mesh, or agent framework is required for the MVP.

## 2. Business problem and expected outcome

**Example scenario.** A NatGeo media researcher needs pictures or clips showing “elephants walking near a watering hole.” Searching filenames and manually maintained tags is slow and misses relevant assets. MediaMesh retrieves visually similar content while enforcing the permissions assigned by an administrator.

**Business outcomes:**
- Reduce time spent finding relevant visual media.
- Centralize access to approved media and ownership metadata.
- Ensure only permitted employees can upload, search, preview and download assets.
- Provide a clear path from upload to searchable asset, including failure status and retry.

**Primary users:** Administrator, Media Uploader, Media Researcher. A single employee may hold more than one role. Users with no assigned role may authenticate but cannot use protected business operations.

## 3. MVP scope and explicit exclusions

| In scope (required for demo) | Out of scope (defer unless approved) |
|---|---|
| Federated OIDC SSO and JWT-protected API | Building a custom identity provider |
| Admin-managed Uploader/Researcher roles | Complex organization-wide IAM workflows |
| Private S3 uploads, previews and downloads | Custom distributed file replication |
| Image semantic search with CLIP-compatible text/image embeddings | LLM agents, Ollama, MCP workflow orchestration |
| Video search through selected keyframes and timestamps | Native temporal video-understanding model |
| Asset metadata, status and access rules in PostgreSQL | Multi-tenant SaaS and elaborate per-asset ACL policy engine |
| Content-similarity recommendation (“Related media”) | Personalized collaborative filtering |
| Basic health checks, error handling and CI tests | Kafka, Kubernetes and service mesh |

**Audio:** Audio may be stored as a file, but semantic *audio-content* search is not guaranteed in the MVP. Uploaded title/description metadata can be searched. Transcription or audio embeddings are an optional extension.

**Authorization choice for minimum effort:** Roles control operations globally. Restrict access to assets by owner/collection visibility only if a restricted-collection demonstration is a stakeholder requirement. Otherwise start with a single internal media library where all Researcher-role users can view approved assets.

## 4. Proposed logical system architecture

```mermaid
flowchart TB
    U[Employees: Admin / Uploader / Researcher]
    UI[React Web Portal]
    IDP[Enterprise SSO / OIDC Identity Provider]
    API[Spring Boot API<br/>JWT validation and RBAC]
    DB[(PostgreSQL + pgvector<br/>Users / Roles / Assets / Vectors / Jobs)]
    S3[(Private Amazon S3<br/>Original media / previews)]
    ML[Python ML Service<br/>CLIP embeddings / similarity retrieval]
    WORKER[Background indexer<br/>Python worker or task runner]

    U --> UI
    UI -->|OIDC sign-in| IDP
    IDP -->|Authentication response| UI
    UI -->|Bearer JWT + HTTPS| API
    API -->|Fetch public signing keys as needed| IDP
    API -->|Read and write users, roles, assets, jobs| DB
    DB -->|Metadata and authorization results| API
    API -->|Request presigned upload/download URLs| S3
    UI -->|Upload/download using presigned URL| S3
    S3 -->|Media response| UI
    API -->|Authorize and request search| ML
    ML -->|Vector similarity queries| DB
    DB -->|Ranked vector matches| ML
    WORKER -->|Fetch uploaded files| S3
    WORKER -->|Generate embeddings| ML
    WORKER -->|Update job and asset status| DB
    API -->|Record durable index job| DB
```

**Deployment interpretation:** These are logical responsibilities, not necessarily separate microservices. The ML inference code and indexing worker may run from the same Python codebase. React, Spring Boot, Python and PostgreSQL can run in Docker Compose locally; S3 and the SSO provider are managed external services.

### Component responsibilities

| Component | Owns | Does not own |
|---|---|---|
| Identity provider | Employee authentication, SSO, OIDC protocol, token issuance | MediaMesh roles and media authorization |
| React portal | Login redirect, upload and search UI, result presentation | Security enforcement |
| Spring Boot API | JWT validation, role checks, asset lifecycle, presigned URLs, result authorization | Large binary file transfer or ML inference |
| PostgreSQL + pgvector | User-role assignments, media metadata, processing jobs, vectors | Original media bytes |
| S3 | Private object storage, managed durability | Semantic similarity search |
| Python ML | Image/text embedding, keyframe processing, vector similarity, related items | Assigning user roles |

## 5. Roles and access model

| Action | Admin | Uploader | Researcher | Unassigned employee |
|---|:---:|:---:|:---:|:---:|
| View/manage user-role assignments | Yes | No | No | No |
| Upload media | Optional* | Yes | No | No |
| Edit own media metadata | Optional* | Yes | No | No |
| Search media | Optional* | No | Yes | No |
| Preview/download search results | Optional* | No | Yes | No |

*For least privilege, **Admin is not implicitly a content user**. An admin can explicitly assign themselves Uploader or Researcher roles if needed. The table intentionally keeps administrative powers separate from content operations.

**Access rule:** Every protected API endpoint validates identity *and* current effective roles. UI visibility is convenience only. Role assignment/revocation persists in the application database and is checked for every request, so a long-lived JWT containing old roles does not override a revocation. JWT contains identity and standard security claims; application roles remain authoritative in PostgreSQL.

## 6. Primary sequence diagrams

### Flow A — Employee SSO login, JWT APIs and session expiry

```mermaid
sequenceDiagram
    autonumber
    actor Employee
    participant Web as React Portal
    participant IdP as OIDC SSO Provider
    participant API as Spring Boot API
    participant DB as PostgreSQL
    Employee->>Web: Open MediaMesh and select Sign In
    Web->>IdP: Authorization code flow with PKCE
    IdP->>Employee: Authenticate through enterprise SSO
    Employee->>IdP: Complete sign-in
    IdP-->>Web: Authorization code
    Web->>IdP: Exchange code securely for tokens
    IdP-->>Web: ID/access tokens and expiry
    Web->>API: GET /api/me with access JWT
    API->>API: Validate signature, issuer, audience, expiry
    API->>DB: Load current roles by identity subject
    DB-->>API: Assigned roles
    API-->>Web: Identity and authorized capabilities
    Note over Web,IdP: Token/session expiry controlled by IdP and app settings
    alt Access token expires and refresh is permitted
        Web->>IdP: Renew using supported secure session mechanism
        IdP-->>Web: New short-lived access token
    else Renewal not available / session expired
        Web->>Employee: Require SSO reauthentication
    end
```

**Security decisions:** OIDC Authorization Code + PKCE for the browser application; short-lived access JWTs; signature, `iss`, `aud`, and `exp` validation in Spring Security; protected tokens not stored in localStorage. Preferred implementation is a BFF/secure HttpOnly cookie or the identity provider's documented secure SPA flow. Application logout clears local session and requests IdP logout as supported. A locally cached JWT cannot automatically be revoked everywhere, so keep it short-lived and check RBAC against the database.

### Flow B — Administrator assigns or revokes a role

```mermaid
sequenceDiagram
    autonumber
    actor Admin
    participant Web as React Portal
    participant API as Spring Boot API
    participant DB as PostgreSQL
    Admin->>Web: Open User Management
    Web->>API: GET /api/admin/users with access JWT
    API->>API: Validate JWT and require ADMIN role
    API->>DB: Read users and role assignments
    DB-->>API: User list
    API-->>Web: User/role data
    Admin->>Web: Assign UPLOADER or RESEARCHER
    Web->>API: PUT /api/admin/users/{id}/roles
    API->>API: Validate JWT and require ADMIN role
    API->>DB: Save role assignment and audit event
    DB-->>API: Commit successful
    API-->>Web: Updated roles
    Note over API,DB: Next request reads current roles
    Note over API,DB: Changes apply before JWT expiry
```

**Failure behavior:** A user without `ADMIN` gets HTTP `403`. Missing/expired/invalid JWT gets HTTP `401`. Admin cannot grant arbitrary unsupported role names.

### Flow C — Media upload and semantic indexing

```mermaid
sequenceDiagram
    autonumber
    actor Uploader
    participant Web as React Portal
    participant API as Spring Boot API
    participant DB as PostgreSQL
    participant S3 as Private S3
    participant Worker as Indexing Worker
    participant ML as Python ML Service
    Uploader->>Web: Select image/video and enter metadata
    Web->>API: POST /api/assets/initiate with JWT and metadata
    API->>API: Validate JWT and require UPLOADER
    API->>DB: Create asset with UPLOADING status
    API-->>Web: Asset ID + short-lived S3 upload URL
    Web->>S3: Upload bytes directly via presigned URL
    S3-->>Web: Upload success
    Web->>API: POST /api/assets/{id}/complete with JWT
    API->>S3: Verify object exists / expected type and size
    API->>DB: Commit asset PROCESSING + durable indexing job
    API-->>Web: Accepted with PROCESSING status
    Worker->>DB: Claim next pending indexing job
    Worker->>S3: Read asset (private service credentials)
    opt Video
        Worker->>Worker: Extract sampled frames and timestamps
    end
    Worker->>ML: Generate CLIP image/frame embeddings
    ML-->>Worker: Embeddings
    Worker->>DB: Upsert vectors and mark asset READY
    Web->>API: GET /api/assets/{id}/status with JWT
    API-->>Web: READY
    alt Processing fails
        Worker->>DB: Record failure and retry count
        Note over Worker,DB: Retry with bounded attempts and mark FAILED after limit
    end
```

**Reliability choice:** A minimal **database-backed indexing job**, not a volatile in-process callback, prevents lost work after restart without introducing SQS. Each job is idempotent by asset ID + model version. Support manual retry for `FAILED` assets. Set file-type/size limits, use private buckets and reject unexpected content.

### Flow D — Authorized natural-language semantic search

```mermaid
sequenceDiagram
    autonumber
    actor Researcher
    participant Web as React Portal
    participant API as Spring Boot API
    participant DB as PostgreSQL
    participant ML as Python ML Service
    Researcher->>Web: Search "elephants near a watering hole"
    Web->>API: POST /api/search with query and JWT
    API->>API: Validate JWT and require RESEARCHER
    API->>DB: Resolve user's allowed asset scope
    DB-->>API: Effective visibility rules
    API->>ML: Search with query + allowed scope
    ML->>ML: Embed natural-language text (CLIP text encoder)
    ML->>DB: Vector similarity query within allowed scope
    DB-->>ML: Ranked asset/frame IDs + scores
    ML-->>API: Candidate results and timestamps
    API->>DB: Re-check result access and retrieve metadata
    DB-->>API: Authorized asset metadata
    API-->>Web: Ranked results, thumbnails, matching video timestamps
    Note over Web,API: Never return unauthorized asset IDs, metadata or URLs.
```

**MVP retrieval:** CLIP-compatible text and image embeddings are in the same space. Images have one or more vectors; videos are indexed using a limited number of keyframes. The standalone Python ML service encodes natural-language queries and stored images using a compatible CLIP model, ranks candidates via pgvector, and returns only asset IDs, scores, and frame timestamps to the Spring Boot API. A user can request **related media** by searching with the chosen asset's embedding.

### Flow E — Authorized preview/download, expiration and denial

```mermaid
sequenceDiagram
    autonumber
    actor Researcher
    participant Web as React Portal
    participant API as Spring Boot API
    participant DB as PostgreSQL
    participant S3 as Private S3
    Researcher->>Web: Open a search result
    Web->>API: POST /api/assets/{id}/access-url with JWT
    API->>API: Validate JWT and require RESEARCHER
    API->>DB: Verify this asset is visible to the user
    alt Authorized
        DB-->>API: Allowed
        API-->>Web: Short-lived presigned S3 GET URL
        Web->>S3: GET media/preview using signed URL
        S3-->>Web: Media bytes
    else Not authorized
        DB-->>API: Denied
        API-->>Web: 403 Forbidden
    end
    Note over Web,S3: Presigned URLs expire, S3 objects remain private.
```

**Important limitation:** An already issued S3 presigned URL can remain usable until its short expiry even if an admin revokes access moments later. Keep expiry brief and do not expose URLs to unrelated users. Stronger immediate revocation would require proxying content through a permission-checking service, which is outside this MVP.

## 7. Minimal data model

```mermaid
erDiagram
    USER ||--o{ USER_ROLE : has
    ROLE ||--o{ USER_ROLE : assigned
    USER ||--o{ ASSET : uploads
    ASSET ||--o{ ASSET_EMBEDDING : indexed_as
    ASSET ||--o{ INDEX_JOB : processed_by
    USER {
        uuid id PK
        string oidc_subject UK
        string email
    }
    ROLE {
        string name PK
    }
    USER_ROLE {
        uuid user_id FK
        string role_name FK
    }
    ASSET {
        uuid id PK
        uuid owner_id FK
        string s3_key
        string media_type
        string title
        string description
        string status
        datetime uploaded_at
    }
    ASSET_EMBEDDING {
        uuid id PK
        uuid asset_id FK
        float timestamp_seconds
        string model_version
        vector embedding
    }
    INDEX_JOB {
        uuid id PK
        uuid asset_id FK
        string status
        int attempts
        datetime updated_at
    }
```

For the lean MVP, roles are application-managed; user authentication remains managed by the IdP. Add `collection_id` and collection membership/visibility data **only if** restricted collections are approved as required scope. Avoid implementing asset-specific grants and multi-tenancy by default.

## 8. Standalone multimodal semantic retrieval service

The Python ML service is a **standalone MediaMesh component** built for the NatGeo-style media library. It exposes internal endpoints to the Spring Boot API and runs a background worker to index uploaded media. The browser cannot invoke the ML API directly.

| Component | MVP responsibility |
|---|---|
| Python Flask API | Implement internal semantic search and optional related-media retrieval |
| CLIP-compatible model | Generate text and image embeddings in one shared vector space |
| FFmpeg keyframe extractor | Sample a limited number of frames from short video clips and record timestamps |
| PostgreSQL + pgvector | Persist versioned asset/frame vectors and run similarity queries |
| Background indexing worker | Claim durable jobs, fetch media privately from S3, upsert vectors, and update processing status |
| Ranking and retrieval | Return ranked asset IDs, similarity scores, and optional video timestamps |

### Internal ML contract

`POST /internal/search` accepts a natural-language query and requested result limit. It returns only ranked candidates; it does not issue public media links or decide user roles.

```json
{
  "query": "elephants drinking at a watering hole at sunset",
  "limit": 20
}
```

```json
{
  "matches": [
    {"assetId": "example-asset-id", "score": 0.83, "frameTimestampSeconds": 72},
    {"assetId": "example-image-id", "score": 0.79, "frameTimestampSeconds": null}
  ]
}
```

The Spring Boot endpoint `POST /api/search` performs JWT and role validation, calls the internal ML endpoint, and performs a final authorized asset and metadata lookup before returning results to the user. The ML endpoint stays private. **No existing external recommendation engine is a documented project dependency.**

### Model and indexing constraints

- Use one explicitly versioned compatible CLIP text/image model and record embedding dimensionality in configuration and migrations.
- Generate one vector for an image and separately indexed timestamped keyframes for videos.
- Support idempotent reindexing by asset, model version, and frame position.
- Keep only a small evaluation set of images/videos for the MVP; avoid native audio embeddings and expensive video understanding.
- Teammate 2 owns asset metadata and indexing-job schema; Teammate 3 owns vectors, ML inference, and worker execution. Agree on job claiming and state transitions before integration.

## 9. Security, failure handling and operating assumptions

- **Identity:** Use one OIDC-capable SSO provider; no password database in MediaMesh.
- **JWT:** Validate signed access tokens on every protected API; reject expired tokens with `401`.
- **RBAC:** Persist role assignments centrally; enforce on each business endpoint (`403` if authenticated but unauthorized).
- **Session:** Agree on access-token lifetime, session idle limit and reauthentication behavior before implementation; do not choose values implicitly. Secure cookie/BFF is preferred if its implementation burden fits.
- **Storage:** S3 bucket is private, with least-privilege IAM credentials and presigned URLs for time-limited client access.
- **Metadata:** Store S3 keys, ownership, statuses and versioned embeddings; avoid storing binary media in PostgreSQL.
- **Processing:** Durable database jobs, bounded retries, idempotent indexing and visible processing errors.
- **Observability:** Structured logs, API health checks and minimal audit events for role changes and media actions.
- **Deployment:** Docker Compose for local development and demonstration; managed S3/identity services. Jenkins builds and tests on GitHub changes.

## 10. Minimum acceptance criteria

| Scenario | Acceptance criterion |
|---|---|
| Employee signs in | Enterprise SSO completes; API accepts valid JWT and rejects expired/invalid JWT (`401`). |
| Admin assigns role | Only Admin can grant/revoke roles; updated privileges take effect on subsequent calls. |
| Unauthorized upload/search | Authenticated user lacking required role receives `403`. |
| Authorized upload | Image/video stored privately in S3; metadata persisted and indexing status visible. |
| Semantic image search | Natural-language query returns relevant indexed images from a curated evaluation set. |
| Semantic video search | Query returns relevant video and representative matching timestamp/keyframe. |
| Secure media retrieval | Only authorized users receive short-lived media URLs; S3 bucket is not public. |
| Background job failure | Failed indexing is visible and can be retried without duplicate asset records. |
| Engineering workflow | Services run via documented Docker setup; CI runs builds and tests. |

**Suggested demonstration:** Admin grants Uploader and Researcher roles; Uploader adds wildlife images and a short clip; indexing completes; Researcher searches “elephants near a watering hole” and previews matches; unassigned user is denied search; demonstrate expired JWT and revoked role handling.

## 11. Decisions requested from stakeholders

| Decision | Recommended MVP default | Alternative / impact |
|---|---|---|
| Is NatGeo illustrative or the actual client? | Illustrative scenario with sample or licensed content | Actual integration requires enterprise approval, legal/data review |
| Role separation | Admin, Uploader, Researcher; multiple roles per user | Combining roles simplifies UI but weakens demonstration |
| Asset visibility | All approved internal assets searchable by Researchers | Restricted collections add a permissions data model and tests |
| SSO provider | One supported OIDC provider, ideally existing team access | Deploying Keycloak adds local infrastructure |
| Session behavior | Short-lived JWT + secure renewal or re-login | Exact idle/absolute expiry values to decide |
| Media scope | Image semantic search + video keyframes | Native audio semantics and detailed video models are deferred |
| Vector storage | PostgreSQL + pgvector | FAISS requires separate persistence/permissions integration |
| Indexing reliability | Database-backed job table | SQS adds AWS infrastructure and operational setup |
| Deployment | Docker Compose + S3 + hosted IdP | Cloud deployment separately if specifically graded |
| CI/CD | GitHub + Jenkins pipeline | Other CI services only if course permits |

**Recommendation:** Approve the MVP above, choose a single SSO provider, decide whether restricted collections are essential, and agree on session-expiry rules. Only after these decisions should tasks be allocated across the team.

## 12. Non-goals and project success boundary

MediaMesh is **not** rebuilding S3, training a multimodal foundation model, building a generalized autonomous agent, or delivering a production-grade National Geographic integration. It is a demonstrable enterprise-style application built around managed identity, role-enforced APIs, secure media storage and a standalone multimodal semantic retrieval service.

**MVP is complete when the end-to-end demonstration works securely, is reproducible, and meets the acceptance criteria in §10.**
