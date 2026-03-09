---
name: wso2-iam-sdk-spec
description: >
  Author, extend, or review documentation and specifications for WSO2 IAM SDKs
  (Asgardeo / WSO2 Identity Server). Use this skill whenever the user wants to
  write or update an SDK specification, design a new SDK layer, define an API
  contract, document identity lifecycle operations (sign-in, sign-up, sign-out,
  recovery), design UI component catalogues for IAM SDKs, define framework
  integration patterns (React, Angular, Vue, Flutter, iOS, Android, Python,
  Node.js), or apply the WSO2 IAM SDK layered architecture. Also trigger for
  questions about how WSO2 app-native authentication flows work, the Flow
  Execution API, or how OAuth2/OIDC maps to SDK method signatures.
---

# WSO2 IAM SDK Specification Skill

This skill encodes the conventions, architecture, and content decisions made in
the WSO2 IAM SDK Specification. Use it to author new sections, review
contributions from external parties, extend the spec for new platforms, or
answer questions about SDK design decisions.

---

## Core Principles (non-negotiable)

These apply to everything — method names, prose, diagrams, tables:

1. **Terminology is strict.** Always `signIn`, `signUp`, `signOut`. Never
   `login`, `logout`, `register` in any public API surface, prose description,
   or example. OIDC protocol values (`prompt=login`) and field names inherited
   from standards (`loginHint`, `post-logout`) are exempt.

2. **Spec is language-agnostic.** Pseudocode only in API contracts. No
   TypeScript, Swift, Kotlin, or Python syntax in the core spec body. Platform
   examples belong in Section 11 (Platform & Language Guidelines) or in
   framework-specific implementation guides.

3. **No type-definition code blocks in operation sections.** Identity lifecycle
   operations (sign-in, sign-up, recovery, etc.) are described in prose and
   tables — not as pseudocode structs. Method signatures belong only in the
   Client Interface section (Section 7.1).

4. **Configuration is tables only.** No pseudocode schema blocks for config or
   preferences. Use grouped tables (Core, OAuth2, Redirect URIs, etc.).

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

| Layer | What it does | Has UI? | JS example |
|-------|-------------|---------|------------|
| **Agnostic SDK** | Full `IAMClient` implementation. All OAuth2/OIDC protocol logic, JWT handling, error model. Zero platform or framework deps. | No | JavaScript SDK |
| **Platform SDK** | Extends Agnostic SDK with platform-specific capabilities: native crypto, storage, redirect handling. | No | Browser SDK, Node.js SDK |
| **Core Lib SDK** | Extends Platform SDK with reactive framework primitives. Single developer entry point. UI components optional. | Optional | React SDK, Vue SDK |
| **Framework Specific SDK** | Thin integration on top of Core Lib SDK. Adds SSR, routing conventions, opinionated DX. | Optional | Next.js, Nuxt, Angular |

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

## Operational Modes

Every SDK MUST support both modes transparently behind the same public API:

| Mode | Mechanism | Key rule |
|------|-----------|----------|
| **Redirect-Based** | OAuth2 Authorization Code + PKCE. Browser redirect to IAM sign-in page. | PKCE `S256` is mandatory. State param is mandatory for CSRF. |
| **App-Native** | Direct API calls. No browser redirects. WSO2-proprietary. | Uses `/oauth2/authorize?response_mode=direct` to initiate, then `/oauth2/authn` for each step. For sign-up/recovery, uses Flow Execution API (`POST /flow/execute`). |

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
| Switch organization | `switchOrganization()` |

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
| **Auth UI** | `SignIn`, `SignUp`, `AcceptInvite`, `InviteUser` |
| **User** | `User`, `UserDropdown`, `UserProfile` |
| **Organization** | `Organization`, `OrganizationList`, `OrganizationProfile`, `OrganizationSwitcher`, `CreateOrganization` |
| **Other** | `LanguageSwitcher` |

### Two-Layer Pattern

Every non-trivial component MUST have:
- `<ComponentName>` — styled default, zero configuration needed
- `Base<ComponentName>` — unstyled, logic-only, for full style override

---

## Configuration — Writing Guide

Always use grouped tables. Groups: Core, Redirect URIs, OAuth2/OIDC,
Application Identity, Token Security, Session, Storage & Platform, UI.

Key constraints to always include:
- `baseUrl` must be HTTPS — HTTP MUST be rejected
- `clientSecret` is for confidential clients only — MUST NOT be used in
  browser/public clients
- `allowedExternalUrls` applies only when storage type is `webWorker`
- `syncSession` warning: may fail due to third-party cookie restrictions

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
  Web → in-memory (never `localStorage` by default); `webWorker` for isolation.
- Never log credentials, tokens, passwords, or OTPs at any log level.
- `decodeJwtToken()` has no signature verification → MUST NOT be used for
  authorization decisions.

---

## Document Structure Reference

The full spec has 15 sections. When extending or reviewing, use these as anchors:

| # | Section |
|---|---------|
| 1 | Introduction |
| 2 | SDK Architecture (four layers, Mermaid diagrams, JS example) |
| 3 | Guiding Principles |
| 4 | Operational Modes |
| 5 | SDK Initialization & Configuration |
| 6 | Identity Lifecycle Operations |
| 7 | Framework Integration (Client Interface, Init Patterns, Exposed API) |
| 8 | UI Components |
| 9 | API Design & Method Signatures |
| 10 | Error Handling |
| 11 | Security Requirements |
| 12 | Platform & Language Guidelines |
| 13 | Extensibility & Customization |
| 14 | Compliance & Standards |
| 15 | Glossary |

---

## Common Tasks

### Adding a new SDK to the spec

1. Classify it into one of the four layers (Section 2.2 table)
2. Add it to the ecosystem table in Section 2.5
3. Update the Mermaid diagram for its ecosystem in Section 2.4 (or add a new
   diagram if it's a new ecosystem)
4. Add platform-specific storage, crypto, and UI notes to Section 11
5. Add component name adaptations to Section 8.4 if it has UI

### Adding a new authenticator

1. Add to the authenticator tables in Section 6.1 (sign-in) with prompt type
   and required params
2. Add to the MFA step handler notes if it is an MFA factor
3. Update the UI component `SignIn` description in Section 8.4 if it needs
   new UI treatment

### Adding a new configuration field

1. Identify which group it belongs to (Core, OAuth2, Token Security, etc.)
2. Add a row to the correct grouped table in Section 5.2
3. If it's UI-related, add to the Preferences tables in Section 5.3
4. Add to the Glossary (Section 15) if it needs a definition

### Writing a new operation

Follow the writing guide above. Prose + tables. No code blocks defining types.
Check method name against the canonical method names table first.
