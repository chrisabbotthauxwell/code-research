Rebuilding the Android app natively with **Kotlin and Jetpack Compose** focuses on intentional, screen-by-screen implementation to ensure robust navigation and state management, deliberately avoiding fragile wholesale SwiftUI-to-Compose conversions. Large Language Models (LLMs) like [Cursor](https://cursor.so/) or [Claude](https://www.anthropic.com/products/claude) are used selectively for translating model and interface code, streamlining mechanical tasks while keeping Compose screens idiomatic and maintainable. The pilot phase targets rapid delivery of three production-ready flows with secure authentication and error handling, omitting cross-platform tech and non-native frameworks to maximize clarity and minimize onboarding friction for iOS-developer familiarity. Success is defined by fully functional flows verified on real hardware and distributed internally for QA.

Key findings:
- Compose’s declarative paradigm closely maps to SwiftUI, easing team ramp-up.
- LLMs effectively automate model/interface translation but are unsuitable for bulk UI conversion.
- The pilot phase’s concrete deliverables establish best practices for navigation, authentication, and API communication.
- Excluding Kotlin Multiplatform and Flutter/React Native reduces complexity and preserves native fidelity.
