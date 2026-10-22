# Pattern: Forms & Multi-Step Wizards

**When to use:** a data-entry form too big for a dialog (create/edit page), or a multi-step flow (checkout, onboarding, multi-part application).

First apply the dialog-or-page rule in SKILL.md: a small form (≤ ~4 simple fields) is a dialog
(`patterns/dialogs.md` → "Form dialogs"), not a page.

## A) Form page (create/edit)

The page shell — `PageHeader` with Cancel + Save, Save linked to the form by `form="<id>"`, the form component
in `components/pages/<route>/` used by both `create.vue` and `[id]/edit.vue` — is in
`patterns/page-header.md` → "Form pages". This section shows the body of that form component.

```vue
<!-- components/pages/admin/events/EventForm.vue (body) -->
<script setup lang="ts">
interface Props { id?: number, title: string, subtitle?: string, initial?: Partial<EventInput> }
const props = defineProps<Props>()
const router = useRouter()
const categories = ['Workshop', 'Seminar'] // from the API if it has an endpoint

const refForm = ref()
// EventInput: every field `T | null` — empty is always null (SKILL.md "Empty values")
const form = ref<EventInput>({ name: null, category: null, date: null, start_time: null, price: null, notes: null, ...props.initial })
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
    const res = props.id
      ? await $api<{ data: { id: number } }>(`/admin/events/${props.id}`, { method: 'PUT', body: form.value })
      : await $api<{ data: { id: number } }>('/admin/events', { method: 'POST', body: form.value })
    router.push(`/admin/events/${res.data.id}`) // Nuxt: await navigateTo(...)
  }
  catch (e) {
    const error = e as FetchError<{ message?: string, errors?: Record<string, string[]> }> // import type { FetchError } from 'ofetch'
    if (error.status === 422) // error shape from data-patterns.md Step A
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
  <!-- PageHeader with Cancel + <VBtn type="submit" form="event-form" :loading="isSubmitting"> — see page-header.md -->
  <VAlert v-if="formError" type="error" variant="tonal" class="mb-4">{{ formError }}</VAlert>
  <VCard> <!-- no title: the PageHeader names the page -->
    <VCardText>
      <VForm id="event-form" ref="refForm" @submit.prevent="onSubmit">
        <VRow>
          <VCol cols="12" md="6">
            <VTextField v-model="form.name" label="Name *" maxlength="100" :rules="[requiredValidator]" :error-messages="fieldErrors.name" />
          </VCol>
          <VCol cols="12" md="6">
            <VSelect v-model="form.category" label="Category *" :items="categories" :rules="[requiredValidator]" :error-messages="fieldErrors.category" />
          </VCol>
          <VCol cols="12" md="6">
            <DateTimeField v-model="form.date" label="Date *" :rules="[requiredValidator]" :error-messages="fieldErrors.date" />
          </VCol>
          <VCol cols="12" md="6">
            <DateTimeField v-model="form.start_time" mode="time" label="Start Time" :error-messages="fieldErrors.start_time" />
          </VCol>
          <VCol cols="12" md="6">
            <NumberField v-model="form.price" currency label="Price *" :rules="[requiredValidator]" :error-messages="fieldErrors.price" />
          </VCol>
          <VCol cols="12">
            <VTextarea v-model="form.notes" label="Notes" rows="3" maxlength="500" counter :error-messages="fieldErrors.notes" />
          </VCol>
        </VRow>
      </VForm>
    </VCardText>
  </VCard>
</template>
```

- Long forms: split into several untitled-or-titled cards (titled cards get a `CardIcon`), all inside the one `VForm`.

## B) Multi-step wizard

**Source (full-version):** `pages/wizard-examples/checkout.vue`
**Key component:** `AppStepper` (catalog) paired with `VWindow`/`VWindowItem`.

```vue
<script setup lang="ts">
const steps = [
  { title: 'Cart', icon: 'ri-shopping-cart-line' },
  { title: 'Address', icon: 'ri-map-pin-line' },
  { title: 'Payment', icon: 'ri-bank-card-line' },
]

const currentStep = ref(0)
const stepForms = ref<any[]>([]) // one VForm per step
// isSubmitting + submit(): same shape as onSubmit in section A

const next = async () => {
  const { valid } = await stepForms.value[currentStep.value].validate()
  if (valid)
    currentStep.value++
}
</script>

<template>
  <div>
    <PageHeader title="Checkout" subtitle="Review your order and pay" />

    <VCard>
      <VCardText>
        <AppStepper
          v-model:current-step="currentStep"
          :items="steps"
          :direction="$vuetify.display.mdAndUp ? 'horizontal' : 'vertical'"
        />
      </VCardText>

      <VDivider />

      <VCardText>
        <VWindow v-model="currentStep" class="disable-tab-transition" :touch="false">
          <VWindowItem v-for="(s, i) in steps" :key="s.title">
            <VForm :ref="el => (stepForms[i] = el)">
              <!-- step fields, same field rules as section A -->
            </VForm>
          </VWindowItem>
        </VWindow>

        <div class="d-flex justify-space-between mt-6">
          <VBtn color="secondary" variant="outlined" :disabled="currentStep === 0" @click="currentStep--">Previous</VBtn>
          <VBtn v-if="currentStep < steps.length - 1" @click="next">Next</VBtn>
          <VBtn v-else :loading="isSubmitting" @click="submit">Submit</VBtn>
        </div>
      </VCardText>
    </VCard>
  </div>
</template>
```

**Notes:**
- Validate the current step's `VForm` before moving forward; the final submit validates the last step, then sends everything in one request (`data-patterns.md` → "Forms and mutations").
- Prev/Next/Submit stay at the bottom of the wizard card — the `PageHeader` Save rule is for single-form pages, not wizards.
- For a small wizard, inline each step's markup; extract a step into `components/pages/<route>/` (imported explicitly, never a `views/` folder) only once it gets genuinely large.
- Use `app-form-elements/CustomRadios*` / `CustomCheckboxes*` (catalog) for "card-style" selectable option groups (delivery method, plan choice).
