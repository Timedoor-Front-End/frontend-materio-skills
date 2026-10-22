# Starter kit additions (Timedoor Materio starter kit)

The Timedoor starter kit is Materio (Vue, Nuxt or Laravel) **plus** the pieces
below. The SKILL.md conventions and the patterns require them (e.g. "every
page starts with `PageHeader`", "numeric inputs are `NumberField`"). They are
not part of stock Materio, so check they exist before generating a page.

`<src>` is the root found in SKILL.md Step 0: `resources/ts` (Laravel),
`src` (Vue), `app` (Nuxt 4) or `.` (Nuxt 3). Every path below is relative to it
unless it says otherwise.

## Components — `<src>/components/global/<category>/` (auto-imported)

`components/global/` is auto-imported under the bare component name
(`PageHeader`, not `GlobalPagePageHeader`); `components/pages/<route>/` holds one
page's own parts and is always imported explicitly. How the auto-import is set up
differs per variant — see "Global config" below.

| File (under `components/global/`) | Role | Required by |
|---|---|---|
| `page/PageHeader.vue` | Title, subtitle, automatic back button, `#status` and `#actions` slots | every page — `patterns/page-header.md` |
| `actions/ActionIconBtn.vue` | The only icon-only button: outlined rounded square, required `tooltip` | row actions, back button — `patterns/tables.md` |
| `page/CardIcon.vue` | Tonal icon tile in a titled card's `#prepend` | every card header — `patterns/cards.md` |
| `form/NumberField.vue` | cleave-zen numeric input; `currency` → v-model integer in minor units (default 2 decimals, `Rp`); `decimals` for plain decimals; `min`/`max` | every numeric input |
| `form/DateTimeField.vue` | `@vuepic/vue-datepicker` (v14) date/time input behind a `VTextField`; model `yyyy-MM-dd` / `HH:mm` | every date/time input |
| `dialogs/ConfirmActionDialog.vue` | Confirm before a server action; server error shown in its slot | delete/cancel/deactivate — `patterns/dialogs.md` |

Props and gotchas of each: `component-catalog.md` → "Starter kit components".

## Composables and utils — auto-imported

| File | Exports | Notes |
|---|---|---|
| `composables/useServerTable.ts` | `useServerTable<T>(path, () => filters)` | `VDataTableServer` state; filters/search/sort debounced 500ms, paging immediate — `patterns/tables.md`. On Nuxt it is built on the `useFetch`-based `useApi`: read the file once for its exact return names. |
| `utils/format.ts` | `formatRupiah(sen)` → `Rp 1.250.000,00`, `formatDate('2026-10-07')` → `Wed, 7 Oct 2026`, `initials(name)` | `formatDate` must be imported explicitly from `@/utils/format` (stock `@core/utils/formatters` exports a different `formatDate`) |
| `utils/api.ts` | `$api`, `apiErrorMessage(error)` | `apiErrorMessage` returns the first field message of a 422 business rule, else `message` |

## Removed from stock Materio

Unused demo components are not in the kit: `AppPricing`, `AppSearchHeader`, and the demo dialogs
(`ConfirmDialog`, `AddEditAddressDialog`, `AddEditRoleDialog`, `AddEditPermissionDialog`, `CardAddEditDialog`,
`AddPaymentMethodDialog`, `PaymentProvidersDialog`, `TwoFactorAuthDialog`, `EnableOneTimePasswordDialog`,
`AddAuthenticatorAppDialog`, `UserInfoEditDialog`, `UserUpgradePlanDialog`, `PricingPlanDialog`,
`ShareProjectDialog`, `ReferAndEarnDialog`, `CreateAppDialog`). Never reference them.

## Global config and styles

| Where | What |
|---|---|
| Vue / Laravel: `vite.config.ts` (project root) | `Components({ dirs: ['<src>/@core/components', '<src>/components/global'] })` — `components/pages/` is not in `dirs` |
| Nuxt: `nuxt.config.ts` (project root) | `components` option registers `~/components/global` (and `@core/components`) with `pathPrefix: false`, so names stay bare. Page-specific components are still imported explicitly, for consistency with the other variants. |
| `plugins/vuetify/defaults.ts` | `VForm: { validateOn: 'blur lazy' }` — validation on blur and submit, never on first render |
| the project's main `styles.scss` (Laravel: `resources/styles/`, Vue/Nuxt: under `<src>/assets/styles/`) | `.bg-info-block` (tinted read-only block, table-header tone) and `.v-card .v-card-subtitle` at 13px |

## npm dependencies

`cleave-zen` (NumberField) and `@vuepic/vue-datepicker` **v14** (DateTimeField —
the component uses v14's `dp-input` slot, `formats.input`, `timeConfig`,
`actionRow`; another major version needs the component adapted first).

## Conventions the kit assumes

- **Money is an integer in minor units** (100 per unit by default — cents, sen) in the
  API; `NumberField currency` (`minorUnits` 100) and the money formatter (kit:
  `formatRupiah`) expect that. A backend that stores whole units uses
  `<NumberField currency :minor-units="1">` and needs the formatter adjusted —
  confirm with the developer first; never divide/multiply in pages. Another
  currency: `currency-prefix` on `NumberField` and its own formatter.
- Dates travel as `yyyy-MM-dd`, times as `HH:mm`; `DateTimeField` only changes
  the display.

## Per-project settings inside kit components

The global components contain no project code. One value is meant to be set per project:

| Where | Setting | Default |
|---|---|---|
| `components/global/page/PageHeader.vue` | `AREA_PREFIXES` — first URL segments that only mark an area, so `/admin/orders` counts as top level (no back button) | `['admin']` |

For another currency or minor unit, see `NumberField`'s `currency-prefix` / `minor-units` props in `component-catalog.md`.

The Frontend Developer Checklist adds a few more app-wide pieces (offline banner,
page titles, `PageHeader` as `h1`, `DateTimeField birth-date`, Sentry, …). They are
checked separately and don't block: `checklist.md` → "Project setup".

## Check before generating (SKILL.md Step 0 — once per session)

Look for the six component files, the two composable/util files, the
auto-import config, the `VForm` default, the style utility and both npm
packages. If anything is missing, **stop and tell the developer which pieces
are missing** — the project was probably created from stock Materio or an
older kit. Don't fall back to the stock equivalent (`IconBtn`, `type="number"`,
`type="date"`, `AppDateTimePicker`) without their go-ahead.

Quick check — set `SRC` to the root from Step 0, run from the project root; it
works for every variant and prints only what's missing:

```bash
SRC=src   # resources/ts (Laravel) | src (Vue) | app (Nuxt 4) | . (Nuxt 3)
for f in components/global/page/{PageHeader,CardIcon}.vue \
         components/global/form/{NumberField,DateTimeField}.vue \
         components/global/actions/ActionIconBtn.vue \
         components/global/dialogs/ConfirmActionDialog.vue \
         composables/useServerTable.ts utils/format.ts; do
  [ -f "$SRC/$f" ] || echo "missing file: $SRC/$f"; done
if [ -f nuxt.config.ts ]; then
  grep -q "components/global" nuxt.config.ts || echo "missing: nuxt.config.ts 'components' must register ~/components/global (pathPrefix: false)"
else
  grep -q "components/global'" vite.config.ts || echo "missing: vite.config.ts Components({ dirs }) must include $SRC/components/global"
fi
grep -qs "apiErrorMessage" "$SRC/utils/api.ts" || echo "missing: apiErrorMessage in $SRC/utils/api.ts"
grep -qs "decimals" "$SRC/components/global/form/NumberField.vue" || echo "outdated: NumberField has no decimals/min/max props"
grep -qs "validateOn: 'blur lazy'" "$SRC/plugins/vuetify/defaults.ts" || echo "missing: VForm validateOn default in $SRC/plugins/vuetify/defaults.ts"
grep -rqs "bg-info-block" --include='*.scss' --exclude-dir=node_modules --exclude-dir=.nuxt --exclude-dir=.output --exclude-dir=vendor . || echo "missing: .bg-info-block in the project styles"
for p in cleave-zen @vuepic/vue-datepicker; do grep -q "\"$p\"" package.json || echo "missing package: $p"; done
```

A file that exists somewhere else (e.g. `$api` defined in another util) is not
"missing" — check with `grep -rn` before reporting it, and tell the developer
where it actually lives.
