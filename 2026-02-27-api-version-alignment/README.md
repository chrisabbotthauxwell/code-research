# API Version-Alignment Strategy for Mobile App + Backend API

<!-- AI-GENERATED-NOTE-->
> [!NOTE]
> This is an AI-generated research report. All text and code in this report was created by an LLM (Large Language Model). For more information on how these reports are created, see the [main research repository](https://github.com/simonw/research).
<!-- /AI-GENERATED-NOTE-->

---

## 1. Executive Summary

Adopt **URL-path-based whole-API versioning** (`/v1/`, `/v2/`) as the primary versioning scheme. Mobile clients embed the API version they were built for in the URL prefix; the server maintains all currently-supported major versions in parallel. A new major version is released only for breaking changes; all additive, non-breaking changes are deployed in place to the current version. Support any major version for **12 months** after the next major version ships, or until that old version drops below 5 % of active sessions, whichever is later. Clients signal the version they need in the URL; the server signals deprecation and end-of-life dates using the standard `Deprecation` and `Sunset` HTTP response headers. When a client calls a version past its end-of-life, the API returns `HTTP 410 Gone` with a structured error body. On Azure Container Apps, run each live major-version's API as a **separate Container App** and place **Azure API Management (APIM)** in front to handle path-routing, observability, and policy enforcement. Canary/staged rollouts *within* a version use ACA's built-in revision traffic-splitting. Do **not** use query-parameter versioning, endpoint-level-only versioning, or unversioned API-with-flags as the sole strategy.

---

## 2. Assumptions & Goals

| Assumption | Detail |
|---|---|
| **Mobile release cadence** | App store releases every 2–6 weeks; app store review adds 1–7 days latency. |
| **Offline / slow-upgrade users** | A meaningful proportion of users delay upgrading. Targeting zero forced-upgrades except for critical security issues. |
| **Forced-upgrade constraint** | Force-upgrade is a last resort; we prefer a generous deprecation window and clear in-app messaging instead. |
| **App store constraints** | Both iOS App Store and Google Play are in scope; neither guarantees instant delivery to all devices. |
| **Backend deployment** | Azure Container Apps; Azure API Management (APIM) may already be in use or can be added. |
| **Breaking change definition** | Any change that would cause an existing client (running the current production app) to fail silently or error. This includes: removing fields, changing field types, removing endpoints, changing auth requirements, changing error shapes. |
| **Non-breaking change** | Additive: new optional fields, new endpoints, new optional query parameters, relaxed validation. |
| **Teams** | One mobile team (iOS + Android) and one backend team; contract between them is the API. |

**Goals:**
- Mobile users are never unexpectedly broken by a server-side deployment.
- The backend team can evolve the API without being blocked indefinitely by old clients.
- Clients always know which version they are talking to.
- Operations remains tractable: we don't maintain more than ~2 major versions at a time.
- The deprecation and end-of-life timeline is predictable and communicated in-band.

---

## 3. Options Evaluated

### 3.1 Whole-API URL-Path Versioning (Recommended)

Every request is prefixed with the API version: `https://api.example.com/v2/orders/42`.

**Pros:**
- Completely explicit — visible in logs, proxy access logs, APIM analytics, crash reports.
- Routes trivially in any gateway or reverse proxy.
- Mobile clients hard-code the version at build time; no runtime version negotiation needed.
- Works perfectly with CDN caching (version is part of the cache key).
- Easy to sunset a whole version atomically.

**Cons:**
- URL "proliferates" with each major version (minor issue in practice).
- REST purists note the resource identifier includes a non-resource concept (version).
- Requires copying/forking route handlers for each major version (mitigated by code-sharing patterns).

### 3.2 Endpoint/Resource-Level Versioning

Individual endpoints are versioned independently: `GET /users/v2/profile` while `GET /v1/orders` remains on v1.

**Pros:**
- Fine-grained: only changed endpoints get a new version number.
- Consumers that don't use the changed endpoint are unaffected.

**Cons:**
- **Combinatorial explosion**: a mobile app depends on 20 endpoints; if each can be on a different version, the total "support state" is 20-dimensional and impossible to reason about.
- Harder to document: "which version of the whole API does app v4.2 use?" has no single answer.
- Sunset logic is per-endpoint; operational complexity multiplies.
- ❌ **Not recommended** as the primary strategy; acceptable only for truly orthogonal microservices.

### 3.3 Non-Versioned API with Strong Backward Compatibility + Feature Flags

No version in the URL. The API evolves in place; backward compatibility is guaranteed forever (or feature flags gate new behaviour).

**Pros:**
- Single URL surface; clients never need to update base URLs.
- Feature flags enable safe rollouts to specific users/cohorts.

**Cons:**
- **Cannot remove deprecated fields/endpoints** without breaking someone — forces accumulation of dead code.
- No contractual version for a client to declare: the client cannot say "I only rely on the API as it existed in month X".
- Feature flags solve rollout, not lifecycle management.
- When the API *must* break (e.g., auth refactor, schema change), there is no clean mechanism.
- ❌ **Not recommended** as the primary strategy; feature flags are a *complement* to versioning, not a replacement.

### 3.4 Header-Based Versioning

Clients pass a custom header: `X-API-Version: 2` or use content negotiation: `Accept: application/vnd.myapp.v2+json`.

**Pros:**
- Clean URLs.
- Enables per-request version selection (powerful for experimentation).

**Cons:**
- Version is invisible in logs and APIM analytics without extra configuration.
- Mobile SDK complexity: every HTTP call must attach the header; easy to forget.
- Content-negotiation media-type versioning is poorly supported by many HTTP clients and testing tools.
- Not a good default for mobile where debugging in the field is already hard.
- ❌ **Not recommended as the primary approach.** Can be used as a *secondary* signal (see §4).

### 3.5 Date-Based Versioning (Azure Pattern)

Azure's own APIs use date-string versions: `api-version=2024-11-01`. Stripe uses a similar scheme.

**Pros:**
- Communicates *when* a version was frozen, which is meaningful for changelog reading.
- Each date-stamp is a full snapshot of the API contract.

**Cons:**
- Less intuitive for mobile developers who think in app version numbers.
- String comparison ("is 2024-11-01 newer than 2024-03-15?") is more error-prone than integer comparison.
- Still requires a migration path identical to integer URL versioning.

**Verdict:** Valid alternative; recommended only if the team already uses Azure-style date versioning elsewhere. For new projects, integer major-version URL-path versioning is clearer.

### 3.6 Relevant Standards and Tools

| Tool / Standard | Purpose |
|---|---|
| **Semantic Versioning (semver.org)** | Convention for expressing breaking vs non-breaking changes in version numbers. For REST APIs, only *major* version increments are exposed in the URL path. |
| **RFC 8594 — `Sunset` header** | Standard HTTP response header for announcing endpoint retirement date. |
| **IETF draft — `Deprecation` header** | Standard header for marking a response as deprecated (not yet RFC but widely adopted). |
| **OpenAPI 3.x** | Machine-readable API contract; enables automated diff and breaking-change detection. |
| **`oasdiff` / `openapi-diff`** | CLI tools that compare two OpenAPI specs and flag breaking changes; integrate in CI. |
| **Pact (pact.io)** | Consumer-driven contract testing framework; mobile app publishes a Pact file that backend CI must pass. |
| **Azure API Management (APIM)** | Managed gateway with native version and revision support, policy engine, analytics. |
| **Azure Container Apps revisions** | Immutable deployment snapshots with percentage-weighted traffic splitting; enables canary deployments within a version. |

---

## 4. Recommendation

### Primary approach: URL-path whole-API major versioning + APIM gateway

```
https://api.example.com/v{N}/resource/id
```

- **Major version** (`v1`, `v2`, …) increments on any breaking change.
- **All non-breaking changes** (new fields, new endpoints, relaxed validation) ship to the current major version immediately.
- The mobile app hard-codes the version prefix at build time; there is no runtime version negotiation.
- APIM sits in front of all Container Apps and routes `/v1/` → `api-v1` Container App, `/v2/` → `api-v2` Container App.
- **Also include the app version in a request header** (`X-App-Version: 4.2.1`) for server-side observability and analytics, but this header does *not* affect routing.

### What we would NOT do and why

| ❌ Rejected approach | Reason |
|---|---|
| Query-parameter versioning (`?api-version=2`) | Accidentally omittable by client code; CDN caching varies; not idiomatic for mobile |
| Accept-header / media-type versioning only | Hidden from logs; complex for mobile HTTP clients; poor tooling support |
| Endpoint-level versioning as sole strategy | Combinatorial support complexity; no whole-app version contract |
| No versioning + backward-compat-only | Cannot remove deprecated behaviour; accumulates technical debt indefinitely |
| Date-based versioning (as default) | Less intuitive for mobile teams; integer major versions are simpler |
| Infinite version support | Operationally unsustainable; target max 2 live major versions at a time |

---

## 5. Compatibility & Deprecation Policy

### 5.1 How Clients Declare the Required API Version

Clients embed the version in the URL path. No additional header is required for routing. Optionally, clients also send:

```http
X-App-Version: 4.2.1
X-App-Platform: ios
```

These headers are logged by APIM for analytics (which app version is calling which API version) but do not influence routing.

### 5.2 How the Server Signals Version, Deprecation, and End-of-Life

All responses from a deprecated API version MUST include:

```http
Deprecation: Sun, 01 Jun 2025 00:00:00 GMT
Sunset: Sun, 01 Jun 2026 00:00:00 GMT
Link: <https://api.example.com/v2/>; rel="successor-version",
      <https://developer.example.com/migrate-v1-to-v2>; rel="deprecation"
```

- **`Deprecation`** (IETF draft, widely implemented): date when this version was declared deprecated.
- **`Sunset`** (RFC 8594): date when the API version will cease to respond (returns 410).
- **`Link`** header: points to the successor version URL and a migration guide page.

These headers must be emitted by APIM policy (not application code) so they are applied consistently.

### 5.3 Suggested Support Window

| Phase | Duration | Action |
|---|---|---|
| **Current** | Until next major version ships | Fully supported; new non-breaking features added here. |
| **Deprecated** | 12 months after next major version ships | All requests receive `Deprecation` + `Sunset` headers. In-app upgrade banners shown based on `Sunset` date returned in a response envelope field. |
| **End-of-life** | After Sunset date | API returns `410 Gone` with error body (see §5.4). |
| **Exception** | If sessions on old version drop below 5 % before 12 months | May retire earlier with 60 days additional notice. |

A maximum of **2 major versions** live at any one time (current + one deprecated). If a v3 ships, v1 is immediately moved to its end-of-life schedule regardless of the v1→v2 deprecation window (with emergency notice to any remaining v1 clients).

### 5.4 What Happens When a Client Calls an Unsupported Version

**HTTP Status: `410 Gone`**

```json
{
  "error": {
    "code": "API_VERSION_RETIRED",
    "message": "API version v1 has been retired. Please upgrade to v2.",
    "sunset_date": "2026-06-01T00:00:00Z",
    "successor_version_url": "https://api.example.com/v2/",
    "migration_guide_url": "https://developer.example.com/migrate-v1-to-v2",
    "support_contact": "support@example.com"
  }
}
```

`410 Gone` is preferred over `404 Not Found` because it is permanent and semantically correct (the resource existed and will not return). Mobile clients must handle this response and show the user an upgrade prompt.

**User messaging strategy:**
1. **In-app banner (warning):** Starting 90 days before Sunset, show a soft banner: "A new version of this app is available with improvements. Please update."
2. **In-app modal (required action):** 30 days before Sunset, show a non-dismissable upgrade prompt if the app detects the `Sunset` header is within 30 days (read the header from any API response and store it).
3. **Blocked state:** After Sunset, the app receives `410 Gone`, shows a blocking screen: "This version of the app is no longer supported. Please update in the App Store."

---

## 6. Azure Container Apps: Hosting Multiple API Versions in Parallel

### Pattern A: Separate Container App per Major Version (Recommended)

Deploy each major API version as an independent Container App within the same Container Apps Environment.

```
ACA Environment: api-env
├── Container App: api-v1  (image: api:1.x.y)
│   └── Revision: api-v1--abc123 (active, 100% traffic)
└── Container App: api-v2  (image: api:2.x.y)
    ├── Revision: api-v2--def456 (stable, 90% traffic)
    └── Revision: api-v2--ghi789 (canary, 10% traffic)

APIM:
├── API: MyApp v1  →  backend: https://api-v1.internal.env.azurecontainerapps.io
└── API: MyApp v2  →  backend: https://api-v2.internal.env.azurecontainerapps.io
```

**Routing in APIM:**

Configure APIM with two APIs, each with its own version prefix:

```
https://api.example.com/v1/* → proxies to api-v1 Container App
https://api.example.com/v2/* → proxies to api-v2 Container App
```

APIM inbound policy strips `/v1` from the URL before forwarding to the Container App (or the app handles it natively):

```xml
<!-- APIM inbound policy for v1 API -->
<set-backend-service base-url="https://api-v1.internal.env.azurecontainerapps.io" />
<rewrite-uri template="@(context.Request.Url.Path.Replace("/v1", ""))" />
```

**Pros:**
- Complete isolation: v1 and v2 have independent scaling, resource limits, and deployment pipelines.
- Rollback is trivial: redeploy the v1 image tag without touching v2.
- Different Node/runtime versions, different dependencies possible per major version.
- Clear cost attribution per version (separate Container App billing).

**Cons:**
- Operational overhead: two Container Apps to monitor, scale-configure, and manage secrets for.
- If code is largely shared, maintaining two repo branches can diverge; mitigate with shared library packages.

**Operational considerations:**

| Concern | Approach |
|---|---|
| Cost | Scale v1 (deprecated) to minimum replicas (0–1) to reduce cost; alert if v1 usage spikes unexpectedly |
| Routing | APIM policies; no code change required to add/remove a version |
| Monitoring | Use APIM's built-in analytics to track per-version call volumes and error rates |
| Rollback | Re-activate a previous ACA revision (`az containerapp revision activate`) or redeploy from a pinned image tag |
| Secrets | Use managed identity + Azure Key Vault references; each Container App gets its own managed identity |

---

### Pattern B: ACA Revisions with Traffic Splitting (for Canary Within One Version)

Use this pattern for **canary deployments and staged rollouts within a single major version**, not for supporting multiple major API versions simultaneously.

```bash
# Enable multiple-revision mode
az containerapp update \
  --name api-v2 \
  --resource-group my-rg \
  --revisions-mode multiple

# Deploy new canary revision (20% traffic)
az containerapp update \
  --name api-v2 \
  --image api:2.5.0-rc1 \
  --revision-suffix "rc1"

az containerapp ingress traffic set \
  --name api-v2 \
  --resource-group my-rg \
  --revision-weight \
      api-v2--stable=80 \
      api-v2--rc1=20
```

ACA assigns a unique URL to each revision: `https://api-v2--rc1.internal.env.azurecontainerapps.io`. This can be used to pin specific integration tests against the canary.

**Pros:**
- Zero-downtime deploys; instant rollback by shifting weight back to stable.
- No APIM changes required; traffic splitting is transparent to clients.
- ACA handles the load balancing natively.

**Cons:**
- Not suitable for hosting different *major* API versions (the canary and stable must be contract-compatible).
- Traffic weights are approximate (not user-sticky by default without session affinity).
- Maximum 100 active revisions per Container App; old revisions should be deactivated.

---

### Pattern C: APIM Versions + Single Container App (Lightweight Alternative)

If isolation between major versions is not required (shared codebase, internal versioning), use APIM's native version groups and a single Container App that handles both `/v1/` and `/v2/` internally.

```
APIM Version Group: MyApp
├── Version: v1  (URL path: /v1)
└── Version: v2  (URL path: /v2)
         ↓
   Single Container App: api (handles both versions internally)
```

**Pros:** Simpler to operate; one Container App; one CI/CD pipeline.

**Cons:** Versions share resources (a bug in v2 can take down v1); harder to retire v1 independently; requires disciplined code structure to avoid v1/v2 bleed.

**Recommendation:** Use Pattern C only during the early stages of a project when the versioning surface is small; migrate to Pattern A as the API matures.

---

## 7. Implementation Checklist

### Phase 1: Foundation (Sprint 1–2)

- [ ] Adopt OpenAPI 3.x for all API endpoints; commit spec file to source control (`openapi.yaml`).
- [ ] Add `oasdiff` or `openapi-diff` to CI pipeline; PR must not introduce breaking changes without a major version bump.
- [ ] Establish URL-path versioning convention: all routes prefixed `/v{N}/`.
- [ ] Add `X-App-Version` and `X-App-Platform` header logging in APIM.
- [ ] Define "breaking change" formally in the API team's contributing guide.

### Phase 2: Deprecation Infrastructure (Sprint 3–4)

- [ ] Implement APIM inbound/outbound policies to inject `Deprecation` and `Sunset` headers for any API marked deprecated.
- [ ] Add `sunset_date` and `successor_version_url` fields to a standard API error envelope.
- [ ] Mobile app: read `Sunset` header from any API response; store it locally; drive in-app upgrade prompt UI.
- [ ] Mobile app: handle `410 Gone` gracefully with a blocking upgrade screen.

### Phase 3: Multi-Version Hosting on ACA (Sprint 5–6)

- [ ] Configure ACA in multi-revision mode for canary deployments within each major version.
- [ ] Set up APIM with separate backend pools pointing to `api-v1` and `api-v2` Container Apps.
- [ ] Set APIM routing policies: strip `/v1` prefix before forwarding to `api-v1` backend.
- [ ] Configure ACA minimum replicas: current version ≥ 2; deprecated version = 1 (scale-to-zero allowed with cold-start warning).
- [ ] Set up Azure Monitor alerts on per-version error rates and call volumes.

### Phase 4: Contract Testing (Sprint 7–8)

- [ ] Mobile team: integrate Pact into iOS and Android CI; publish pact files to PactFlow (or self-hosted Pact Broker).
- [ ] Backend CI: add Pact provider verification step; block deploy if any consumer contract is broken.
- [ ] Add `can-i-deploy` check to backend release pipeline.

### Phase 5: Deprecation Process (Ongoing)

- [ ] When a new major version ships, update APIM policy for the old version to emit `Deprecation` + `Sunset` headers.
- [ ] Post migration guide to developer portal; email registered developer accounts.
- [ ] Monitor old-version call volume weekly; confirm it is declining.
- [ ] At Sunset date: update APIM policy to return `410 Gone` for all requests to the old version.
- [ ] Deactivate (do not delete) old Container App for 30 days post-sunset; delete after.

---

## 8. Appendix

### A. Example Request/Response Patterns

#### URL-path versioning (recommended)

**Request — current version:**
```http
GET /v2/orders/42 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
X-App-Version: 5.1.0
X-App-Platform: ios
```

**Response — current version:**
```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Request-Id: a1b2c3d4

{
  "id": 42,
  "status": "shipped",
  "estimated_delivery": "2026-03-05"
}
```

---

**Request — deprecated version:**
```http
GET /v1/orders/42 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
X-App-Version: 3.8.2
X-App-Platform: android
```

**Response — deprecated version (headers injected by APIM):**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Deprecation: Sat, 01 Mar 2025 00:00:00 GMT
Sunset: Tue, 01 Sep 2026 00:00:00 GMT
Link: <https://api.example.com/v2/>; rel="successor-version",
      <https://developer.example.com/migrate-v1-to-v2>; rel="deprecation"

{
  "id": 42,
  "status": "shipped"
}
```

---

**Request — retired version (post-sunset):**
```http
GET /v1/orders/42 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
```

**Response — 410 Gone:**
```http
HTTP/1.1 410 Gone
Content-Type: application/json

{
  "error": {
    "code": "API_VERSION_RETIRED",
    "message": "API version v1 has been retired. Please upgrade to v2.",
    "sunset_date": "2026-02-01T00:00:00Z",
    "successor_version_url": "https://api.example.com/v2/",
    "migration_guide_url": "https://developer.example.com/migrate-v1-to-v2",
    "support_contact": "support@example.com"
  }
}
```

---

#### Header-based versioning (NOT recommended as primary; shown for comparison)

```http
GET /orders/42 HTTP/1.1
Host: api.example.com
X-API-Version: 2
Accept: application/vnd.myapp.v2+json
```

This approach hides the version from logs and requires every mobile HTTP call to include the header correctly. Rejected in favour of URL-path versioning.

---

#### Date-based versioning (Azure style, alternative)

```http
GET /orders/42?api-version=2026-01-15 HTTP/1.1
Host: api.example.com
```

Used natively by Azure's own ARM APIs. Valid approach but less intuitive for most mobile teams.

---

### B. Azure CLI Snippets

```bash
# Deploy api-v1 Container App
az containerapp create \
  --name api-v1 \
  --resource-group my-rg \
  --environment api-env \
  --image myregistry.azurecr.io/api:1.12.0 \
  --ingress internal \
  --target-port 8080 \
  --min-replicas 1 \
  --max-replicas 10

# Deploy api-v2 Container App  
az containerapp create \
  --name api-v2 \
  --resource-group my-rg \
  --environment api-env \
  --image myregistry.azurecr.io/api:2.3.0 \
  --ingress internal \
  --target-port 8080 \
  --min-replicas 2 \
  --max-replicas 20

# Enable multiple revisions for canary deployments on v2
az containerapp update \
  --name api-v2 \
  --resource-group my-rg \
  --revisions-mode multiple

# Set canary traffic split (90/10)
az containerapp ingress traffic set \
  --name api-v2 \
  --resource-group my-rg \
  --revision-weight \
      api-v2--stable=90 \
      api-v2--canary=10
```

---

### C. APIM Routing Policy (v1 → api-v1 Container App)

```xml
<policies>
  <inbound>
    <base />
    <!-- Route to the api-v1 Container App -->
    <set-backend-service base-url="https://api-v1.internal.env.azurecontainerapps.io" />
    <!-- Strip /v1 prefix so the Container App sees /orders/42 not /v1/orders/42 -->
    <rewrite-uri template="@(context.Request.Url.Path.Replace("/v1", ""))" />
  </inbound>
  <outbound>
    <base />
    <!-- Inject deprecation headers on all v1 responses -->
    <set-header name="Deprecation" exists-action="override">
      <value>Sat, 01 Mar 2025 00:00:00 GMT</value>
    </set-header>
    <set-header name="Sunset" exists-action="override">
      <value>Tue, 01 Sep 2026 00:00:00 GMT</value>
    </set-header>
    <set-header name="Link" exists-action="override">
      <value>&lt;https://api.example.com/v2/&gt;; rel="successor-version", &lt;https://developer.example.com/migrate-v1-to-v2&gt;; rel="deprecation"</value>
    </set-header>
  </outbound>
</policies>
```

---

### D. References

| Source | URL |
|---|---|
| RFC 8594: The Sunset HTTP Header Field | https://www.rfc-editor.org/rfc/rfc8594 |
| IETF Draft: The Deprecation HTTP Header Field | https://www.ietf.org/archive/id/draft-ietf-httpapi-deprecation-header-05.html |
| Azure Container Apps — Revisions | https://learn.microsoft.com/en-us/azure/container-apps/revisions |
| Azure Container Apps — Traffic Splitting | https://learn.microsoft.com/en-us/azure/container-apps/traffic-splitting |
| Azure API Management — Versions | https://learn.microsoft.com/en-us/azure/api-management/api-management-versions |
| ACA Labels for Environment-Based Routing | https://techcommunity.microsoft.com/blog/appsonazureblog/leveraging-azure-container-apps-labels-for-environment-based-routing-and-feature/4372249 |
| API Versioning Strategy for Mobile Clients | https://yrkan.com/blog/api-versioning-mobile/ |
| Best Practices for Mobile App API Versioning | https://www.techneosis.com/insights/best-practices-for-mobile-app-api-versioning/ |
| Mastering API Versioning (nerdleveltech) | https://nerdleveltech.com/mastering-api-versioning-strategies-tradeoffs-and-best-practices |
| API Versioning Strategies (Java Guides) | https://www.javaguides.net/2024/12/api-versioning-strategies.html |
| Pact vs OpenAPI (Speakeasy) | https://www.speakeasy.com/blog/pact-vs-openapi |
| API-First Development & Contract Testing | https://dasroot.net/posts/2026/02/api-first-development-contract-testing/ |
| API Deprecation Best Practices (Antler Digital) | https://antler.digital/blog/api-deprecation-best-practices |
| API Lifecycle Management: Deprecation and Sunsetting (Axway) | https://blog.axway.com/learning-center/apis/api-management/api-lifecycle-management-deprecation-and-sunsetting |
| Traffic Splitting in ACA (Thorsten Hans) | https://www.thorsten-hans.com/traffic-split-in-azure-container-apps/ |
| Semantic Versioning | https://semver.org/ |
| oasdiff (OpenAPI diff CLI) | https://github.com/Tufin/oasdiff |
| Pact consumer-driven contract testing | https://pact.io/ |
