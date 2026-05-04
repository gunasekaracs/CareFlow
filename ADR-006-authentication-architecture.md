# ADR-006: Authentication Architecture for Age-Care Platform

| Field | Detail |
|---|---|
| **ADR Number** | ADR-006 |
| **Title** | Authentication Architecture for Age-Care Platform |
| **Status** | Accepted |
| **Date** | 2026-05-04 |
| **Deciders** | Engineering Team, Product Owner |
| **Reviewed By** | Infrastructure Lead, Age-Care Operations |

---

## Context

The age-care platform has grown beyond the initial room cleaning tracker to encompass four collaborative subsystems:

- **Room Cleaning Tracker** — BLE-based detection of cleaner presence and dwell time
- **Cleaning Audit** — supervisory review and sign-off of cleaning records
- **Laundry Management** — tracking of linen jobs and laundry cycles
- **Laundry Audit** — audit trail for laundry operations and compliance

All four subsystems share a common user base (age-care staff) and a common role model. A single authentication strategy is required so that a user logs in once and can access the subsystems appropriate to their role without re-authenticating.

Additionally, the platform includes a dedicated **Admin Panel** used by age-care administrators to manage users, assign roles, and control access. The requirement is that user management is owned entirely by the admin panel — age-care administrators must not need to interact with any third-party auth UI (e.g. Keycloak's own admin console) to perform day-to-day user operations.

The backend is built on ASP.NET Core. The frontend is a React SPA using PKCE for secure token acquisition. The platform runs in Docker/Kubernetes.

---

## Decision Drivers

- **Single sign-on across all subsystems** — one login session must work across all four services
- **Role-based access control** — distinct roles per subsystem must be enforced at the API level
- **Admin panel owns user management UX** — no exposure of Keycloak admin UI to age-care staff
- **Minimal auth code per service** — each .NET API should not implement its own auth logic
- **Self-hosted** — user credentials and PII must remain on-premises or in a controlled environment, not a third-party SaaS
- **Proven security** — token issuance, refresh rotation, revocation, and brute-force protection must not be hand-rolled
- **Extensible** — must support a future mobile app and potential additional subsystems without re-architecture

---

## Options Considered

### Option 1 — Auth Logic Baked into One API

Authentication logic (login, JWT issuance, refresh) is implemented inside one of the existing .NET APIs (e.g. the cleaning API), with other services validating tokens via a shared secret.

**Pros:**
- Simple to start — no additional service to deploy
- No new infrastructure

**Cons:**
- Does not scale to four independent subsystems — all services must either call the auth API per request (latency, coupling) or share a symmetric secret (security risk)
- All auth security concerns (brute force, token rotation, revocation) must be built from scratch
- Extracting to a proper auth service later requires significant rework

**Verdict:** Rejected. Insufficient for a multi-subsystem platform.

---

### Option 2 — Custom Auth Microservice

A purpose-built .NET auth microservice is developed and deployed. It issues RS256 JWTs. All other services validate tokens using the auth service's public key exposed via a JWKS endpoint.

**Pros:**
- Full control over behaviour and data model
- No external dependency

**Cons:**
- The team owns the entire security surface: token storage, refresh rotation, revocation, brute-force protection, PKCE implementation, JWKS endpoint — all non-trivial to implement correctly
- Significant ongoing maintenance burden
- Effectively re-implementing what Keycloak already provides

**Verdict:** Rejected. The security risk of hand-rolling auth outweighs the control benefit at this scale.

---

### Option 3 — Cloud Identity Provider (Auth0 / Azure AD B2C)

Delegate authentication entirely to a managed cloud IdP. The React SPA authenticates via the IdP; .NET APIs validate tokens using the IdP's public key.

**Pros:**
- Zero infrastructure to manage
- Enterprise-grade security out of the box

**Cons:**
- Vendor lock-in — migrating away later is costly
- Age-care staff PII (names, credentials) is stored with a third-party vendor
- Cost scales with active users
- Less control over token claims and user attribute schema

**Verdict:** Rejected. Data sovereignty and vendor dependency concerns outweigh the operational convenience.

---

### Option 4 — Keycloak (Self-Hosted) ✅ Selected

Deploy Keycloak as a self-hosted identity provider within the platform's Docker/Kubernetes environment. Keycloak acts as the sole token issuer for all four subsystems. The React SPA authenticates via PKCE. Each .NET API validates JWTs locally using Keycloak's published JWKS endpoint — no Keycloak call is made per request.

The Admin Panel communicates with Keycloak exclusively via the **Keycloak Admin REST API**, proxied through a dedicated Admin .NET API. Hotel administrators manage users and roles entirely within the Admin Panel UI — the Keycloak admin console is not exposed to or used by hotel staff.

User data is split across two stores:
- **Keycloak** — credentials, roles, enabled/disabled status, email
- **Platform database** — hotel domain attributes (employee ID, floor assignment, department, hire date), linked to Keycloak by the Keycloak user UUID

**Pros:**
- Battle-tested auth engine — token issuance, PKCE, refresh rotation, revocation, brute-force protection all built in
- Single realm covers all four subsystems with no re-authentication
- RBAC built in — roles embedded in JWT, validated locally by each .NET API
- Admin Panel retains full ownership of user management UX via Admin REST API
- Self-hosted — no PII leaves the controlled environment
- Free and open source
- Runs in Docker/Kubernetes alongside existing services
- Future mobile app and additional subsystems slot in with no auth re-architecture

**Cons:**
- Keycloak requires ~512MB RAM minimum — infrastructure overhead compared to a lightweight custom service
- Configuration learning curve (realms, clients, scopes)
- Admin REST API integration adds implementation effort to the Admin Panel

**Verdict:** Accepted. Keycloak provides enterprise-grade auth with full data sovereignty, covers all four subsystems under a single realm, and allows the Admin Panel to own the user management UX without exposing Keycloak internals to age-care staff.

---

## Decision

**Keycloak (self-hosted)** is selected as the authentication provider for the age-care platform.

---

## Architecture

### High-Level Flow

```
React SPA (Admin Panel / Subsystem UIs)
        │
        ├──→  Keycloak              PKCE login, token issuance, refresh
        │
        ├──→  Admin .NET API        User/role management via Keycloak Admin REST API
        ├──→  Cleaning .NET API     JWT validation via JWKS endpoint
        ├──→  Audit .NET API        JWT validation via JWKS endpoint
        ├──→  Laundry .NET API      JWT validation via JWKS endpoint
        └──→  Laundry Audit API     JWT validation via JWKS endpoint
```

### User Management Flow (Admin Panel)

All user management operations originate from the Admin Panel and are proxied through the Admin .NET API, which holds the Keycloak service account credentials. The React frontend never communicates with Keycloak's admin API directly.

```
Admin Panel  →  Admin .NET API  →  Keycloak Admin REST API  (identity + roles)
                     │
                     └──────────→  Platform DB               (age-care domain attributes)
```

On user creation, the Admin API:
1. Creates the user in Keycloak via `POST /admin/realms/age-care/users`
2. Assigns the appropriate realm role via `POST /admin/realms/age-care/users/{id}/role-mappings/realm`
3. Persists age-care domain attributes to the platform database, linked by the Keycloak-issued UUID
4. On failure at step 3, executes a compensating transaction to delete the Keycloak user

### Token Validation (Per .NET API)

Each subsystem API validates incoming JWTs locally without calling Keycloak on every request. Keycloak's public keys are fetched once at startup and cached, refreshed automatically via the JWKS endpoint.

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://keycloak.internal/realms/age-care";
        options.Audience  = "age-care-platform";
        // Public keys fetched automatically from Keycloak's JWKS endpoint
    });
```

### Token Specification

| Property | Value |
|---|---|
| Algorithm | RS256 (asymmetric — each service needs only the public key) |
| Access token lifetime | 15 minutes |
| Refresh token lifetime | 7 days, rotation enabled |
| Refresh token reuse | Not permitted — each use issues a new refresh token and invalidates the old |
| Roles claim | `realm_access.roles` embedded in JWT |
| Frontend storage | Access token in memory; refresh token in httpOnly cookie |

---

## Role Model

A single set of realm roles covers all four subsystems. Each .NET API enforces the subset of roles relevant to its operations.

| Role | Cleaning Tracker | Cleaning Audit | Laundry Mgmt | Laundry Audit | Admin Panel |
|---|---|---|---|---|---|
| `agecare_admin` | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ Full |
| `agecare_manager` | ✅ Read/Write | ✅ Read | ✅ Read/Write | ✅ Read | ❌ |
| `floor_supervisor` | ✅ Read/Write | ✅ Read | ❌ | ❌ | ❌ |
| `cleaner` | ✅ Own records | ❌ | ❌ | ❌ | ❌ |
| `laundry_staff` | ❌ | ❌ | ✅ Read/Write | ❌ | ❌ |
| `auditor` | ❌ | ✅ Read | ❌ | ✅ Read | ❌ |

---

## User Data Split

| Attribute | Store | Rationale |
|---|---|---|
| Email, password hash | Keycloak | Credential management is Keycloak's responsibility |
| Realm roles | Keycloak | Embedded in JWT at token issuance |
| Enabled / disabled flag | Keycloak | Controls login access and active sessions |
| Employee ID | Platform DB | Age-care domain concept, not an auth attribute |
| Floor assignment | Platform DB | Operational data belonging to the platform domain |
| Department | Platform DB | Age-care organisational structure |
| Hire date | Platform DB | HR data — no auth relevance |
| Keycloak UUID | Platform DB (FK) | Foreign key linking the two stores |

---

## Deployment

Keycloak is deployed as a Docker service within the existing Kubernetes cluster. It requires its own persistent database (PostgreSQL recommended). Keycloak is not exposed to the public internet — it is accessible only within the cluster and via the internal network.

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-db:5432/keycloak
      KC_HOSTNAME: keycloak.internal
    command: start
    depends_on:
      - keycloak-db

  keycloak-db:
    image: postgres:16
    volumes:
      - keycloak-db-data:/var/lib/postgresql/data
```

---

## Consequences

**Positive:**
- Age-care staff experience seamless single sign-on across all four subsystems
- Each .NET API enforces roles from the JWT with no inter-service auth calls at runtime
- The Admin Panel provides a complete, age-care-branded user management experience — age-care administrators have no exposure to Keycloak internals
- Adding a fifth subsystem or a mobile app requires no changes to the auth architecture
- All user credentials and PII remain within the age-care facility's controlled infrastructure

**Negative:**
- Keycloak adds infrastructure overhead (~512MB RAM, PostgreSQL dependency)
- The Admin API must implement compensating transactions for the dual-write (Keycloak + platform DB) on user creation and deletion
- Keycloak realm and client configuration must be version-controlled and managed as infrastructure-as-code to avoid configuration drift

---

## Links and References

- Keycloak documentation: https://www.keycloak.org/documentation
- Keycloak Admin REST API: https://www.keycloak.org/docs-api/latest/rest-api/
- Microsoft.AspNetCore.Authentication.JwtBearer: https://learn.microsoft.com/en-us/aspnet/core/security/authentication/
- ADR-002: Indoor positioning technology (BLE Beacons selected)
- ADR-003: MQTT broker hosting (Stackhero selected)
- ADR-004: BLE scanning approach (Raspberry Pi selected)
- ADR-005: Cleaner wearable tag (GAO RFID Key Fob selected)

---

*ADR-006 | Status: Accepted | Last updated: 2026-05-04*
