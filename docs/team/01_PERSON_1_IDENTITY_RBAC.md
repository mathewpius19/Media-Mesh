# Teammate 1 — Enterprise SSO, JWT Security, RBAC and Audit

**Mission:** Make the NatGeo pilot an enterprise-controlled application: real OIDC sign-in, correctly validated API JWTs, administrator-assigned roles, and session expiry. You own backend identity/security/admin code, **not** media APIs or the web UI.

## You own
- Spring Boot packages `security/`, `identity/`, `admin/` and associated tests.
- SQL migrations `V1xx__...sql`: `users`, `user_roles`, `audit_events`.
- Spring Security token validation configuration, security-focused application settings, login callback/session integration as agreed with teammate 4.
- `GET /api/me`, `GET /api/admin/users`, `PUT /api/admin/users/{id}/roles`.

## Tasks in implementation order

- [ ] **T1.1 — Decide and configure one OIDC provider.** Recommend Keycloak locally; configure issuer, audience/client, JWKS, redirect URLs and Authorization Code + PKCE. Use standard OIDC libraries, not custom login or JWT issuance.
- [ ] **T1.2 — Validate bearer JWT on every protected `/api/**` call.** Reject invalid signature, wrong issuer/audience and expired tokens as 401. Public health/auth endpoints must be explicit exceptions.
- [ ] **T1.3 — Provision user on first login.** Store stable OIDC subject and display/email, with **zero assigned roles** initially. Define first Admin bootstrap (seed one explicit admin subject in dev configuration).
- [ ] **T1.4 — Implement DB-backed roles.** `ADMIN`, `UPLOADER`, `RESEARCHER`; support multiple roles per user. Provide a reusable authorization service or method annotation for teammate 2. Read *current* roles on each protected request rather than relying on role claims in an old token.
- [ ] **T1.5 — Admin user-management APIs.** List provisioned users and assign/revoke roles. Only ADMIN may change roles. Atomic role update + audit event, including actor and timestamp.
- [ ] **T1.6 — Session expiry.** Propose access token TTL 15 min, idle timeout 30 min, absolute session 8 h; implement with provider/browser-session strategy agreed with teammate 4. A raw JWT cannot be invalidated just by clearing frontend state; enforce limits in actual session flow. Prefer secure HttpOnly cookies for renewal material; no long-lived refresh token in localStorage.
- [ ] **T1.7 — Security tests.** 401 vs 403, assigned/unassigned roles, role combinations, revoked role while existing JWT still valid, admin-only updates, forced expiry and logout.
- [ ] **T1.8 — Admin audit record.** Role grants/revokes logged with actor, subject, action, outcome. Teammate 2 may call an audit interface for downloads; you own its implementation.

## Contract you publish at Checkpoint A

```http
GET /api/me
Authorization: Bearer <access_jwt>
```

```json
{"id":"uuid","email":"researcher@example.org","displayName":"Footage Researcher","roles":["RESEARCHER"]}
```

```http
GET /api/admin/users
PUT /api/admin/users/{id}/roles
Content-Type: application/json
Authorization: Bearer <admin_jwt>
```

```json
{"roles":["UPLOADER","RESEARCHER"]}
```

The `PUT` **replaces** the user's entire role set. Define error JSON consistently with teammate 2. Unauthenticated -> 401; valid identity lacking required role -> 403. Admin UI consumes these routes via teammate 4.

## Dependencies / non-overlap
- Coordinate OIDC redirect and session strategy with **4**; 4 owns React callbacks/guards, you own security implementation.
- Give **2** one reusable backend role-check interface. Do not implement upload/search endpoints yourself.
- Agree `users.id` and `user_roles` schema with **2** before `assets.owner_id` foreign key is created.
- You own Spring Security filters/config. Teammate 2 calls security services but does not modify security files.

## Independent demo
1. First-time logged-in employee receives `roles: []`.
2. An Admin assigns `UPLOADER`; `/api/me` returns new role.
3. Revoke role; next protected call is 403 using existing unexpired JWT.
4. Invalid/expired JWT -> 401; logout/idle timeout triggers reauthentication.

## Done when
- [ ] OIDC, JWT, RBAC and admin APIs are tested and documented.
- [ ] All protected endpoints have a server-side role gate for the relevant action.
- [ ] First-admin bootstrap and expiration behavior are reproducible.
- [ ] You can demonstrate both a denied request and a permitted request.
