# Pattern: Page Header (every page)

**When to use:** the first element of **every** page in the `default` and `public`
layouts. Not on `blank` pages (login, forgot/reset password, errors) — those
keep their own auth/error layout.

**Component:** `<src>/components/global/page/PageHeader.vue` (auto-imported, no import line).

Shape (matches the design reference): optional back button (`ActionIconBtn`,
40px, tooltip "Back") · title (`<h1 class="text-h4">`, the page's only `h1`; checklist S7) · muted subtitle
(`text-body-2`) · optional actions on the right.

| Prop / slot | Type | Notes |
|---|---|---|
| `title` | string, required | Page name, or the record's name on detail pages (`Budi`, `Wed, 30 Sep 2026`). |
| `subtitle` | string | One short line describing the page or record. |
| `back` | string | Override the back target. Rarely needed. |
| `hide-back` | boolean | Hide the back button on a nested page. Rarely needed. |
| `#status` | slot | Status chip(s) of the record, shown right after the title (`Paid`, `Inactive`). |
| `#actions` | slot | Page-level **buttons** on the right (`Add Customer`, Save/Cancel). Never status chips. |

**Back button rule (automatic — don't pass anything):**
- Top-level pages have **no** back button: one segment after the area prefix —
  `/admin`, `/admin/customers`, `/admin/settings`, `/products`, `/gallery`.
- Nested pages **do**, pointing at the parent path:
  `/admin/orders/create` → `/admin/orders`, `/admin/customers/3` → `/admin/customers`,
  `/admin/orders/3/edit` → `/admin/orders/3`, `/gallery/5` → `/gallery`.

```vue
<!-- List page (top level): title + primary action, no back button -->
<template>
  <div>
    <PageHeader title="Customers" subtitle="Everyone who has ordered">
      <template #actions>
        <VBtn prepend-icon="ri-add-line" @click="openCustomerDialog(null)">Add Customer</VBtn> <!-- small form → dialog -->
      </template>
    </PageHeader>

    <VCard> <!-- no `title` on the card: the header already names the page -->
      ...
    </VCard>
  </div>
</template>
```

```vue
<!-- Detail/edit page (nested): render the header once the record has loaded -->
<template>
  <VAlert v-if="error" type="error" variant="tonal">Customer not found.</VAlert>
  <template v-else-if="data">
    <PageHeader :title="data.data.name" subtitle="Orders and contact details" />
    <VRow>...</VRow>
  </template>
</template>
```

**Rules:**
- Type scale: page title `text-h4` (24px) > card title (18px); page subtitle `text-body-2`; card subtitle 13px (global override in `styles.scss`). Don't restyle either per page.
- Exactly one `PageHeader` per page, before any alert or card.
- Don't repeat the page title as a `VCard` title below it. Cards inside the
  page get their own section titles (`Items`, `History`, `Details`).
- Primary page actions go in `#actions`, not inside the first card.

**Form pages (create/edit) — only for forms too big for a dialog (see the
dialog-or-page rule in SKILL.md): Save and Cancel go in the header, never at the
bottom of the form.** The shared form component
(`components/pages/<route>/XForm.vue`, imported explicitly) renders the `PageHeader` itself — it owns
the submit/loading state — and takes `title`/`subtitle` props from the page.
The card below has no title and no buttons. The Save button sits outside the
`<form>` but is linked by id, so it is still the form's submit button (Enter
in a field submits too):

```vue
<!-- components/pages/admin/orders/OrderForm.vue -->
<template>
  <PageHeader :title="props.title" :subtitle="props.subtitle">
    <template #actions>
      <VBtn color="secondary" variant="outlined" :to="cancelTo">Cancel</VBtn>
      <VBtn type="submit" form="order-form" :loading="isSubmitting">Save Changes</VBtn>
    </template>
  </PageHeader>

  <VCard>
    <VCardText>
      <VForm id="order-form" @submit.prevent="onSubmit">
        <!-- fields only -->
      </VForm>
    </VCardText>
  </VCard>
</template>

<!-- pages/admin/orders/[id]/edit.vue -->
<OrderForm v-if="data" :id="data.data.id" title="Edit Order" :subtitle="`#${data.data.number}`" :initial="data.data" />
```
Order: Cancel (outlined, secondary) then the primary Save on the far right.
Give each form a unique `id` (`<resource>-form`).
