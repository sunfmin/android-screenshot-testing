# Android Screenshot Testing on JVM

A Claude Code skill for writing Android integration tests that render real UI pages and capture pixel-accurate screenshots — entirely on JVM, no emulator needed.

## What is this?

This is a [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that teaches Claude how to set up and write Robolectric-based screenshot tests for Android projects. When installed, Claude gains deep knowledge of:

- Rendering real Android pages on JVM using Robolectric's native graphics mode
- Intercepting network calls with MockWebServer at the HTTP boundary
- Solving the 5 common blockers (SDK crashes, KeyStore, endpoint URLs, singleton lifecycle, auth state)
- Capturing pixel-accurate PNGs from Activity views
- Structuring a reusable `BaseScreenshotTest` class

## Why?

Traditional Android UI testing is painful:

- **Instrumented tests** need an emulator or device — slow, flaky, expensive in CI
- **Mocked unit tests** don't catch integration bugs between layers
- **Manual QA** isn't repeatable or automated

This approach runs full-stack integration tests in `./gradlew test` — Fragment → ViewModel → Repository → Retrofit → OkHttp → MockWebServer — and captures a real PNG of the rendered screen. All in 10-15 seconds, no emulator.

## Installation

```bash
npx skills add sunfmin/android-screenshot-testing
```

Then use Claude Code as usual. When you ask it to write screenshot tests, set up visual testing infrastructure, or create integration tests with screenshots, the skill activates automatically.

## Usage

Once installed, just ask Claude Code naturally:

```
> Set up screenshot testing for this Android project
> Write a screenshot test for the checkout page
> Add visual regression tests for the settings screen
> Create a base class for Robolectric screenshot tests
```

Claude will apply the patterns from this skill — MockWebServer setup, Robolectric configuration, lifecycle-aware state management, and proper screenshot capture.

## What's in the skill

| Section | What it covers |
|---------|----------------|
| Architecture | Full-stack test diagram, mock-at-the-wire principle |
| Quick Reference | Gradle deps, annotations, minimal test structure |
| Five Problems | SDK crashes, KeyStore, URL redirect, singleton lifecycle, auth state |
| Base Class Design | Reusable test infrastructure pattern |
| Screenshot Capture | Software rendering, bitmap capture, xxhdpi explained |
| Asserting on Rendered Content | Text and visibility assertions: binding access, view-tree walks, Espresso, when-to-use matrix |
| Common Mistakes | Pitfalls and fixes table |
| Scaling Tips | Naming conventions, CI artifacts, visual diffing, multi-state/multi-size |

## Requirements

Your Android project needs:

- **Robolectric 4.10+** (4.12+ recommended for best native graphics support)
- **API 28+ target** (for `GraphicsMode.Mode.NATIVE`)
- **Retrofit + OkHttp** (or any HTTP client compatible with MockWebServer)
- Standard Android test dependencies (`androidx.test:core`, `androidx.test.ext:junit`)

## License

MIT
