---
name: wso2-iam-sdk-spec
description: >
  Author, extend, or review documentation and specifications for WSO2 IAM SDKs
  (Asgardeo / WSO2 Identity Server). Use this skill whenever the user wants to
  write or update an SDK specification, design a new SDK layer, define an API
  contract, document identity lifecycle operations (sign-in, sign-up, sign-out,
  recovery), design UI component catalogues for IAM SDKs, define framework
  integration patterns (React, Angular, Vue, Flutter, iOS, Android, Python,
  Node.js), set up SDK repositories, write sample applications, or apply the
  WSO2 IAM SDK layered architecture. Also trigger for questions about how WSO2
  app-native authentication flows work, the Flow Execution API, or how
  OAuth2/OIDC maps to SDK method signatures.
---

# WSO2 IAM SDK Specification Skill

You are an agent assisting with the WSO2 IAM SDK Specification. This document
is your authoritative reference for all conventions, architecture decisions, and
content rules. When generating code, writing spec prose, reviewing
contributions, or answering questions about SDK design, you MUST follow these
rules strictly.

**How to use this skill:**
- When **writing or editing spec sections** in the README, follow the writing
  guides and structural rules below exactly. Do not invent new conventions.
- When **generating SDK implementation code**, use the canonical method names,
  error codes, component names, and architectural patterns defined here. Adapt
  to the target language's idioms per Section 12 guidance, but never rename
  public API methods or violate the layering rules.
- When **reviewing code or documentation**, check against the rules below and
  flag any violations (wrong terminology, layer-skipping, missing error codes,
  insecure defaults, etc.).
- When **answering questions**, cite the relevant section number from the spec.
- The full specification lives in `README.md` at the repository root. Always
  read it for the complete details — this skill is a condensed reference, not a
  replacement.

---

## Core Principles (non-negotiable)

These apply to everything — method names, prose, diagrams, tables:

1. **Terminology is strict.** Always `signIn`, `signUp`, `signOut`. Never
   `login`, `logout`, `register` in any public API surface, prose description,
   or example. OIDC protocol values (`prompt=login`) and field names inherited
   from standards (`loginHint`, `post-logout`) are exempt.

2. **Spec is language-agnostic.** Pseudocode only in API contracts. No
   TypeScript, Swift, Kotlin, or Python syntax in the core spec body. Platform
   examples belong in Section 12 (Platform & Language Guidelines) or in
   framework-specific implementation guides.

3. **No type-definition code blocks in operation sections.** Identity lifecycle
   operations (sign-in, sign-up, recovery, etc.) are described in prose and
   tables — not as pseudocode structs. Method signatures belong only in the
   Client Interface section (Section 7.1).

4. **Configuration is tables only.** No pseudocode schema blocks for config or
   preferences. Use grouped tables (Core, Redirect URIs, OAuth2/OIDC,
   Application Identity, Token Security, Session, Storage & Platform, UI).

5. **Component-driven IAM.** Every UI-capable SDK MUST ship a component
   library. Components follow the `Base*` / styled two-layer pattern. See
   Section 8 for the canonical component catalogue.

6. **Single developer entry point per framework.** One hook (React), one
   service (Angular), one composable (Vue). Internal helpers are implementation
   details and MUST NOT be required in application code.

---

## SDK Architecture — The Four Layers

All SDK implementations fit into one of four layers. When designing or
documenting any new SDK, classify it first:

| Layer | What it does | Has UI? | Example |
|-------|-------------|---------|---------|
| **Agnostic SDK** | Language-specific implementation of the full `IAMClient` interface and all protocol logic. No platform, browser, or framework deps. | No | JavaScript SDK, Swift SDK, Python SDK |
| **Platform SDK** | Extends Agnostic SDK with platform-specific capabilities: native storage, redirect/callback handling, platform crypto, platform-native session management. | No | Browser SDK, Node.js SDK, iOS SDK, Android SDK |
| **Core Lib SDK** | Extends Platform SDK with a framework's reactive primitives (state, lifecycle, context). Single developer-facing entry point. UI components optional. | Optional | React SDK, Vue SDK |
| **Framework Specific SDK** | Thin integration on top of Core Lib SDK. Adds SSR, routing conventions, opinionated DX. Builds on a Core Lib SDK. | Optional | Next.js SDK, Nuxt SDK, Angular SDK |

**Flutter exception:** Flutter has no Agnostic layer of its own. It sits at
Core Lib and bridges to iOS SDK + Android SDK via platform channels. Flutter
UI widgets are written in Dart — not reusing native components.

**Angular exception:** Angular's DI model makes it a Framework Specific SDK
that builds directly on the Browser SDK, skipping a Core Lib layer.

When writing architecture sections, always:
- Use a Mermaid diagram for visual depiction of a specific ecosystem
- Keep the canonical four-layer table language-agnostic
- Use the JavaScript ecosystem only as a *worked example*, not as the default

---

## Ecosystem Table

| Ecosystem | Agnostic | Platform | Core Lib | Framework Specific |
|-----------|----------|----------|----------|--------------------|
| **JavaScript** | JavaScript SDK | Browser SDK, Node.js SDK | React SDK, Vue SDK | Express SDK, Next.js SDK, Nuxt SDK, React Router SDK, TanStack Router SDK |
| **Mobile — Apple** | Swift SDK | iOS SDK | SwiftUI SDK | — |
| **Mobile — Android** | Kotlin SDK | Android SDK | Jetpack Compose SDK | — |
| **Cross-platform Mobile** | — | iOS SDK + Android SDK *(via platform channels)* | Flutter SDK (Dart) | — |
| **Python** | Python SDK | — *(server-side, no platform layer needed)* | Django / FastAPI SDK | — |

> **Router SDKs:** `react-router` and `tanstack-router` sit at the Framework
> Specific layer. They build on top of the React Core Lib SDK and add
> router-specific concerns (protected routes, callback route handling,
> navigation guards).

---

## Layer Rules

- A layer MUST depend on exactly one parent layer (except where platform channels bridge two Platform SDKs, as in Flutter)
- A layer MUST NOT re-implement logic already present in its parent layer
- A layer MUST NOT expose internal implementation details of its parent layer through its own public API
- Each SDK MUST declare its minimum required version of its parent SDK as a versioned dependency
- A breaking change in any layer requires a **major version bump** in that layer and all layers that depend on it
- All SDK layers MUST document which version of this specification they implement

---

## Operational Modes

Every SDK MUST support both modes transparently behind the same public API:

| Mode | Mechanism | Key rule |
|------|-----------|----------|
| **Redirect-Based** | OAuth2 Authorization Code + PKCE. Browser redirect to IAM sign-in page. | PKCE `S256` is mandatory. State param is mandatory for CSRF. |
| **App-Native** | Direct API calls. No browser redirects. WSO2-proprietary. | Uses `/oauth2/authorize?response_mode=direct` to initiate, then `/oauth2/authn` for each step. For sign-up/recovery, uses Flow Execution API (`POST /flow/execute`). |

Mode is inferred from minimal configuration, not declared explicitly. For
example, supplying a `signInPath` tells the SDK to switch to App-Native mode
automatically.

---

## App-Native Flow APIs

### Sign-In — Application-Native Authentication API

- **Initiation:** `GET /oauth2/authorize` with `response_mode=direct`
- **Step handling:** `POST /oauth2/authn` with `flowId` + `selectedAuthenticator`
- **Step types:** `AUTHENTICATOR_PROMPT` (single authenticator) or
  `MULTI_OPTIONS_PROMPT` (user picks)
- **Prompt types within a step:**
  - `USER_PROMPT` — collect user input (username, password, OTP)
  - `INTERNAL_PROMPT` — collect from client context (device info, origin)
  - `REDIRECTION_PROMPT` — redirect to external IdP; handle callback; resume
- **Completion:** `flowStatus: SUCCESS_COMPLETED` → `authData.code` →
  standard token exchange

### Sign-Up / Recovery — Flow Execution API

- **Endpoint:** `POST /api/server/v1/flow/execute` (no auth header required)
- **flowType values:** `REGISTRATION`, `INVITED_USER_REGISTRATION`,
  `PASSWORD_RECOVERY`
- **Response types:**

| `type` | SDK behaviour |
|--------|--------------|
| `VIEW` | Render `data.components`; collect user input; submit `flowId` + `actionId` + `inputs` |
| `REDIRECTION` | Redirect to `data.url`; handle callback; resume with `flowId` + callback params |
| `WEBAUTHN` | Run WebAuthn ceremony with `data.webAuthn`; Base64url-encode result; submit |
| `INTERNAL_PROMPT` | Collect client context from `data.requiredParams`; submit without user interaction |

- **Completion:** `flowStatus: COMPLETE`. If auto sign-in is enabled, response
  contains a short-lived `userAssertion` JWT (~2s expiry) in `data` for
  immediate authentication.

---

## Identity Lifecycle Operations — Writing Guide

When documenting any operation, follow this structure — **no type-definition
code blocks**:

1. **Redirect Mode** — one prose paragraph describing what the SDK does
2. **App-Native Mode** — prose + step-type table if relevant
3. **Authenticator support** — table (authenticator name, prompt type, required
   params)
4. **Implementor notes** — bullet points for edge cases and MUST rules

### Canonical Method Names

| Operation | Method |
|-----------|--------|
| Initiate authentication | `signIn()` |
| Initiate registration | `signUp()` |
| Terminate session | `signOut()` |
| Passive authentication | `signInSilently()` |
| Complete invited registration | covered by `signUp()` with `INVITED_USER_REGISTRATION` flow |
| Password recovery | `initiatePasswordRecovery()` |
| Username recovery | `initiateUsernameRecovery()` |
| Get current user | `getUser()` |
| Get full profile | `getUserProfile()` |
| Update profile | `updateUserProfile()` |
| Change password | `changePassword()` |
| Switch organization | `switchOrganization()` |
| Get all organizations | `getAllOrganizations()` |
| Get my organizations | `getMyOrganizations()` |
| Get current organization | `getCurrentOrganization()` |
| Get access token | `getAccessToken()` |
| Decode JWT (no verification) | `decodeJwtToken()` |
| Get verified user info | `getUserInfo()` |
| Exchange token (RFC 8693) | `exchangeToken()` |

---

## Client Interface (IAMClient)

All SDK implementations MUST provide a client interface that maps to the
canonical `IAMClient`. Key design points:

- `signIn()` and `signUp()` are **overloaded** — one signature for redirect
  mode, another for app-native/embedded mode
- `signInSilently()` uses `prompt=none` via iframe (redirect mode only)
- `reInitialize()` accepts a **partial** config for updating specific fields
  without full re-initialization
- `isLoading()` is **synchronous** unlike all other state queries

Categories exposed by the single entry point:

| Category | Methods / Properties |
|----------|---------------------|
| Authentication state | `isAuthenticated`, `isLoading`, `user` |
| Auth actions | `signIn()`, `signOut()`, `signInSilently()` |
| Registration | `signUp()` |
| Token | `getAccessToken()`, `exchangeToken()` |
| Profile | `getUserProfile()`, `updateUserProfile()` |
| Organizations | `getAllOrganizations()`, `getMyOrganizations()`, `getCurrentOrganization()`, `switchOrganization()` |
| Lifecycle | `reInitialize()` |

---

## UI Component Catalogue

When specifying UI components for any platform SDK, use this canonical set.
Adapt component names to platform conventions (e.g., `SignInView` on Android,
`SignInWidget` in Flutter) but preserve the semantic mapping.

### Categories

| Category | Components |
|----------|-----------|
| **Actions** | `SignInButton`, `SignOutButton`, `SignUpButton` |
| **Auth Flow** | `Callback` (handles OAuth2 redirect callback) |
| **Control / Guard** | `SignedIn`, `SignedOut`, `Loading` |
| **Presentation — Auth UI** | `SignIn`, `SignUp`, `AcceptInvite`, `InviteUser` |
| **Presentation — User** | `User`, `UserDropdown`, `UserProfile` |
| **Presentation — Organization** | `Organization`, `OrganizationList`, `OrganizationProfile`, `OrganizationSwitcher`, `CreateOrganization` |
| **Other** | `LanguageSwitcher` |

### Two-Layer Pattern

Every non-trivial component MUST have:
- `<ComponentName>` — styled default, zero configuration needed
- `Base<ComponentName>` — unstyled, logic-only, for full style override

All presentation components with non-trivial layout MUST provide a `Base*`
variant (e.g., `BaseSignIn`, `BaseUserProfile`, `BaseOrganizationSwitcher`).

---

## Configuration — Writing Guide

Always use grouped tables. Groups: **Core**, **Redirect URIs**, **OAuth2/OIDC**,
**Application Identity**, **Token Security**, **Session**, **Storage &
Platform**, **UI**.

Key constraints to always include:
- `baseUrl` must be HTTPS — HTTP MUST be rejected
- `clientSecret` is for confidential clients only — MUST NOT be used in
  browser/public clients
- `allowedExternalUrls` applies only when storage type is `webWorker`
- `syncSession` warning: may fail due to third-party cookie restrictions
- `clientId` is required for OAuth/OIDC (Redirect) mode but conditional overall
- `applicationId` is used for branding and sign-up URL resolution
- `organizationHandle` is required when a custom domain is configured

### Preferences (UI Config)

Preferences apply only to SDKs with bundled UI components:

- **ThemePreferences** — `mode` (light/dark/system), `direction` (ltr/rtl),
  `inheritFromBranding`, `overrides`
- **I18nPreferences** — `language`, `fallbackLanguage` (default `en-US`),
  `bundles`, `storageStrategy` (cookie/localStorage/none), `storageKey`,
  `cookieDomain`, `urlParam`
- **resolveFromMeta** — resolve theme from the server Flow Meta API
  (`GET /flow/meta`); applicable only for Asgardeo V2 / Thunder platform

---

## API Design Conventions

### Naming

| Operation Category | Method Prefix |
|-------------------|---------------|
| Authentication | `signIn`, `signOut` |
| Registration | `signUp`, `completeRegistration` |
| Recovery | `initiate*Recovery`, `confirm*Recovery` |
| Session | `refreshSession`, `getSession` |
| Token | `getTokens`, `decodeIDToken`, `getUserInfo` |
| Profile | `getUser*`, `updateUser*`, `changePassword` |
| Lifecycle | `initialize`, `reset` |

### Options Types

`SignInOptions`, `SignOutOptions`, `SignUpOptions` MUST be open, extensible
map/record types (not closed structs). Typed convenience properties MAY be
layered on top.

### Async Contract

All network I/O operations MUST be asynchronous using platform-native patterns:

| Platform | Pattern |
|----------|---------|
| JavaScript/TypeScript | `Promise<T>` / `async-await` |
| Java/Android | `CompletableFuture<T>` or callback with `Result<T, IAMError>` |
| Kotlin/Android | `suspend fun` returning `T` (coroutines) |
| Swift/iOS | `async throws` + `Result<T, IAMError>` |
| Python | `async def` returning `Awaitable[T]` |
| Dart/Flutter | `Future<T>` |

### Input Validation

- Required fields must be present and non-empty
- Email fields must match a valid email format
- Password fields must meet server-reported `PasswordPolicy`
- Invalid inputs MUST throw `InvalidInputException` synchronously

### Idempotency Rules

- `signOut()` when no session exists MUST succeed silently
- `initialize()` called more than once MUST throw `AlreadyInitializedException`
  unless `reset()` was called first
- `refreshSession()` called concurrently MUST deduplicate — only one refresh
  in flight at a time

---

## Error Handling

### Error Model

```
IAMError {
  code: ErrorCode
  message: String
  cause: Error?
  requestId: String?
  statusCode: Integer?
}
```

### Error Code Categories

| Category | Codes |
|----------|-------|
| **Configuration** | `SDK_NOT_INITIALIZED`, `ALREADY_INITIALIZED`, `INVALID_CONFIGURATION`, `INVALID_REDIRECT_URI` |
| **Authentication** | `AUTHENTICATION_FAILED`, `USER_ACCOUNT_LOCKED`, `USER_ACCOUNT_DISABLED`, `SESSION_EXPIRED`, `MFA_REQUIRED`, `MFA_FAILED`, `INVALID_GRANT`, `CONSENT_REQUIRED` |
| **Registration** | `USER_ALREADY_EXISTS`, `INVALID_INPUT`, `INVITATION_CODE_INVALID`, `INVITATION_CODE_EXPIRED`, `REGISTRATION_DISABLED` |
| **Recovery** | `RECOVERY_FAILED`, `CONFIRMATION_CODE_INVALID`, `CONFIRMATION_CODE_EXPIRED` |
| **Network & Server** | `NETWORK_ERROR`, `REQUEST_TIMEOUT`, `SERVER_ERROR`, `UNKNOWN_ERROR` |

### Error Conventions

- All errors MUST be catchable via standard platform error-handling
- SDK MUST NOT swallow errors silently
- Network retries MUST NOT be automatic (except token refresh)
- Error messages MUST NOT include sensitive data
- `requestId` MUST be populated from `Correlation-ID` / `X-Request-ID` headers

---

## Security — Non-Negotiable Rules

Always include these when writing security sections:

- PKCE: `S256` only. Plain method MUST NOT be used. `code_verifier` in memory
  only — never persisted.
- State param: cryptographically random, validated on callback. Mismatch →
  `AUTHENTICATION_FAILED`.
- ID token validation: verify signature (JWKS), `iss`, `aud`, `exp`, `nonce`.
  Support key rotation by re-fetching JWKS on failure before erroring.
- Token storage defaults: iOS → Keychain; Android → EncryptedSharedPreferences;
  Web → in-memory (never `localStorage` by default); `webWorker` for isolation;
  Desktop → OS credential store; Server-side → env var or OS keyring.
- Never log credentials, tokens, passwords, or OTPs at any log level.
- `decodeJwtToken()` has no signature verification → MUST NOT be used for
  authorization decisions.
- Access tokens MUST be refreshed before expiry (recommended: 60s before `exp`).
- Refresh tokens MUST be rotated on use; stored tokens updated atomically.
- On sign-out, revoke refresh token server-side (RFC 7009) before clearing
  local state.
- `allowedExternalUrls` MUST be enforced in `webWorker` storage mode — tokens
  only attached to allowlisted URLs.
- Log sanitization: mask access tokens (log only type + expiry), never log
  refresh tokens/passwords/OTPs, mask emails and phone numbers.

---

## Extensibility & Customization

The SDK MUST support the following extension points:

| Extension | Interface | Notes |
|-----------|-----------|-------|
| **Custom Storage** | `StorageAdapter { store, retrieve, delete, clear }` | Passed via `SDKConfig.storage` |
| **Custom Logger** | `LoggerAdapter { debug, info, warn, error }` | Default is no-op; sanitized data only |
| **Custom HTTP Client** | `HTTPAdapter { request }` → `HTTPResponse { statusCode, headers, body }` | For proxy, custom TLS, or testing |
| **Event Hooks** | `IAMClient.on(event, handler)` | Events: `SIGN_IN_SUCCESS`, `SIGN_IN_FAILED`, `SIGN_OUT`, `TOKEN_REFRESHED`, `TOKEN_REFRESH_FAILED`, `SESSION_EXPIRED`, `MFA_STEP_REQUIRED` |

---

## Repository Setup

### Philosophy

**Decision rule:** Start with a monorepo. Split into separate repositories only
when native toolchains are incompatible or when platform-specific team ownership
makes a shared repository impractical.

### JavaScript / TypeScript — Monorepo

**Repository:** `asgardeo/javascript`

| Package | Layer | Path |
|---------|-------|------|
| `javascript` | Agnostic | `packages/javascript` |
| `browser` | Platform | `packages/browser` |
| `node` | Platform | `packages/node` |
| `react` | Core Lib | `packages/react` |
| `vue` | Core Lib | `packages/vue` |
| `express` | Framework Specific | `packages/express` |
| `nextjs` | Framework Specific | `packages/nextjs` |
| `nuxt` | Framework Specific | `packages/nuxt` |
| `react-router` | Framework Specific | `packages/react-router` |
| `tanstack-router` | Framework Specific | `packages/tanstack-router` |

Rules: independently versioned + published to npm; workspace protocol during
dev; no layer skipping.

### Multi-Platform — Separate Repositories

| Repository | Contents | Layer |
|------------|----------|-------|
| `asgardeo/ios` | Swift SDK | Agnostic + Platform |
| `asgardeo/android` | Kotlin SDK | Agnostic + Platform |
| `asgardeo/flutter` | Dart SDK (delegates via Platform Channels) | Core Lib |

### Naming Conventions

| Ecosystem | Package Name | Convention |
|-----------|-------------|------------|
| JavaScript/TypeScript | `@asgardeo/*` | npm scope |
| iOS (Swift) | `Asgardeo` | PascalCase module |
| Android (Kotlin) | `io.asgardeo.*` | Reverse-domain |
| Flutter (Dart) | `asgardeo_flutter` | snake_case |
| Python | `asgardeo` | Lowercase |
| Go | `github.com/asgardeo/asgardeo-go` | Repository import path |

### Repository Checklist

Every new SDK repository must include before first release:
- `README.md` — installation, quick-start, link to this specification
- `CONTRIBUTING.md` — branching strategy, commit convention, PR process
- `LICENSE` — Apache 2.0
- `SECURITY.md` — responsible disclosure process
- CI pipeline — lint, build, test on every PR
- Release pipeline — automated publish on tag push
- Issue templates — bug report and feature request
- Sample application(s) — at least one runnable sample per SDK

---

## Documentation Requirements

Every SDK release MUST be accompanied by published documentation. An SDK
without docs MUST NOT be considered shippable.

- **Quickstart Guide** per SDK: zero to working sign-in in the shortest path.
  Published to both portals:
  - Asgardeo: `https://wso2.com/asgardeo/docs/quick-starts/<sdk-name>/`
  - WSO2 IS: `https://is.docs.wso2.com/en/latest/quick-starts/<sdk-name>/`
- **API Reference** — generated docs covering every public method, type, and
  config option. Indexed at:
  - Asgardeo: `https://wso2.com/asgardeo/docs/sdks/`
  - WSO2 IS: `https://is.docs.wso2.com/en/latest/integrations/`

---

## Sample Applications

Every SDK at Core Lib or Framework Specific layer MUST include at least one
runnable sample. Samples are first-class deliverables — an SDK MUST NOT be
considered shippable without them.

### Location

Samples live under `<repo-root>/samples/`, each a self-contained project with
its own `package.json` (or equivalent) and `README.md`.

### Required Samples

| SDK | Sample | What it demonstrates |
|-----|--------|---------------------|
| React | `b2c-react` | SPA: sign-in, profile, sign-out, protected route |
| Vue | `b2c-vue` | Same as React, with Vue 3 composable |
| Next.js | `b2c-nextjs` | App Router: public page, sign-in, protected server component |
| Nuxt | `b2c-nuxt` | Equivalent to Next.js, with Nuxt 3 |
| Angular | `b2c-angular` | SPA: sign-in, AuthGuard-protected profile, sign-out |
| Express | `express-protected` | Public `/health` + protected `/api/me` with bearer token |
| Node.js | `node-protected` | Same as Express but plain HTTP server / serverless |
| React Router | `b2c-react-router` | React Router v7: protected routes, callback route |
| TanStack Router | `b2c-tanstack-router` | TanStack Router: same scope as React Router |
| iOS | `b2c-ios` | SwiftUI: sign-in sheet, profile view, sign-out |
| Android | `b2c-android` | Jetpack Compose: sign-in, profile, sign-out |
| Flutter | `b2c-flutter` | Cross-platform: sign-in, profile, sign-out |
| Django | `django-protected` | Public index + protected `/profile` view |
| FastAPI | `fastapi-protected` | Public root + protected `/me` endpoint |

### Quality Standards

- Runs against a live Asgardeo trial org or local WSO2 IS with only `.env.example` vars
- Uses public SDK API exclusively (no internal imports)
- No hardcoded secrets or disabled security features
- CI runs sample build on every PR

### B2C Reference Flow (Minimum)

Client-side: unauthenticated state → sign-in → complete auth → authenticated
state with display name/email/avatar → sign-out → unauthenticated state.

Server-side: `GET /public` → 200; `GET /protected` without token → 401;
`GET /protected` with valid bearer → 200 with user claims.

---

## Document Structure Reference

The full spec has 18 sections. When extending or reviewing, use these as anchors:

| # | Section |
|---|---------|
| 1 | Introduction |
| 2 | SDK Architecture (four layers, Mermaid diagrams, ecosystem table, layer rules) |
| 3 | Guiding Principles |
| 4 | Operational Modes |
| 5 | SDK Initialization & Configuration (init patterns, config reference, preferences) |
| 6 | Identity Lifecycle Operations |
| 7 | Framework Integration (Client Interface, Init Patterns, Exposed API) |
| 8 | UI Components |
| 9 | API Design & Method Signatures |
| 10 | Error Handling |
| 11 | Security Requirements |
| 12 | Platform & Language Guidelines |
| 13 | Extensibility & Customization |
| 14 | Compliance & Standards |
| 15 | Repository Setup |
| 16 | Documentation Requirements |
| 17 | Sample Applications |
| 18 | Glossary |

---

## Common Tasks

### Adding a new SDK to the spec

1. Classify it into one of the four layers (Section 2.2 table)
2. Add it to the ecosystem table in Section 2.5
3. Update the Mermaid diagram for its ecosystem in Section 2.4 (or add a new
   diagram if it's a new ecosystem)
4. Add platform-specific storage, crypto, and UI notes to Section 12
5. Add component name adaptations to Section 8.4 if it has UI
6. Add the repository to Section 15 (monorepo package or separate repo)
7. Define the required sample in Section 17.3
8. Add quickstart guide and API reference to Section 16

### Implementing a mobile SDK (iOS / Android / Flutter)

Three separate repos, co-located in a `mobile-sdks/` workspace alongside a
`docs-is/` checkout:

| Repo | Package | Layer | Min versions |
|------|---------|-------|--------------|
| `ios-sdk` | SPM: `Asgardeo` | Agnostic + Platform | iOS 15.0, Swift 5.9+ |
| `android-sdk` | Maven: `io.asgardeo.android` | Agnostic + Platform | SDK 24, Kotlin 1.9+ |
| `flutter-sdk` | pub.dev: `asgardeo_flutter` | Core Lib | Flutter 3.16+, Dart 3.2+ |

Flutter bridges via `MethodChannel("io.asgardeo.flutter/sdk")` and
`EventChannel("io.asgardeo.flutter/events")`. It must NOT re-implement
protocol logic.

Implementation phases (in order):
0. Repo scaffolding + `docs-is` checkout
1. Models & configuration
2. Adapters & utilities (StorageAdapter, LoggerAdapter, HTTPAdapter, EventEmitter)
3. OIDC discovery & PKCE
4. Token management
5. Platform auth handlers (Keychain/EncryptedPrefs, ASWebAuthSession/ChromeTabs)
6. App-native authentication
7. AsgardeoClient assembly (all 19 methods + 3 properties)
8. SwiftUI / Jetpack Compose integration
9. Flutter platform channel bridge
10. Flutter UI widgets
11. Sample applications
12. Documentation & release prep

### Contributing documentation to docs-is

All SDK docs (quickstarts, API reference) live in `wso2/docs-is` and are
submitted as a PR. File locations:

- Quickstart: `en/includes/quick-starts/<sdk-name>.md`
- API reference: `en/includes/sdks/<sdk-name>/`
- Nav entries: added to **both** `en/asgardeo/mkdocs.yml` and
  `en/identity-server/next/mkdocs.yml`

Nav placement:
- Quickstart → `Get started > Connect App > <SDK name>`
- API reference → `SDK Documentation > <SDK name>`

Follow the React SDK as the reference pattern:
`en/includes/quick-starts/react.md` and `en/includes/sdks/react/`.

### Adding a new authenticator

1. Add to the authenticator tables in Section 6.1 (sign-in) with prompt type
   and required params
2. Add to the MFA step handler notes if it is an MFA factor
3. Update the UI component `SignIn` description in Section 8.4 if it needs
   new UI treatment

### Adding a new configuration field

1. Identify which group it belongs to (Core, Redirect URIs, OAuth2, Application
   Identity, Token Security, Session, Storage & Platform, UI)
2. Add a row to the correct grouped table in Section 5.2
3. If it's UI-related, add to the Preferences tables in Section 5.3
4. Add to the Glossary (Section 18) if it needs a definition

### Writing a new operation

Follow the writing guide above. Prose + tables. No code blocks defining types.
Check method name against the canonical method names table first.

### Adding a new error code

1. Add to the appropriate error code category table in Section 10.2
2. Document the trigger condition clearly
3. If the error is surfaced by a specific operation, mention it in that
   operation's implementor notes in Section 6

### Creating a new sample application

1. Follow the naming pattern in Section 17.3
2. Place under `samples/` in the SDK repository
3. Must be self-contained with its own README and `.env.example`
4. Must meet the quality standards in Section 17.4
5. Must demonstrate at least the B2C reference flow in Section 17.5

---

## Agent Guardrails

When using this skill, you MUST observe the following rules:

1. **Never use `login`, `logout`, or `register`** in any generated code,
   documentation, variable names, or comments. The only terms are `signIn`,
   `signUp`, `signOut`. If you catch yourself using the wrong term, correct it
   immediately.

2. **Never generate `localStorage`-based token storage** as a default in any
   web SDK code. Default to in-memory. Offer `webWorker` as the recommended
   upgrade.

3. **Never skip PKCE** in redirect-mode code. Every authorization request MUST
   include `code_challenge` + `code_challenge_method=S256`.

4. **Never put type-definition code blocks in operation sections** of the spec.
   Operations (Section 6) use prose + tables only. Method signatures belong in
   Section 7.1.

5. **Always check the layer** before writing import statements or dependency
   declarations. A Framework Specific SDK must not import directly from the
   Agnostic SDK — it goes through Platform → Core Lib → Framework Specific.

6. **Always use open map types for options** (`SignInOptions`, etc.), not closed
   structs. The user must be able to pass arbitrary server-specific parameters.

7. **When generating framework integration code**, expose exactly ONE entry
   point: `useAsgardeo()` (React), `AsgardeoService` (Angular),
   `useAsgardeo()` (Vue). Do not require multiple hooks or services.

8. **When writing error handling code**, use the exact error codes from the
   Error Code Categories table. Do not invent new codes without adding them to
   the spec first.

9. **When creating UI components**, always provide both `<ComponentName>` and
   `Base<ComponentName>` variants for non-trivial components.

10. **For the full specification**, always refer to `README.md` in the
    repository root. This skill is a condensed guide — the README is the
    source of truth.
