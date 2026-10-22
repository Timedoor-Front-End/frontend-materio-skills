# Auth modes

Read when a page logs in/out, reads the current user, or needs a route
guard. Pick **one** mode per project in Data patterns Step A, then follow
only that section. Never mix modes.

## Detect

| Mode | Detect by |
|---|---|
| A. Bearer token | Docs say token/JWT auth, or the helpers attach `Authorization: Bearer` from a cookie/storage (Materio default: `accessToken` cookie) |
| B. Session cookie | Docs say session/cookie auth (Laravel web/Sanctum SPA, Django session, Rails); frontend and API share an origin or a parent domain; helpers send a CSRF header |
| C. Nuxt server-side | Nuxt with `server/api/` handling auth, or a `nuxt-auth-utils` / `@sidebase/nuxt-auth` module in `nuxt.config.ts` |
| D. No auth | Public-only app or page |

Docs and helpers disagree → stop and ask (see Data patterns Step A).

## A. Bearer token

- Login: `$api('<login endpoint>', { method: 'POST', body })` → store the
  token where the helpers read it (Materio: `useCookie('accessToken').value = token`).
- Helpers attach `Authorization: Bearer <token>` (Materio's `useApi`/`$api`
  already do).
- Logout: call the logout endpoint if the API has one, then clear the token
  cookie and any user store; navigate to the login page.
- Current user: from the login response or a "me" endpoint, kept in a store
  or cookie (Materio full-version uses a `userData` cookie).
- 401 from any request → clear the token, go to login.
- Caveat: a JS-readable token is exposed to XSS. Fine when the API is on
  another origin or used by other clients too; otherwise prefer mode B.

## B. Session cookie

The browser holds an httpOnly session cookie; JavaScript never sees a token.

**Helpers** (`useApi` + `$api`) must:
- send `Accept: application/json` so the backend answers 401/422 as JSON
  instead of redirecting;
- send the CSRF header on every request — framework names:

  | Backend | Cookie / source | Header |
  |---|---|---|
  | Laravel (web / Sanctum SPA) | `XSRF-TOKEN` cookie (URL-encoded → `decodeURIComponent`) | `X-XSRF-TOKEN` |
  | Django | `csrftoken` cookie | `X-CSRFToken` |
  | Rails | `<meta name="csrf-token">` | `X-CSRF-Token` |

- contain **no** Bearer / `accessToken` logic.
- Cross-origin (API on another subdomain only): also `credentials: 'include'`
  and a backend CORS config with credentials; Laravel Sanctum SPA also needs
  `GET /sanctum/csrf-cookie` before login. Same origin needs neither.

**Flow**
- Login: `$api('<login endpoint>', { method: 'POST', body })`, then fetch the
  current user (`GET <me endpoint>`) and navigate to the post-login page.
  Store only the user object (Pinia or local), never credentials.
- Logout: `$api('<logout endpoint>', { method: 'POST' })`, clear the user
  store, navigate to login.
- Guard: on protected routes, load the current user once (cache it); a 401
  means logged out → redirect to login. The backend still enforces auth on
  every endpoint — the guard is UX only.
- 401 from any request → clear the user store, go to login.
- 419 (Laravel CSRF mismatch / expired session) → reload or send to login.

## C. Nuxt server-side

- Auth lives in `server/` (session in an httpOnly cookie, e.g.
  `nuxt-auth-utils` `useUserSession()`), not in page code.
- Pages read the user via the module's composable; protect pages with route
  middleware in `<src>/middleware/`, not with per-page checks.
- During SSR, calls to an external API must forward the cookie
  (`useRequestHeaders(['cookie'])`); check whether `useApi` already does.

## D. No auth

Use `useApi` / `$api` as-is. Public pages must not call protected endpoints
"just to check" — they'd trigger 401 handling.

## All modes

- Login, register, logout, forgot/reset password and OTP send/verify are all
  `POST`. Credentials never travel in a query string.
- Login form errors: show the server's message (e.g. wrong credentials,
  throttled) in a `VAlert`; don't reveal which field was wrong unless the
  API does.
- Forgot-password: always show the same generic success message.
- Password reset only through a link or OTP sent to the user's **registered**
  email or phone. Never build a form that takes an email plus a new password
  directly. Show the link/OTP validity as a countdown
  (`patterns/authentication.md` → "Countdown").
- Logout clears **every** client-side trace of the session: the token and user
  cookies (Materio: `accessToken`, `userData`), CASL rules
  (`ability.update([])`), every Pinia store holding user data (reset its refs),
  and anything the app stored in `localStorage`/`sessionStorage` for this user.
  Do this even when the logout request fails, for example when offline or with
  an already-expired token. The 401 handler uses the same clear function.
- Public forms that write data (register, forgot password, contact, guest
  booking) send a reCAPTCHA token (`patterns/authentication.md` →
  "reCAPTCHA"). Login relies on throttling and gets reCAPTCHA only when the PRD
  asks for it.
- Never log or display tokens, cookies, or raw error bodies.
