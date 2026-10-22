# Data patterns

Read this before wiring any page to a real API. It is backend-agnostic: the
backend may be Laravel, Express/Nest, Django, Rails, Go, or anything that
speaks HTTP + JSON. `<src>` is the root from SKILL.md Step 0. If the code
disagrees with this file, trust the code.

## Step A — learn the project's API conventions (once per session)

Before the first fetch, fill in this table from the sources below. Every
later page reuses it. Don't guess a row; if no source answers it, ask.

| Convention | Example values |
|---|---|
| Base URL | `/api` (same origin), `https://api.example.com`, env var |
| Auth mode | see `references/auth-modes.md` (bearer token, session cookie, Nuxt SSR, none) |
| Validation error shape | Laravel `422 { message, errors: { field: [msg] } }`, Django REST `400 { field: [msg] }`, Nest `400 { message: string[] }` |
| Success shape | bare object, `{ data }`, `{ message, data }` |
| Pagination shape + params | Laravel paginator (`data`, `current_page`, `total`, `?page=`), `{ items, total }`, cursor |
| Key casing | `snake_case` / `camelCase` |

Sources, in order:
1. **Project docs** — `CLAUDE.md`, `docs/API.md`, `docs/ARCHITECTURE.md`, or
   whatever the repo uses as its API contract. In a Laravel project with
   `[BE]` items, `docs/API.md` is written by those items
   (`backend-laravel.md`) and is the contract — but if it disagrees with the
   backend code, the code wins: fix the doc and tell the developer.
2. **Fetch helpers** — `<src>/composables/useApi.ts`, `<src>/utils/api.ts`
   (base URL, headers, token/cookie handling).
3. **Backend code in this repo** — Laravel `routes/*.php` + FormRequests,
   Nuxt `server/api/`, etc.
4. **The prompt / PRD.**

If docs and helpers disagree (e.g. docs say session cookie, helper still
attaches a Bearer token), **stop and tell the developer**: the helper is
probably mid-migration. Don't wire pages against a helper that is about to
change.

## Helpers Materio ships

| Helper | File | Use for |
|---|---|---|
| `useApi` | `<src>/composables/useApi.ts` | Reactive GET on page load (lists, detail). VueUse `createFetch`. |
| `createUrl` | `<src>/@core/composable/createUrl.ts` | Reactive URL + query string for `useApi`. |
| `$api` | `<src>/utils/api.ts` | Imperative calls (POST, PUT, PATCH, DELETE, one-off GET). `ofetch` instance. |
| `paginationMeta` | `<src>/utils/paginationMeta.ts` | "1 - 10 of 57" footer text. |

All four are auto-imported; don't add import lines. Don't add axios or
another fetch wrapper. Auth headers/cookies belong **inside these two
helpers only**, never in a page. Custom header values must be ASCII:
`encodeURIComponent` anything else, because fetch throws on characters outside
ISO-8859-1, such as a name with accents in a header.

On Nuxt the same two names exist with different internals — `useApi` wraps
`useFetch` (`data` / `pending` / `error` / `refresh`, runs on server and
client) and `$api` wraps `$fetch`. Use `useApi` for page-load data and
`$api` inside handlers; never call `useApi` from a click handler. Read the
actual `composables/useApi.ts` before relying on its return shape.

**Not shipped in the starter-kit:** a toast/snackbar store, an auth store,
a router auth guard, a registered CASL `abilitiesPlugin`. Check the repo
first — the project may have added them. If a page needs one that doesn't
exist, flag it as an escalation (see SKILL.md).

## Hard rules

- Never invent an endpoint, a response shape, query parameter names, or a
  permission name. Take them from the Step A sources. If none say, leave a
  `// TODO: endpoint` and say so in your response. (Laravel, scope FE + BE:
  a missing endpoint means a `[BE]` item is missing — add it to
  `docs/PLAN.md` before this page instead of leaving a TODO.)
- Use the project's names for pagination, response, and error fields
  exactly (Step A). Don't assume `page`, `limit`, `meta`, or `data`.
- Use the project's key casing in request bodies. Don't convert casing in a
  page; if conversion is needed, it belongs in the helpers.
- If you find two conflicting patterns in the repo, ask which one is correct.

## Lists and page-load data

- **List pages:** `useServerTable` + `VDataTableServer` — shape, loading and
  debounce rules in `patterns/tables.md`. Don't hand-roll paging state with
  `useApi` + `createUrl`. Only small detail lists the API returns whole stay
  client-side `VDataTable`.
- **Detail / other page-load data:** `useApi` (reactive; refetches when its URL
  changes, so reactive params need no watcher). Field names must match the
  real API response (Step A).
- If several independent fetches run on mount, run them in parallel with
  `Promise.all`.

### Loading and empty states

Every fetch shows that it is loading, and every list shows something when it
is empty. Tables do both through `VDataTableServer` (`patterns/tables.md`).
Everything else:

- **First load** of a detail page or block: `<VSkeletonLoader type="card" />`
  (or `"table"`, `"list-item-two-line"`, matching the shape) in place of the
  content while `isFetching` is true and there's no data yet. **Refetch**: keep
  the content and set `:loading="isFetching"` on its `VCard`.
- **Load error**: `VAlert type="error"` with `apiErrorMessage(error.value)` and a
  Retry button (`@click="execute()"`), never a blank card.
- **Empty list** that isn't a table (cards grid, timeline, chips): a
  `v-if="!items.length"` block with an icon, one line saying what's missing
  ("No branches yet"), and the page's primary action if it has one. When search
  or filters are active, say so instead ("No branches match your search").
- **Submit / action buttons**: `:loading` while their request runs (see "Forms
  and mutations").

## URLs

- Never put credentials, tokens, OTPs or personal identifiers (national ID,
  card/bank number, medical data) in a URL the app builds: route params, query
  strings or links. URLs end up in server logs, browser history and the
  `Referer` header. Route params use the record `id`. The only exception is an
  emailed one-time reset/verify link, which is single-use and expires.
- `VITE_*` / `NUXT_PUBLIC_*` values ship to the browser. Use them only for
  public values (API base URL, reCAPTCHA site key, Sentry DSN), never for
  secrets.

## Forms and mutations

```ts
import type { FetchError } from 'ofetch' // $api (ofetch / Nuxt $fetch) throws this

const isSubmitting = ref(false)
const fieldErrors = ref<Record<string, string[]>>({})
const formError = ref('')

const onSubmit = async () => {
  const { valid } = await refForm.value!.validate()
  if (!valid)
    return

  isSubmitting.value = true
  fieldErrors.value = {}
  formError.value = ''
  try {
    await $api('/items', { method: 'POST', body: form.value })
    isDialogVisible.value = false
    refresh() // useServerTable's / useApi's refresh; from a dialog component: emit('saved', ...)
  }
  catch (e) {
    // adapt to the validation error shape from Step A — Laravel shown here
    const error = e as FetchError<{ message?: string, errors?: Record<string, string[]> }>
    if (error.status === 422)
      fieldErrors.value = error.data?.errors ?? {}
    else
      formError.value = apiErrorMessage(e) // generic text for 5xx/network (checklist S3)
  }
  finally {
    isSubmitting.value = false
  }
}
```

```vue
<VTextField v-model="form.name" :error-messages="fieldErrors.name" />
<VAlert v-if="formError" type="error" variant="tonal">{{ formError }}</VAlert>
```

- Server-side validation is the source of truth (business rules live
  there). Client rules in `:rules` are only for instant feedback; always
  show the server's field errors next to the matching field.

### Client-side validation (project standard)

- **When:** on blur and on submit only. The global default
  `VForm: { validateOn: 'blur lazy' }` (`plugins/vuetify/defaults.ts`) does
  this: `lazy` = no errors on first render even though fields are empty,
  `blur` = a field is checked when it loses focus, and `validate()` in the
  submit handler checks everything. Don't set `validate-on` per form.
- **Required fields:** label ends with ` *` (`label="Venue *"`) and the field
  has `:rules="[requiredValidator]"`. Optional fields get neither.
- **Rules:** reuse `@core/utils/validators` (`requiredValidator`,
  `emailValidator`, `confirmedValidator(v, other)`, …); cross-field rules as
  inline arrows: `(v: string) => !form.start_time || v > form.start_time || 'End time must be after the start time.'`.
- **Submit:** `ref="refForm"` on `VForm`, and the handler begins with
  ```ts
  const { valid } = await refForm.value!.validate()
  if (!valid)
    return // no request while client rules fail
  ```
- **Dialogs:** call `refForm.value?.resetValidation()` when the dialog opens,
  so a reopened dialog starts without old errors.
- `DateTimeField` handles its own validation timing (see `component-catalog.md`). Cross-field rules (end after start) should read the form values, not the rule argument.
- `NumberField` takes `:rules` like any field (rules see the formatted text, so
  `requiredValidator` works; `0,00` counts as filled). For a range use its
  `:min`/`:max` props, not `betweenValidator` — that one would get the formatted text.
- Clear a field's server error when the user edits it, if the form is long.
- Put the loading flag on the submit button: `<VBtn :loading="isSubmitting">`.
- Always reset loading in `finally`.
- Don't show raw response bodies or tokens to the user. Every error message
  shown to the user comes from `apiErrorMessage()`, which turns 5xx and network
  failures into friendly text (checklist S3).
- Delete: only add a delete action if the PRD asks for one. Confirm it with
  a dialog first (`ConfirmActionDialog`, showing the server error in its slot), then refetch.

### Input limits

Every input has an upper bound, written the same in three places: the backend
rule, the column, and the field. Take it from the PRD / business rule first,
else the tables below, else the fallback. A fallback is a guess — say so in the
response: `⚠️ "<field>" uses the default limit 255 — check it fits.`

**FE:** text fields and textareas get the native `maxlength="100"` attribute (the
browser stops typing and trims pastes — no rule needed); textareas from 500
also get `counter`. Numbers: `<NumberField :min="1" :max="9999">` (`min` defaults
to 0; `max` defaults to the money cap with `currency`, else to the largest
precise JS integer).

| Text field | Max characters |
|---|---|
| Person / entity name | 100 |
| Username | 30 |
| Email | 254 (RFC 5321) |
| Password | min 8, max 72 (bcrypt reads only the first 72 bytes) |
| Phone | 20 (E.164: 15 digits, plus `+`, spaces, dashes) |
| Code / SKU / reference number | 50 |
| Title / short label | 150 |
| URL | 2048 |
| Address | 255 |
| Short description / note | 500 |
| Bio | 1000 |
| Long text (notes, reports) | 5000 |
| Rich text (`TiptapEditor`) | 200000, counted on the HTML — see "Rendering HTML" |
| Fixed-length identifier (national ID, tax ID, postal code) | exact length from the PRD or the country's standard; FE `maxlength` + `lengthValidator(v, n)` |

`lengthValidator` checks an **exact** length (its message says "at least", but
it isn't a minimum) — use it only for fixed-length identifiers, never as a max.

| Number field | Range (model units) |
|---|---|
| Money | 0 – 9_000_000_000_000_000 minor units (90,000,000,000,000.00 with 2 decimals) — the `NumberField currency` default; the largest round amount a JS number holds exactly |
| Quantity | 1 – 9999 |
| Count (stock, quota, capacity) | 0 – 999999 |
| Percentage | 0 – 100 |
| Age in years | 0 – 150 |
| Measurement (weight, height, temperature, …) | from the PRD (ask if it doesn't say), with `:decimals` |

| Other | Limit |
|---|---|
| File upload | image 2 MB, document 5 MB, plus a type whitelist: `accept` on FE plus a size check before uploading (instant feedback); `mimes:` / `File::types()` on BE, which checks the file content (`backend-laravel.md` → "Uploads") |
| Array (multi-select, add/remove rows) | 50 items, each item limited by its own type |

**Fallback** (no row fits — flag it):

| Field | Fallback |
|---|---|
| Single-line text | 255 (Laravel's `string()` column default) |
| Textarea | 1000 |
| Integer | 0 – 999999999 (fits a 32-bit `INT`) |
| Decimal | what the column holds: `decimal(10,2)` → 99999999.99 |

Money never falls back — it always uses the money row.

## Rendering HTML

- `v-html` only for fields the backend sanitizes — rich-text fields
  (`TiptapEditor`). Plain text always renders with `{{ }}`, which escapes it;
  plain text is never sanitized either (escaping already makes it safe, and a
  sanitizer would mangle text like `a < b`).
- Laravel FE + BE: sanitizing is the `[BE]` item's job (`backend-laravel.md` →
  "Rich text"); the page renders the field as is.
- Backend not in this repo (Vue/Nuxt, Laravel FE only): `v-html` only when the
  API docs say the field is sanitized; otherwise sanitize with DOMPurify before
  rendering (Nuxt SSR: `isomorphic-dompurify`). That's a new dependency — flag
  it first (`⚠️ needs <package> — please confirm`).

## Auth and permissions

Read `references/auth-modes.md` when a page logs in/out, needs the current
user, or sits behind a guard. Short version:

- Pages never touch tokens or CSRF headers; the helpers do.
- Don't hardcode role checks (`if (user.role === 'admin')`) in pages. If
  access control is needed and no guard/CASL exists, flag it as
  infrastructure work.
- Once CASL is registered, hide UI with `can(action, subject)` from
  `@layouts/plugins/casl`, and control nav items with their `action`/`subject`
  fields.

## State boundaries

Page state (query, rows, dialogs, form, loading) stays in local refs. Use
Pinia only for state that is shared across pages or persisted, such as the
logged-in user.

Pinia stores use the setup style: `defineStore('auth', () => { const user =
ref<User | null>(null); ...; return { user, login } })`, never the options
object (`state` / `getters` / `actions`). Calls inside go through `$api`, typed
like any page call.
