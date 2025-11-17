# Middleware Support Spec

## Overview

React Router middleware (stable since 7.9.0) lets us run the AuthKit session refresh once per request and share the resulting auth payload with every loader/action via the router context. Today each loader invokes `authkitLoader`/`authkitAction` in isolation, so cookies are parsed multiple times and refresh logic may run repeatedly. This spec redesigns AuthKit around a root middleware that performs the refresh, caches the result in context, and appends new `Set-Cookie` headers while unwinding the middleware stack. The existing APIs continue to work for apps that have not enabled middleware yet.

## Goals

- Refresh tokens and manage `Set-Cookie` headers inside middleware so loaders/actions can rely on shared context.
- Expose auth data (`AuthorizedData | UnauthorizedData`) plus helpers (`getAccessToken`) through a typed context created with `unstable_createContext`.
- Mirror the ergonomics used by Sergio’s `remix-utils` middleware (factory that returns `[middleware, helper]`).
- Document how to enable `future.v8_middleware`, update `getLoadContext`/`getContext`, and migrate loaders progressively.

## Non-Goals

- Changing WorkOS token refresh semantics or storage layout.
- Forcing immediate migration away from `authkitLoader`/`authkitAction`.
- Introducing new persistence/caching beyond the router context.

## Lessons from Remix Utils Middleware

Sergio’s middleware catalog demonstrates patterns that map cleanly onto WorkOS AuthKit:

- **Factory outputs middleware + helpers**: Every `createXMiddleware()` returns a tuple so route modules simply export `middleware = [mw]` and import `getX(context)` wherever needed. We should adopt the same API to keep AuthKit usage symmetrical with other middleware ecosystems.
- **Single source of truth per-request**: Session/singleton/batcher middleware write to router context exactly once, guaranteeing loaders/actions operate on the same instance. Our auth middleware must provide the exact same session instance (headers/access token) everywhere, avoiding repeated cookie parsing.
- **Post-response hooks**: Session middleware auto-commits when data changes, logger/timing middleware read/modify responses after `await next()`. AuthKit needs this to append refreshed cookies even if downstream loaders already returned a `Response`.
- **Optional hooks for customization**: Logger middleware accepts `formatMessage`, session middleware accepts `shouldCommit`. AuthKit can expose `onSessionCommitted`/`shouldCommitSession` to allow advanced control (e.g., skipping writes when headers already set).
- **AsyncLocal shortcuts**: Context-storage middleware exposes zero-arg getters bundling router context + request. We can optionally ship a companion getter so userland code can call `getAuth()` without threading loader args, while still using the same middleware context.

Leveraging these patterns keeps our API surface idiomatic for React Router users already familiar with remix-utils helpers.

### Example Wiring

```tsx filename=app/root.tsx
import { Links, Meta, Outlet } from "react-router";
import { createAuthMiddleware } from "~/authkit/middleware";

export const [
  sessionMiddleware,
  getAuth,
  requireAuth,
] = createAuthMiddleware({
  ensureSignedIn: false, // root just refreshes + shares session
});

export const middleware = [sessionMiddleware];

export async function loader({ context }: Route.LoaderArgs) {
  const auth = getAuth(context); // safe even for anonymous users
  return { user: auth.auth.user };
}

export default function Root() {
  return (
    <html>
      <head>
        <Meta />
        <Links />
      </head>
      <body>
        <Outlet />
      </body>
    </html>
  );
}
```

```tsx filename=app/routes/dashboard.tsx
import { json, redirect, type Route } from "react-router";
import { requireAuth } from "~/authkit/middleware";

export async function loader({ context }: Route.LoaderArgs) {
  const auth = requireAuth(context); // throws redirect if not signed in
  const profile = await fetchProfile(auth.getAccessToken());
  return json({ profile });
}

export async function action({ request, context }: Route.ActionArgs) {
  const auth = requireAuth(context);
  const formData = await request.formData();
  await updateSettings(auth.getAccessToken(), formData);
  return redirect("/dashboard");
}
```

### Route-level Enforcement

Routes that must be authenticated can register an additional middleware instance (or just call `requireAuth`), letting public routes coexist in the same tree:

```tsx filename=app/routes/_auth.tsx
import { createAuthMiddleware, requireAuth } from "~/authkit/middleware";

export const [authOnlyMiddleware] = createAuthMiddleware({
  ensureSignedIn: true, // redirects when no session
});

export const middleware = [authOnlyMiddleware];

export async function loader({ context }: Route.LoaderArgs) {
  const auth = requireAuth(context);
  return json({ user: auth.auth.user });
}
```

Sibling routes that omit `authOnlyMiddleware` stay public but still benefit from the root `sessionMiddleware` (shared refresh + cookies). API/resource routes can follow the same pattern: export `middleware = [authOnlyMiddleware]` on protected handlers and skip it on public endpoints.

## Architecture

### Shared Auth Evaluation

Introduce `resolveAuthForRequest(request, options)` that encapsulates the existing loader/action logic:

- Validates or refreshes tokens via `updateSession`.
- Returns `{ auth, accessToken, sessionHeaders, refreshed, redirect? }`.
- Fires `onSessionRefreshSuccess`/`onSessionRefreshError` at most once.
- Provides a `commit(response)` helper to append refreshed cookies (similar to Remix Utils session middleware’s auto-commit behavior).

### Context & Factory API

```ts
import { unstable_createContext, type Route } from "react-router";

interface AuthContextValue {
  auth: AuthorizedData | UnauthorizedData;
  getAccessToken(): string | null;
  sessionHeaders?: Record<string, string>;
  refreshed: boolean;
}

export const authContext = unstable_createContext<AuthContextValue | null>(null);
```

`createAuthMiddleware(options?: AuthKitLoaderOptions)` returns a tuple mirroring `createSessionMiddleware`:

```ts
type AuthMiddlewareTuple = [
  Route.MiddlewareFunction,
  (context: unknown) => AuthContextValue,
  (context: unknown) => AuthContextValue & { ensureSignedIn(): asserts auth is AuthorizedData },
];
```

Usage example:

```ts
const [authMiddleware, getAuth, requireAuth] = createAuthMiddleware({ ensureSignedIn: true });

export const middleware = [authMiddleware];

export async function loader({ context }: Route.LoaderArgs) {
  const auth = requireAuth(context);
  const profile = await fetchProfile(auth.getAccessToken());
  return json({ profile });
}
```

### Server Middleware Behavior

1. Call `resolveAuthForRequest(request, options)`.
2. If it returns a redirect (unauthenticated + `ensureSignedIn`), throw immediately.
3. Store the `AuthContextValue` via `context.set(authContext, value)`.
4. `const response = await next();`
5. If `sessionHeaders?.["Set-Cookie"]` exists, append it before returning (just like Remix Utils session middleware auto-commits).
6. Optionally expose a `shouldCommit` hook akin to `createSessionMiddleware(storage, shouldCommit)` so apps can skip writing cookies (useful for read-only routes).

### Client Middleware (Future)

A matching `createClientAuthMiddleware()` can hydrate the same context on purely client-side navigations, similar to how logger/server-timing middleware ship both server and client versions. This can ship later without blocking server middleware adoption.

### Loader/Action Integration

Provide `getAuthFromContext(args)` that:

1. Detects middleware mode by checking for `.get/.set` on `context`.
2. Returns the stored value when middleware ran.
3. Falls back to `withAuth(args)` or `resolveAuthForRequest` when middleware isn’t enabled.

Update `authkitLoader`, `authkitAction`, and `withAuth` to call `getAuthFromContext` first so they become thin compatibility wrappers.

### Optional AsyncLocal Storage Hook

Pairing with a `createContextStorageMiddleware` (from `remix-utils`) lets us expose zero-arg helpers:

```ts
export function getAuth() {
  const context = getContext();
  return getAuthFromRouterContext(context);
}
```

This mirrors Sergio’s pattern where batcher/singleton getters no longer require passing `context` explicitly.

## Migration & Usage Guide

1. **Enable middleware flag**
   ```ts
   export default { future: { v8_middleware: true } } satisfies Config;
   ```
2. **Create middleware**
   ```ts
   export const [authMiddleware, getAuth, requireAuth] = createAuthMiddleware({ ensureSignedIn: true });
   export const middleware = [authMiddleware];
   ```
3. **Custom servers** – `getLoadContext` must return `new RouterContextProvider()` and seed legacy fields via `.set` (mirrors Remix docs for middleware).
4. **Loaders/actions** – either:
   - Use plain loaders calling `getAuth(context)` / `requireAuth(context)`; or
   - Keep `authkitLoader`/`authkitAction` during migration—they detect middleware context automatically.
5. **Client/data routers** – optionally add `getContext()` to hydrate a base context for SPA navigations; client middleware hook can be introduced later.

## Testing Strategy

- Unit tests for `resolveAuthForRequest` (redirects, refresh success/error hooks, returned headers).
- Middleware tests ensuring:
  - Refresh runs once per request and memoizes auth in context.
  - `Set-Cookie` is appended exactly once even if downstream loaders return `Response`s with their own headers (similar to session middleware tests in Remix Utils).
  - `getAuth`/`requireAuth` throw informative errors when middleware isn’t registered.
  - Legacy mode (no middleware) still works.
- Integration test covering opt-in flow: enable flag, register middleware, consume `getAuth` in a nested loader, verify tokens refresh & headers propagate.

## Open Questions / Future Work

- Do we ship client middleware immediately or wait for a clear SPA-only requirement?
- Should we expose a `useAuth()` hook for components analogous to `useLoaderData`, powered by the context value?
- Should we offer helpers for multi-tenant middleware stacks (e.g., ability to register per-route middlewares for different auth policies)?
- Document patterns for forcing middleware to run on every navigation (e.g., add a root loader) as highlighted in the React Router docs.
