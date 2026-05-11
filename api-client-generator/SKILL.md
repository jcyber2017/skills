---
name: api-client-generator
description: 'Create or refactor a frontend TypeScript apiClient.ts using Axios with request and response interceptors, AWS auth token injection, request rate limiting, singleton initialization, and a React hook accessor. Use when the user asks for an API client scaffold, standardized HTTP client, or typed network layer in frontend projects.'
argument-hint: 'Target file path and required API modules/endpoints'
license: MIT
---

# API Client Generator

Generate a production-ready, well-typed `apiClient.ts` for frontend TypeScript projects.

The produced client must include:
- `axios.create` setup
- request interceptor with rate limiter + auth headers
- response interceptor with normalized error handling
- singleton instance lifecycle (`initializeApiClient` + exported instance)
- hook-style accessor (`useApiClient`) that throws if not initialized
- typed API modules/getters and typed user-level methods

## When To Use

Use this skill when the user asks to:
- create a new `apiClient.ts`
- standardize API calls around Axios interceptors
- add AWS authentication token handling
- add response error toast notifications to API calls
- enforce request throttling/rate limiting
- expose API client as a singleton with a hook accessor

## Inputs To Collect

Collect these inputs from the user request or existing code:
- target file path (default: `frontend/src/http/apiClient.ts`)
- auth provider symbol and methods (default pattern: `awsAuth.getCurrentUser()`, `awsAuth.fetchAuthSession()`)
- error UX integration (example: `toast`, `toApiClientError`, `markToastShown`)
- rate limiting parameters (`MAX_CREDITS`, `PERIOD_MS`, `COOLDOWN_MS`)

If the user does not provide one of these, infer from existing project conventions before introducing new patterns.

## Procedure

1. Inspect existing HTTP and auth conventions.
2. Create or update the target `apiClient.ts` with strongly typed imports and exported types.
3. Build the `ApiClient` class with:
  - private `axios: AxiosInstance`
  - optional domain context state only if required by module routing (for example scoped paths)
  - constructor that initializes `axios.create` with JSON headers
4. Add request interceptor logic:
  - consume a request credit via `RequestRateLimiter`
  - resolve current user; if authenticated, fetch session and ID token
  - set `Authorization: Bearer <token>` when token exists
  - throw on auth-token retrieval failure (hard-fail default)
5. Add response interceptor logic:
   - normalize error shape with `toApiClientError`
   - show destructive toast with user-facing message
   - rethrow via `Promise.reject(markToastShown(apiError))`
6. Add class API surface:
   - `isAuthenticated()`
   - typed module getters
   - typed `user` methods (for example user account endpoints)
7. Add singleton exports:
   - `export let apiClient: ApiClient;`
   - `initializeApiClient(baseUrl: string)`
   - `useApiClient()` guard that throws if uninitialized
8. Validate TypeScript correctness and fix errors introduced by the change.

## Decision Points

- Auth strategy:
  - If no authenticated user exists, skip token injection.
  - If session fetch fails for an authenticated user, throw a clear re-authentication error (default behavior).
- Header token source:
  - Prefer `session.tokens?.idToken` and support both string and token object via `.toString()`.
- Error propagation:
  - Always preserve rejected promise flow after toast side effects.

## Quality Checklist

Before finishing, ensure:
- all public methods and responses are explicitly typed
- interceptor callbacks return the expected Axios values/promises
- no `any` is introduced unless explicitly justified
- singleton exports are present and usable from React code
- hook accessor throws a clear initialization error
- rate limiter is initialized with documented constants
- imports align with project alias and lint conventions

## Output Requirements

Return a concise summary with:
- files created/edited
- key behaviors implemented (auth, rate limiting, error handling, singleton, hook)
- any assumptions made
- follow-up options (for example tests or endpoint module expansion)
