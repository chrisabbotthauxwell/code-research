# SwiftUI → Android: Shipping Native Kotlin/Jetpack Compose Fast

**Research date:** 2026-02-27  
**Context:** Existing iOS app (SwiftUI + Azure HTTP API). iOS-only team. Goal: ship Android MVP quickly with good UX parity. Long-term plan: separate native codebases (SwiftUI + Kotlin/Compose).

---

## 1. Executive Summary

Rebuild the Android app natively in **Kotlin + Jetpack Compose**. Do not attempt a wholesale automatic conversion of SwiftUI screens; instead, use LLM assistance (Cursor/Claude/GPT-4) selectively for mechanical translation of models, DTOs, and Retrofit interfaces, then write each screen intentionally in idiomatic Compose. This approach gives the iOS team the smallest conceptual ramp-up because Compose's declarative/state-hoisting paradigm maps almost 1-to-1 to SwiftUI's.

**Do NOT:**
- Attempt to auto-convert SwiftUI Views to Composables wholesale — navigation, lifecycle, and state restoration semantics differ too much for LLM output to be production-safe without screen-by-screen review.
- Introduce Kotlin Multiplatform (KMP) in the first phase — it adds an extra build-system layer before the team has Android confidence.
- Choose Flutter or React Native — they introduce Dart or JavaScript, a third language, and non-native UI rendering.

**First 2 weeks pilot (deliverable-driven):**
| Day | Deliverable |
|-----|-------------|
| 1–2 | Android Studio project created, Hilt wired, `AppNavHost` skeleton with 3 stub screens |
| 3–4 | Retrofit client + OkHttp interceptor for bearer token auth; Kotlin models from existing Swift DTOs |
| 5–6 | First real screen: read-only data list from Azure API (ViewModel + Repository + UI) |
| 7–8 | Auth flow: MSAL login, token storage in `EncryptedSharedPreferences`, automatic token refresh |
| 9–10 | Second screen: detail view (navigation + back-stack working end-to-end) |
| 11–12 | Third screen: first mutation (POST/PUT to API with loading/error states) |
| 13–14 | Firebase Crashlytics wired; Compose UI test for each screen; internal TestFlight-equivalent (Firebase App Distribution) build |

The pilot is complete when three real vertical flows work end-to-end on a physical device and an internal build has been shared for review.

---

## 2. Context, Constraints, and Assumptions

| Item | Current state |
|------|---------------|
| iOS app | SwiftUI, production; communicates with Azure-hosted REST/JSON API |
| Android app | Does not exist |
| Team | iOS engineers only; Kotlin/Android experience: none or minimal |
| AI tooling | Cursor (Claude/GPT-4 backend) already in use for iOS development |
| Android target | Kotlin + Jetpack Compose; separate codebase from iOS forever |
| Parity goal | Staged: MVP first (core flows), then iterate to full parity |
| Offline-first | Not required now; noted in Appendix where it would change |
| Auth | Assumed: Azure AD / Microsoft Entra ID bearer tokens (confirm before Phase 1) |
| API contract | REST/JSON assumed; OpenAPI spec existence unconfirmed — priority to check |

**Key assumption:** The Azure API is a standard HTTP REST/JSON API with bearer token authentication. The research recommends MSAL for Android as the auth library; if the API uses a custom JWT or API-key scheme, auth implementation simplifies further but MSAL is still the right choice for Azure AD-backed systems.

---

## 3. Options Evaluated

### A. Rebuild Android Natively in Kotlin + Jetpack Compose ✅ *Primary recommendation*

Build the Android app from scratch in Kotlin with Jetpack Compose for UI. No code is shared between iOS and Android at the source level; the API contract is the integration point.

**Why it wins for this team:**
- Compose's declarative, state-driven model is the closest thing Android has to SwiftUI. The iOS team's mental model transfers with modest adaptation.
- Google's officially recommended path; largest community; best LLM training data; most StackOverflow coverage.
- Architecture pattern (MVVM, ViewModel, Repository) maps directly to ObservableObject + Combine patterns the team already knows.
- The resulting codebase is fully idiomatic Android — future Android hires can work in it without friction.

**Tradeoffs:** Full rebuild of every screen. No shared code with iOS. However, this is explicit in the long-term plan and is not a tradeoff for this team.

### B. Kotlin Multiplatform (KMP) — share networking/models ⏳ *Worth evaluating at Phase 3*

KMP allows sharing Kotlin code (models, repositories, networking) across Android and iOS, with native UI on each platform (Compose for Android, SwiftUI for iOS). As of 2024–2025, Compose Multiplatform (the UI layer) is also available but not yet production-stable for all use cases.

**Evaluate now?** No. KMP is compelling but adds a second learning curve (KMP gradle build system, `expect`/`actual` declarations, Xcode integration) before the team has Android confidence. The shared-networking benefit is real but the iOS app already works and has its own tested networking layer. Revisit in Phase 3 when the team is comfortable with Android and there is evidence of API model drift between platforms.

**References:**
- [Kotlin Multiplatform docs](https://kotlinlang.org/docs/multiplatform.html)
- [KMP vs React Native comparison (official)](https://kotlinlang.org/docs/multiplatform/kotlin-multiplatform-react-native.html)

### C. Cross-platform rewrite: Flutter ❌

Flutter uses Dart (a third language) and renders all UI via its own engine (not native Android/iOS components). For an iOS team, this means learning Dart, Flutter's widget system, and platform-specific integration patterns. The result is non-native widgets that can diverge from Android Material Design defaults, creating an extra UX alignment burden.

**Rejected because:** Dart ramp-up cost, non-native rendering, and no preservation of SwiftUI investment. The only win over native Compose is "one codebase" — but the iOS app remains SwiftUI regardless, so this win does not materialise.

### D. Cross-platform rewrite: React Native ❌

React Native uses JavaScript/TypeScript and bridges to native components. For an iOS-Swift team, JS is a new language. The New Architecture (Fabric) improves performance, but the bridge model introduces an extra debugging surface. Hiring flexibility is the main argument, but that's a business decision beyond the scope of this research.

**Rejected because:** Language mismatch (Swift → JS, not Kotlin), bridge complexity, and less direct mapping to the native Android UX idioms the team needs to learn anyway.

### E. LLM-assisted translation: Swift → Kotlin ✅ *Selective use, with guardrails*

LLMs (Cursor/Claude/GPT-4) can mechanically translate well-scoped units of Swift code to Kotlin, significantly reducing time-to-first-build for models, DTOs, and network interface definitions.

**Safe to LLM-translate:**
- Codable DTOs → Kotlin `@Serializable` data classes
- URLSession endpoint wrappers → Retrofit interface methods with suspend functions
- Simple utility/extension functions (string formatting, date helpers)
- Unit tests for the above

**Unsafe to LLM-translate wholesale:**
- SwiftUI Views with NavigationLink/NavigationStack → Compose Navigation (navigation semantics differ fundamentally)
- `onAppear`/`onDisappear` lifecycle → `LaunchedEffect`/`DisposableEffect` (must be understood, not just converted)
- SwiftUI sheets, alerts, and custom gestures → Compose `ModalBottomSheet`, `AlertDialog`, and gesture modifiers
- Concurrency patterns: Swift structured concurrency (`async/await` + `Task`) maps to Kotlin coroutines conceptually, but `viewModelScope.launch` and `StateFlow` are the idiomatic Android equivalents — generated code must be reviewed carefully

**Pragmatic LLM workflow (see Section 7 for full detail).**

### F. Backend-driven UI (BDUI) / thin client ❌

BDUI pushes layout descriptions from the server to the client, allowing UI updates without app releases. This materially speeds delivery only if the team must ship >30–50 screens simultaneously with near-zero native investment.

**Rejected because:** Building a BDUI server layer is significant backend work (the Azure API serves data, not layouts). It introduces a new architectural tier with its own rendering bugs, accessibility gaps, and a locked server team dependency. The team size and scope do not justify this complexity.

---

## 4. Recommendation and Rationale

### Primary recommendation

**Native Kotlin + Jetpack Compose, MVVM architecture, LLM assistance for mechanical translation.**

This is the lowest-risk path for an iOS-only team because:
1. Compose and SwiftUI share the same conceptual foundation (declarative, reactive, state-driven). The learning gap is about idioms and library APIs, not a paradigm shift.
2. Cursor/LLM tooling is most effective for Kotlin/Android — strong training data, official docs, and community resources.
3. The architecture (MVVM + Repository + Retrofit) will be familiar within days for a team that has written SwiftUI with ObservableObject + async networking.
4. Future Android hires will find a clean, idiomatic codebase; no knowledge of a bespoke framework required.

### Fallback

If the team finds Android native ramp-up slower than expected (e.g., build tooling complexity, Gradle, testing setup), the most pragmatic fallback is to hire one Android contractor for Phase 0–1 (setup + first vertical slice), then transfer ownership to the iOS team for Phase 2 onwards. This is preferable to switching to Flutter or RN.

### Phased roadmap

#### Phase 0 — Setup and architecture skeleton (Week 1)
- Create Android Studio project with recommended structure (feature-module or single-module to start)
- Wire Hilt, Navigation Compose, Retrofit+OkHttp skeleton, Firebase Crashlytics
- Establish layer boundaries: `ui/`, `viewmodel/`, `repository/`, `network/`, `model/`
- CI: GitHub Actions build + lint + unit test on every PR
- Decision gate: Does the app build, navigate between 3 stub screens, and pass a mock API call?

#### Phase 1 — MVP vertical slice (Weeks 2–4)
- Select 2–3 core flows from the iOS app (e.g., auth → list → detail)
- Implement each flow end-to-end: UI → ViewModel → Repository → Retrofit → Azure API
- Auth: MSAL for Android, token stored in `EncryptedSharedPreferences`
- Error handling: sealed `Result<T, AppError>` type; loading states in ViewModel
- Internal distribution: Firebase App Distribution
- Decision gate: iOS engineers can demo the 2–3 flows on an Android device without assistance

#### Phase 2 — Parity and hardening (Weeks 5–8)
- Complete remaining screens to reach MVP parity with iOS
- Auth edge cases: token refresh, session expiry, forced re-login
- Error handling hardening: retry with exponential backoff, network-unavailable states
- Performance: baseline Compose rendering (skip recompositions, `remember`, `key`)
- Observability: OkHttp logging interceptor (debug builds), correlation IDs on requests
- Testing: ViewModel unit tests (JUnit 5 + MockK), Flow tests (Turbine), MockWebServer for repository
- Decision gate: All MVP screens pass manual QA; unit test coverage ≥ 70% for ViewModel + Repository

#### Phase 3 — Polish, accessibility, instrumentation (Weeks 9–12)
- Material 3 theming, dark mode, dynamic colour
- Android accessibility: TalkBack semantics (contentDescription, `semantics {}` modifiers)
- Adaptive layouts for tablets/foldables if relevant
- Production Crashlytics dashboard review
- Evaluate KMP: if API model drift between iOS and Android is already observed, scope a KMP spike to share models/repositories
- Play Store release checklist: ProGuard/R8 rules, app signing, Privacy Policy, Data Safety form

---

## 5. Proposed Android Technical Stack

### UI Framework
**Jetpack Compose + Material 3**

The only serious choice for new Android development. Material 3 (Material You) is Google's current design system with dynamic colour, updated components (TopAppBar, NavigationBar, NavigationDrawer), and strong Compose support.

- [Compose documentation](https://developer.android.com/develop/ui/compose/documentation)
- [Material 3 for Compose](https://developer.android.com/develop/ui/compose/designsystems/material3)

### Networking
**Retrofit 2 + OkHttp 4**

Retrofit provides a type-safe HTTP client using annotated interfaces. Its annotation-driven style maps directly to OpenAPI generator output (`openapi-generator-cli generate -g kotlin --library jvm-retrofit2`). OkHttp provides the HTTP engine with built-in connection pooling, TLS, and an interceptor API for logging and auth.

```kotlin
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDto

    @POST("orders")
    suspend fun createOrder(@Body request: CreateOrderRequest): OrderDto
}
```

OkHttp `Interceptor` for bearer token injection:
```kotlin
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer ${tokenProvider.getToken()}")
            .build()
        return chain.proceed(request)
    }
}
```

- [Retrofit docs](https://square.github.io/retrofit/)
- [OkHttp docs](https://square.github.io/okhttp/)

**Why not Ktor?** Ktor is the right choice when sharing networking code via KMP. For an Android-only codebase, Retrofit's interface-annotation style has better tooling support (OpenAPI generator, Mockito/MockK-friendly interfaces), and more community resources — important for a team learning Android.

### Concurrency
**Kotlin Coroutines + Flow**

Coroutines are the standard Kotlin async model. `viewModelScope.launch` for fire-and-forget; `suspend fun` for repository/network calls; `StateFlow` for observable UI state (Combine analogue). `Flow` for streams (pagination, real-time updates).

```kotlin
class UserViewModel(private val repo: UserRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.value = when (val result = repo.getUser(id)) {
                is Result.Success -> UserUiState.Success(result.data)
                is Result.Error   -> UserUiState.Error(result.message)
            }
        }
    }
}
```

- [Coroutines overview](https://kotlinlang.org/docs/coroutines-overview.html)
- [Asynchronous Flow](https://kotlinlang.org/docs/flow.html)

### Persistence
**Start minimal: Jetpack DataStore (Preferences) for settings/tokens. Add Room later for caching.**

DataStore (Preferences flavour) replaces SharedPreferences with a coroutine-native, type-safe API. Use `EncryptedSharedPreferences` (from Jetpack Security) for storing auth tokens.

Room (SQLite abstraction) should not be added until the team identifies caching requirements (Phase 2/3 or when adding offline support).

- [DataStore docs](https://developer.android.com/topic/libraries/architecture/datastore)
- [Room docs](https://developer.android.com/training/data-storage/room)

### Dependency Injection
**Hilt**

Hilt is Google's officially recommended DI framework for Android, built on Dagger. Compile-time validation catches missing/mismatched dependencies at build time rather than runtime — important for a team learning Android where DI misconfigurations are a common early stumble.

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel()
```

**Why not Koin?** Koin is excellent for prototyping and multiplatform scenarios, but its runtime resolution means DI errors surface in prod or testing rather than at compile time. Hilt's overhead in setup is offset by the safety net it provides during ramp-up.

- [Hilt docs](https://developer.android.com/training/dependency-injection/hilt-android)
- [2024 Hilt vs Koin benchmarks (Droidcon)](https://www.droidcon.com/2024/12/03/benchmarking-koin-vs-dagger-hilt-in-modern-android-development-2024/)

### Navigation
**Navigation Compose**

Single-activity architecture with Navigation Compose. Type-safe destinations available via Navigation 2.8+ (using `@Serializable` route objects).

```kotlin
@Serializable data class UserDetailRoute(val userId: String)

NavHost(navController, startDestination = HomeRoute) {
    composable<HomeRoute> { HomeScreen(navController) }
    composable<UserDetailRoute> { backStackEntry ->
        val route: UserDetailRoute = backStackEntry.toRoute()
        UserDetailScreen(route.userId, navController)
    }
}
```

- [Navigation for Compose](https://developer.android.com/develop/ui/compose/navigation)

### Image Loading
**Coil 3**

If the app displays images from URLs, use Coil — a Kotlin-first image loading library with Compose support (`AsyncImage`). Built on coroutines and OkHttp. Coil 3 supports multiplatform.

- [Coil docs](https://coil-kt.github.io/coil/)

### Crash Reporting / Logging
**Firebase Crashlytics**

Azure App Center is entering retirement (most features sunset by mid-2026 per Microsoft). Firebase Crashlytics is free, real-time, and actively developed. Integrate with Firebase App Distribution for pre-release builds.

If the team is deeply invested in Azure DevOps for CI, consider [Sentry](https://sentry.io) as an alternative to Crashlytics (vendor-neutral, supports both Azure and GCP).

- [Firebase Crashlytics docs](https://firebase.google.com/docs/crashlytics/)

### Testing
| Layer | Library |
|-------|---------|
| Unit tests (ViewModel, Repository) | JUnit 5 + MockK |
| Coroutine/Flow tests | Turbine (`app.cash.turbine`) |
| Network layer mocking | MockWebServer (`com.squareup.okhttp3:mockwebserver`) |
| Compose UI tests | `androidx.compose.ui:ui-test-junit4` |
| Hilt in tests | `hilt-android-testing` |

- [Compose UI testing](https://developer.android.com/develop/ui/compose/testing)
- [Turbine (GitHub)](https://github.com/cashapp/turbine)
- [MockWebServer](https://github.com/square/okhttp/tree/master/mockwebserver)

---

## 6. Architecture Guidance for iOS Engineers

### Recommended pattern: MVVM

Google officially recommends MVVM for Android with Jetpack libraries. It maps cleanly onto SwiftUI patterns:

| iOS/SwiftUI | Android/Compose | Purpose |
|-------------|----------------|---------|
| `ObservableObject` class | `ViewModel` class | State holder, survives config changes |
| `@Published var items: [Item]` | `MutableStateFlow<List<Item>>` | Observable state property |
| `@StateObject var vm = VM()` | `val vm: MyVM = hiltViewModel()` | Scoped ViewModel in a composable |
| `.task { await vm.load() }` | `LaunchedEffect(Unit) { vm.load() }` | Side-effect on composable entry |
| `@EnvironmentObject` | `CompositionLocal` or Hilt-injected VM | Dependency injection through tree |

### Recommended layering

```
UI (Composables)
    ↕ observe StateFlow / call ViewModel methods
ViewModel (viewModelScope, StateFlow/MutableStateFlow)
    ↕ suspend fun calls
[Use Cases — optional, add for complex business logic]
    ↕ suspend fun calls
Repository (interface)
    ↕ suspend fun calls
API Client (Retrofit interface)        DataStore / Room
    ↕ HTTP
Azure REST API
```

- **Repository** is the single source of truth. It decides whether to fetch from network, cache, or both.
- **ViewModel** transforms Repository data into `UiState` sealed classes and exposes them via `StateFlow`.
- **Composables** are pure functions of state; they call ViewModel methods for events.

### Loading, error, and empty states

Define a sealed UI state per screen rather than separate `isLoading`/`error` booleans:

```kotlin
sealed interface UserListUiState {
    data object Loading : UserListUiState
    data class Success(val users: List<User>) : UserListUiState
    data class Error(val message: String) : UserListUiState
    data object Empty : UserListUiState
}
```

The Composable `when`-expression over this type gives exhaustive, compile-safe handling — analogous to SwiftUI's `switch` over an enum.

### Retry and pagination

- **Retry:** wrap Repository calls in a `retry(times = 3)` + exponential backoff in a Flow operator, or use OkHttp's `RetryInterceptor` at the HTTP layer for idempotent requests.
- **Pagination:** Jetpack Paging 3 library provides `Pager` + `PagingData` + `collectAsLazyPagingItems()` for Compose — use when list screens require server-side pagination.

### API contract strategy

**If an OpenAPI spec exists for the Azure API (strongly recommended if it doesn't):**

```sh
openapi-generator-cli generate \
  -i api-spec.yaml \
  -g kotlin \
  --library jvm-retrofit2 \
  --additional-properties useCoroutines=true,dateLibrary=kotlinx-datetime \
  -o app/src/main/java/com/example/generated
```

Generated output includes Retrofit interfaces and `@Serializable` data classes. Check generated code into a dedicated `generated/` package and treat it as read-only. Apply custom logic in the Repository layer, never in generated code.

**Preventing iOS/Android model drift:**
- Store the OpenAPI spec in the shared API repository (or the iOS repo alongside the iOS app)
- Run `openapi-diff` or `oasdiff` in CI on every backend PR that changes the spec
- Both iOS (Swift codegen) and Android (Kotlin codegen) client modules regenerate from the same source spec on each backend release

- [OpenAPI Generator Kotlin docs](https://openapi-generator.tech/docs/generators/kotlin/)
- [oasdiff — OpenAPI diff tool](https://github.com/Tufin/oasdiff)

---

## 7. LLM Conversion Reality Check: SwiftUI → Compose

### What translates mechanically (safe to LLM-assist)

| Swift/iOS | Kotlin/Android equivalent | Notes |
|-----------|--------------------------|-------|
| `Codable` struct with `CodingKeys` | `@Serializable` data class | Near 1:1; field name mapping via `@SerialName` |
| `URLSession` endpoint wrapper | Retrofit `@GET`/`@POST` interface method | Provide the method signature + URL; LLM fills annotations |
| `enum` with associated values (Result type) | `sealed class` / `sealed interface` | Direct structural equivalent |
| Date/time parsing | `kotlinx-datetime` or `java.time` | Confirm date format with API |
| String formatting / extensions | Kotlin extension functions | Safe; review generated tests |
| `Codable` test fixtures | MockWebServer test JSON + data class assertions | Highly mechanical |

### What must be redesigned (do NOT wholesale-translate)

| SwiftUI construct | Why it doesn't translate directly |
|-------------------|----------------------------------|
| `NavigationStack` + `NavigationLink` | Compose Navigation uses a `NavHost` + routes + `NavController`; the mental model and back-stack API are fundamentally different |
| `@Environment(\.dismiss)` | Compose equivalent is `navController.popBackStack()` or a `dismiss` lambda hoisted to the composable |
| `.sheet(isPresented:)` | Compose uses `ModalBottomSheet` or a flag in UI state + conditional composable |
| `onAppear` / `task` | `LaunchedEffect(Unit)` / `LaunchedEffect(key)` in Compose — semantics around recomposition differ |
| `onDisappear` | `DisposableEffect` with `onDispose` — important for cleanup |
| Custom gestures | Compose `Modifier.pointerInput` API; gesture detection differs significantly from SwiftUI's `DragGesture`/`TapGesture` |
| `@FocusState` | `FocusRequester` + `Modifier.focusRequester` in Compose |

### Risks of LLM-based conversion

1. **Deprecated APIs:** Compose had significant API changes before stable 1.0. LLMs trained on pre-1.0 code can generate deprecated `Scaffold` or `BottomNavigation` patterns. Always check against current Compose release notes.
2. **Missing tests:** LLMs rarely generate tests unprompted. Every translated unit must have a test — prompt explicitly: *"also write a unit test for the ViewModel and a MockWebServer test for the repository"*.
3. **Hidden platform differences:** `rememberCoroutineScope` vs `viewModelScope` — the former is tied to composable lifecycle and can leak if misused. LLMs sometimes confuse the two.
4. **Navigation back-stack:** LLM-generated navigation code may not handle the Android back button correctly; test on a physical device, not just in preview.

### Pragmatic LLM workflow

1. **Translate DTOs first.** Provide the Swift `Codable` struct and a sample JSON response; ask for the equivalent `@Serializable` data class plus a MockWebServer test.
2. **Generate Retrofit interfaces.** Provide the list of iOS URLSession calls (endpoint, method, parameters, response type); ask for the Retrofit interface definition plus a MockWebServer test.
3. **Translate one screen at a time.** Provide the SwiftUI View, the corresponding ViewModel/ObservableObject, and a description of the navigation context. Ask for the Compose screen + ViewModel. Review navigation, lifecycle effects, and state handling manually before merging.
4. **Require tests at each step.** Add to your Cursor prompt: *"Generate JUnit + Turbine tests for the ViewModel and a Compose UI test for the screen."*
5. **Code review gate.** Every LLM-produced Android screen must be reviewed by at least one engineer who has read the [Compose state docs](https://developer.android.com/develop/ui/compose/state) before merging.

---

## 8. Azure API Considerations

### Authentication

**Assumption:** The Azure API is protected by Azure AD (Microsoft Entra ID) bearer tokens.

**Recommended library:** MSAL for Android (`com.microsoft.identity.client:msal`)

MSAL handles the OAuth2/OIDC flow, token caching, and silent refresh. The acquired access token is passed as a `Bearer` header via an OkHttp `AuthInterceptor`.

Key MSAL flow:
1. `PublicClientApplication.createSingleAccountPublicClientApplication(context, config)` — initialise from `msal_config.json` in `res/raw/`
2. `acquireToken(activity, scopes, callback)` — interactive sign-in on first launch
3. `acquireTokenSilent(scopes, account)` — silent refresh for subsequent calls
4. Store the account in `EncryptedSharedPreferences`; refresh tokens are managed by MSAL's cache automatically

If the API uses Azure AD B2C (consumer identity), replace the authority in `msal_config.json` with the B2C authority URL. If the API uses a custom JWT or API key scheme, MSAL is not needed — use a simple `AuthInterceptor` that reads the token from encrypted storage.

- [MSAL for Android](https://learn.microsoft.com/en-us/entra/msal/android/)
- [Acquire and cache tokens with MSAL](https://learn.microsoft.com/en-us/entra/identity-platform/msal-acquire-cache-tokens)
- [ms-identity-android-kotlin sample](https://github.com/Azure-Samples/ms-identity-android-kotlin)

### API versioning and deprecation signals

Assumptions from the [API Version Alignment research](../2026-02-27-api-version-alignment/README.md):
- URL-path versioning (e.g., `/api/v1/`) is likely in use
- The Android app should declare the same API version as the iOS app at launch
- Monitor `Sunset` and `Deprecation` response headers (RFC 8594 / IETF draft) in the OkHttp interceptor; log a warning when either is present so the team is notified before a breaking change

```kotlin
class SunsetWarningInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        response.header("Sunset")?.let { sunsetDate ->
            Log.w("API", "Sunset header received: $sunsetDate — plan for migration")
        }
        return response
    }
}
```

### Observability

- **Correlation IDs:** The iOS app presumably sends a correlation/request ID header (e.g., `X-Correlation-ID`). Add the same header in an OkHttp interceptor on Android, using `UUID.randomUUID().toString()` per request.
- **Request logging:** Use `HttpLoggingInterceptor` from OkHttp in debug builds. Set level to `BODY` for debug, `NONE` for release.
- **Retries/backoff:** For idempotent `GET` requests, use an `OkHttp` retry interceptor or a Kotlin Flow `retry` operator with exponential backoff. Do NOT automatically retry mutating requests (`POST`/`PUT`/`DELETE`) without idempotency guarantees from the API.
- **Firebase Performance Monitoring** (optional): Available as an OkHttp interceptor via `FirebasePerformance.getInstance().newHttpMetric(...)` — useful for latency baselines.

---

## 9. Appendix

### A. Swift/SwiftUI → Kotlin/Compose Concept Mapping Table

| Swift/SwiftUI | Kotlin/Compose | Notes |
|---------------|---------------|-------|
| `async/await` | Kotlin coroutines (`suspend fun`) | Both model structured concurrency; `Task { }` ≈ `viewModelScope.launch { }` |
| `Combine` `Publisher` | `kotlinx.coroutines.flow.Flow` | Cold streams; `StateFlow` = `@Published` + `CurrentValueSubject` |
| `@Published` | `MutableStateFlow<T>` | Push updates to observers |
| `@State` | `remember { mutableStateOf() }` | Local composable state |
| `@Binding` | Pass `MutableState<T>` or lambdas down | State hoisting |
| `@StateObject` / `@ObservedObject` | `ViewModel` + `collectAsState()` | Long-lived shared state |
| `@EnvironmentObject` | `CompositionLocal` / Hilt ViewModel | Ambient/injected dependency |
| `Codable` / `Decodable` | `kotlinx.serialization` `@Serializable` | JSON ↔ class mapping |
| `URLSession` | OkHttp + Retrofit | HTTP client + type-safe API interface |
| `URLSession.shared` | `OkHttpClient` singleton (via Hilt) | Shared HTTP session |
| `Result<Success, Failure>` | `sealed class Result<T>` or Kotlin `Result<T>` | Success/failure wrapping |
| `enum` with associated values | `sealed class` / `sealed interface` | Exhaustive state machines |
| `struct` (value type model) | `data class` (value-copy semantics with `copy()`) | Immutable model objects |
| `protocol` | `interface` | Dependency inversion; enables mocking |
| `extension` | Extension function | Same concept; `fun String.trimSlashes()` |
| `guard let` / `if let` | `?.let { }` / `?:` (Elvis) | Null safety; Kotlin uses nullable types |
| `Optional<T>` | `T?` | Explicit nullable type |
| `SwiftUI.View` (`some View`) | `@Composable fun` | Both are functions describing UI |
| `NavigationStack` | `NavHost` + `NavController` | Navigation graph; routes differ |
| `NavigationLink(destination:)` | `navController.navigate(route)` | Programmatic push navigation |
| `sheet(isPresented:)` | `ModalBottomSheet` or `Dialog` composable | Overlay presentation |
| `.task { }` | `LaunchedEffect(key)` | Coroutine side-effect on composable lifecycle |
| `onAppear` | `LaunchedEffect(Unit)` (once on entry) | Lifecycle hook |
| `onDisappear` | `DisposableEffect` `onDispose { }` | Cleanup on composable leaving composition |
| `List { ForEach }` | `LazyColumn { items() }` | Lazy vertical list |
| `HStack` / `VStack` / `ZStack` | `Row` / `Column` / `Box` | Layout containers |
| `Image(systemName:)` | `Icon(Icons.Default.Name, ...)` | System icon sets |
| `AsyncImage` | `AsyncImage` (Coil) | Image URL loading |
| `@FocusState` | `FocusRequester` + `Modifier.focusRequester` | Focus management |
| `GeometryReader` | `BoxWithConstraints` | Measure available space |
| `.foregroundColor` | `color = Color.X` in `Text`/`Icon` | Colour modifiers |
| `Modifier.frame(width:height:)` | `Modifier.size(dp)` / `.fillMaxWidth()` | Size modifiers |

### B. Testing Stack Summary

| Concern | Library | Purpose |
|---------|---------|---------|
| ViewModel logic | JUnit 5 + MockK | Unit test coroutines + state transitions |
| Flow emissions | Turbine (`app.cash.turbine`) | Assert `StateFlow` sequences |
| API responses | MockWebServer | Inject canned JSON responses; test Repository |
| Compose UI | `ui-test-junit4` | Tap, assert text, node existence |
| DI in tests | `hilt-android-testing` | Replace Hilt modules in test |

### C. If We Later Add Offline Support

The current app is **not** offline-first. If that changes, here is what to add:

| Component | Change required |
|-----------|----------------|
| **Local cache** | Add Room (`@Entity` + `@Dao`) for cacheable resources; Repository pattern already in place, just add a second data source |
| **Cache invalidation** | Implement staleness policy in Repository: cache-first with TTL, or network-first with fallback |
| **Background sync** | Add `WorkManager` for periodic sync or retry-on-connectivity tasks |
| **Conflict resolution** | Decide on "last write wins" vs server authority vs CRDT depending on mutation type |
| **Connectivity state** | `ConnectivityManager` + `Flow` to observe network availability; expose to ViewModel |
| **Optimistic updates** | Write to Room immediately on user action; reconcile with server response asynchronously |

References:
- [Room docs](https://developer.android.com/training/data-storage/room)
- [WorkManager docs](https://developer.android.com/topic/libraries/architecture/workmanager)
- [Offline-first guide (Android)](https://developer.android.com/topic/architecture/data-layer/offline-first)

---

*Sources: All primary documentation URLs are in [notes.md](./notes.md). Prefer primary sources (developer.android.com, kotlinlang.org, square.github.io, ktor.io, learn.microsoft.com); blogs cited only for practical gotchas with caveats noted inline.*
