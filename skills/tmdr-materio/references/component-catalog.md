# Materio Component Catalog (Layer 2 fallback)

Only use this layer when no pattern in `patterns/*.md` covers the requirement.
Every item below actually exists in this project — never invent a component
or prop name that isn't listed here or in the pattern files.

`<src>` is the root from SKILL.md Step 0. The starter-kit components are the
same in every variant; the stock Materio lists below were taken from the
Laravel kit, so on Vue/Nuxt check that a stock file exists before using it.

## `<src>/layouts/`
- `default.vue` — main app shell (top nav / vertical nav, used for anything behind login).
- `blank.vue` — no nav/chrome, used for auth pages, error pages, misc standalone pages.
- `components/DefaultLayoutWithVerticalNav.vue` — vertical sidebar nav variant of default layout.
- `components/DefaultLayoutWithHorizontalNav.vue` — horizontal top nav variant.
- `components/*` (Footer, nav bar pieces, UserProfile) — app chrome, not used in pages.

## `<src>/components/global/` (auto-imported; `components/pages/<route>/` is per-page and imported explicitly)

### Starter kit components (Timedoor additions — see `starter-kit.md`)

Required by the conventions; use these instead of the stock equivalents.

- `actions/ActionIconBtn.vue` — **starter kit, the only icon-only button allowed in pages**: outlined rounded square (34px, light theme border, 20px icon) with a tooltip. Props `icon`, `tooltip` (both required — the tooltip doubles as `aria-label`, and vue-tsc fails without it); everything else (`to`, `size`, `color`, `loading`, `disabled`, `@click`) passes through to `VBtn`. Table action rows: `<div class="d-flex flex-nowrap gap-2">` of these. Don't use the template's `IconBtn` alias in pages (it's for navbar chrome).
- `form/DateTimeField.vue` — **starter kit, the only date/time input**: `@vuepic/vue-datepicker` (v14) whose
  `dp-input` slot renders a `VTextField`, so label, `*`, `:rules`, `:error-messages` behave like other fields.
  Props `modelValue`, `mode` (`'date'` default | `'time'`), `maxDate`, `birthDate` (date of birth: opens on the
  year list with the calendar at 10 years back, no future dates, field stays empty until picked; added by
  checklist S8, so apply that first if the component lacks it). Date model `yyyy-MM-dd` (display
  `Wed, 7 Oct 2026`, a click on a day applies it), time model `HH:mm` (24h, applied only with the picker's
  **Save** button; Cancel or clicking outside keeps the old value). Validates when the picker closes (not on the text field's blur — opening
  the picker moves focus into its menu, so the inner `VTextField` uses `validate-on="submit lazy"`), on submit,
  and again on change while it shows an error. Picker colors are mapped to the Vuetify theme (`--dp-*` vars), light and dark; menu is teleported.
  ```vue
  <DateTimeField v-model="form.date" label="Date *" :rules="[requiredValidator]" />
  <DateTimeField v-model="form.start_time" mode="time" label="Start Time *" :rules="[requiredValidator]" />
  <DateTimeField v-model="form.transaction_date" label="Date *" :max-date="new Date()" />
  ```
  - Gotcha (already handled inside): the picker's `.dp--input-wrap` sets `box-sizing: unset`; with Materio's
    `box-sizing: inherit` reset the field grew to 72px until border-box was restored.
- `form/NumberField.vue` — **starter kit, the only numeric input**: `VTextField` formatted live by cleave-zen (`formatNumeral` + `registerCursorTracker`, delimiter `.`, decimal mark `,`). Props `modelValue` (number | null), `currency`, `currencyPrefix` (default `'Rp'`), `minorUnits` (`1 | 10 | 100 | 1000`, default `100`), `decimals` (default `0`), `min` (default `0`), `max`. `currency`: v-model is an integer in minor units — default 2 decimals (`125000050` ↔ `1.250.000,50`); `:minor-units="1"` for a backend that stores whole units (no decimals). Blur re-shows the full decimals. Without `currency`: a plain integer, or with `:decimals="1"` a decimal number (`65,5` ↔ `65.5`). `min`/`max` are in model units (minor units for currency) and checked by a built-in rule on the model value — don't use `betweenValidator` here, rules get the formatted text. `max` defaults to `9_000_000_000_000_000` with `currency` (the money cap), else `Number.MAX_SAFE_INTEGER`. Other `VTextField` props, `:rules` included, pass through. Never use `type="number"`.
  ```vue
  <NumberField v-model="form.price" currency label="Price *" :rules="[requiredValidator]" />
  <NumberField v-model="form.quantity" label="Quantity *" :min="1" :max="9999" :rules="[requiredValidator]" />
  <NumberField v-model="form.weight" :decimals="1" :max="500" label="Weight (kg)" />
  ```
  - cleave-zen gotcha: `unformatNumeral` must always get `{ numeralDecimalMark: ',' }`, or the `.` thousands delimiters are read as decimals.
- `page/CardIcon.vue` — **starter kit, required on every card header**: tonal primary rounded-square icon tile (38px avatar, 22px icon). Prop `icon`. Put it in the `#prepend` slot of a titled `VCard` or `VCardItem`:
  `<VCard title="History"><template #prepend><CardIcon icon="ri-history-line" /></template>...</VCard>`
- `page/PageHeader.vue` — **starter kit, required on every page** (`default`/`public` layouts). Props, slots and usage: `patterns/page-header.md`; per-project setting `AREA_PREFIXES`: `starter-kit.md`.

### Stock Materio components

- `app/AppLoadingIndicator.vue` — global route-change loading bar, already wired app-wide, don't re-add per page.
- `app/ErrorHeader.vue` — big status-code + title + description block. Use for 404/500/error pages.

## `<src>/components/global/dialogs/` (all are `v-model:is-dialog-visible` popups — see `patterns/dialogs.md` for usage shape)
The starter kit removed Materio's unused demo dialogs (address, role/permission, payment, 2FA, pricing, share, refer, create-app,
user-info) and the demo `ConfirmDialog`. Don't reference them; write a new dialog per `patterns/dialogs.md` instead.
- `ConfirmActionDialog.vue` — **starter kit**: confirm before a server action (delete, cancel, deactivate). Props `title`, `message`, `confirm-text`, `cancel-text`, `confirm-color`, `loading`; emits `confirm`; default slot for extra fields or the server's error `VAlert`. The page closes it after the request succeeds.

## `<src>/@core/components/`
- `AppDrawerHeaderSection.vue` — standard header (title + close button) for `VNavigationDrawer` side panels — use this for any custom "Add New X" drawer.
- `AppStepper.vue` — horizontal/vertical stepper header (props: `items: {title, icon?, subtitle?}[]`, `v-model:current-step`). Pair with `VWindow`/`VWindowItem` for step content — see `patterns/forms-wizard.md`.
- `DialogCloseBtn.vue` — small X button positioned top-right of a `VDialog`/`VCard`, use inside every custom dialog.
- `DropZone.vue` — drag-and-drop file upload area with preview (uses `@vueuse/core`).
- `MoreBtn.vue` — "⋮" icon button that opens a `VMenu` from a `menu-list` prop — use instead of writing a custom kebab menu.
- `TiptapEditor.vue` — rich text editor (Tiptap) wrapper, `v-model` bound, for any rich-text field.
- `TablePagination.vue` — prev/next + page-size footer for a table (props: `page`, `itemsPerPage`, `totalItems`) — prefer it over re-inlining footer markup.
- App chrome / demo only, **never used in pages:** `AppBarSearch`, `BuyNow`, `CustomizerSection`, `I18n`, `Notifications`, `ProductDescriptionEditor`, `ScrollToTop`, `Shortcuts`, `TheCustomizer`, `ThemeSwitcher`, `cards/AppCardCode`.

### `app-form-elements/`
- `AppDateTimePicker.vue` — **don't use**; use `DateTimeField` instead.
- `CustomCheckboxes.vue` / `CustomCheckboxesWithIcon.vue` / `CustomCheckboxesWithImage.vue` — styled checkbox groups (plain / with icon / with image), `v-model` + `:options` array.
- `CustomRadios.vue` / `CustomRadiosWithIcon.vue` / `CustomRadiosWithImage.vue` — same idea for radio groups.

### `cards/`
- `AppCardActions.vue` — card wrapper with collapse/refresh/remove header actions — use when a card's content needs to be collapsible.
- `CardStatisticsHorizontal.vue` — stat card, icon+value laid out horizontally (props: `title`, `color`, `icon`, `stats`, `change`).
- `CardStatisticsVertical.vue` — stat card, icon+value stacked vertically (same props + `subtitle`).
- `CardStatisticsWithIcon.vue` — stat card, icon-accented (props: `title`, `value`, `change`, `desc`, `icon`, `iconColor`).
- `CardStatisticsWithImages.vue` — stat card variant with an illustration image instead of an icon.

## Vuetify primitives (always available, already themed by Materio — see `plugins/vuetify/defaults.ts` for defaults like `VBtn` primary color, `VTextField` outlined/comfortable)
`VCard`, `VCardText`, `VCardTitle`, `VRow`, `VCol`, `VBtn`, `VTextField`, `VSelect`, `VAutocomplete`, `VCheckbox`, `VRadio`, `VSwitch`, `VChip`, `VAvatar`, `VIcon`, `VDivider`, `VDataTableServer`, `VDataTable`, `VDialog`, `VForm`, `VTooltip`, `VMenu`, `VList`/`VListItem`, `VNavigationDrawer`, `VWindow`/`VWindowItem`, `VTabs`/`VTab`, `VSpacer`. Only reach for raw Vuetify composition (rung 3 of the escalation) after checking layers 1 and 2.
