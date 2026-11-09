# Interface freeze and integration rules

The canonical MVP contract is documented in [the team overview](../team/00_TEAM_OVERVIEW.md), including routes, response shapes, role names, error codes, indexing statuses and database migration prefixes.

## Before coding

1. Agree on an OIDC provider, a first-admin bootstrap procedure and whether admin implicitly has other roles (proposed: no).
2. Freeze REST paths and the exact JSON fields and error representation; publish minimal example fixtures.
3. Choose one CLIP model and record the embedding dimension and version in the vector schema.
4. Confirm media format limits and a test S3 bucket policy.
5. Assign owners for shared Spring Boot startup files and shared Docker configs.

## Interface requirements

- Browser traffic goes through Spring Boot; internal Python endpoints are never exposed to the browser.
- Spring Security validates bearer JWTs, including expiration, issuer, audience and signature; the application checks role assignments from its own database on protected operations.
- Unauthenticated/expired = `401`, authenticated but insufficient role = `403`.
- Each asset has a durable ID and state: `UPLOADING`, `PROCESSING`, `READY`, or `FAILED`.
- Search results return asset IDs and optional video-frame timestamps; asset metadata and S3 access URLs come from authorized backend routes.
- Indexing and `complete` must be idempotent, so retries cannot create duplicate indexed vectors.

## Collaboration policy

Code ownership is exclusive per package and directory; request contract changes through pull requests. Frontend uses mocks while APIs are built, and media backend uses a mock ML service until Python integration is available. The README documents the intended system; real setup commands should only be added after they are tested.
