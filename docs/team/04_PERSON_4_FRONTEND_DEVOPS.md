# Teammate 4 — NatGeo Editorial Portal, Containers, CI and End-to-End Demo

**Mission:** Build the actual NatGeo-facing web experience and make the whole team deliverable/testable. You own React, mock contracts, Docker Compose and Jenkins/GitHub CI. You do **not** edit backend role/upload implementations or ML algorithms.

## You own
- Entire `frontend/` directory and UI tests.
- `infra/`, root `compose.yaml`, `Jenkinsfile` (or `.github/workflows/` if team approves), demo seed instructions and end-to-end test harness.
- `docs/contracts/` as steward of **agreed** API examples; changes require backend/ML owners' approval.

## Main NatGeo screens

1. **SSO entry / logged-in state** — show employee identity, session expiry handling, sign out.
2. **Role-aware navigation** — only show Admin, Upload and Search menus when `/api/me` grants roles; backend remains source of truth.
3. **Admin dashboard** — list signed-in employees and grant/revoke Uploader/Researcher using teammate 1's APIs.
4. **Uploader page** — select wildlife photo/video, enter species/location/date/contributor/usage note, show upload progress and processing status.
5. **Researcher page** — natural-language query, optional media-type/location/species/date filters; list result thumbnails, titles and video timestamp; secure preview/download by requesting backend access URL.

## Tasks in implementation order

- [ ] **T4.1 — React skeleton and API client.** Setup routing, shared layout, typed API DTOs, error display and environment-specific backend URL. Start with mocked response JSON from shared contract.
- [ ] **T4.2 — OIDC browser integration.** Use established OIDC client with Authorization Code + PKCE per teammate 1's identity provider configuration. Handle session/renewal/re-login without storing long-lived refresh tokens in localStorage. On API 401, go through auth renewal or login; on 403, show an access-denied state.
- [ ] **T4.3 — Role-aware screens.** Build Admin screen (list + assign roles); no admin API should be callable without valid server-side authorization. Show an informative unassigned-user screen after sign-in.
- [ ] **T4.4 — Uploader workflow.** Form fields as agreed with teammate 2, call `initiate`, upload file directly via presigned S3 PUT, call `complete`, poll `GET /api/assets/{id}` until READY/FAILED. Display progress/errors.
- [ ] **T4.5 — Research UI.** Natural-language search box, simple filters, ranked photos/video frame results; call `access-url` only on preview/download click. Display rights note before download to remind humans to verify license.
- [ ] **T4.6 — Developer environment.** Docker Compose for React, Spring Boot, PostgreSQL+pgvector and Python ML/worker; reference a *real* private AWS S3 bucket for final demo, with mock test credentials/storage for CI. Own `.env.example` and startup script; no committed secrets.
- [ ] **T4.7 — CI.** GitHub webhook/Jenkins pipeline installs/builds/tests frontend, backend, Python, and checks containers. Reuse course Jenkins approach; simple green build is enough—no Kubernetes.
- [ ] **T4.8 — Integrated demo.** Document setup/seeded NatGeo dataset, role assignment, photo/video upload, search and preview. Add minimal smoke tests exercising main UI path; report and route API issues to the owning teammate.

## Mock-first API contract you consume

- `GET /api/me` -> user profile and `roles` array (teammate 1).
- `GET /api/admin/users`, `PUT /api/admin/users/{id}/roles` (teammate 1).
- `POST /api/assets/initiate`, browser PUT to `uploadUrl`, `POST /api/assets/{id}/complete`, `GET /api/assets/{id}`, `POST /api/assets/{id}/access-url` (teammate 2).
- `POST /api/search` -> `{results:[...]}` containing assetId, title, mediaType, score, species/location and optional video timestamp (teammate 2). **Never call Python directly from browser.**

## Dependencies / non-overlap
- **1** owns backend identity and session enforcement; you own browser login flow and role-aware navigation. Agree issuer, client, redirects and logout URL before coding.
- **2** owns media endpoints; you use fixture JSON until API is ready. You own UI, never storage/backend modules.
- **3** owns ML. You only display Spring's search results, do not implement query embeddings.
- You own all shared Compose and Jenkins files. Other teammates provide Dockerfile/API env needs to you instead of editing these files concurrently.

## Independent demo
1. React mock login establishes the intended UI flow and displays unassigned state.
2. Admin screen assigns/revokes roles against mock API; upload and search interfaces work with fixtures.
3. Docker Compose starts development stack and Jenkins runs tests.
4. After integration, full SSO -> grant role -> upload -> natural-language search -> preview/download works.

## Done when
- [ ] All three role-specific views work and show appropriate unauthorized/expired states.
- [ ] Full UI workflow uses only Spring public APIs; media goes directly to S3 via presigned URLs.
- [ ] Team can start project with documented commands, and CI passes.
- [ ] Final NatGeo demo is documented and reproducible without modifying other owners' code.
