# Frontend Developer Checklist

The team's release checklist (Notion "Developer Check List Web", 59 items). This
file says where each item is handled, so generated code already passes what code
can pass, and an audit can report the rest with evidence. `<src>` is the root
from SKILL.md Step 0.

Every item is one of three kinds:

- **Rule**: enforced while generating, by a SKILL.md convention or a reference
  rule (the "Handled by" column). Step 5 runs the "Checks" below on your files.
- **Setup**: done once per project as app-wide code or config. See "Project
  setup" (S1–S13).
- **Manual**: needs a person, a browser, a design file or the server. The skill
  never marks these checked; it says exactly what to look at.

The skill only ever fills **Developer Check**. **Senior Dev Check** belongs to
the reviewer and always stays `Not Checked`.

## Items

Names are verbatim from Notion, so audit rows can be matched back.

### Core Functionality & Reliability
| ID | Name | Kind | Handled by |
|---|---|---|---|
| C1 | All core features work as expected. | Rule + Manual | Step 5 (lint, typecheck, feature tests); a click-through of each flow is manual |
| C2 | Navigation between pages is smooth and logical. | Rule | SKILL.md "Sidenav" (`navActiveLink`), `PageHeader` back button |
| C3 | All buttons, links, and navigation elements work correctly. | Rule | SKILL.md "Links and buttons" |
| C4 | All form inputs have proper validation (including max length/value). | Rule | SKILL.md "Form validation", "Input limits"; BE form requests |
| C5 | If a form has validation errors, it should not submit. | Rule | `validate()` guard at the top of every submit handler |
| C6 | The app handles being offline gracefully (e.g., shows a message). | Setup | S1 (offline banner) + S3 (network error message) |
| C7 | Users can log out from any authenticated page. | Setup | S6 (logout in the default layout's user menu) |

### UI/UX & Design
| ID | Name | Kind | Handled by |
|---|---|---|---|
| U1 | The layout and design match the Figma/XD mockups. | Rule + Manual | SKILL.md "Design file"; visual comparison is manual |
| U2 | The layout is responsive and looks good on desktop, tablet, and mobile. | Rule + Manual | SKILL.md "Responsive"; check at 360 / 768 / 1280 px |
| U3 | No text or elements are cut off or hidden on any screen size. | Rule + Manual | same as U2 |
| U4 | Styles (colors, fonts, spacing) are consistent across all pages. | Rule | SKILL.md: no manual colors/spacing, theme defaults only |
| U5 | Hover effects, transitions, and animations work as intended. | Rule + Manual | Vuetify's built-in states only, no custom transitions |
| U6 | All modals, dropdowns, and tooltips can be closed properly. | Rule | SKILL.md "Dialogs and menus close" |
| U7 | A clear "empty state" is shown when there is no data. | Rule | `patterns/tables.md` "Empty state", `data-patterns.md` "Loading and empty states" |
| U8 | A loading state is always shown when fetching data or submitting a form. | Rule | tables `:loading`, submit `:loading`, `data-patterns.md` "Loading and empty states" |
| U9 | Show a confirmation popup before any destructive action (like deleting data). | Rule | SKILL.md "Destructive actions" (`ConfirmActionDialog`) |
| U10 | Use type="password" for all password input fields. | Rule | SKILL.md "Password fields" |
| U11 | Date of birth inputs should default to 10 years ago and start with year selection. | Rule (+ Setup S8) | `<DateTimeField birth-date>` |
| U12 | Add a tooltip to any button that only has an icon and no text. | Rule | `ActionIconBtn` (tooltip required) |
| U13 | Show a countdown timer for OTP or password reset links. | Rule | `patterns/authentication.md` "Countdown" |
| U14 | There should be no unwanted horizontal scrolling. | Rule + Manual | same as U2 |

### Accessibility & SEO
| ID | Name | Kind | Handled by |
|---|---|---|---|
| A1 | All clickable elements are easy to use on mobile devices. | Rule + Manual | SKILL.md "Responsive" (kit control sizes) |
| A2 | Heading structure follows a logical order (h1 → h2 → h3, etc.). | Rule + Setup S7 | SKILL.md "Headings" |
| A3 | Each page has the correct meta tags (title, description). | Rule + Setup S2 | SKILL.md "Page title" |
| A4 | All images have relevant alt text. | Rule | SKILL.md "Images" |

### Security: Authentication & Authorization
| ID | Name | Kind | Handled by |
|---|---|---|---|
| SA1 | All pages require authentication, except for public ones. | Setup S6 + Rule | route guard; `public: true` only on auth/public pages; BE `auth:*` on every non-public route |
| SA2 | Login and other authentication processes use POST requests, not GET. | Rule | `auth-modes.md` "All modes" |
| SA3 | API requests are secured with an Authorization Token. | Rule | `auth-modes.md`: Bearer token, or a Sanctum session cookie + CSRF (both count) |
| SA4 | Authentication error messages are generic (e.g., "Invalid credentials") and don't reveal which part was wrong. | Rule | `auth-modes.md` "All modes", `backend-laravel.md` auth endpoints |
| SA5 | Password resets require verification via email or OTP sent to the user's registered contact. | Rule | `auth-modes.md` "All modes", `backend-laravel.md` auth endpoints |
| SA6 | All user session data is cleared from the client-side on logout. | Rule | `auth-modes.md` "All modes" |
| SA7 | Access to pages, assets, and features is restricted based on user roles (authorization). | Rule | `action`/`subject` on pages and nav; BE policies/middleware; private files behind an authorized endpoint |

### Security: Data & Input Handling
| ID | Name | Kind | Handled by |
|---|---|---|---|
| SD1 | Sanitize any user input that accepts HTML or Markdown to prevent XSS attacks. | Rule | `data-patterns.md` "Rendering HTML", `backend-laravel.md` "Rich text" |
| SD2 | Disable autocomplete on forms with sensitive data. | Rule | SKILL.md "Password fields", "Sensitive fields" |
| SD3 | Limit the file types and sizes that can be uploaded. | Rule | `data-patterns.md` "Input limits" (file row) |
| SD4 | Validate uploaded files by their content/headers, not just their file extensions. | Rule | `backend-laravel.md` "Uploads" |
| SD5 | Use Google reCAPTCHA on public forms (e.g., contact forms). | Rule + Setup S9 | `patterns/authentication.md` "reCAPTCHA" |
| SD6 | Validate the length of user inputs to prevent buffer overflow issues. | Rule | "Input limits" (FE `maxlength`, BE `max:`, column size) |

### Security: Infrastructure & Communication
| ID | Name | Kind | Handled by |
|---|---|---|---|
| SI1 | All communication and API endpoints must use HTTPS. | Setup S4 + Manual | certificate on the server is manual |
| SI2 | Automatically redirect all HTTP traffic to HTTPS. | Manual | web server / hosting (snippet under "Server snippets") |
| SI3 | Never send sensitive data in URL parameters. | Rule | `data-patterns.md` "URLs" |
| SI4 | User-facing error messages are friendly and do not reveal technical details. | Setup S3 + Manual | `APP_DEBUG=false` on production is manual |
| SI5 | No sensitive data is written to or stored in logs. | Rule + Setup S5 | `backend-laravel.md` "Logs"; console stripped from builds |
| SI6 | If sending custom headers, ensure they only contain ASCII characters. | Rule | `data-patterns.md` "Helpers Materio ships" |
| SI7 | Set up Basic Auth to protect the development and staging environments from public access. | Manual | web server (snippet under "Server snippets") |
| SI8 | Use strong, randomly generated credentials for Basic Auth. | Manual | `openssl rand -base64 24` |

### Code & Dependency Management
| ID | Name | Kind | Handled by |
|---|---|---|---|
| CD1 | No credentials, tokens, or API keys are hardcoded. Use environment variables (.env) instead. | Rule | `backend-laravel.md` "Env keys", `data-patterns.md` "URLs" (`VITE_*` are public) |
| CD2 | No console.log or other debug code is in the production build. | Rule + Setup S5 | SKILL.md "No debug code" |
| CD3 | Validation and authentication logic are grouped into centralized modules. | Rule | `@core/utils/validators`, the fetch helpers own auth, BE form requests + policies |
| CD4 | Regularly update third-party libraries, plugins, and dependencies. | Manual | the audit reports outdated packages; upgrades are the developer's call |
| CD5 | Ensure all third-party libraries are secure and not deprecated. | Rule + Manual | no new package without confirmation; the audit runs `pnpm audit` / `composer audit` |
| CD6 | Explicitly initialize all variables with default values. | Rule | SKILL.md "Empty values", "Reactivity" |
| CD7 | Implement versioning | Setup S10 | app version in the footer, API under `/api/v1` |
| CD8 | The application is integrated with SonarQube for static code analysis. | Setup S11 + Manual | Sonar project and CI token are manual |

### Assets & External Content
| ID | Name | Kind | Handled by |
|---|---|---|---|
| AE1 | A custom favicon is used. | Setup S12 | |
| AE2 | Use a standard placeholder image for dynamic content that is missing an image (https://placehold.co/). | Rule | SKILL.md "Images" |
| AE3 | All external links work and open in a new tab. | Rule | SKILL.md "External links"; rich text forced by `SafeHtml` |
| AE4 | Ensure all external URLs point to safe and trusted sources. | Rule | SKILL.md "External links"; BE `url:https` on user-entered URLs |

### Monitor & Operations
| ID | Name | Kind | Handled by |
|---|---|---|---|
| MO1 | The application is integrated with Sentry for error tracking and performance monitoring. | Setup S13 | |

## Project setup (once per session, after the starter kit check)

App-wide pieces the checklist needs that a page can't add by itself. Missing
pieces **don't block** page generation, with one exception: S8 blocks a page that
has a date-of-birth field. List the open items to the developer and offer to apply
them. Each one is an app-wide change, so it needs confirmation first, like any
escalation. Record the result under the `docs/PLAN.md` header:
`> Checklist setup: OK (<date>)` or `> Checklist setup: open S5, S13 (<date>)`.

Run from the project root. The script prints only what's open:

```bash
SRC=resources/ts   # resources/ts (Laravel) | src (Vue) | app (Nuxt 4) | . (Nuxt 3)
APP=$( [ -f nuxt.config.ts ] && { [ -f app/app.vue ] && echo app/app.vue || echo app.vue; } || echo "$SRC/App.vue" )
CFG=$( [ -f nuxt.config.ts ] && echo nuxt.config.ts || echo vite.config.ts )
grep -qs "useOnline" "$APP"                                   || echo "open S1: offline banner in $APP"
grep -rqs "document.title\|titleTemplate" "$SRC/plugins" "$APP" || echo "open S2: page title from meta.title"
grep -qs ">= 500" "$SRC/utils/api.ts"                          || echo "open S3: apiErrorMessage hides 5xx/network details"
[ -f artisan ] && { grep -rqs "forceScheme" app/Providers      || echo "open S4: URL::forceScheme('https') in production"; }
grep -qs "drop:" "$CFG"                                        || echo "open S5: build strips console/debugger ($CFG)"
grep -rqsl "beforeEach\|defineNuxtRouteMiddleware" "$SRC/plugins" "$SRC/middleware" || echo "open S6: route auth guard"
grep -rqs "logout" "$SRC/layouts/components"                   || echo "open S6: logout in the user menu"
grep -qs "<h1" "$SRC/components/global/page/PageHeader.vue"    || echo "open S7: PageHeader title as <h1>"
grep -qs "birthDate" "$SRC/components/global/form/DateTimeField.vue" || echo "open S8: DateTimeField birth-date (only if a DOB field exists)"
grep -qs "__APP_VERSION__\|appVersion" "$CFG"                  || echo "open S10: app version"
[ -f sonar-project.properties ]                                || echo "open S11: sonar-project.properties"
[ "$(md5sum public/favicon.ico 2>/dev/null | cut -c1-32)" = 057b56a6c39c7ed81047d029daebc29f ] && echo "open S12: favicon is still Materio's"
grep -qs "sentry" package.json                                 || echo "open S13: Sentry (FE)"
[ -f artisan ] && { grep -qs "sentry/sentry-laravel" composer.json || echo "open S13: Sentry (BE)"; }
```

S9 (reCAPTCHA) applies only when the project has a public form that writes data,
such as register, forgot password, contact or guest booking. Check it when the
plan has one: `[ -f "$SRC/utils/recaptcha.ts" ]`.

Snippets are written for Vue/Laravel; Nuxt differences follow each one. Kit-level
text stays English. Translate page copy only.

### S1 — offline banner (C6)
`App.vue` (Nuxt: `app.vue`), inside `VApp`. `useOnline` comes from `@vueuse/core`
and is auto-imported:
```vue
<script setup lang="ts">
const isOnline = useOnline()
</script>

<VSnackbar :model-value="!isOnline" :timeout="-1" color="error" location="bottom">
  You're offline. Changes can't be saved until the connection is back.
</VSnackbar>
```

### S2 — page title and description (A3)
Pages set `meta.title` (SKILL.md "Page title"). Add `title?: string` to
`RouteMeta` in `env.d.ts`, then in `plugins/1.router/guards.ts`:
```ts
router.afterEach(to => {
  document.title = to.meta.title ? `${to.meta.title} | ${themeConfig.app.title}` : themeConfig.app.title
})
```
Put `<meta name="description" content="<one line from the PRD>">` in `index.html`
(Laravel: `resources/views/application.blade.php`) and replace the Materio `<title>`.
Admin apps keep `<meta name="robots" content="noindex, nofollow">`.
Nuxt: pages use `definePageMeta({ title })`. In `app.vue`, add
`const route = useRoute(); useHead({ title: () => route.meta.title as string | undefined, titleTemplate: t => t ? `${t} | <App>` : '<App>' })`,
and put the description in `app.head.meta` of `nuxt.config.ts`.

### S3 — friendly API errors (C6, SI4)
`utils/api.ts`. The server's own text is shown only for 4xx. For a 5xx in debug
mode that text is a stack trace or SQL.
```ts
import type { FetchError } from 'ofetch'

export const apiErrorMessage = (error: unknown) => {
  const e = error as FetchError<{ message?: string, errors?: Record<string, string[]> }>
  if (!e?.response) // network failure: offline, DNS, CORS, timeout
    return navigator.onLine ? 'Could not reach the server. Please try again.' : 'You are offline. Check your connection and try again.'
  if (e.response.status >= 500)
    return 'Something went wrong on our side. Please try again.'
  const errors = e.data?.errors

  return (errors && Object.values(errors)[0]?.[0]) ?? e.data?.message ?? 'Something went wrong. Please try again.'
}
```
Nuxt (SSR): `navigator` only exists on the client. Use `import.meta.client && !navigator.onLine`.

### S4 — HTTPS (SI1)
Laravel, in `AppServiceProvider::boot()`: `if ($this->app->isProduction()) { URL::forceScheme('https'); }`.
If TLS ends at a proxy or load balancer, also trust it in `bootstrap/app.php`
(`$middleware->trustProxies(at: '<proxy IP or *>')`) so `$request->secure()` is
right. For all variants, the production API base URL (`VITE_API_BASE_URL` /
`NUXT_PUBLIC_*`) starts with `https://`. Never hardcode `http://` URLs.

### S5 — no console in production builds (CD2, SI5)
`vite.config.ts`: turn the config into a function so dev keeps its console:
```ts
export default defineConfig(({ command }) => ({
  // ...existing config unchanged...
  esbuild: { drop: command === 'build' ? ['console', 'debugger'] : [] },
}))
```
Nuxt: in `nuxt.config.ts` add `vite: { esbuild: { drop: process.env.NODE_ENV === 'production' ? ['console', 'debugger'] : [] } }`.
This applies to both client and server bundles.

### S6 — auth guard and logout everywhere (SA1, C7)
This is a report only, because building it is the auth module's job (`auth-modes.md`).
It counts as done when two things hold. First, a global guard (Vue/Laravel:
`router.beforeEach` in `plugins/1.router/`; Nuxt: `middleware/*.global.ts`)
sends every route without `meta.public` to login. Second, the default layout's
`UserProfile` menu calls the real logout flow (`auth-modes.md`), not just
`router.push('/login')`.

### S7 — PageHeader heading (A2)
In `components/global/page/PageHeader.vue`, change the title tag
`<h4 class="text-h4">` to `<h1 class="text-h4">`. The class keeps the look, and the
tag gives the page its single `h1`.

### S8 — DateTimeField `birth-date` (U11)
`components/global/form/DateTimeField.vue`: add the prop and three picker bindings.
The field stays empty. Only the calendar *opens* at 10 years back, starting from
the year list, because prefilling a fake birth date would be wrong data.
```ts
const props = withDefaults(defineProps<{
  modelValue: string | null | undefined
  mode?: 'date' | 'time'
  maxDate?: Date | string
  birthDate?: boolean
}>(), { mode: 'date', maxDate: undefined, birthDate: false })

const tenYearsAgo = computed(() => {
  const d = new Date()
  d.setFullYear(d.getFullYear() - 10)

  return d
})
```
```vue
:max-date="props.maxDate ?? (props.birthDate ? new Date() : undefined)"
:start-date="props.birthDate ? tenYearsAgo : undefined"
:flow="props.birthDate ? { steps: ['year', 'month', 'calendar'] } : undefined"
```
Add the prop to the header comment too. After applying, open the picker once in
the browser to confirm it starts on the year list (vue-datepicker v14 `flow`).

### S9 — reCAPTCHA v3 (SD5)
Only for public forms that write data. Env: `VITE_RECAPTCHA_SITE_KEY` (Nuxt:
`NUXT_PUBLIC_RECAPTCHA_SITE_KEY`) and `RECAPTCHA_SECRET_KEY` (backend). Add both to
the plan's Environment table. v3 is invisible, so the form doesn't change. No
package is needed: `utils/recaptcha.ts` (auto-imported) loads Google's script on
first use:
```ts
declare global {
  interface Window { grecaptcha: { ready: (cb: () => void) => void, execute: (key: string, o: { action: string }) => Promise<string> } }
}

const siteKey = import.meta.env.VITE_RECAPTCHA_SITE_KEY as string | undefined
const loader = ref<Promise<void> | null>(null)

const load = () => loader.value ??= new Promise((resolve, reject) => {
  const script = document.createElement('script')
  script.src = `https://www.google.com/recaptcha/api.js?render=${siteKey}`
  script.onload = () => window.grecaptcha.ready(resolve)
  script.onerror = () => {
    loader.value = null // allow a retry on the next submit
    reject(new Error('reCAPTCHA failed to load'))
  }
  document.head.append(script)
})

/** Token for one submit, sent as `recaptcha_token`. Empty when no site key is set (local dev). */
export const recaptchaToken = async (action: string) => {
  if (!siteKey)
    return ''
  await load()

  return window.grecaptcha.execute(siteKey, { action })
}
```
Laravel: `config/services.php` → `'recaptcha' => ['secret' => env('RECAPTCHA_SECRET_KEY')]`, and
`app/Rules/Recaptcha.php`:
```php
class Recaptcha implements ValidationRule
{
    public bool $implicit = true; // runs even when the token is missing

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $secret = config('services.recaptcha.secret');
        if (! $secret && ! app()->isProduction()) {
            return; // local/dev without keys; production must set them
        }

        $response = Http::asForm()->timeout(5)->post('https://www.google.com/recaptcha/api/siteverify', [
            'secret' => $secret,
            'response' => $value,
            'remoteip' => request()->ip(),
        ]);

        if (! $response->json('success') || $response->json('score', 0) < 0.5) {
            $fail('Verification failed. Please try again.');
        }
    }
}
```
In the form request: `'recaptcha_token' => [new Recaptcha]`. Add a feature test for
the failure path with `Http::fake()`.

### S10 — versioning (CD7)
- App version: `vite.config.ts` → `define: { __APP_VERSION__: JSON.stringify(process.env.npm_package_version) }`.
  Add `declare const __APP_VERSION__: string` in `env.d.ts`. The footer
  (`layouts/components/Footer.vue`) shows it via `const version = __APP_VERSION__`
  in its script, because templates can't read globals. Nuxt:
  `runtimeConfig: { public: { appVersion: process.env.npm_package_version } }`.
- Release flow: bump `version` in `package.json` (semver) for each release and tag
  it (`git tag v1.2.0`).
- API: routes under `/api/v1` (`backend-laravel.md` Step A). A breaking change
  goes to `/api/v2`, never into v1.

### S11 — SonarQube (CD8)
`sonar-project.properties` at the root. The template's own code is excluded:
```properties
sonar.projectKey=<project-key>
sonar.sources=resources/ts,app
sonar.tests=tests
sonar.exclusions=**/@core/**,**/@layouts/**,**/node_modules/**,vendor/**,public/build/**,**/*.d.ts
```
Vue: `sonar.sources=src`; Nuxt: the source folders. `SONAR_HOST_URL` and
`SONAR_TOKEN` are CI secrets, never in `.env` or the repo. Creating the Sonar
project and the CI step is manual.

### S12 — favicon (AE1)
`public/favicon.ico` with md5 `057b56a6c39c7ed81047d029daebc29f` is Materio's.
Ask the developer for the logo and never make one up. Also replace the Materio
logo and name in `themeConfig.ts` if they're still there.

### S13 — Sentry (MO1)
These are new packages, so confirm first. They also add env keys: `VITE_SENTRY_DSN`
(Nuxt: `NUXT_PUBLIC_SENTRY_DSN`) and `SENTRY_LARAVEL_DSN`. Without a DSN, Sentry
stays off (`↪ alternative: disabled until DSN`).
- Vue/Laravel FE: `@sentry/vue`, in `plugins/sentry.ts`. Plugins load in file-name
  order, after `1.router`:
  ```ts
  import * as Sentry from '@sentry/vue'
  import type { App } from 'vue'
  import { router } from '@/plugins/1.router'

  export default function (app: App) {
    const dsn = import.meta.env.VITE_SENTRY_DSN
    if (!dsn)
      return
    Sentry.init({ app, dsn, environment: import.meta.env.MODE, integrations: [Sentry.browserTracingIntegration({ router })], tracesSampleRate: 0.1 })
  }
  ```
- Nuxt: the `@sentry/nuxt` module, following its setup docs, with the DSN from
  runtime config.
- Laravel: `sentry/sentry-laravel`. In `bootstrap/app.php`, add
  `->withExceptions(fn (Exceptions $e) => Integration::handles($e))` and set
  `SENTRY_TRACES_SAMPLE_RATE=0.1`.
- Keep `sendDefaultPii` off (the default): no emails, IPs or request bodies in Sentry.

## Server snippets (Manual items: hand them to whoever runs the server)

```nginx
# SI2: HTTP → HTTPS
server { listen 80; server_name example.com; return 301 https://$host$request_uri; }

# SI7: Basic Auth on dev/staging only (inside the HTTPS server block)
auth_basic "Restricted";
auth_basic_user_file /etc/nginx/.htpasswd;   # htpasswd -c /etc/nginx/.htpasswd <user>
```
SI8: generate the password with `openssl rand -base64 24`, store it in the team's
password manager and never in the repo. Production: `APP_DEBUG=false`,
`APP_ENV=production` (SI4).

## Checks (Rule items as greps)

Step 5 runs these on **your files**. The audit runs them on the whole project
(`<src>/pages`, `<src>/components`, `<src>/layouts`, plus `app/` and `routes/`
for BE), excluding `@core`, `@layouts` and `node_modules`. A match means "look
at it", because multi-line tags make some of these approximate. Confirm each
match before reporting it.

| ID | Look for | Violation when |
|---|---|---|
| U12 | `IconBtn\b`, `<VBtn[^>]*\bicon[ =>]` | any match in pages/components (use `ActionIconBtn`) |
| C4, SD6 | `<VTextField\|<VTextarea` | without `maxlength`; `NumberField` without a deliberate `:max` |
| C4 | `type="number"\|type="date"\|type="time"\|AppDateTimePicker` | any match |
| C5 | `@submit.prevent` handlers | handler doesn't start with the `validate()` guard |
| U9 | `method: 'DELETE'` and other destructive calls (cancel, deactivate) | not behind `ConfirmActionDialog` |
| U10, SD2 | `password` fields | no `'password'` type, or a set-password form without `autocomplete="new-password"` |
| U7 | `VDataTableServer\|VDataTable\b`, `v-for` over fetched lists | no `no-data-text` / no empty block |
| U8 | `useApi(`, `$api(` in handlers | no `:loading` / skeleton for it |
| U6 | `persistent` | on any `VDialog` |
| U11 | birth / dob / lahir fields | `DateTimeField` without `birth-date` |
| A2 | `<h[1-6]` | levels skip, or more than one `h1` per page |
| A3 | `definePage(\|definePageMeta(` | no `title` in its meta (pages with a layout) |
| A4 | `<VImg\|<img` | no `alt` |
| AE2 | `:src=` on dynamic images | no `?? 'https://placehold.co/...'` fallback (people use initials) |
| AE3, AE4 | `href="http`, `target="_blank"` | missing `target="_blank"` or `rel="noopener noreferrer"`; `http://` URL |
| C3 | `href="#"`, `:to=`, `RouterLink` | `#` links; a route name/path that doesn't exist under `<src>/pages/` |
| SD1 | `v-html` | bound to a field the backend doesn't sanitize |
| SI3 | `?token=\|?password=\|query: {` | credentials, tokens or personal identifiers in a URL |
| CD2 | `console\.\|debugger` | any match in app code |
| CD1 | `(sk_live\|AKIA[0-9A-Z]{16}\|-----BEGIN\|secret\s*[:=]\s*['"][^'"]{8,})`, `env(` outside `config/` | any match |
| SD4 | `extensions:` in `app/Http/Requests` | used without `mimes:` / `File::types()` |
| SI5 | `Log::[a-z]+\(.*(request\(\)->all\|password\|token)` | any match |
| SA1, SA7 | `php artisan route:list --except-vendor -v` | a non-public route without an `auth:*` middleware; a write route without a role/policy check |
| CD4, CD5 | `pnpm outdated`, `pnpm audit --prod`, `composer outdated --direct`, `composer audit` | report the results; never upgrade without the developer |

## Audit mode ("run the checklist", "cek checklist", "audit checklist")

Usually run before a release or a senior review.

1. Run the Project setup script and every row of "Checks" on the whole project.
   Read the code where a grep can't decide (C2, C7, SA6, U8).
2. Write `docs/CHECKLIST.md` and update it in place, never as a second file:
   ```markdown
   # Developer Checklist
   > Audited: <date> · Stack: Laravel · Commit: <short sha, or "not a git repo">

   | ID | Name | Type | Developer Check | Senior Dev Check | Evidence / to do |
   |---|---|---|---|---|---|
   | C1 | All core features work as expected. | Core Functionality & Reliability | Manual | Not Checked | 32 feature tests pass; click through the booking flow |
   | U12 | Add a tooltip to any button that only has an icon and no text. | UI/UX & Design | Checked | Not Checked | no `IconBtn`/`<VBtn icon>` outside @core |
   ```
   All 59 items, in the order above, with `Name` and `Type` verbatim. Developer Check is one of:
   - `Checked`: verified. Evidence is a `file:line`, a command and its result, or the setup item that's in place.
   - `Not Checked`: fails or is missing. Say what and where.
   - `Manual`: needs a person. Say exactly what to verify (pages, screen sizes, server).
   - `N/A`: the feature doesn't exist (no upload, no DOB, no public form). Say why.

   Senior Dev Check is always `Not Checked`.
3. Reply with the count per status and the `Not Checked` items, each with its
   proposed fix. Fix nothing in the same turn. Fixes become plan items, or a
   one-off change once the developer agrees.
