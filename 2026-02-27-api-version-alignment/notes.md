# Research Notes: API Version-Alignment Strategy for Mobile + Backend

## Session: 2026-02-27

### Goal
Investigate version-alignment strategies for a mobile app that depends on a backend API where:
- Changes to either the app or the API can be breaking
- Multiple-version support may be required
- Azure Container Apps is the hosting platform

---

### Sources Checked

1. **API Versioning for Mobile Apps**
   - https://yrkan.com/blog/api-versioning-mobile/ — excellent deep-dive on backward compat constraints for mobile
   - https://www.techneosis.com/insights/best-practices-for-mobile-app-api-versioning/ — practical mobile-specific advice
   - https://dev.to/theodo/mastering-api-versioning-strategies-for-seamless-frontend-backend-communication-in-mobile-apps-12o7
   - https://daily.dev/blog/api-versioning-strategies-best-practices-guide
   - https://nerdleveltech.com/mastering-api-versioning-strategies-tradeoffs-and-best-practices
   - https://calmops.com/api-design/api-versioning-deep-dive/

2. **Azure Container Apps / APIM**
   - https://learn.microsoft.com/en-us/azure/container-apps/revisions — ACA revisions docs (official MS)
   - https://learn.microsoft.com/en-us/azure/container-apps/traffic-splitting — ACA traffic splitting
   - https://learn.microsoft.com/en-us/azure/api-management/api-management-versions — APIM versioning docs
   - https://techcommunity.microsoft.com/blog/appsonazureblog/leveraging-azure-container-apps-labels-for-environment-based-routing-and-feature/4372249
   - https://www.thorsten-hans.com/traffic-split-in-azure-container-apps/

3. **Deprecation Headers / Standards**
   - https://www.rfc-editor.org/rfc/rfc8594 — RFC 8594: `Sunset` header field
   - https://www.ietf.org/archive/id/draft-ietf-httpapi-deprecation-header-05.html — IETF Draft: `Deprecation` header
   - https://blog.axway.com/learning-center/apis/api-management/api-lifecycle-management-deprecation-and-sunsetting
   - https://antler.digital/blog/api-deprecation-best-practices

4. **Contract Testing / OpenAPI**
   - https://www.speakeasy.com/blog/pact-vs-openapi
   - https://yrkan.com/blog/contract-testing-microservices-pact/
   - https://support.smartbear.com/swagger/contract-testing/docs/en/user-guide/contract-testing/bi-directional-contract-testing/compatibility-checks.html
   - https://dasroot.net/posts/2026/02/api-first-development-contract-testing/

---

### Key Decisions Made

**URL-path versioning as primary approach** — chosen because:
- Explicit: easy to debug, log, and route; no hidden headers needed
- Works well with CDNs, proxies, API gateways without extra configuration
- Mobile clients benefit from being able to see the version in logs
- APIM natively supports path-based versioning for routing

**Whole-API versioning over endpoint-level** — because:
- Mobile apps have a contract with the whole API surface; granular endpoint versioning creates a combinatorial explosion of supported "states" that is hard to reason about
- Simpler to document: clients know "I support API v2" not "I support /users v3 but /orders v1"
- App store update cycle means you can manage version bundles more cleanly

**Why we reject: endpoint-level-only versioning without coordination**
- N endpoints × M versions = huge support surface
- Harder to sunset safely because dependency graph between endpoints is not explicit in URLs

**Why we reject: non-versioned API with only backward compatibility + feature flags**
- Feature flags solve rollout, not version-lifecycle management
- Very hard to remove deprecated behaviour when old clients never force-upgrade
- No explicit contract for clients to declare what they depend on

**Support window decision:**
- Time-based window preferred over N-version window for mobile because app store metrics are more reliable than counting "version distance"
- Suggested: 12 months from GA of a new major version (or when <5% of sessions still on old version, whichever is later)

**Azure hosting pattern decision:**
- Separate Container Apps per major API version is the cleanest for full isolation; APIM in front handles routing
- ACA revisions for canary/staging within one version — not for multi-major-version support

**What we explicitly would NOT do:**
- Query-parameter versioning for mobile (caching problems, easy to omit accidentally)
- Versioning via Accept/Content-Type media type alone (too complex for mobile SDKs, hard to debug)
- Keeping unlimited old versions alive indefinitely (operationally unsustainable)
- Big-bang forced upgrades without notice (destroys user trust)

---

### Alternatives Considered but Not Recommended

- **GraphQL** — solves additive evolution naturally but introduces a completely new query model; out of scope for REST API evolution
- **gRPC with protobuf** — strong backward compat primitives but requires significant tooling shift; relevant if greenfield
- **API key bundles (tying app version to an API key)** — elegant but creates key management complexity; covered instead by version declaration in header
- **Date-based versioning** (`api-version=2024-01-15`, Azure style) — valid alternative to semantic integer versioning; noted in appendix as Azure default; our recommendation uses integer major versions for clarity but date-versioning is noted

---

### Open Questions / Caveats Noted

- Force-upgrade policy must align with legal/product constraints (some regulated industries cannot force upgrades)
- App stores have review latency (~1-7 days); version timeline must account for this
- Contract testing with Pact requires mobile team to publish pact files — needs mobile CI pipeline setup
- OpenAPI diff tooling (e.g., `oasdiff`, Speakeasy) should be in CI on every PR that changes the API spec

---

## Session: 2026-02-27 (revision)

### Changes made in response to PR review

**Decision: Replace APIM with Cloudflare Tunnel (`cloudflared`)**
- Reason: APIM has significant cost (Developer tier ~$50/mo, Production tier ~$300/mo+); Cloudflare Tunnel is free/cheap and fits ACA's outbound-connectivity model well
- Referenced article: https://medium.com/@asafshakarzy/deploy-and-protect-azure-container-apps-aca-with-cloudflare-024a42836317 (Medium, restricted; sourced technique from Cloudflare docs directly)
- Key Cloudflare source: https://blog.cloudflare.com/many-services-one-cloudflared/ — path-based multi-service routing with one cloudflared instance
- Ingress rules: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/configure-tunnels/local-management/ingress/

**Architecture changes:**
- `cloudflared-router` Container App with external ingress replaces APIM as the gateway
- `api-v1` and `api-v2` Container Apps are now internal-only ingress (no public IPs)
- Deprecation headers now injected by application middleware (not gateway policy), applied at the Container App level
- Added alternative: hostname-per-version routing (api-v1.example.com, api-v2.example.com) — noted as less ideal since it breaks URL-path versioning convention

**Decision: 410 Gone vs 426 Upgrade Required**
- Researched RFC 9110 sections 15.5.11 (410) and 15.5.27 (426)
- 426 is for protocol-level upgrade negotiation (HTTP version, TLS) — not for API versioning
- 410 is the correct and industry-standard choice (used by Salesforce, Atlassian, GitHub)
- 301/308 redirect noted as valid alternative only if new version is backward-compatible

**Sections removed/replaced:**
- Appendix C: APIM Routing Policy XML → replaced with Cloudflare Tunnel config.yaml
- §6 Pattern C: "APIM Versions + Single Container App" → replaced with "Single Container App with internal version routing" + Cloudflare config
- §3.6 tools table: APIM row → replaced with Cloudflare Tunnel row
- All inline APIM policy references updated to Cloudflare/application-middleware alternatives

