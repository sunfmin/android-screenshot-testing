---
name: android-screenshot-testing
description: Use when writing Android integration tests that render real UI pages and capture screenshots on JVM without emulators. Use when needing visual verification of Android screens, testing full UI rendering pipelines, or setting up Robolectric-based screenshot infrastructure for any Android project. Triggers on "screenshot test", "render page test", "Robolectric screenshot", "visual test without emulator", "integration test with screenshot", "Android JVM screenshot".
---

# Android Screenshot Testing on JVM

## Overview

Render real Android pages and capture pixel-accurate screenshots entirely on JVM — no emulator, no device. Uses Robolectric (native graphics) + MockWebServer to exercise the full code stack (Fragment/Activity → ViewModel → Repository → Retrofit → OkHttp) while only faking HTTP responses at the network boundary.

**Core principle:** Mock at the lowest possible layer. All your Kotlin/Java code runs for real. Only the bytes coming back from the wire are fake.

## Why This Approach

### The problem with traditional Android UI testing

| Approach | Drawback |
|----------|----------|
| Instrumented tests (androidTest) | Requires emulator/device, slow CI, flaky |
| Unit tests with mocked ViewModels | Doesn't test real data flow, misses integration bugs |
| Manual QA screenshots | Not repeatable, not in CI |
| Compose Preview screenshots | Only works for Compose, not Views |

### What this technique gives you

- **Runs in `./gradlew test`** — no emulator, 10-15 seconds, works in any CI
- **Full vertical integration** — Fragment observes real LiveData/StateFlow, ViewModel calls real Repo, Repo serializes real data, OkHttp sends real HTTP to MockWebServer
- **Visual regression baseline** — PNG output can be diffed across commits
- **Catches real bugs** — adapter binding logic, data transformation chains, visibility conditions all execute for real

## Architecture

```
┌─────────────────────────────────────────────┐
│  Your Test                                   │
│  ┌────────────┐  ┌────────────────────────┐ │
│  │ Test Data  │  │ MockWebServer          │ │
│  │ (JSON /    │  │ (returns test HTTP     │ │
│  │  protobuf) │  │  responses by path)    │ │
│  └─────┬──────┘  └──────────▲─────────────┘ │
│        │                    │               │
│  ALL REAL CODE BELOW THIS LINE              │
│  ┌─────▼──────────────────────────────────┐ │
│  │ Fragment → ViewModel → Repository      │ │
│  │     → Retrofit → OkHttp ──────────┘    │ │
│  └────────────────────────────────────────┘ │
│        │                                    │
│  ┌─────▼──────────────────────────────────┐ │
│  │ View.draw(Canvas) → PNG file           │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## Quick Reference

### Dependencies (app/build.gradle)

```groovy
testImplementation 'org.robolectric:robolectric:4.12.2'
testImplementation 'androidx.test:core:1.6.1'
testImplementation 'androidx.test.ext:junit:1.2.1'
testImplementation 'com.squareup.okhttp3:mockwebserver:4.12.0'

android {
    testOptions {
        unitTests { includeAndroidResources = true }
    }
}
```

### Test class annotations

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(
    sdk = [34],
    application = TestApp::class,           // skip third-party SDK init
    qualifiers = "w360dp-h780dp-xxhdpi"     // real phone density
)
@GraphicsMode(GraphicsMode.Mode.NATIVE)     // real pixel rendering
```

### Minimal test structure

```kotlin
class MyPageScreenshotTest : BaseScreenshotTest() {

    // 1. Mock page-specific APIs
    override fun onDispatch(path: String, request: RecordedRequest) = when {
        path.contains("/api/my-endpoint") -> jsonResponse(buildTestData())
        else -> null  // fall through to base defaults
    }

    // 2. Test = launch + interact + capture
    @Test
    fun renderMyPage() = launchAndCapture("my_page") { scenario ->
        // set up any required app state...
        navigateTo(scenario, R.id.target_destination)
    }
}
```

## Five Problems You Must Solve

Every Android screenshot test on JVM hits these five walls. Solve them once in a base class.

### 1. Third-party SDKs crash on JVM

Firebase, analytics SDKs, crash reporters, and similar libraries use native code or system services unavailable on JVM.

**Solution:** Create a `TestApp` extending your Application class that catches init errors:

```kotlin
// Make your Application class open
open class MyApp : Application() { ... }

class TestApp : MyApp() {
    override fun onCreate() {
        // Disable features that need hardware (see #2)
        try { super.onCreate() }
        catch (_: Throwable) { /* SDK init failures are expected */ }
    }
}
```

### 2. AndroidKeyStore unavailable on JVM

`EncryptedSharedPreferences` → `MasterKey` → `AndroidKeyStore` — this is a hardware-backed system service with no JVM equivalent.

**Solution:** Add a test bypass flag so encrypted storage falls back to plain SharedPreferences:

```kotlin
object SecureStorage {
    @VisibleForTesting var disableEncryption = false

    fun getPrefs(name: String, encrypted: Boolean = false): SharedPreferences {
        if (encrypted && !disableEncryption)
            return getEncryptedSharedPreferences(name)  // real path
        return context.getSharedPreferences(name, MODE_PRIVATE)  // plain fallback
    }
}
```

### 3. Endpoint URLs point to real servers

Repositories use Retrofit services initialized with production URLs. Must redirect to MockWebServer before first access.

**Solution:** Add a `baseUrlOverride` to your endpoint configuration:

```kotlin
object ApiConfig {
    @VisibleForTesting var baseUrlOverride: String? = null

    val baseUrl: String
        get() = baseUrlOverride ?: BuildConfig.API_BASE_URL
}
```

Then in test setup: `ApiConfig.baseUrlOverride = mockWebServer.url("/").toString()`

### 4. Singleton lifecycle ordering

Activity lifecycle callbacks often reset global state. If you set state before launching the Activity, it gets wiped.

**Solution:** Always set app state AFTER Activity creation + first idle:

```kotlin
val scenario = ActivityScenario.launch(MainActivity::class.java)
idleSafe()                          // let lifecycle callbacks finish
// NOW set your app state
AppState.currentUser = testUser
AppState.selectedConfig = testConfig
```

### 5. Views depend on user authentication state

Many views check authentication status and render differently (or skip rendering) for unauthenticated users.

**Solution:** Set up a fake authenticated user before capture:

```kotlin
// Use your app's auth mechanism, or reflection for private fields
AuthManager.setTestUser(TestUser(
    token = "test-token",
    expiresAt = "2099-12-31T23:59:59Z",
    userId = "TEST001"
))
```

## Base Class Design

Extract all five solutions plus screenshot capture into a reusable base class:

```
BaseScreenshotTest
├── @Before: MockWebServer + endpoint redirect
├── @After:  cleanup singletons + shutdown server
├── initAppState(): auth, config, any global providers
├── launchAndCapture(name) { lambda }:  full lifecycle orchestration
├── Subclass hooks:
│   ├── onDispatch(path, request): MockResponse? ← page-specific APIs
│   ├── buildTestData(): Any                     ← test fixtures
│   └── configureAppState()                      ← optional overrides
├── Response helpers:
│   ├── jsonResponse(obj): MockResponse
│   ├── protobufResponse(msg): MockResponse
│   ├── imageResponse(bytes): MockResponse
│   └── generateTestImage(label, color): ByteArray
├── Navigation helpers:
│   ├── navigateTo(scenario, destinationId)
│   └── idleSafe()
├── Reflection utilities:
│   └── setPrivateField, setAtomicBoolean, etc.
└── Screenshot: forceSoftwareRendering + draw + PNG save
```

Subclasses typically need only 50-90 lines of page-specific code.

## Screenshot Capture Details

```kotlin
// Force software rendering (bypasses hardware layers that don't draw to Canvas)
fun forceSoftwareRendering(view: View) {
    view.setLayerType(View.LAYER_TYPE_SOFTWARE, null)
    view.setWillNotDraw(false)
    if (view is ViewGroup) {
        for (i in 0 until view.childCount)
            forceSoftwareRendering(view.getChildAt(i))
    }
}

// Use Activity's own layout dimensions (don't re-measure — breaks RecyclerView)
val rootView = activity.window.decorView.rootView
val bitmap = Bitmap.createBitmap(rootView.width, rootView.height, Bitmap.Config.ARGB_8888)
rootView.draw(Canvas(bitmap))
bitmap.compress(Bitmap.CompressFormat.PNG, 100, FileOutputStream(file))
```

### Why `xxhdpi` qualifier matters

Without it, Robolectric defaults to mdpi (1x). A 1080px-wide canvas at mdpi = 1080dp — a giant tablet. At xxhdpi (3x), 1080px = 360dp — a real phone. Text, spacing, and layouts all render at correct proportions.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Set app state before Activity launch | Lifecycle callbacks reset it. Set AFTER `idleSafe()` |
| Forget to set authenticated user | Views skip rendering for anonymous users |
| Use manual `measure/layout` for screenshot | Breaks RecyclerView. Use Activity's natural dimensions |
| Only call `submitList` without forcing sync | DiffUtil is async, may not complete in Robolectric. Call `notifyDataSetChanged` or idle the looper |
| Expect real image decoding | Robolectric shows "Failed to create image decoder". Use `generateTestImage()` for colored placeholders |
| Wait with `Thread.sleep` only | Must also call `Shadows.shadowOf(Looper.getMainLooper()).idle()` to process messages |
| Missing `@GraphicsMode(NATIVE)` | Without native graphics mode, views render as empty bitmaps |

## Running

```bash
# Single test class
./gradlew :app:testDebugUnitTest --tests "com.example.MyPageScreenshotTest"

# All screenshot tests (use a naming convention)
./gradlew :app:testDebugUnitTest --tests "com.example.*ScreenshotTest"

# Output location (configure in your base class)
build/test-results/screenshots/my_page.png

# Open on macOS
open build/test-results/screenshots/my_page.png
```

## Tips for Scaling

- **Naming convention:** Suffix all screenshot tests with `ScreenshotTest` so they can be run as a group
- **Output directory:** Save PNGs to a consistent directory (e.g., `build/test-results/screenshots/`) for easy CI artifact collection
- **Visual diffing:** Compare PNGs across commits using tools like `pixelmatch`, `reg-suit`, or simple `diff` on file hashes
- **Multiple states:** Capture the same page in different states (empty, loading, error, populated) by varying MockWebServer responses
- **Screen sizes:** Run the same test with different `@Config(qualifiers = ...)` to test tablet/phone layouts
