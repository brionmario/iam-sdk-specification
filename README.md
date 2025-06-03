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

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

- **SDK:** The provided Software Development Kit.  
- **Client:** An application or service integrating an SDK.  
- **Developer:** An individual or team integrating IAM functionality into their product using the SDK.  
- **Platform:** The programming language, environment, or runtime targeted by an SDK.

---

## 3. Implementation

### 3.1 Architecture

SDKs **SHALL** be structured into three layers:

#### 3.1.1 Core Layer

- Contains platform-agnostic logic, common utilities, and shared protocol implementations (e.g., OIDC flows, token validation, PKCE).  
- **MUST NOT** depend on platform-specific libraries or frameworks.  
- **SHALL** be reusable across SDKs.

#### 3.1.2 Platform Layer

- Contains platform-specific integrations, user-facing APIs, and runtime-specific behaviors (e.g., React Hooks, Node.js middleware, Android Activities).  
- **SHALL** adapt the core layer to idiomatic patterns and developer expectations of the target platform.

#### 3.1.3 Framework Layer

- Provides framework-specific bindings or helpers for popular frameworks or libraries (e.g., React, Angular, Vue, Spring Boot, Django).  
- **SHALL** extend the platform layer offering higher-level abstractions, components, or annotations aligned with framework conventions.  
- **MAY** be published as separate packages or modules to keep core and platform layers lightweight.

### 3.2 SDK Configuration

- Each SDK **SHALL** support a standardized configuration mechanism initializing the SDK with necessary parameters.  
- The configuration **SHALL** include at minimum:

  | Key              | Description                                                       | Notes                                      |
  |------------------|-------------------------------------------------------------------|--------------------------------------------|
  | `clientId`       | The client’s unique identifier registered with the identity server. | Required                                    |
  | `baseUrl`        | Base URL of the OpenID Connect-compliant identity provider.       | Required                                    |
  | `clientSecret`   | Client secret for confidential client flows.                      | Only if confidential flow is used           |
  | `afterSignInUrl` | URL to redirect after successful sign-in.                        | OAuth code received here during auth flow   |

- SDK **SHALL** validate configuration at initialization and provide clear error messages for missing/invalid fields.  
- Examples:  
  - React or Angular: Configuration via Provider or Module.  
  - Vue.js: Configuration via plugin installation (`app.use()`).

### 3.3 Public API

#### 3.3.1 Entry Point

- Each SDK **SHALL** expose an entry point tailored to the respective platform or framework providing access to core functionalities with consistent behavior across environments.  
- Examples:  
  - Angular: Expose a module (`AuthModule`) and a service (`AuthService`) integrated with Angular DI.  
  - React: Expose a context provider component (`AuthProvider`) and hooks (e.g., `useAuth()`).

#### 3.3.2 Components (If applicable)

##### 3.3.2.1 UI / Full-stack Framework SDKs

- The SDK **MAY** provide prebuilt UI components for simpler integration and better developer experience.

###### 3.3.2.1.1 Action Components

| Component       | Purpose          | Props                    |
|-----------------|------------------|--------------------------|
| `SignInButton`  | Initiate login   | `children`: Optional custom label |
| `SignOutButton` | Trigger logout   | `children`: Optional custom label |

###### 3.3.2.1.2 Control Components

| Component   | Purpose                               | Props                   |
|-------------|-------------------------------------|-------------------------|
| `SignedIn`  | Renders content when user is signed in | `children`: Content to render |
| `SignedOut` | Renders content when user is signed out | `children`: Content to render |

- Components **SHALL** follow design system conventions of their framework and allow customization (styling, event hooks).  
- Components **MUST** expose props/configuration for seamless client UI integration.  
- Components **SHOULD** be distributed as part of the main SDK package.

#### 3.3.3 Route Guards (If applicable)

- SDKs targeting frameworks with routing (e.g., Angular, Vue Router) **SHALL** provide route guards/navigation guards that:  
  - Restrict access to authenticated users.  
  - Redirect to login/error pages if unauthenticated/unauthorized.  
  - Provide hooks/callbacks for post-guard actions (messages, redirects).  

- For React (Vanilla) without built-in routing:  
  - SDK **SHALL** follow Bring Your Own Router (BYOR) approach.  
  - Provide documented recipes/examples for route guarding with popular routers (React Router, Wouter, TanStack Router).

- Route guards **SHALL** work with both redirect-based and first-party (app-native) flows.

#### 3.3.4 Composables / Hooks / Services

SDKs **SHALL** expose platform-idiomatic mechanisms for accessing auth state, user info, and actions:

##### 3.3.4.1 Composables

- For Vue 3, Nuxt 3, SolidJS (Composition API):  
- Expose a composable `useAsgardeo` with:  
  - Reactive auth state (e.g., `isAuthenticated`, `user`, `tokens`).  
  - Methods (`login()`, `logout()`, `getAccessToken()`, etc.).

##### 3.3.4.2 Hooks

- For React, Preact, React Native:  
- Expose hooks such as `useAuth()` or `useAsgardeo()` with the same capabilities as composables.

##### 3.3.4.3 Services

- For Angular, Ember, Flutter, native platforms:  
- Provide injectable services or singleton classes exposing auth APIs.

##### 3.3.4.4 Others

- Other idiomatic platform APIs as necessary (e.g., Observables in RxJS, Callbacks).

### 3.4 Naming Conventions

#### 3.4.1 General Rules

- Use `camelCase` for variables and functions.  
- Use `PascalCase` for types, classes, and components.  
- Constants are UPPERCASE with underscores.  
- Avoid abbreviations unless widely recognized.

#### 3.4.2 Platform-Specific Guidelines

- React:  
  - Hooks prefixed with `use` (e.g., `useAuth`).  
  - Components use PascalCase (e.g., `<SignInButton />`).

- Angular:  
  - Services suffixed with `Service` (e.g., `AuthService`).  
  - Modules suffixed with `Module` (e.g., `AuthModule`).

- Vue:  
  - Composables prefixed with `use` (e.g., `useAuth`).  
  - Components in PascalCase.

#### 3.4.3 Configuration Keys

- Use lowercase with camelCase keys (e.g., `clientId`, `baseUrl`).

#### 3.4.4 Package Naming

- Use the prefix `@asgardeo` followed by platform/framework, e.g.:  
  - `@asgardeo/auth-react`  
  - `@asgardeo/auth-angular`  
  - `@asgardeo/auth-core`

---

## 4. Coding Style

### 4.1 Formatting

- Follow Prettier formatting with 2 spaces indentation.  
- Use Unix-style line endings (LF).  
- Max line length 100 characters.  

### 4.2 File Naming

- Use kebab-case for file names (e.g., `sign-in-button.tsx`).  
- Keep file names descriptive and concise.

### 4.3 Variable and Function Naming

- Descriptive, meaningful names.  
- Avoid single-character or overly abbreviated names except in short loops.

### 4.4 Constants Naming and Scoping

- Use `const` for constants.  
- Name constants in uppercase with underscores (e.g., `MAX_RETRIES`).  
- Keep constants scoped as locally as possible.

### 4.5 Enums, Interfaces and Types

- Enums in PascalCase (e.g., `AuthStatus`).  
- Interfaces prefixed with `I` or not based on team preference (consistent throughout).  
- Use explicit types for all public APIs.  
