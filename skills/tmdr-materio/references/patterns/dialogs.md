# Pattern: Dialogs / Popups

**When to use:** any confirm/add/edit/share popup triggered from a button or a table row action.

**Source (full-version):** `pages/pages/dialog-examples/index.vue`
**Cross-reference:** the starter kit keeps only `ConfirmActionDialog` in `<src>/components/global/dialogs/` (Materio's demo dialogs were removed) — use it for confirmations and write a new dialog for anything else (see `component-catalog.md`).

**Shape:** a boolean `ref` toggled by a trigger button, passed into the dialog component via `v-model:is-dialog-visible`. The dialog components themselves are self-contained (they render their own `VDialog` + `VCard` + `DialogCloseBtn`).

```vue
<script setup lang="ts">
const isDeleteDialogVisible = ref(false)
const deleteError = ref('')
const isDeleting = ref(false)

const confirmDelete = async () => {
  isDeleting.value = true
  deleteError.value = ''
  try {
    await $api(`/admin/items/${item.value.id}`, { method: 'DELETE' })
    isDeleteDialogVisible.value = false
    refresh()
  }
  catch (e) {
    deleteError.value = apiErrorMessage(e)
  }
  finally {
    isDeleting.value = false
  }
}
</script>

<template>
  <VBtn @click="isDeleteDialogVisible = true">
    Delete
  </VBtn>

  <ConfirmActionDialog
    v-model:is-dialog-visible="isDeleteDialogVisible"
    title="Delete item?"
    message="This can't be undone."
    confirm-text="Delete"
    :loading="isDeleting"
    @confirm="confirmDelete"
  >
    <VAlert v-if="deleteError" type="error" variant="tonal" class="mt-4">{{ deleteError }}</VAlert>
  </ConfirmActionDialog>
</template>
```

**Writing a NEW dialog** (anything that isn't a plain confirmation):

```vue
<script setup lang="ts">
interface Props { isDialogVisible: boolean }
const props = defineProps<Props>()
const emit = defineEmits<{ (e: 'update:isDialogVisible', v: boolean): void }>()
</script>

<template>
  <VDialog
    :model-value="props.isDialogVisible"
    max-width="500"
    @update:model-value="val => emit('update:isDialogVisible', val)"
  >
    <VCard class="pa-5">
      <DialogCloseBtn @click="emit('update:isDialogVisible', false)" />
      <VCardText>
        <!-- form fields -->
      </VCardText>
    </VCard>
  </VDialog>
</template>
```

**Notes:**
- **Never derive visibility from the dialog's data** (`:is-dialog-visible="!!target"` + `target = null` on close). The text vanishes while the close animation still runs. Keep a separate boolean and leave the data alone when closing; overwrite it on the next open:
  ```ts
  const deleteTarget = ref<Customer | null>(null)
  const isDeleteDialogVisible = ref(false)
  const openDelete = (c: Customer) => { deleteTarget.value = c; isDeleteDialogVisible.value = true }
  // success: isDeleteDialogVisible.value = false  (deleteTarget stays)
  ```
  `<ConfirmActionDialog v-model:is-dialog-visible="isDeleteDialogVisible" :message="`${deleteTarget?.name} ...`" />`
- Always use the `v-model:is-dialog-visible` + `DialogCloseBtn` shape for consistency with every existing dialog in this project.
- If the PRD describes a dialog that's clearly a variant of one already in the project (e.g. another resource's add/edit dialog), follow that one's shape rather than inventing a new one.

## Form dialogs (small create/edit forms)

Use for forms that pass the rule in SKILL.md (≤ ~4 simple fields, nothing
conditional, no uploads/dynamic lists, no deep link needed). One component per
resource in `components/pages/<route>/<Resource>Dialog.vue` (imported explicitly by the page) handles both modes:
create when the record prop is empty, edit when it's set.

- Title `Add <Resource>` / `Edit <Resource>`, `DialogCloseBtn`, fields stacked
  in one column, then Cancel (outlined, secondary) + primary Save at the bottom
  right. Save is `type="submit" form="<resource>-dialog-form"` so Enter submits.
- Fill the form in a `watch` on open; don't clear it on close (see the
  visibility note above). Reset `fieldErrors` on open.
- 422 → `:error-messages` per field; other errors → `VAlert` with
  `apiErrorMessage(e)`.
- On success: `emit('saved', record)`, close. The caller refreshes
  (`refresh()` of `useServerTable` or of the detail `useApi`) — no navigation.
- Open it from the list's `PageHeader` "Add" button, the row's Edit
  `ActionIconBtn`, and the detail page's Edit button.
- Bind `@saved="() => refresh()"` (not `@saved="refresh"`: VueUse's `execute`
  takes a boolean, so passing it the record is a type error).

```vue
<!-- components/pages/admin/customers/CustomerDialog.vue (trimmed) -->
<script setup lang="ts">
interface Props { isDialogVisible: boolean, customer?: Customer | null }
const props = defineProps<Props>()
const emit = defineEmits<{
  (e: 'update:isDialogVisible', value: boolean): void
  (e: 'saved', customer: Customer): void
}>()

const refForm = ref()
const form = ref<CustomerInput>({ name: null, phone: null, notes: null }) // fields `string | null`
const isSubmitting = ref(false)
const fieldErrors = ref<Record<string, string[]>>({})
const formError = ref('')

watch(() => props.isDialogVisible, open => {
  if (!open)
    return
  form.value = { name: props.customer?.name ?? null, phone: props.customer?.phone ?? null, notes: props.customer?.notes ?? null }
  fieldErrors.value = {}
  formError.value = ''
  refForm.value?.resetValidation()
})

const onSubmit = async () => {
  const { valid } = await refForm.value!.validate()
  if (!valid)
    return

  isSubmitting.value = true
  fieldErrors.value = {}
  formError.value = ''
  try {
    const res = props.customer
      ? await $api<{ data: Customer }>(`/admin/customers/${props.customer.id}`, { method: 'PUT', body: form.value })
      : await $api<{ data: Customer }>('/admin/customers', { method: 'POST', body: form.value })
    emit('saved', res.data)
    emit('update:isDialogVisible', false)
  }
  catch (e) {
    // validation error shape comes from data-patterns.md Step A — Laravel 422 shown here
    const error = e as FetchError<{ message?: string, errors?: Record<string, string[]> }> // import type { FetchError } from 'ofetch'
    if (error.status === 422)
      fieldErrors.value = error.data?.errors ?? {}
    else
      formError.value = apiErrorMessage(e)
  }
  finally {
    isSubmitting.value = false
  }
}
</script>

<template>
  <VDialog max-width="560" :model-value="props.isDialogVisible" @update:model-value="v => emit('update:isDialogVisible', v)">
    <VCard :title="props.customer ? 'Edit Customer' : 'Add Customer'">
      <DialogCloseBtn @click="emit('update:isDialogVisible', false)" />
      <VCardText>
        <VAlert v-if="formError" type="error" variant="tonal" class="mb-4">{{ formError }}</VAlert>
        <VForm id="customer-dialog-form" ref="refForm" @submit.prevent="onSubmit">
          <VTextField v-model="form.name" label="Name *" maxlength="100" :rules="[requiredValidator]" :error-messages="fieldErrors.name" />
          <!-- other fields: same `:error-messages="fieldErrors.<field>"` and their own `maxlength` -->
        </VForm>
      </VCardText>
      <VCardText class="d-flex justify-end gap-4">
        <VBtn color="secondary" variant="outlined" @click="emit('update:isDialogVisible', false)">Cancel</VBtn>
        <VBtn type="submit" form="customer-dialog-form" :loading="isSubmitting">{{ props.customer ? 'Save Changes' : 'Add Customer' }}</VBtn>
      </VCardText>
    </VCard>
  </VDialog>
</template>

<!-- list page: import CustomerDialog from '@/components/pages/admin/customers/CustomerDialog.vue' -->
<CustomerDialog v-model:is-dialog-visible="isCustomerDialogVisible" :customer="editTarget" @saved="() => refresh()" />
```

