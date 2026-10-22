---
name: tmdr-materio
description: Generate Materio Vuetify pages (.vue) — and, in Laravel projects, the Laravel backend (migrations, models, requests, controllers, routes, tests) behind them — from a prompt plus a supporting document (PRD, data model, or similar). Use whenever the user asks to create/build/generate a page, screen, module, feature, or its API for this project — in any language or phrasing ("generate page for...", "build the invoice module", "buat API customer") — with or without an attached file. Also handles resuming a previously planned multi-item generation ("continue the plan", or the equivalent in the user's language), and auditing the project against the team's Frontend Developer Checklist ("run the checklist", "cek checklist", "audit checklist").
---

# tmdr-materio — Materio page (and Laravel backend) generator

Turns a prompt (+ optional PRD/data-model file) into real `.vue` pages that
match this project's existing Materio/Vuetify conventions, reusing what
already exists instead of inventing new UI. In the **Laravel** variant it can
also build the backend each page needs, so one plan covers FE and BE.

**Language:** reply, and write any new plan text, in the language of the
developer's prompt. Code, file names and identifiers stay English.

## Step 0 — detect the stack

Check which Materio variant this project is, and use its root as `<src>`
in every path in this skill and its references:

| Variant | Detect by | `<src>` | Endpoints come from | Backend in scope |
|---|---|---|---|---|
| Laravel | `artisan` + `resources/ts/pages/` exist | `resources/ts` | `docs/API.md` (kept up to date by `[BE]` items), Laravel `routes/*.php`, or the prompt/PRD | **Yes** — `[BE]` items, see `references/backend-laravel.md` |
| Vue (Vite) | `src/pages/` + `vite.config.ts`, no `artisan` | `src` | project API docs, or the prompt/PRD (backend lives elsewhere) | No — FE only |
| Nuxt | `nuxt.config.ts` exists | `app` if `app/pages/` exists (Nuxt 4), else the project root (Nuxt 3) | project API docs, `server/api/` if present, else the prompt/PRD | No — FE only (`server/api/` is read, never written) |

None match (React/Next, plain HTML, …) → stop and tell the developer this
skill only supports Materio Vue, Nuxt, and Laravel. Don't guess paths.

**Backend scope (Laravel only).** Default is FE + BE. If the developer says the
backend is done elsewhere or by someone else ("FE only", "backend sudah ada"),
plan and build FE only and treat the API like the Vue variant. Record the
choice in the plan header (`> Scope: FE + BE` or `> Scope: FE only`). In the
Vue/Nuxt variants, backend requirements in a PRD are **not** planned; mention
once that they're out of scope for this repo.

### Nuxt deltas

Everything in this skill and its references applies to Nuxt too, with these
substitutions — the reference files are written Vue-first:

| Vue / Laravel | Nuxt |
|---|---|
| `definePage({ meta: { layout: 'blank' } })` | `definePageMeta({ layout: 'blank' })` |
| `useApi` (VueUse `createFetch`) | `useApi` — same name, but a `useFetch` wrapper; SSR-aware, returns `data`/`pending`/`error`/`refresh` |
| `$api` (`ofetch` instance) | `$api` — same name, `$fetch` wrapper |
| route guard in `router/` | route middleware in `<src>/middleware/` |
| `import.meta.env.VITE_*` | `useRuntimeConfig().public.*` |
| `RouterLink` / `useRouter()` | `NuxtLink` / `navigateTo()` |
| `useServerTable` / `useApi` return names | read `composables/` once — the Nuxt versions wrap `useFetch` |

`composables/` and `utils/` are auto-imported — no import lines. Components:
`components/global/` is auto-imported with bare names (per `nuxt.config.ts`,
see `references/starter-kit.md`); `components/pages/<route>/` parts are still
imported explicitly, same as the other variants. Wrap anything browser-only in
`<ClientOnly>` rather than reaching for `process.client` checks.

### Starter kit check (once per session)

This skill targets the Timedoor starter kit: Materio **plus** project-level
components, composables, config and npm packages that the rules below require
(`PageHeader`, `ActionIconBtn`, `CardIcon`, `NumberField`, `DateTimeField`,
`ConfirmActionDialog`, `useServerTable`, …). Before the first `[FE]` item of a
session, read `references/starter-kit.md` and run its check. Missing pieces →
stop and list them for the developer; don't silently fall back to stock
Materio components. (A session that only does `[BE]` items may skip it.)

Run it **once per session**, not per page. If `docs/PLAN.md` exists, add
or refresh a line `> Starter kit: OK (<date>)` under its header after a passing
check; a new session that finds this line may skip the check (re-run it if a
required component turns out to be missing while generating).

### Checklist setup check (once per session)

Every generated item must pass the team's Frontend Developer Checklist
(`references/checklist.md`). Most of it is enforced by the conventions below. A
few items need app-wide pieces: offline banner, page titles, friendly API
errors, no console in builds, Sentry, and so on. Right after the starter kit
check, run `references/checklist.md` → "Project setup". Open items **don't
block** page generation, except S8 for a page with a date-of-birth field. List
them once and offer to apply them; each one needs the developer's go-ahead.
Record the result in `docs/PLAN.md` (`> Checklist setup: ...`), same as the
starter kit line.

## Step 1 — scope decision

- **Single item, clearly scoped** (e.g. "build the login page", "buat API
  customer") → skip straight to Step 3 (`[FE]`) or Step 3B (`[BE]`). In
  Laravel FE + BE scope, a single page that needs an endpoint that doesn't
  exist yet is two items: build the `[BE]` part first, then the page — in the
  same run is fine when both are small; otherwise write a short plan (Step 2).
- **Multiple pages / a whole module / a PRD covering several features** → do
  Step 2 first (plan), then stop and show the plan. Do not generate code in the
  same turn as producing a new plan — let the developer review it first.
- **No new prompt, just "continue the plan" (in any language)** → read
  `docs/PLAN.md`, find the first unchecked `- [ ]` item, run the env check
  for its module (its `Env:`, Step 2 → "Environment keys"), and do only that one:
  `[BE]` → Step 3B, `[FE]` → Step 3. Then update its checkbox. Skip Step 2.
  A single item outside a plan gets the same env check for the keys it needs.
- **"Run the checklist" / "cek checklist" / "audit checklist"** → audit the
  whole project per `references/checklist.md` → "Audit mode". It writes
  `docs/CHECKLIST.md` and reports the gaps. Nothing gets fixed in the same turn.

## Step 2 — write/update `docs/PLAN.md`

**Legacy file:** if `docs/PAGE_PLAN.md` exists and `docs/PLAN.md` doesn't,
rename it (`git mv docs/PAGE_PLAN.md docs/PLAN.md` when in git), keep every
checkbox state, and prefix existing items with `[FE]`. Never keep both files.

Group items by module. Infer module boundaries from how the PRD itself groups
features/sections — don't force a rigid taxonomy. Order modules by
dependency (e.g. Authentication before anything that assumes a logged-in
user).

Item types:

- `[BE]` (Laravel FE + BE scope only) — the API for **one resource/feature**:
  migration(s), model, factory, form requests, policy if needed, controller,
  API resource, routes, feature tests, and its entry in `docs/API.md`. List the
  endpoints it delivers after `→`.
- `[FE]` — one page (with its own dialogs/parts). List the endpoints it uses
  after `→ uses:`.

Within a module, every `[BE]` item comes **before** the `[FE]` items that use
its endpoints, so a page is always wired to an API that already exists and is
documented. Shared backend groundwork (auth, roles/permissions, a lookup table
several modules use) goes in the earliest module that needs it.

Always update this file in place — never create `PLAN_v2.md` or similar.
This file (not chat memory) is what lets work resume in a brand-new
conversation, since a new session has no memory of this one.

```markdown
# Plan
> Generated from: <source file/prompt> (<date>)
> Stack: Laravel · Scope: FE + BE
> Starter kit: OK (<date>)
> Checklist setup: open S11, S13 (<date>)

## Environment
| Key | Module | For | Status |
|---|---|---|---|
| DB_DATABASE, DB_USERNAME, DB_PASSWORD | 1 | database | ✅ set |
| MAIL_MAILER, MAIL_HOST, MAIL_USERNAME, MAIL_PASSWORD | 1 | password reset email | ❌ missing |
| PAYMENT_SERVER_KEY, PAYMENT_CLIENT_KEY | 3 | payment gateway | ↪ alternative: sandbox keys until production |

## Module 1: Authentication
Status: pending · Env: `DB_*`, `MAIL_*`
- [ ] [BE] Auth API → POST /api/auth/login, POST /api/auth/logout, GET /api/auth/me
- [ ] [FE] Login page → uses: POST /api/auth/login, GET /api/auth/me

## Module 2: Customers
Status: pending · Env: —
- [ ] [BE] Customers API → GET/POST /api/customers, GET/PUT/DELETE /api/customers/{id}
- [ ] [FE] Customer list page (+ create/edit dialog) → uses: /api/customers
```

Module `Status`: `pending` → `in progress` once any of its items is checked →
`done` when all are. Vue/Nuxt plans (or Laravel `FE only`) contain only `[FE]`
items and say so in the header (`> Stack: Nuxt · Scope: FE only`).

### Environment keys

While writing the plan, list every env key it needs, so the developer can
prepare them before work starts.

- **Sources:** what the stack needs to run (Laravel: `APP_URL`, `DB_*`, plus
  `SANCTUM_STATEFUL_DOMAINS` / `SESSION_DOMAIN` for session auth across
  subdomains; Vue: the `VITE_*` API base URL; Nuxt: `NUXT_PUBLIC_*`), every
  external service the PRD mentions (mail, payment gateway, object storage,
  SMS/WhatsApp, maps, push notifications), the checklist's services (Sentry DSN
  always; reCAPTCHA keys when the PRD has a public form; `checklist.md` S9/S13),
  and `.env.example` — reuse its names, never a second name for the same thing.
- Only keys this repo reads. In FE-only scope the backend's keys are not
  planned; say once that the backend team provides them.
- **Placement:** the `## Environment` table under the header, and
  `· Env: ...` on each module's `Status:` line with the keys that must be set
  before its first item (`Env: —` when none), each key in backticks — on a line
  of its own, Markdown would merge it into `Status:` and read `_*` as italics.
  A key belongs to the earliest module that needs it.
- **Status** comes from `.env`, checking presence only — never read, print or
  copy a value. Prints only the names of missing or empty keys:
  ```bash
  for k in DB_DATABASE MAIL_HOST; do
    grep -qE "^$k=['\"]?[^'\"[:space:]#]" .env 2>/dev/null || echo "missing: $k"; done
  ```
- In the reply that shows the plan, also list the ❌ keys, so the developer can
  start getting them (accounts, sandbox credentials) while reviewing.
- An item that introduces a key adds it to `.env.example` (empty, or a safe
  local default) and to this table.

**Env check before an item.** Run the check above for the module's `Env:`
keys. Any key missing and not marked `↪ alternative` → **stop before writing
code**: list the missing keys and what each is for, and ask the developer to
either fill them in `.env`, or go ahead with an alternative that works without
the value (e.g. `MAIL_MAILER=log`, local disk instead of object storage, the
provider's sandbox, the feature hidden behind a config check). Propose the
alternative that fits; wait for the answer. Record the choice in the table
(`↪ alternative: <what>`), then continue.

After writing/updating the plan, stop and show it to the developer — don't
auto-generate from it in the same turn.

## Step 3 — `[FE]` item: generate a page

For each page, work through two layers before writing any code:

1. **Layer 1 — `references/patterns/*.md`.** Find the file whose domain
   matches the page (auth, invoice, user/role/permission, dashboard, dialogs,
   cards, forms-wizard, tables). Adapt its skeleton — real conventions from
   this template, already trimmed to be copy-adaptable.
2. **Layer 2 — `references/component-catalog.md`.** If no pattern fits,
   compose the page from components/layouts that actually exist in this
   project, per the catalog. Never reference a component or prop that isn't
   in a pattern file or the catalog.
3. **Escalation — new component.** Only after both layers fail to cover the
   need, compose a new component from raw Vuetify primitives (`VCard`,
   `VIcon`, etc. — already themed by this project's `plugins/vuetify/`
   config, no manual styling needed). When this happens, **explicitly flag
   it** in your response to the developer, e.g.:
   `⚠️ No existing pattern/component covered "X" — created new component Y, please review closely.`
   Never silently invent a component.

Conventions to follow regardless of layer. Each rule is stated once here;
props, examples and gotchas live in the file it points to.

- File-based routing: place the page under `<src>/pages/...` matching
  this project's existing route structure.
- Component folders (never `views/`):
  - `<src>/components/global/<category>/` — reusable across pages, **auto-imported**
    (no import line). Categories: `app/` (app chrome), `page/` (page/card
    scaffolding), `form/` (inputs), `actions/` (buttons), `dialogs/` (generic
    dialogs). Add a new category folder only when none fits.
  - `<src>/components/pages/<route path>/` — parts made for one page only
    (its form, its dialogs), mirroring the route: `pages/admin/customers/index.vue`
    → `components/pages/admin/customers/CustomerDialog.vue`. **Not auto-imported**:
    `import CustomerDialog from '@/components/pages/admin/customers/CustomerDialog.vue'`.
  - Used by a second page → move it to `global/<category>/` (and drop the imports).
- `<script setup lang="ts">` style, matching the pattern files.
- **Props and emits** are typed: `interface Props { ... }` +
  `defineProps<Props>()` (defaults via `withDefaults`), and
  `defineEmits<{ (e: 'saved', id: number): void }>()` — never the runtime
  array/object form (`defineProps(['title'])`, `defineEmits(['saved'])`).
- **No `any`.** Every API response and payload has an interface, passed as the
  generic (`$api<{ data: Customer }>(...)`, `useServerTable<Customer>(...)`).
  Caught errors are typed as in `references/data-patterns.md` → "Forms and
  mutations". Truly unknown values are `unknown`, then narrowed.
- **Reactivity:** state lives in `ref`/`reactive`, never a plain `let`. Derived
  values are `computed`; `watch` only for side effects (refetch, sync to the
  URL, reset a form) — never a `watch` whose job is to set another ref.
- **Lifecycle hooks** (`onMounted`, …) only for DOM/browser work (focus,
  listeners — removed again in `onBeforeUnmount`). Page-load data goes through
  `useApi` / `useServerTable` (Nuxt: SSR-aware), never `onMounted` + `$api`.
- Pick layout via `definePage({ meta: { layout: '...' } })` — `blank` for
  auth/error/standalone pages, `default` for anything behind the main nav.
- Don't add manual colors/spacing that fight Vuetify's theme — components
  are already styled by Materio's defaults.
- **Page header:** every page in the `default`/`public` layouts starts with
  `<PageHeader>`; never hand-roll a page title or put it on the first card.
  Create/edit pages put Cancel + Save in its `#actions`, not under the form.
  → `references/patterns/page-header.md`.
- **Create/edit: dialog or page?** Small forms are a dialog for both create and
  edit — no `create.vue` / `[id]/edit.vue` routes. Use a dialog when ALL hold:
  ≤ ~4 simple fields (text, select, radio, short textarea); no field that
  appears/disappears based on another; no file upload and no add/remove-row
  list; nobody needs a direct URL to the form. Otherwise it's a page (many
  fields or sections, conditional fields, uploads/dynamic lists, rules that need
  context). → `references/patterns/dialogs.md` → "Form dialogs".
- **Icon-only buttons:** always `<ActionIconBtn icon="..." tooltip="...">`
  (tooltip required), never `IconBtn` or `<VBtn icon>`.
  → `references/component-catalog.md`.
- **Form validation:** on blur and on submit only (global `validateOn: 'blur lazy'`
  — don't set `validate-on`). Required fields: label ends with ` *` and
  `:rules="[requiredValidator, ...]"`. Submit handler starts with
  `const { valid } = await refForm.value!.validate(); if (!valid) return`.
  → `references/data-patterns.md` → "Client-side validation".
- **Date/time inputs:** always `<DateTimeField>` (model `yyyy-MM-dd` / `HH:mm`),
  never `type="date"/"time"` or `AppDateTimePicker`.
  → `references/component-catalog.md`.
- **Numeric inputs:** always `<NumberField>` (money: `currency`, model an integer
  in minor units; decimals: `:decimals="n"`), never `type="number"`. Display money
  only with the project's money formatter (starter kit: `formatRupiah`).
  → `references/component-catalog.md`.
- **Input limits:** every input has an upper bound, the same one the backend rule
  and the column use: text gets the native `maxlength` attribute (+ `counter` on
  textareas from 500), `NumberField` gets `:min`/`:max`. Limit from the PRD, else
  the defaults table, else the fallback — and flag every fallback.
  → `references/data-patterns.md` → "Input limits".
- **Empty values:** a field without a value is `null` — in form defaults and in
  edit prefill (`initial?.name ?? null`); multi-select `[]`, checkbox/switch
  `false`. Never `''` or `undefined`: `''` in `VSelect`/`VAutocomplete` counts
  as a selected item (floating label, clear icon on an empty field), and
  `undefined` is dropped from the JSON body, so an edit can't clear the field.
  Type form fields `T | null` (`ref<CustomerInput>(...)`), or `ref({ name: null })`
  infers `null` and v-model fails type-checking.
- **Sidenav:** a new top-level page gets a nav item in `<src>/navigation/`
  (vertical, plus horizontal if the layout uses it) with the same
  `action`/`subject` as its `definePage`. Every page under a nav item that isn't
  the item's own route (`create`, `[id]`, `[id]/edit`, …) sets `navActiveLink` to
  that item's route **name, as a string**:
  `definePage({ meta: { navActiveLink: 'customers' } })` (route names join path
  segments with `-`: `pages/customers/index.vue` → `customers`).
  `isNavLinkActive` in `<src>/@layouts/utils.ts` compares names with `===`, so
  `{ name: 'customers' }` type-checks but never highlights the item.
- **Raw HTML:** `v-html` only for HTML the backend sanitizes (rich-text fields);
  everything else renders with `{{ }}`. → `references/data-patterns.md` →
  "Rendering HTML".
- **Avatars of people** (account, users, members) show initials, never a generic user
  icon: `<VAvatar color="primary" variant="tonal">{{ initials(name) }}</VAvatar>`
  (`initials()` from `utils/format.ts`).
- **Card headers:** every card with a `title` gets `<CardIcon icon="ri-...">` in
  its `#prepend` slot; cards without a header (filter/table cards, forms under a
  `PageHeader`) get none. → `references/patterns/cards.md`.

Checklist rules (`references/checklist.md` maps every checklist item to its rule):

- **Page title:** every page sets `meta.title` in `definePage` (Nuxt:
  `definePageMeta`). Use the page's name in the UI language, the same as its
  `PageHeader` title. On detail pages, use the kind of record ("Customer
  Detail"), not the record's name.
- **Headings:** the tag follows the structure and the class sets the size.
  `PageHeader` is the page's only `h1`. Auth/error pages use
  `<h1 class="text-h4">` for their own title. Section headings below go
  `h2` → `h3` without skipping (`<h2 class="text-h5">`). Never pick a tag for its
  size.
- **Responsive:** lay out with `VRow`/`VCol`. Every `VCol` has `cols="12"` plus
  `sm`/`md` widths for wider screens. Button and filter rows use
  `d-flex flex-wrap gap-4`. No fixed px widths or heights on containers
  (`max-width` on dialogs is fine). Tables scroll inside their card, and the page
  never scrolls sideways. Keep the kit's control sizes: no `size="x-small"` /
  `"small"` on action buttons, since they become too small to tap.
- **Links and buttons:** every `:to` / `RouterLink` points to a route that exists
  under `<src>/pages/`. No `href="#"`, and no button without a handler. Something
  not available yet is `:disabled` and says why.
- **Destructive actions** (delete, cancel, deactivate) always go through
  `ConfirmActionDialog` first. → `references/patterns/dialogs.md`.
- **Dialogs and menus close:** every dialog has `DialogCloseBtn` and a Cancel
  button, and closes on Esc or an outside click (no `persistent`). Menus and
  tooltips keep Vuetify's own open/close behaviour.
- **Password fields:** `type="password"` with the eye toggle
  (`references/patterns/authentication.md`). Forms that *set* a password (create
  user, change/reset password) also get `autocomplete="new-password"`, so the
  browser doesn't fill in the admin's own password.
- **Sensitive fields:** a form with personal, financial or medical identifiers
  (national ID, bank/card number, diagnosis) gets `autocomplete="off"` on its
  `VForm`. OTP inputs use `autocomplete="one-time-code"`. Such values never go
  in a URL (`references/data-patterns.md` → "URLs").
- **Date of birth:** `<DateTimeField birth-date>` opens on the year list,
  10 years back, with no future dates. If the prop is missing, apply checklist S8
  first.
- **Loading and empty states:** tables follow `references/patterns/tables.md`.
  Every other fetched list or detail block follows
  `references/data-patterns.md` → "Loading and empty states".
- **Images:** every `VImg`/`img` has an `alt` saying what it shows (`alt=""` only
  for pure decoration). A dynamic image whose URL may be missing falls back to
  `https://placehold.co/<w>x<h>?text=No+Image` at the slot's size. People get
  initials instead (see "Avatars").
- **External links:** every link that leaves the app gets
  `target="_blank" rel="noopener noreferrer"`. Use only `https://` URLs, taken
  from the PRD or from the record itself. In-app navigation uses `RouterLink` /
  `:to`, never `href`.
- **No debug code:** no `console.*`, `debugger`, commented-out code or dummy data
  in generated files. Production builds strip console as a safety net (S5).
- **Design file:** when the prompt or PRD links a Figma/XD design, follow it
  wherever it differs from the pattern's default shape (still using kit
  components), and list what you couldn't match.

## Step 3B — `[BE]` item: build the Laravel backend (Laravel variant only)

Read and follow `references/backend-laravel.md`. In short: learn the
backend's conventions from the existing code once per session, then build one
resource/feature end to end (migration → model → requests → controller →
resource → routes → tests), shaped exactly as the FE side expects (validation
`422`, the pagination/sort params `useServerTable` sends, money in integer minor units, dates as
`Y-m-d`/`H:i`), and record its endpoints in `docs/API.md`. Never run a
destructive database command; ask before running migrations against the
developer's database.

## Step 4 — wire data (only if the page talks to an API)

Read and follow `references/data-patterns.md` before writing any fetch, form
submit, loading state, or permission check. Do its Step A (learn the
project's API conventions) once per session; the backend can be any stack.
When a `[BE]` item of this plan built the endpoint, `docs/API.md` is the
contract — use its paths, fields and error shapes as written.
If the page logs in/out, reads the current user, or needs a guard, also read
`references/auth-modes.md` and follow only the project's mode.

## Step 5 — validate

Check only the files you created or changed.

**FE.** Don't run `npm run lint`: it runs `eslint --fix` on the whole repo and
can rewrite unrelated files.

- Lint: `npx eslint --fix <your files>`. `--fix` is fine here because it only
  touches your files.
- Types: if `package.json` has a `typecheck` script, run it. It checks the
  whole project but changes nothing, so only fix errors in your files and
  report any errors you find elsewhere.

**BE.** Per `references/backend-laravel.md` → "Validate": `vendor/bin/pint
<your files>` (never bare `pint`), `php artisan route:list --path=<prefix>`, and
the item's feature tests — only after confirming the test database is not the
developer's database.

**Checklist.** Run `references/checklist.md` → "Checks" on your files and fix
what they find. In the response, list the Manual items this item touches,
such as which screen sizes to check or which Figma frame to compare, so the
developer knows what to verify by hand.

## Step 6 — update the plan

If this generation came from `docs/PLAN.md`, mark the item's checkbox
`[x]` in that same file once it is written and validated, and update the
module's `Status`. If a `[BE]` item's endpoints ended up different from what
the plan listed, fix the `→` / `→ uses:` text of the affected items too.
Refresh the `## Environment` statuses (the check in Step 2), and add any key
the item introduced.

## Scope limits

- FE: UI-only by default; wire data only per Step 4.
- BE: Laravel variant only, and only what the PRD/prompt asks for — no extra
  endpoints, columns or features "for later".
- Do one plan item per run for quality control (a small `[BE]` + its `[FE]`
  page may go together, see Step 1) — don't batch-generate an entire plan
  unprompted.
- If a requirement is genuinely ambiguous in a way that affects what gets
  built (fields, business rules, who may do what), ask rather than guessing.
