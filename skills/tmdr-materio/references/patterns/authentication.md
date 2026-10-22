# Pattern: Authentication (login / register / forgot password)

**When to use:** any login, register, forgot-password, reset-password, verify-email, or 2FA page.

**Layout:** `blank` — no nav chrome, and no `PageHeader` (auth pages keep their own heading).
Vue/Laravel: `definePage({ meta: { layout: 'blank', public: true } })`; Nuxt: `definePageMeta({ layout: 'blank', public: true })`.

**Source (full-version, for deeper reference only, not present in starter-kit):**
`pages/pages/authentication/login-v1.vue`, `register-v1.vue`, `forgot-password-v1.vue`

**Shape:** centered `VCard` (max-width ~448px) on a plain background, app name + title, a `VForm`, a link to the
sibling auth page (login ↔ register ↔ forgot-password), no sidebar/topbar.

What happens on submit (where the token or session goes, what to do after login) depends on the project's auth
mode — read `references/auth-modes.md` and follow only that mode.

```vue
<script setup lang="ts">
definePage({ meta: { layout: 'blank', public: true } })

const refForm = ref()
const form = ref<{ email: string | null, password: string | null, remember: boolean }>({ email: null, password: null, remember: false })
const isPasswordVisible = ref(false)
const isSubmitting = ref(false)
const formError = ref('')

const onSubmit = async () => {
  const { valid } = await refForm.value!.validate()
  if (!valid)
    return
  isSubmitting.value = true
  formError.value = ''
  try {
    await $api('/auth/login', { method: 'POST', body: form.value }) // endpoint from Step A
    // then: per auth-modes.md (store token / fetch current user) and navigate
  }
  catch (e) {
    formError.value = apiErrorMessage(e) // server's message; don't reveal which field was wrong
  }
  finally {
    isSubmitting.value = false
  }
}
</script>

<template>
  <div class="auth-wrapper d-flex align-center justify-center pa-4">
    <VCard class="auth-card pa-sm-4 pa-md-7 pa-0" max-width="448">
      <VCardText>
        <h1 class="text-h4 mb-1">Welcome! 👋🏻</h1> <!-- the page's h1, sized by class -->
        <p class="mb-0">Please sign in to your account</p>
      </VCardText>

      <VCardText>
        <VAlert v-if="formError" type="error" variant="tonal" class="mb-4">{{ formError }}</VAlert>
        <VForm ref="refForm" autocomplete="off" @submit.prevent="onSubmit">
          <VRow>
            <VCol cols="12">
              <VTextField v-model="form.email" autofocus label="Email *" type="email" maxlength="254" :rules="[requiredValidator, emailValidator]" />
            </VCol>

            <VCol cols="12">
              <VTextField
                v-model="form.password"
                label="Password *"
                maxlength="72"
                :rules="[requiredValidator]"
                :type="isPasswordVisible ? 'text' : 'password'"
                :append-inner-icon="isPasswordVisible ? 'ri-eye-off-line' : 'ri-eye-line'"
                @click:append-inner="isPasswordVisible = !isPasswordVisible"
              />

              <div class="d-flex align-center flex-wrap justify-space-between my-5 gap-4">
                <VCheckbox v-model="form.remember" label="Remember me" />
                <RouterLink class="text-primary" :to="{ name: 'forgot-password' }">Forgot Password?</RouterLink>
              </div>

              <VBtn block type="submit" :loading="isSubmitting">Login</VBtn>
            </VCol>

            <VCol cols="12" class="text-center text-base">
              <span>New on our platform?</span>
              <RouterLink class="text-primary d-inline-block" :to="{ name: 'register' }">Create an account</RouterLink>
            </VCol>
          </VRow>
        </VForm>
      </VCardText>
    </VCard>
  </div>
</template>
```

**Notes:**
- Route names (`{ name: 'register' }`) must match this project's actual generated routes (file-based routing) — check `<src>/pages/` before wiring links. Nuxt: `NuxtLink`.
- Forgot-password always shows the same generic success message (`auth-modes.md` → "All modes").
- Password fields: `type="password"` + eye toggle as above. Login's `VForm` has `autocomplete="off"`
  (the checklist asks for it on sensitive forms; a browser's password manager may still offer saved logins,
  which is accepted). Set/reset password forms put `autocomplete="new-password"` on both password fields.
  OTP input: `autocomplete="one-time-code"`, `inputmode="numeric"`, `maxlength` = the code length.

## Countdown (OTP and reset links)

Pages that wait on something sent to the user show how long it stays valid, and
when it can be resent: the OTP verify page, and the "check your email" step after
forgot-password. The seconds come from the API response (`expires_in`,
`resend_in`). Never hardcode them in the page, because the backend owns the
timing and also throttles resends. The countdown is computed from a deadline, not
by decrementing every tick, because background tabs throttle timers.

```ts
const useSecondsLeft = () => {
  const left = ref(0)
  const deadline = ref(0)
  const { pause, resume } = useIntervalFn(() => {
    left.value = Math.max(0, Math.ceil((deadline.value - Date.now()) / 1000))
    if (!left.value)
      pause()
  }, 500, { immediate: false })

  return { left, start: (seconds: number) => { deadline.value = Date.now() + seconds * 1000; left.value = seconds; resume() } }
}

const mmss = (s: number) => `${Math.floor(s / 60)}:${String(s % 60).padStart(2, '0')}`

const expiry = useSecondsLeft()
const resendCooldown = useSecondsLeft()
// after a successful send: expiry.start(res.expires_in); resendCooldown.start(res.resend_in)
```

```vue
<p class="text-body-2">
  {{ expiry.left.value ? `The code expires in ${mmss(expiry.left.value)}` : 'The code has expired. Request a new one.' }}
</p>
<VBtn variant="text" :disabled="resendCooldown.left.value > 0" :loading="isResending" @click="resend">
  {{ resendCooldown.left.value ? `Resend in ${mmss(resendCooldown.left.value)}` : 'Resend code' }}
</VBtn>
```

When the code has expired, the Verify button is disabled. Page copy is in the
project's UI language.

## reCAPTCHA (public forms)

Register, forgot password, contact, guest booking: any form a visitor can submit
without logging in. Needs checklist S9 (`utils/recaptcha.ts`, the backend rule
and the env keys). v3 is invisible, so the form doesn't change:

```ts
await $api('/auth/forgot-password', { method: 'POST', body: { ...form.value, recaptcha_token: await recaptchaToken('forgot_password') } })
```

A failed check comes back as a 422 on `recaptcha_token`, which `apiErrorMessage()`
shows in the form's `VAlert`.
- Decorative tree/mask images (`@images/misc/...`) in the original are cosmetic only — safe to drop; don't fabricate image assets that don't exist in this project.
- `AuthProvider` (social login buttons) is a full-version-only view component, not present in starter-kit — omit unless the PRD explicitly asks for social login, in which case flag it as a new component per the escalation rule.
