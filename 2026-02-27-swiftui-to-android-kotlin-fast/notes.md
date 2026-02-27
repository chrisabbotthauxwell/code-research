# Research Notes: SwiftUI → Android (Kotlin/Jetpack Compose) Strategy

## Session: 2026-02-27

### Goal
Investigate the fastest, lowest-risk approach for an iOS-only team to ship an Android app
that mirrors an existing SwiftUI + Azure HTTP API app, using Kotlin/Jetpack Compose as the
target platform. No offline-first requirement initially.

---

## Sources Checked

### Android Architecture & Jetpack Compose
- https://developer.android.com/develop/ui/compose/documentation — Official Compose docs (primary)
- https://developer.android.com/topic/architecture — Guide to App Architecture (primary)
- https://developer.android.com/topic/architecture/recommendations — Architecture recommendations
- https://developer.android.com/courses/pathways/android-architecture — Modern Android App Architecture course
- https://developer.android.com/topic/libraries/architecture/viewmodel — ViewModel overview
- https://developer.android.com/codelabs/jetpack-compose-state — State in Jetpack Compose codelab
- https://github.com/android/compose-samples — Official Compose samples (NowInAndroid, Jetsnack, etc.)
- https://android-developers.googleblog.com/2024/05/whats-new-in-jetpack-compose-at-io-24.html — What's new in Compose (I/O '24)

### Kotlin Coroutines & Flow
- https://kotlinlang.org/docs/coroutines-overview.html — Coroutines overview
- https://kotlinlang.org/docs/coroutines-guide.html — Coroutines guide
- https://kotlinlang.org/docs/flow.html — Asynchronous Flow documentation
- https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/ — Flow API reference

### Networking
- https://square.github.io/retrofit/ — Retrofit docs (primary)
- https://square.github.io/okhttp/ — OkHttp docs (primary)
- https://ktor.io/docs/client-engines.html — Ktor client engine docs
- https://carrion.dev/en/posts/migrating-retrofit-okhttp-to-ktor-kmp/ — Retrofit→Ktor KMP migration guide (blog; practical gotchas)
- https://www.androidmetro.com/2024/01/android-ktor-vs-retrofit.html — Android-specific comparison (blog)

### Dependency Injection
- https://developer.android.com/training/dependency-injection/hilt-android — Hilt official docs (primary)
- https://insert-koin.io/docs/quickstart/android-compose — Koin for Compose (primary)
- https://www.droidcon.com/2024/12/03/benchmarking-koin-vs-dagger-hilt-in-modern-android-development-2024/ — 2024 Hilt vs Koin benchmarks (Droidcon)

### OpenAPI Code Generation
- https://openapi-generator.tech/docs/generators/kotlin/ — OpenAPI Generator Kotlin docs (primary)
- https://github.com/OpenAPITools/openapi-generator/blob/master/docs/generators/kotlin.md — Detailed config options

### Authentication
- https://learn.microsoft.com/en-us/entra/msal/android/ — MSAL for Android (Microsoft primary)
- https://github.com/AzureAD/microsoft-authentication-library-for-android — MSAL Android library (primary)
- https://learn.microsoft.com/en-us/entra/identity-platform/msal-acquire-cache-tokens — Acquire/cache tokens
- https://github.com/Azure-Samples/ms-identity-android-kotlin — MSAL Android Kotlin sample

### Crash Reporting
- https://firebase.google.com/docs/crashlytics/ — Firebase Crashlytics docs (primary)
- https://learn.microsoft.com/en-us/appcenter/diagnostics/ — Azure App Center Diagnostics
- https://www.trustradius.com/compare-products/firebase-crashlytics-vs-visual-studio-app-center — Comparison (note: App Center retirement confirmed)

### Cross-Platform Framework Comparisons
- https://kotlinlang.org/docs/multiplatform/kotlin-multiplatform-react-native.html — KMP vs React Native (Kotlin primary)
- https://dev.to/forge-stackobea/flutter-react-native-or-kotlin-multiplatform-choosing-the-right-stack-in-2025-22g3 — Practical comparison (blog)
- https://speednetsoftware.com/cross-platform-frameworks-analysis-kmp-vs-react-native-vs-flutter/ — Framework analysis

### SwiftUI ↔ Compose Concept Mapping
- https://omzzz.dev/swiftui-to-jetpack-compose-and-vice-era-reference-guide/ — SwiftUI↔Compose reference guide (blog; well-cited)
- https://foso.github.io/Jetpack-Compose-Playground/compose_for/swiftui_devs/ — "Compose for SwiftUI Developers"
- https://developer.android.com/develop/ui/compose/documentation — Official Compose learning resources (primary)

### Navigation Compose
- https://developer.android.com/develop/ui/compose/navigation — Navigation for Compose (primary)

### DataStore / Room
- https://developer.android.com/topic/libraries/architecture/datastore — Jetpack DataStore (primary)
- https://developer.android.com/training/data-storage/room — Room (primary)

### Image Loading
- https://coil-kt.github.io/coil/ — Coil docs (primary)

### Testing
- https://developer.android.com/develop/ui/compose/testing — Compose UI testing (primary)
- https://github.com/cashapp/turbine — Turbine for Flow testing (primary)
- https://github.com/square/okhttp/tree/master/mockwebserver — MockWebServer docs (primary)

---

## Key Decisions Made

### Primary recommendation: Native Kotlin + Jetpack Compose (Option A)
**Rationale:**
- Smallest conceptual leap for an iOS/SwiftUI team: Compose has almost identical declarative UI
  paradigm, state-hoisting pattern, ViewModel = ObservableObject, StateFlow ≈ @Published + Combine
- No new language boundary (Kotlin is well-documented, Cursor/LLM tooling is excellent for Kotlin)
- Google's recommended architecture pattern; largest ecosystem, best long-term support
- Most job-market coverage for future Android hires
- KMP and Flutter both require more infrastructure or a new language (Dart); not justified now

### Networking: Retrofit + OkHttp (chosen over Ktor)
**Rationale:**
- Team is Android-only for now; no KMP requirement immediately
- Retrofit's interface-annotation style maps well to OpenAPI generator output
- Larger community, more StackOverflow answers, more familiar to LLM-assisted workflows
- OkHttp interceptor pattern for bearer tokens is well-documented and widely used
- If KMP added later, Ktor migration is well-documented (one codebase boundary to change)

### DI: Hilt (chosen over Koin)
**Rationale:**
- Hilt is compile-time safe; errors surface at build time rather than runtime
- Google official recommendation; best integration with ViewModel/Compose lifecycle
- More authoritative documentation available for LLM-assisted development
- Koin is excellent for small teams/prototypes, but Hilt pays off at scale
- For a team learning Android from scratch, catching DI errors at compile time reduces ramp-up risk

### Crash Reporting: Firebase Crashlytics (chosen over Azure App Center)
**Rationale:**
- Azure App Center is entering retirement (most features sunset by mid-2026)
- Firebase Crashlytics is free, real-time, battle-tested on Android, and not being deprecated
- App Center integration with Azure Pipeline is a plus only if team is heavily Azure DevOps — noting that the API is Azure but the mobile CI pipeline needs to be decided separately

### Architecture: MVVM (chosen over MVI)
**Rationale:**
- MVVM is Google's explicitly recommended pattern with first-party support
- Maps directly to SwiftUI's ObservableObject/StateObject model — easiest transfer of mental model
- MVI is compatible but adds boilerplate (sealed Intent classes, reducers) that increases ramp-up
- If the app later requires complex UI state machines, migrating VM state to UDF/MVI-style sealed
  state classes is incremental

### LLM-assisted translation: yes, with guardrails
**Rationale:**
- Swift models (Codable structs) → Kotlin data classes with kotlinx.serialization: highly mechanical
- Retrofit interfaces from existing URLSession endpoint calls: highly mechanical
- Full screen translation (SwiftUI View → Compose): feasible for simple screens, risky for
  navigation, sheets, animation, custom gestures; each must be reviewed and tested
- Rule: every LLM-translated unit must have a corresponding test before merging

---

## Alternatives Considered and Rejected

### KMP (Kotlin Multiplatform) — now
- Rejected for Phase 0/1 because: team is iOS-only, adding KMP means learning two new
  technology layers simultaneously (Kotlin/Android AND KMP gradle build system)
- It makes more sense to add KMP in Phase 3/4 once the team has Android confidence
- KMP also cannot share UI; every screen still must be re-implemented in Compose

### Flutter
- Rejected because: introduces Dart (third language), UI rendering departs from native Android
  components (non-native widget rendering can fail accessibility and platform-specific expectations)
- Team would need to learn Dart + Flutter widget tree AND maintain the existing SwiftUI iOS app
- If the goal were a single shared codebase from day one, Flutter would be considered more seriously

### React Native
- Rejected because: iOS team is Swift-focused, not JavaScript/TypeScript-focused; RN adds a JS
  bridge layer to debug; tooling maturity for complex Compose-equivalent UIs is still lower than
  native Compose

### LLM "wholesale" SwiftUI → Compose automatic conversion
- Rejected because: navigation model is fundamentally different (SwiftUI NavigationStack/NavigationLink
  vs Compose Navigation); lifecycle management differs (onAppear vs LaunchedEffect/DisposableEffect);
  state restoration and back-stack semantics differ; gesture system differs
- Safe to mechanically translate: DTOs, model layer, network interface signatures
- Unsafe to wholesale translate: screens containing navigation, sheets, animations, accessibility modifiers

### Backend-driven UI / thin client (BDUI)
- Evaluated briefly; rejected because: requires significant backend investment to emit layout
  descriptions in addition to data; the existing Azure API presumably serves data, not UI layout;
  this would be a new architectural layer to build and maintain; justified only if team wants to
  ship >50 screens simultaneously with zero native ramp-up (not the stated goal)

---

## Open Questions and Risks

1. **Auth mechanism on existing Azure API** — not specified in the brief. Assuming Azure AD /
   Microsoft Entra ID bearer tokens + MSAL, but could be custom JWT, API key, or Azure AD B2C.
   The team must confirm this before Phase 1.

2. **OpenAPI spec existence** — if the Azure API already has an OpenAPI spec, code generation
   saves significant time. If not, the spec must be written (or inferred from the iOS app's
   URLSession calls). This is a prerequisite for the API client generation approach.

3. **Team size for Android work** — the problem states iOS-only team; it's unclear if an
   Android contractor will be hired, or if the iOS engineers are expected to write the Android
   app directly. This affects timeline estimates.

4. **Play Store account and signing setup** — not researched; must be set up before Phase 1 build.

5. **Deep links / push notifications** — not mentioned in brief; if the iOS app uses push
   notifications (APNs), the Android equivalent (FCM) will need wiring. Not blocking MVP.

6. **Accessibility requirements** — Android TalkBack semantics differ from iOS VoiceOver; this
   should be addressed in Phase 2/3, not Phase 0/1.

7. **App Center deprecation timeline** — Microsoft confirmed most App Center features being
   retired by June 2026. Confirming exact dates is important if the team is currently using
   App Center for iOS.

8. **Cursor LLM quality for Kotlin** — empirically, Claude and GPT-4-class models generate
   high-quality Kotlin/Compose code; however, models can hallucinate deprecated Compose APIs
   (e.g., pre-1.0 Scaffold APIs). Always cross-reference generated code against the official
   Compose release notes.
