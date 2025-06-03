# IAM SDK Specification

## Table of Contents

1. [Introduction](#introduction)  
2. [Terminology](#terminology)  
3. [Implementation](#implementation)  
   3.1 [Architecture](#architecture)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.1.1 [Core Layer](#core-layer)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.1.2 [Platform Layer](#platform-layer)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.1.3 [Framework Layer](#framework-layer)  
   3.2 [SDK Configuration](#sdk-configuration)  
   3.3 [Public API](#public-api)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.3.1 [Entry Point](#entry-point)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.3.2 [Components (If applicable)](#components-if-applicable)  
   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.3.2.1 [UI / Full-stack Framework SDKs](#ui--full-stack-framework-sdks)  
   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.3.2.1.1 [Action Components](#action-components)  
   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.3.2.1.2 [Control Components](#control-components)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.3.3 [Route Guards (If applicable)](#route-guards-if-applicable)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.3.4 [Composables / Hooks / Services](#composables--hooks--services)  
   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.3.4.1 [Composables](#composables)  
   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.3.4.2 [Hooks](#hooks)  
   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.3.4.3 [Services](#services)  
   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.3.4.4 [Others](#others)  
   3.4 [Naming Conventions](#naming-conventions)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.4.1 [General Rules](#general-rules)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.4.2 [Platform-Specific Guidelines](#platform-specific-guidelines)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.4.3 [Configuration Keys](#configuration-keys)  
   &nbsp;&nbsp;&nbsp;&nbsp;3.4.4 [Package Naming](#package-naming)  
4. [Coding Style](#coding-style)  
   4.1 [Formatting](#formatting)  
   4.2 [File Naming](#file-naming)  
   4.3 [Variable and Function Naming](#variable-and-function-naming)  
   4.4 [Constants Naming and Scoping](#constants-naming-and-scoping)  
   4.5 [Enums, Interfaces and Types](#enums-interfaces-and-types)  

---

## 1. Introduction

This document specifies the strategic design, implementation guidelines, and governance model for the WSO2 IAM SDK family. The goal is to provide a consistent, secure, and developer-friendly set of SDKs across multiple platforms, enabling developers to easily integrate with WSO2 IAM products or any other IAM vendor.

The document defines normative requirements for SDK behavior, packaging, versioning, and documentation, and establishes guiding principles for future SDK development.

---

## 2. Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

- **SDK:** The provided Software Development Kit.  
- **Client:** An application or service integrating an SDK.  
- **Developer:** An individual or team integrating IAM functionality into their product using the SDK.  
- **Platform:** The programming language, environment, or runtime targeted by an SDK.

---

## 3. Implementation

### 3.1 Architecture

SDKs **SHALL** be structured into three layers:

#### 3.1.1 Core Layer

Contains platform-agnostic logic, common utilities, and shared protocol implementations (e.g., OIDC flows, token validation, PKCE). This layer **SHALL NOT** depend on platform-specific libraries or frameworks and **SHALL** be reusable across SDKs.

#### 3.1.2 Platform Layer

Contains platform-specific integrations, user-facing APIs, and runtime-specific behaviors (e.g., React Hooks, Node.js middleware, Android Activities). This layer **SHALL** adapt the core layer to the idiomatic patterns and developer expectations of the target platform.

#### 3.1.3 Framework Layer

Provides **framework-specific bindings** or helpers to integrate the SDK smoothly within popular frameworks or libraries (e.g., React, Angular, Vue, Spring Boot, Django).

This layer **SHALL** extend the platform layer to offer higher-level abstractions, components, or annotations that align with the selected **framework’s conventions**.

Framework layers **MAY** be published as separate packages or modules to keep the core and platform layers lightweight.

### 3.2 SDK Configuration

Each SDK **SHALL** support a standardized configuration mechanism that initializes the SDK with the necessary parameters. This ensures the SDK connects properly to the identity server and aligns with client integration needs.

The configuration **SHALL** include at minimum:


  | Key              | Description                                                       | Notes                                      |
  |------------------|-------------------------------------------------------------------|--------------------------------------------|
  | `clientId`       | The client’s unique identifier registered with the identity server. | Required                                    |
  | `baseUrl`        | Base URL of the OpenID Connect-compliant identity provider.       | Required                                    |
  | `clientSecret`   | Client secret for confidential client flows.                      | Only if confidential flow is used           |
  | `afterSignInUrl` | URL to redirect after successful sign-in.                        | OAuth code received here during auth flow   |

Each SDK **SHALL** validate configuration at initialization and provide clear error messages for missing/invalid fields.  

For example: 
- In **React** or **Angular**, configuration is passed through the **Provider** or **Module**.
- In **Vue**, configuration is passed through the **plugin** installation using `app.use()`.

### 3.3 Public API

#### 3.3.1 Entry Point

Each SDK’s public API **SHALL** expose an entry point tailored to the respective platform or framework. These entry points will provide access to the core SDK functionalities while maintaining consistency across environments.
 
SDKs for **Angular**, which follow a [module pattern](https://angular.dev/guide/ngmodules/overview), **SHALL** expose a module (e.g., `AuthModule`) and **SHOULD** provide a service (e.g., `AuthService`) that integrates with Angular’s dependency injection system.

For **React** and **React-based frameworks** (e.g., React (Vanilla), Next.js, Remix), the entry point **SHALL** be a [context provider](https://legacy.reactjs.org/docs/context.html) (e.g., `AuthProvider`) component that wraps the main application or specific subtrees. It **SHOULD** also expose a React hook(s) (e.g., `useAuth()`) to access authentication state, trigger login/logout, and fetch user details.

#### 3.3.2 Components (If applicable)

Components **SHALL** follow the design system conventions of the respective framework and allow customization (e.g., styling, event hooks). Where applicable, they **MUST** expose props or configuration options to integrate seamlessly into the client application’s UI.

Components **SHOULD** be distributed as part of the main SDK package.

##### 3.3.2.1 UI / Full-stack Framework SDKs

The SDK **MAY** provide prebuilt UI components for simpler integration and better developer experience.

###### 3.3.2.1.1 Action Components

Components that **initiate or end authentication flows**.

- `SignInButton`
  - **Purpose**: Initiate login.
  - **Props**:
    - `children`: Optional custom label.
- `SignOutButton`
  - **Purpose**: Trigger logout.
  - **Props**:
    - `children`: Optional custom label.

###### 3.3.2.1.2 Control Components

Components that **conditionally render** their content based on the user's authentication state. 

- `SignedIn`
  - **Purpose**: Render content for authenticated users.
  - **Props**:
    - `children`: Content to render when signed in.
- `SignedOut`
  - **Purpose**: Render content for unauthenticated users.
  - **Props**:
    - `children`: Content to render when signed out.

#### 3.3.3 Route Guards (If applicable)

SDKs targeting frameworks with built-in routing (e.g., Angular, Vue Router) **SHALL** provide first-class support for protecting routes using **route guards** or **navigation guards**. This includes:
- Guards to restrict access to authenticated users.
- Redirects to login or error pages when unauthenticated or unauthorized.
- Hooks or callbacks to handle post-guard actions (e.g., showing a message or redirecting elsewhere).

For frameworks like **React (Vanilla)**, which do not come bundled with a routing solution, the SDK **SHALL** follow a **Bring Your Own Router (BYOR)** approach. In such cases, the SDK **SHOULD** provide **recipes** or documented examples to implement route guarding using popular routing solutions, such as **React Router**, **Wouter**, or **TanStack Router**. These recipes **MUST** be clearly documented to help developers understand how to integrate guard-like patterns within their chosen router.

SDK route guards **SHALL** be designed to work seamlessly with both **redirect-based flows** and **first-party (app-native) flows**, ensuring consistent behavior across environments.

#### 3.3.4 Composables / Hooks / Services

SDKs **SHALL*** expose platform-idiomatic mechanisms for programmatically accessing authentication state, user information, and authentication actions. Depending on the target framework or platform.

##### 3.3.4.1 Composables

Frameworks like **Vue.js** (Vue 3, Nuxt 3) and **SolidJS** that use the Composition API **SHALL** expose a composable `useAsgardeo` for accessing authentication-related data.

These composable **SHALL** provide:

- **Reactive authentication state**: Reactive references or signals indicating whether the user is authenticated (e.g., `isSignedIn`, `isLoading`).
- **User information**: Access to the user details (e.g., `user` object).
- **Authentication actions**: Functions for login, logout, session management, and token retrieval (e.g., `signIn()`, `signOut()`).

For **Vue 3** and **Nuxt 3**, these composables **SHALL**:
- Integrate with Vue’s reactivity system using `ref()`, `reactive()`, or other Composition API tools.
- Optionally integrate with Nuxt’s auto-import system to make composables available globally without manual imports.

For **SolidJS**, composables **SHALL** utilize Solid's reactive signals to provide real-time updates to the authentication state and user information.

##### 3.3.4.2 Hooks

For **React** and **React-based frameworks** (e.g., React, Next.js, Remix), the SDK **SHALL** expose a hook `useAsgardeo` that provide:

This hook **SHALL** provide:

- **Authentication state**: Variables or states that indicate if the user is authenticated (e.g., `isSignedIn`, `isLoading`).
- **User information***: Hooks that return user details (e.g., `user` object).
- **Authentication actions**: Functions to trigger login, logout, and fetch tokens (e.g., `signIn()`, `signOut()`).

##### 3.3.4.3 Services

For **Angular** applications, the SDK **SHALL** expose an injectable service (e.g., `AsgardeoService`).

This service **SHALL** provide:
- **Authentication state**: Observables (e.g., `isSignedIn$`, `isLoading$`) that components can subscribe to for authentication state changes using RxJS.
- **User information**: Object that returns user details (e.g., `user` object).
- **Authentication actions**: Functions to trigger login, logout, and fetch tokens (e.g., `signIn()`, `signOut()`).

The services **SHALL** integrate into Angular’s Dependency Injection (DI) system and be provided through the main `AsgardeoModule`. These services **SHALL** allow components to reactively update their state based on authentication events.

##### 3.3.4.4 Others

For other platforms or frameworks that do not fit into the above categories, the SDK **SHALL** provide appropriate mechanisms for programmatically accessing authentication state, user information, and authentication actions.

### 3.4 Naming Conventions

To ensure consistency, clarity, and ease of adoption across all SDKs and platforms, the following naming conventions **SHALL** be followed.

#### 3.4.1 General Rules

Use **PascalCase** for class names, component names, service names, and exported types.

For example:
- `AuthProvider`
- `SignInButton`
- `AsgardeoService`


Use **camelCase** for variables, functions, hooks, composables, and configuration keys.

For example:
- `signIn()`
- `useAsgardeo()`
- `clientId`
- `baseUrl`


Use **kebab-case** for package names.

For example:
- `@asgardeo/react`
- `@asgardeo/vue`

Abbreviations within names (e.g., URL, ID, PKCE) SHALL follow **PascalCase** or **camelCase** casing:
- `URL` → `Url` (e.g., `baseUrl`, `redirectUrl`)
- `ID` → `Id` (e.g., `clientId`, `sessionId`)
- `PKCE` → `Pkce` (e.g., `enablePkce`)
- `OIDC` → `Oidc` (e.g., `oidcProviderUrl`)

#### 3.4.2 Platform-Specific Guidelines

- **React / Next.js / Remix**
  - Hooks: `useXxx()` (e.g., `useAsgardeo`)
  - Context: `XxxProvider` (e.g., `AuthProvider`)
  - Components: PascalCase, descriptive (e.g., `SignInButton`, `SignedOut`)

- **Vue / Nuxt**
  - Composables: `useXxx()` (e.g., `useAsgardeo`)
  - File names: `use-xxx.ts` (e.g., `use-asgardeo.ts`)
  - Plugins: `app.use(asgardeo)` convention

- **Angular**
  - Services: `XxxService` (e.g., `AsgardeoService`)
  - Modules: `XxxModule` (e.g., `AsgardeoModule`)
  - Observables: End with $ (e.g., `isSignedIn$`)

#### 3.4.3 Configuration Keys

All configuration keys **SHALL** use **camelCase**.

For example:
- `clientId`
- `baseUrl`
- `redirectUrl`
- `enablePkce`
- `postLogoutRedirectUrl`


#### 3.4.4 Package Naming

Official packages **MUST** follow the format: `@asgardeo/<platform>` or `@asgardeo/<platform>-<framework>`

For example:
- `@asgardeo/javascript`  
- `@asgardeo/node`
- `@asgardeo/react`
- `@asgardeo/nextjs`
- `@asgardeo/angular`

---

## 4. Coding Style

### 4.1 Formatting

- Use **Prettier** with project-default configuration for consistent code formatting.
- Maintain **2 spaces** indentation.
- Use **single quotes** (') for strings except when using template literals or when double quotes improve readability.
- Always include a **semicolon** at the end of statements.
- Limit line length to **100 characters** where possible.
- Use **trailing commas** in multiline objects, arrays, and function parameters for cleaner diffs.
- Keep **blank lines** between logical code blocks to improve readability.

### 4.2 File Naming

Use **PascalCase** for file names that export a default class or component matching the export name.

For example:

```typescript
// File: AuthClient.ts
class AuthClient { ... }

export default AuthClient;
```

Use **camelCase** or **kebab-case** for utility, constant, or helper files that export multiple named exports or default objects.

File extensions should always match the language, e.g., `.ts` for TypeScript, `.tsx` for React components with JSX.

### 4.3 Variable and Function Naming

- Use **camelCase** for variables, functions, and method names.
- For acronyms in names, capitalize only the first letter for readability (e.g., `getPkceStorageKeyFromState` instead of `getPKCEStorageKeyFromState`).
- Use **descriptive and meaningful names**. Avoid vague names like `data`, `item`, or `value` unless in very limited scope.
- Use verbs for function names that perform actions (e.g., `fetchUser`, `calculateSum`).
- Prefix boolean variables or functions that return booleans with `is`, `has`, or `can` (e.g., `isAuthenticated`, `hasAccess`).

### 4.4 Constants Naming and Scoping

- Constants should be **scoped** inside descriptive constant objects instead of declared as loose top-level constants.
- Use **PascalCase** for constant object names (e.g., `PkceConstants`).
- Use **UPPER_SNAKE_CASE*** for keys inside the constant object.
- Mark the constant object as `as const` to ensure immutability and strong typing.
- Export constant objects as **default exports** if they are the main export of the file.

For example:

```typescript
const PkceConstants = {
  PKCE_CODE_VERIFIER: 'pkce_code_verifier',
  PKCE_SEPARATOR: '#',
} as const;

export default PkceConstants;
```

This approach groups related constants logically, avoiding global namespace pollution. Improves discoverability and makes it clear which constants belong together and facilitates easier refactoring and code readability.

### 4.5 Enums, Interfaces and Types

- Enums in PascalCase (e.g., `AuthStatus`).  
- Interfaces prefixed with `I` or not based on team preference (consistent throughout).  
- Use explicit types for all public APIs.  
