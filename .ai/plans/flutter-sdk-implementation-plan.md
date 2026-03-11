# WSO2 IAM Mobile SDKs — Implementation Plan

## Context

We're building three production mobile SDKs following the [WSO2 IAM SDK Specification](https://github.com/brionmario/iam-sdk-specification/blob/main/skills/wso2-iam-sdk-spec/SKILL.md):

1. **iOS SDK** (Swift) — Agnostic + Platform layers
2. **Android SDK** (Kotlin) — Agnostic + Platform layers
3. **Flutter SDK** (Dart) — Core Lib layer bridging to iOS/Android via platform channels

All three support both **Asgardeo** and **WSO2 Identity Server** (configurable via `baseUrl`), implement both **redirect-based** and **app-native** auth modes, and cover the **full specification**.

---

## Repository Structure

Three separate repositories, co-located in a single workspace folder for easy local development:

```
mobile-sdks/                         # Local workspace (not a repo)
├── ios-sdk/                         # Repo 1 — Swift Package (SPM)
│   ├── README.md
│   ├── LICENSE                      # Apache 2.0
│   ├── CONTRIBUTING.md
│   ├── SECURITY.md
│   └── .github/
│       ├── workflows/ios-ci.yml
│       └── ISSUE_TEMPLATE/
│           ├── bug_report.md
│           └── feature_request.md
│
├── android-sdk/                     # Repo 2 — Gradle/Kotlin library
│   ├── README.md
│   ├── LICENSE
│   ├── CONTRIBUTING.md
│   ├── SECURITY.md
│   └── .github/
│       ├── workflows/android-ci.yml
│       └── ISSUE_TEMPLATE/
│           ├── bug_report.md
│           └── feature_request.md
│
├── flutter-sdk/                     # Repo 3 — Flutter plugin package
│   ├── README.md
│   ├── LICENSE
│   ├── CONTRIBUTING.md
│   ├── SECURITY.md
│   └── .github/
│       ├── workflows/flutter-ci.yml
│       └── ISSUE_TEMPLATE/
│           ├── bug_report.md
│           └── feature_request.md
│
└── docs-is/                         # Repo 4 — wso2/docs-is (checked out for doc contributions)
    └── en/
        ├── asgardeo/mkdocs.yml      # Nav entries for Asgardeo product
        ├── identity-server/next/mkdocs.yml  # Nav entries for IS product
        └── includes/
            ├── quick-starts/        # Shared quickstart pages (one file per SDK)
            │   ├── ios.md           # NEW
            │   ├── android.md       # NEW
            │   └── flutter.md       # NEW
            └── sdks/
                ├── ios/             # NEW — iOS API reference
                ├── android/         # NEW — Android API reference
                └── flutter/         # NEW — Flutter API reference
```

> Each SDK ships and is versioned independently. `docs-is` is the upstream WSO2 documentation repo — SDK doc contributions are submitted as PRs to it. The workspace folder is not committed.

---

## iOS SDK Structure (`ios-sdk/`)

Swift Package named `Asgardeo`. Min iOS 15.0, Swift 5.9+. No external dependencies — uses only Foundation, Security, AuthenticationServices, and CryptoKit frameworks.

```
ios-sdk/
├── Package.swift
├── README.md
├── CHANGELOG.md
├── Sources/Asgardeo/
│   ├── AsgardeoClient.swift                    # Main public IAMClient
│   │
│   ├── Config/
│   │   ├── AsgardeoConfig.swift                # Full config struct (all grouped fields from spec)
│   │   ├── SignInOptions.swift                  # typealias [String: Any]
│   │   ├── SignOutOptions.swift
│   │   ├── SignUpOptions.swift
│   │   └── TokenValidationConfig.swift
│   │
│   ├── Models/
│   │   ├── User.swift                          # User, KnownUser, UserProfile
│   │   ├── Organization.swift                  # Organization, AllOrganizationsResponse
│   │   ├── TokenResponse.swift                 # TokenResponse, AccessTokenAPIResponse, IdToken
│   │   ├── OIDCDiscovery.swift                 # OIDC well-known discovery document
│   │   ├── OIDCEndpoints.swift                 # Derived endpoints from discovery
│   │   ├── JWKS.swift                          # JSON Web Key Set models
│   │   ├── AuthState.swift                     # Internal auth state tracking
│   │   └── EmbeddedFlow/
│   │       ├── EmbeddedSignInFlow.swift         # FlowId, FlowStatus, StepType, Authenticator
│   │       ├── EmbeddedSignUpFlow.swift         # FlowExecuteRequest/Response, FlowComponent
│   │       └── EmbeddedFlowTypes.swift          # Shared enums (FlowType, ComponentType, etc.)
│   │
│   ├── Errors/
│   │   ├── IAMError.swift                      # code, message, cause, requestId, statusCode
│   │   └── IAMErrorCode.swift                  # Enum with all canonical error codes
│   │
│   ├── Core/                                   # Agnostic layer — pure protocol logic
│   │   ├── OIDCManager.swift                   # Discovery fetch, endpoint resolution, JWKS cache
│   │   ├── TokenManager.swift                  # Exchange, refresh, rotation, single-flight dedup
│   │   ├── PKCEManager.swift                   # code_verifier + S256 code_challenge generation
│   │   ├── StateManager.swift                  # Cryptographic state param generation + validation
│   │   ├── JWTDecoder.swift                    # JWT base64 decode (no signature verify)
│   │   ├── JWTValidator.swift                  # Full ID token validation (JWKS sig, iss, aud, exp, nonce)
│   │   ├── AuthorizationRequestBuilder.swift   # Build /oauth2/authorize URL with PKCE, state, scopes
│   │   ├── RedirectFlowManager.swift           # Orchestrates redirect-based auth code + PKCE flow
│   │   ├── AppNativeSignInManager.swift        # /oauth2/authorize?response_mode=direct + /oauth2/authn
│   │   ├── AppNativeFlowManager.swift          # /api/server/v1/flow/execute for signUp, recovery
│   │   └── SessionManager.swift                # Session state, expiry monitoring, auto-refresh
│   │
│   ├── Platform/                               # Platform layer — iOS-specific
│   │   ├── KeychainStorage.swift               # Keychain Services wrapper (default StorageAdapter)
│   │   ├── ASWebAuthSessionHandler.swift       # ASWebAuthenticationSession for redirect flow
│   │   ├── CryptoUtils.swift                   # Security framework: SHA256, random bytes
│   │   └── BiometricAuthHelper.swift           # Optional Face ID / Touch ID for token access
│   │
│   ├── Adapters/                               # Extensibility protocols
│   │   ├── StorageAdapter.swift                # Protocol: store, retrieve, delete, clear
│   │   ├── LoggerAdapter.swift                 # Protocol: debug, info, warn, error
│   │   ├── HTTPAdapter.swift                   # Protocol: request -> HTTPResponse
│   │   └── DefaultHTTPAdapter.swift            # URLSession-based default implementation
│   │
│   ├── Events/
│   │   ├── SDKEvent.swift                      # Enum: SIGN_IN_SUCCESS, SIGN_IN_FAILED, etc.
│   │   ├── EventPayload.swift                  # Payload struct per event type
│   │   └── EventEmitter.swift                  # on(event:handler:) / emit(event:payload:)
│   │
│   ├── API/                                    # Network calls
│   │   ├── UserInfoAPI.swift                   # GET /oauth2/userinfo
│   │   ├── SCIM2API.swift                      # GET/PATCH /scim2/Me
│   │   ├── TokenAPI.swift                      # POST /oauth2/token, /oauth2/revoke
│   │   ├── OrganizationAPI.swift               # Organization CRUD endpoints
│   │   └── FlowExecuteAPI.swift                # POST /api/server/v1/flow/execute
│   │
│   ├── Utils/
│   │   ├── LogSanitizer.swift                  # Mask tokens, emails, phones per spec
│   │   ├── URLUtils.swift                      # URL building, query param encoding
│   │   └── ThreadSafety.swift                  # Actor-based or lock-based concurrency helpers
│   │
│   └── SwiftUI/                                # Framework integration
│       ├── AsgardeoEnvironment.swift            # .asgardeoProvider(config:) view modifier
│       ├── AsgardeoViewModel.swift              # ObservableObject exposing IAMClient state
│       └── UseAsgardeo.swift                    # @Environment property wrapper
│
├── Tests/AsgardeoTests/
│   ├── Config/
│   │   └── AsgardeoConfigTests.swift
│   ├── Core/
│   │   ├── PKCEManagerTests.swift
│   │   ├── JWTValidatorTests.swift
│   │   ├── TokenManagerTests.swift
│   │   ├── RedirectFlowManagerTests.swift
│   │   ├── AppNativeSignInManagerTests.swift
│   │   └── AppNativeFlowManagerTests.swift
│   ├── Platform/
│   │   └── KeychainStorageTests.swift
│   ├── Errors/
│   │   └── IAMErrorTests.swift
│   ├── API/
│   │   └── MockHTTPAdapter.swift
│   └── Integration/
│       └── FullFlowIntegrationTests.swift
│
└── samples/
    └── b2c-ios/
        ├── b2c-ios.xcodeproj/
        ├── b2c-ios/
        │   ├── b2cApp.swift                     # @main, .asgardeoProvider(config:)
        │   ├── ContentView.swift                # SignedIn/SignedOut conditional
        │   ├── SignInView.swift                  # Sign-in sheet
        │   ├── ProfileView.swift                # User profile display
        │   └── .env.example
        └── README.md
```

---

## Android SDK Structure (`android-sdk/`)

Kotlin library, package `io.asgardeo.android`. Min SDK 24, Kotlin 1.9+, compileSdk 34.

**Dependencies:** `androidx.security:security-crypto:1.1.0-alpha06`, `androidx.browser:browser:1.7.0`, `com.squareup.okhttp3:okhttp:4.12.0`, `kotlinx-coroutines-android:1.8.0`

```
android-sdk/
├── build.gradle.kts                            # Root build file
├── settings.gradle.kts
├── gradle.properties
├── gradlew / gradlew.bat
├── gradle/wrapper/
├── README.md
├── CHANGELOG.md
│
├── lib/                                        # The SDK library module
│   ├── build.gradle.kts
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml
│       │   └── kotlin/io/asgardeo/android/
│       │       ├── AsgardeoClient.kt           # Main public IAMClient
│       │       │
│       │       ├── config/
│       │       │   ├── AsgardeoConfig.kt       # Data class for configuration
│       │       │   ├── SignInOptions.kt
│       │       │   ├── SignOutOptions.kt
│       │       │   ├── SignUpOptions.kt
│       │       │   └── TokenValidationConfig.kt
│       │       │
│       │       ├── models/
│       │       │   ├── User.kt                 # User, KnownUser, UserProfile
│       │       │   ├── Organization.kt
│       │       │   ├── TokenResponse.kt
│       │       │   ├── OIDCDiscovery.kt
│       │       │   ├── OIDCEndpoints.kt
│       │       │   ├── JWKS.kt
│       │       │   ├── AuthState.kt
│       │       │   └── embedded/
│       │       │       ├── EmbeddedSignInFlow.kt
│       │       │       ├── EmbeddedSignUpFlow.kt
│       │       │       └── EmbeddedFlowTypes.kt
│       │       │
│       │       ├── errors/
│       │       │   ├── IAMError.kt             # Exception class
│       │       │   └── IAMErrorCode.kt         # Enum of all error codes
│       │       │
│       │       ├── core/                       # Agnostic layer (mirrors iOS Core/)
│       │       │   ├── OIDCManager.kt
│       │       │   ├── TokenManager.kt
│       │       │   ├── PKCEManager.kt
│       │       │   ├── StateManager.kt
│       │       │   ├── JWTDecoder.kt
│       │       │   ├── JWTValidator.kt
│       │       │   ├── AuthorizationRequestBuilder.kt
│       │       │   ├── RedirectFlowManager.kt
│       │       │   ├── AppNativeSignInManager.kt
│       │       │   ├── AppNativeFlowManager.kt
│       │       │   └── SessionManager.kt
│       │       │
│       │       ├── platform/                   # Platform layer (Android-specific)
│       │       │   ├── EncryptedStorage.kt     # EncryptedSharedPreferences wrapper
│       │       │   ├── ChromeTabHandler.kt     # Chrome Custom Tabs for redirect
│       │       │   ├── CryptoUtils.kt          # java.security SHA256, SecureRandom
│       │       │   └── RedirectActivity.kt     # Deeplink callback handler Activity
│       │       │
│       │       ├── adapters/
│       │       │   ├── StorageAdapter.kt       # Interface
│       │       │   ├── LoggerAdapter.kt        # Interface
│       │       │   ├── HTTPAdapter.kt          # Interface + HTTPResponse data class
│       │       │   └── DefaultHTTPAdapter.kt   # OkHttp-based default
│       │       │
│       │       ├── events/
│       │       │   ├── SDKEvent.kt
│       │       │   ├── EventPayload.kt
│       │       │   └── EventEmitter.kt
│       │       │
│       │       ├── api/
│       │       │   ├── UserInfoApi.kt
│       │       │   ├── SCIM2Api.kt
│       │       │   ├── TokenApi.kt
│       │       │   ├── OrganizationApi.kt
│       │       │   └── FlowExecuteApi.kt
│       │       │
│       │       └── utils/
│       │           ├── LogSanitizer.kt
│       │           ├── URLUtils.kt
│       │           └── CoroutineUtils.kt       # Mutex for concurrent token refresh dedup
│       │
│       └── test/kotlin/io/asgardeo/android/
│           ├── config/AsgardeoConfigTest.kt
│           ├── core/
│           │   ├── PKCEManagerTest.kt
│           │   ├── JWTValidatorTest.kt
│           │   ├── TokenManagerTest.kt
│           │   ├── RedirectFlowManagerTest.kt
│           │   ├── AppNativeSignInManagerTest.kt
│           │   └── AppNativeFlowManagerTest.kt
│           ├── platform/EncryptedStorageTest.kt
│           ├── errors/IAMErrorTest.kt
│           └── api/MockHTTPAdapter.kt
│
└── samples/
    └── b2c-android/
        ├── app/
        │   ├── build.gradle.kts
        │   └── src/main/
        │       ├── AndroidManifest.xml
        │       ├── kotlin/io/asgardeo/sample/
        │       │   ├── MainActivity.kt
        │       │   └── ui/
        │       │       ├── SignInScreen.kt      # Jetpack Compose
        │       │       ├── ProfileScreen.kt
        │       │       └── theme/Theme.kt
        │       └── res/
        ├── build.gradle.kts
        ├── settings.gradle.kts
        ├── .env.example
        └── README.md
```

---

## Flutter SDK Structure (`flutter-sdk/`)

Plugin package `asgardeo_flutter`. Min Flutter 3.16+, Dart 3.2+.

Bridges to native SDKs via `MethodChannel("io.asgardeo.flutter/sdk")` and `EventChannel("io.asgardeo.flutter/events")`.

```
flutter-sdk/
├── pubspec.yaml
├── README.md
├── CHANGELOG.md
├── analysis_options.yaml
│
├── lib/
│   ├── asgardeo_flutter.dart                   # Barrel export
│   └── src/
│       ├── asgardeo_client.dart                # Public API (delegates via platform channel)
│       │
│       ├── config/
│       │   ├── asgardeo_config.dart
│       │   ├── sign_in_options.dart
│       │   ├── sign_out_options.dart
│       │   └── sign_up_options.dart
│       │
│       ├── models/
│       │   ├── user.dart
│       │   ├── organization.dart
│       │   ├── token_response.dart
│       │   ├── auth_state.dart
│       │   └── embedded/
│       │       ├── embedded_sign_in_flow.dart
│       │       ├── embedded_sign_up_flow.dart
│       │       └── embedded_flow_types.dart
│       │
│       ├── errors/
│       │   ├── iam_error.dart
│       │   └── iam_error_code.dart
│       │
│       ├── platform/
│       │   ├── asgardeo_platform_interface.dart  # Abstract platform interface
│       │   ├── asgardeo_method_channel.dart      # MethodChannel implementation
│       │   └── channel_codec.dart                # Serialize/deserialize models ↔ Map
│       │
│       ├── adapters/
│       │   ├── storage_adapter.dart
│       │   └── logger_adapter.dart
│       │
│       ├── events/
│       │   ├── sdk_event.dart
│       │   ├── event_payload.dart
│       │   └── event_stream.dart                # Stream-based event bus (Dart idiom)
│       │
│       └── widgets/
│           ├── actions/
│           │   ├── sign_in_button.dart          # Styled default
│           │   ├── base_sign_in_button.dart     # Unstyled logic-only
│           │   ├── sign_out_button.dart
│           │   ├── base_sign_out_button.dart
│           │   ├── sign_up_button.dart
│           │   └── base_sign_up_button.dart
│           ├── auth_flow/
│           │   └── callback.dart
│           ├── guards/
│           │   ├── signed_in.dart
│           │   ├── signed_out.dart
│           │   └── loading.dart
│           ├── auth_ui/
│           │   ├── sign_in.dart
│           │   ├── base_sign_in.dart
│           │   ├── sign_up.dart
│           │   ├── base_sign_up.dart
│           │   ├── accept_invite.dart
│           │   ├── base_accept_invite.dart
│           │   ├── invite_user.dart
│           │   └── base_invite_user.dart
│           ├── user/
│           │   ├── user_widget.dart
│           │   ├── base_user_widget.dart
│           │   ├── user_dropdown.dart
│           │   ├── base_user_dropdown.dart
│           │   ├── user_profile.dart
│           │   └── base_user_profile.dart
│           ├── organization/
│           │   ├── organization_widget.dart
│           │   ├── base_organization_widget.dart
│           │   ├── organization_list.dart
│           │   ├── base_organization_list.dart
│           │   ├── organization_profile.dart
│           │   ├── base_organization_profile.dart
│           │   ├── organization_switcher.dart
│           │   ├── base_organization_switcher.dart
│           │   ├── create_organization.dart
│           │   └── base_create_organization.dart
│           ├── other/
│           │   ├── language_switcher.dart
│           │   └── base_language_switcher.dart
│           └── provider/
│               ├── asgardeo_provider.dart       # InheritedWidget wrapping init
│               └── asgardeo_state.dart           # ChangeNotifier for reactive state
│
├── ios/                                        # Native iOS plugin side
│   ├── Classes/
│   │   └── AsgardeoFlutterPlugin.swift         # FlutterPlugin: MethodChannel → AsgardeoClient
│   └── asgardeo_flutter.podspec                # CocoaPods spec, depends on Asgardeo SPM
│
├── android/                                    # Native Android plugin side
│   ├── src/main/kotlin/io/asgardeo/flutter/
│   │   └── AsgardeoFlutterPlugin.kt           # FlutterPlugin: MethodChannel → AsgardeoClient
│   └── build.gradle.kts                        # Depends on io.asgardeo.android lib
│
├── test/
│   ├── asgardeo_client_test.dart
│   ├── models/user_test.dart
│   └── widgets/
│       ├── sign_in_button_test.dart
│       └── guards_test.dart
│
├── integration_test/
│   └── full_flow_test.dart
│
└── example/                                    # b2c-flutter sample app
    ├── pubspec.yaml
    ├── lib/
    │   ├── main.dart
    │   ├── sign_in_screen.dart
    │   └── profile_screen.dart
    ├── ios/
    ├── android/
    ├── .env.example
    └── README.md
```

---

## Key Design Decisions

### 1. Mode Detection

If `signInPath` is set in config → app-native mode. Otherwise with `clientId` + `redirectUri` → redirect mode. `AsgardeoClient` delegates transparently to either `RedirectFlowManager` or `AppNativeSignInManager`.

### 2. Platform Channel Contract

Single `MethodChannel("io.asgardeo.flutter/sdk")` with 1:1 method mapping:

| Dart Method | Channel Method | Arguments | Return |
|---|---|---|---|
| `initialize(config)` | `"initialize"` | `{"config": <serialized>}` | `{"success": bool}` |
| `signIn(options?)` | `"signIn"` | `{"options": <map>?}` | `{"user": <map>}` |
| `signOut(options?)` | `"signOut"` | `{"options": <map>?}` | `{"url": string}` |
| `getAccessToken()` | `"getAccessToken"` | `{}` | `{"token": string}` |
| `getUser()` | `"getUser"` | `{}` | `{"user": <map>}` |
| `getUserProfile()` | `"getUserProfile"` | `{}` | `{"profile": <map>}` |
| `switchOrganization(org)` | `"switchOrganization"` | `{"organization": <map>}` | `{"tokenResponse": <map>}` |

Events via `EventChannel("io.asgardeo.flutter/events")` streaming `SDKEvent` payloads.

Errors thrown as `PlatformException` with `IAMErrorCode` as the `code` field.

### 3. Token Refresh

Scheduled 60s before `exp`. Concurrent calls deduplicated via single-flight pattern (Swift `actor`, Kotlin `Mutex`). Old refresh token atomically replaced on rotation.

### 4. Storage Defaults

| Platform | Default | Implementation |
|---|---|---|
| iOS | Keychain | `KeychainStorage` with `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` |
| Android | EncryptedSharedPreferences | `EncryptedStorage` backed by Android Keystore |
| Flutter | Delegates to native | No Dart-side token storage |

Custom `StorageAdapter` supported on all platforms.

### 5. Async Patterns

| Platform | Pattern |
|---|---|
| iOS/Swift | `async throws` + `Result<T, IAMError>` |
| Android/Kotlin | `suspend fun` (coroutines) |
| Flutter/Dart | `Future<T>` |

### 6. Terminology

Strictly `signIn` / `signUp` / `signOut` everywhere in public API. Never `login` / `logout` / `register`.

---

## Canonical Method Signatures

All SDKs implement these 19 methods + 3 properties:

### Properties
| Name | Type | Description |
|---|---|---|
| `isAuthenticated` | `Bool` | Whether user is currently signed in |
| `isLoading` | `Bool` | Whether an auth operation is in progress |
| `user` | `User?` | Current authenticated user (minimal) |

### Methods
| Method | Return Type | Description |
|---|---|---|
| `signIn(options?)` | `User` | Initiate authentication |
| `signUp(options?)` | `User` | Initiate registration |
| `signOut(options?)` | `String` (URL) | Terminate session |
| `signInSilently(options?)` | `User` | Passive auth (prompt=none) |
| `getUser()` | `User` | Current user (minimal) |
| `getUserProfile()` | `UserProfile` | Full profile data |
| `updateUserProfile(payload)` | `UserProfile` | Update profile |
| `changePassword(current, new)` | `Void` | Change user password |
| `initiatePasswordRecovery()` | `FlowResponse` | Start password recovery |
| `initiateUsernameRecovery()` | `FlowResponse` | Start username recovery |
| `getUserInfo()` | `UserInfo` | Verified info from userinfo endpoint |
| `getAccessToken()` | `String` | Retrieve access token |
| `decodeJwtToken(token)` | `Map` | JWT decode (no verification) |
| `exchangeToken(config)` | `TokenResponse` | RFC 8693 token exchange |
| `getAllOrganizations()` | `[Organization]` | List all organizations |
| `getMyOrganizations()` | `[Organization]` | User's organizations |
| `getCurrentOrganization()` | `Organization` | Active organization context |
| `switchOrganization(org)` | `TokenResponse` | Change organization context |
| `reInitialize(config)` | `Void` | Partial config update |

---

## Error Codes

| Category | Codes |
|---|---|
| **Configuration** | `SDK_NOT_INITIALIZED`, `ALREADY_INITIALIZED`, `INVALID_CONFIGURATION`, `INVALID_REDIRECT_URI` |
| **Authentication** | `AUTHENTICATION_FAILED`, `USER_ACCOUNT_LOCKED`, `USER_ACCOUNT_DISABLED`, `SESSION_EXPIRED`, `MFA_REQUIRED`, `MFA_FAILED`, `INVALID_GRANT`, `CONSENT_REQUIRED` |
| **Registration** | `USER_ALREADY_EXISTS`, `INVALID_INPUT`, `INVITATION_CODE_INVALID`, `INVITATION_CODE_EXPIRED`, `REGISTRATION_DISABLED` |
| **Recovery** | `RECOVERY_FAILED`, `CONFIRMATION_CODE_INVALID`, `CONFIRMATION_CODE_EXPIRED` |
| **Network** | `NETWORK_ERROR`, `REQUEST_TIMEOUT`, `SERVER_ERROR`, `UNKNOWN_ERROR` |

---

## Security Requirements (Non-Negotiable)

- **PKCE**: S256 only. `code_verifier` in memory only — never persisted
- **State param**: Cryptographically random. Validated on callback. Mismatch → `AUTHENTICATION_FAILED`
- **ID token validation**: Verify signature (JWKS), `iss`, `aud`, `exp`, `nonce`. Re-fetch JWKS on key rotation failure
- **Sign-out**: Revoke refresh token server-side (RFC 7009) before clearing local state
- **Never log**: Credentials, tokens, passwords, OTPs at any log level
- **Token refresh**: Before expiry (60s recommended). Refresh token rotation with atomic storage update
- **Log sanitization**: Mask access tokens (log type + expiry only), mask emails and phone numbers

---

## Extensibility Interfaces

| Extension | Interface | Notes |
|---|---|---|
| Custom Storage | `StorageAdapter { store, retrieve, delete, clear }` | Via config |
| Custom Logger | `LoggerAdapter { debug, info, warn, error }` | Default no-op |
| Custom HTTP Client | `HTTPAdapter { request } → HTTPResponse` | For proxy, custom TLS, testing |
| Event Hooks | `on(event, handler)` | Events: `SIGN_IN_SUCCESS`, `SIGN_IN_FAILED`, `SIGN_OUT`, `TOKEN_REFRESHED`, `TOKEN_REFRESH_FAILED`, `SESSION_EXPIRED`, `MFA_STEP_REQUIRED` |

---

## UI Components (Flutter Widgets)

Two-layer pattern: styled `ComponentName` (zero config) + unstyled `BaseComponentName` (full customization).

| Category | Components |
|---|---|
| **Actions** | `SignInButton` + `BaseSignInButton`, `SignOutButton` + `BaseSignOutButton`, `SignUpButton` + `BaseSignUpButton` |
| **Auth Flow** | `Callback` |
| **Guards** | `SignedIn`, `SignedOut`, `Loading` |
| **Auth UI** | `SignIn` + `BaseSignIn`, `SignUp` + `BaseSignUp`, `AcceptInvite` + `BaseAcceptInvite`, `InviteUser` + `BaseInviteUser` |
| **User** | `UserWidget` + `BaseUserWidget`, `UserDropdown` + `BaseUserDropdown`, `UserProfile` + `BaseUserProfile` |
| **Organization** | `OrganizationWidget`, `OrganizationList`, `OrganizationProfile`, `OrganizationSwitcher`, `CreateOrganization` (all with `Base*` variants) |
| **Other** | `LanguageSwitcher` + `BaseLanguageSwitcher` |

---

## Implementation Phases

### Phase 0: Repository Scaffolding

**SDK repos (×3):**

- Top-level files per repo: `README.md`, `LICENSE` (Apache 2.0), `CONTRIBUTING.md`, `SECURITY.md`, `.github/`
- `ios-sdk`: `Package.swift` with target declarations, min iOS 15.0
- `android-sdk`: Gradle project with root + `lib/` module, min SDK 24, Kotlin 1.9+
- `flutter-sdk`: `pubspec.yaml` with plugin declaration, min Flutter 3.16
- GitHub Actions CI stubs for all three platforms

**Docs workspace:**

- Clone `wso2/docs-is` into the workspace alongside the SDK repos
- No commits to docs-is until Phase 12 — used for reference and local preview during development

### Phase 1: Models & Configuration (iOS + Android in parallel)

- All config classes (`AsgardeoConfig` with all grouped fields from spec Section 5.2)
- All model classes: User, Organization, TokenResponse, OIDCDiscovery, OIDCEndpoints, JWKS, AuthState
- All embedded flow models (sign-in steps, flow execute response types)
- Error types: `IAMError` + all `IAMErrorCode` values
- Options types: `SignInOptions`, `SignOutOptions`, `SignUpOptions`
- Unit tests for all models

### Phase 2: Adapters & Utilities (iOS + Android in parallel)

- `StorageAdapter` protocol/interface
- `LoggerAdapter` protocol/interface (default = no-op)
- `HTTPAdapter` protocol/interface + `DefaultHTTPAdapter` (URLSession / OkHttp)
- `EventEmitter` + `SDKEvent` enum
- `LogSanitizer` — mask tokens, emails, phones
- `CryptoUtils` — SHA256, secure random bytes
- `URLUtils` — URL building helpers
- Unit tests for crypto, sanitizer, URL utils

### Phase 3: OIDC Discovery & PKCE (iOS + Android in parallel)

- `OIDCManager` — fetch `/.well-known/openid-configuration`, parse into `OIDCEndpoints`, cache with TTL
- `PKCEManager` — generate `code_verifier` (43+ chars), derive `code_challenge` with S256
- `StateManager` — generate/validate cryptographic `state` parameter
- `AuthorizationRequestBuilder` — construct `/oauth2/authorize` URL with all params
- `JWTDecoder` — decode JWT payload (base64url) without signature verification
- `JWTValidator` — verify JWT signature via JWKS, validate iss, aud, exp, nonce; re-fetch JWKS on key rotation
- Thorough unit tests

### Phase 4: Token Management (iOS + Android in parallel)

- `TokenManager`: exchange (auth code → tokens), refresh (refresh_token grant), revocation (RFC 7009), atomic storage, single-flight dedup
- `SessionManager`: state tracking, auto-refresh 60s before expiry, event emission
- `TokenAPI`: HTTP calls to `/oauth2/token` and `/oauth2/revoke`
- Unit tests with mock HTTP adapter

### Phase 5: Platform Auth Handlers (iOS + Android in parallel)

- **iOS**: `KeychainStorage`, `ASWebAuthSessionHandler`
- **Android**: `EncryptedStorage`, `ChromeTabHandler`, `RedirectActivity`
- `RedirectFlowManager`: build URL → launch browser → callback → exchange code → validate ID token → return User
- Integration tests with mock browser handler

### Phase 6: App-Native Authentication (iOS + Android in parallel)

- `AppNativeSignInManager`: GET `/oauth2/authorize?response_mode=direct` → step loop via POST `/oauth2/authn` → handle all step types → token exchange
- `AppNativeFlowManager`: POST `/api/server/v1/flow/execute` for signUp, recovery → handle VIEW, REDIRECTION, WEBAUTHN, INTERNAL_PROMPT responses
- API classes: `FlowExecuteAPI`, `UserInfoAPI`, `SCIM2API`, `OrganizationAPI`
- Unit tests for each step type, error conditions, MFA branching

### Phase 7: AsgardeoClient Assembly (iOS + Android in parallel)

- Full public `IAMClient` with all 19 methods + 3 properties
- Input validation, idempotency rules (`signOut` with no session = no-op, double `initialize` = error)
- Event hook wiring
- Integration tests for full client lifecycle

### Phase 8: SwiftUI / Jetpack Compose Integration

- **iOS**: `AsgardeoViewModel` (`@Observable`), `.asgardeoProvider(config:)` view modifier, `@Environment` wrapper
- **Android**: `AsgardeoState` composable helper wrapping client in `MutableState`

### Phase 9: Flutter SDK — Platform Channel Bridge

- **Dart side**: `AsgardeoClient`, `AsgardeoMethodChannel`, `ChannelCodec`, `EventStream`
- **iOS plugin**: `AsgardeoFlutterPlugin.swift` — MethodChannel → Swift async calls
- **Android plugin**: `AsgardeoFlutterPlugin.kt` — MethodChannel → Kotlin coroutines
- `AsgardeoProvider` (InheritedWidget) + `AsgardeoState` (ChangeNotifier)
- Tests with mock method channel

### Phase 10: Flutter UI Widgets

- Full widget library per spec Section 8 (all components listed above)
- Two-layer pattern for all non-trivial components
- Widget tests for each component

### Phase 11: Sample Applications

- **b2c-ios**: SwiftUI — sign-in sheet, profile view, sign-out
- **b2c-android**: Jetpack Compose — sign-in screen, profile screen, sign-out
- **b2c-flutter**: Cross-platform — sign-in, profile, sign-out (in `example/` directory)
- All with `.env.example` (ASGARDEO_BASE_URL, ASGARDEO_CLIENT_ID), README, CI build

### Phase 12: Documentation & Release Prep

**In-repo API reference (auto-generated):**

- DocC (iOS), Dokka (Android), dartdoc (Flutter)
- GitHub Actions workflows finalized
- README quickstart guides for each SDK
- Tag all three SDKs as v0.1.0

**docs-is contributions (PR to `wso2/docs-is`):**

Quickstart pages — `en/includes/quick-starts/`:

- `ios.md` — mirrors React quickstart structure: app config, SPM install, `AsgardeoClient` init, sign-in/sign-out, user profile
- `android.md` — same sections for Gradle + Kotlin
- `flutter.md` — same sections for pub.dev + Dart

Nav entries added to **both** `en/asgardeo/mkdocs.yml` and `en/identity-server/next/mkdocs.yml` under `Get started > Connect App`:

```yaml
- iOS:
    - Quickstart: quick-starts/ios.md
    - Complete Guide: complete-guides/ios/introduction.md
- Android:
    - Quickstart: quick-starts/android.md
    - Complete Guide: complete-guides/android/introduction.md
- Flutter:
    - Quickstart: quick-starts/flutter.md
    - Complete Guide: complete-guides/flutter/introduction.md
```

API reference pages — `en/includes/sdks/`:

```
sdks/
├── ios/
│   ├── overview.md
│   ├── client.md                    # AsgardeoClient — all 19 methods + 3 properties
│   ├── configuration.md             # AsgardeoConfig fields
│   ├── models.md                    # User, Organization, TokenResponse, etc.
│   ├── swiftui.md                   # AsgardeoViewModel, .asgardeoProvider, @Environment
│   └── guides/
│       ├── redirect-auth.md
│       ├── app-native-auth.md
│       └── token-management.md
├── android/
│   ├── overview.md
│   ├── client.md
│   ├── configuration.md
│   ├── models.md
│   ├── compose.md                   # AsgardeoState composable helper
│   └── guides/
│       ├── redirect-auth.md
│       ├── app-native-auth.md
│       └── token-management.md
└── flutter/
    ├── overview.md
    ├── client.md
    ├── configuration.md
    ├── provider/
    │   └── asgardeo-provider.md     # AsgardeoProvider InheritedWidget
    ├── widgets/
    │   ├── sign-in-button.md
    │   ├── sign-out-button.md
    │   ├── sign-up-button.md
    │   ├── signed-in.md
    │   ├── signed-out.md
    │   ├── loading.md
    │   ├── user-widget.md
    │   ├── user-dropdown.md
    │   ├── user-profile.md
    │   ├── organization-widget.md
    │   ├── organization-list.md
    │   ├── organization-switcher.md
    │   └── create-organization.md
    └── guides/
        ├── accessing-protected-apis.md
        └── using-guards.md
```

Nav entries added to **both** mkdocs.yml files under `SDK Documentation`:

```yaml
- iOS SDK:
    - Overview: sdks/ios/overview.md
    - APIs:
        - AsgardeoClient: sdks/ios/client.md
        - Configuration: sdks/ios/configuration.md
        - Models: sdks/ios/models.md
        - SwiftUI Integration: sdks/ios/swiftui.md
    - Guides:
        - Redirect Authentication: sdks/ios/guides/redirect-auth.md
        - App-Native Authentication: sdks/ios/guides/app-native-auth.md
        - Token Management: sdks/ios/guides/token-management.md
- Android SDK:
    - Overview: sdks/android/overview.md
    - APIs:
        - AsgardeoClient: sdks/android/client.md
        - Configuration: sdks/android/configuration.md
        - Models: sdks/android/models.md
        - Jetpack Compose Integration: sdks/android/compose.md
    - Guides:
        - Redirect Authentication: sdks/android/guides/redirect-auth.md
        - App-Native Authentication: sdks/android/guides/app-native-auth.md
        - Token Management: sdks/android/guides/token-management.md
- Flutter SDK:
    - Overview: sdks/flutter/overview.md
    - APIs:
        - AsgardeoClient: sdks/flutter/client.md
        - Configuration: sdks/flutter/configuration.md
        - "&lt;AsgardeoProvider /&gt;": sdks/flutter/provider/asgardeo-provider.md
        - Widgets:
            - Actions:
                - "&lt;SignInButton /&gt;": sdks/flutter/widgets/sign-in-button.md
                - "&lt;SignOutButton /&gt;": sdks/flutter/widgets/sign-out-button.md
                - "&lt;SignUpButton /&gt;": sdks/flutter/widgets/sign-up-button.md
            - Guards:
                - "&lt;SignedIn /&gt;": sdks/flutter/widgets/signed-in.md
                - "&lt;SignedOut /&gt;": sdks/flutter/widgets/signed-out.md
                - "&lt;Loading /&gt;": sdks/flutter/widgets/loading.md
            - User:
                - "&lt;UserWidget /&gt;": sdks/flutter/widgets/user-widget.md
                - "&lt;UserDropdown /&gt;": sdks/flutter/widgets/user-dropdown.md
                - "&lt;UserProfile /&gt;": sdks/flutter/widgets/user-profile.md
            - Organization (B2B):
                - "&lt;OrganizationWidget /&gt;": sdks/flutter/widgets/organization-widget.md
                - "&lt;OrganizationList /&gt;": sdks/flutter/widgets/organization-list.md
                - "&lt;OrganizationSwitcher /&gt;": sdks/flutter/widgets/organization-switcher.md
                - "&lt;CreateOrganization /&gt;": sdks/flutter/widgets/create-organization.md
    - Guides:
        - Accessing Protected APIs: sdks/flutter/guides/accessing-protected-apis.md
        - Using Guards: sdks/flutter/guides/using-guards.md
```

---

## Testing Strategy

| Level | Coverage | Tools |
|---|---|---|
| **Unit tests** | 80%+ for `core/` and `errors/` | XCTest (iOS), JUnit5 + MockK (Android), flutter_test + mockito (Flutter) |
| **Integration tests** | Full auth lifecycle, token refresh, app-native multi-step, org switch | Mock HTTP adapter, mock browser |
| **Widget tests** | Guard rendering, button interactions, provider state | flutter_test |
| **Sample app CI** | Build verification on every PR | GitHub Actions |

---

## Dependency Summary

| Platform | Min Version | External Dependencies |
|---|---|---|
| **iOS** | iOS 15.0, Swift 5.9+ | None (Foundation, Security, AuthenticationServices, CryptoKit) |
| **Android** | SDK 24, Kotlin 1.9+, compileSdk 34 | androidx.security:security-crypto, androidx.browser:browser, okhttp3, kotlinx-coroutines |
| **Flutter** | Flutter 3.16+, Dart 3.2+ | plugin_platform_interface + native iOS/Android SDKs |

---

## Verification

1. `swift build` and `swift test` pass for iOS SDK
2. `./gradlew build` and `./gradlew test` pass for Android SDK
3. `flutter test`, `flutter analyze` pass for Flutter SDK
4. All three sample apps build successfully
5. End-to-end: `initialize` → `signIn` → `getUser` → `getUserProfile` → `signOut` works against a live Asgardeo instance or local WSO2 IS
