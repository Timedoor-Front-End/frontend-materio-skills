# Pattern: User List / Roles / Permissions

**When to use:** user management list, role management, permission management pages.

**Layout:** `default`.

**Source (full-version):** `pages/apps/user/list/index.vue`, `pages/apps/roles/index.vue`, `pages/apps/permissions/index.vue`,
adapted to the starter-kit rules.

## A) User list (filterable table + add/edit user)

Same table shape as `patterns/tables.md`, with an initials avatar in the user column and role/status filters.
Add/Edit user: a form dialog when it passes the dialog rule in SKILL.md (e.g. name, email, role, status);
a page when it doesn't (password rules, profile sections, uploads).

```vue
<script setup lang="ts">
import UserDialog from '@/components/pages/admin/users/UserDialog.vue'
import type { User } from '@/utils/domain'

const search = ref('')
const role = ref<string | null>(null)
const status = ref<string | null>(null)

const headers = [
  { title: 'User', key: 'name' },
  { title: 'Role', key: 'role' },
  { title: 'Status', key: 'status' },
  { title: 'Actions', key: 'actions', sortable: false, width: 0 },
]

const { page, itemsPerPage, sortBy, items, total, isLoading, refresh } = useServerTable<User>(
  '/admin/users',
  () => ({ q: search.value, role: role.value, status: status.value }),
)

const roleOptions = [{ title: 'Admin', value: 'admin' }, { title: 'Editor', value: 'editor' }] // from the API if it has an endpoint
const statusColor = (s: string) => ({ active: 'success', pending: 'warning' }[s] ?? 'secondary')

const isUserDialogVisible = ref(false)
const editTarget = ref<User | null>(null)
const openUserDialog = (u: User | null) => { editTarget.value = u; isUserDialogVisible.value = true }
</script>

<template>
  <div>
    <PageHeader title="Users" subtitle="Everyone who can sign in">
      <template #actions>
        <VBtn prepend-icon="ri-add-line" @click="openUserDialog(null)">Add User</VBtn>
      </template>
    </PageHeader>

    <VCard> <!-- filters live in the table card: no title, no CardIcon -->
      <VCardText class="d-flex flex-wrap gap-4">
        <VTextField v-model="search" placeholder="Search name or email" prepend-inner-icon="ri-search-line" density="compact" clearable />
        <VSelect v-model="role" placeholder="Role" :items="roleOptions" density="compact" clearable />
        <VSelect v-model="status" placeholder="Status" :items="['active', 'pending', 'inactive']" density="compact" clearable />
      </VCardText>

      <VDataTableServer
        v-model:page="page"
        v-model:items-per-page="itemsPerPage"
        v-model:sort-by="sortBy"
        :headers="headers"
        :items="items"
        :items-length="total"
        :items-per-page-options="[10, 25, 50, 100]"
        :loading="isLoading"
        item-value="id"
      >
        <template #item.name="{ item }">
          <div class="d-flex align-center gap-3">
            <VAvatar color="primary" variant="tonal" size="34">{{ initials(item.name) }}</VAvatar>
            <div class="d-flex flex-column">
              <span class="text-high-emphasis">{{ item.name }}</span>
              <span class="text-body-2">{{ item.email }}</span>
            </div>
          </div>
        </template>
        <template #item.status="{ item }">
          <VChip :color="statusColor(item.status)" size="small" class="text-capitalize">{{ item.status }}</VChip>
        </template>
        <template #item.actions="{ item }">
          <div class="d-flex flex-nowrap gap-2">
            <ActionIconBtn icon="ri-edit-box-line" tooltip="Edit" @click="openUserDialog(item)" />
            <ActionIconBtn icon="ri-eye-line" tooltip="View" :to="`/admin/users/${item.id}`" />
          </div>
        </template>
      </VDataTableServer>
    </VCard>

    <UserDialog v-model:is-dialog-visible="isUserDialogVisible" :user="editTarget" @saved="() => refresh()" />
  </div>
</template>
```

`UserDialog` follows `patterns/dialogs.md` → "Form dialogs".

## B) Roles page (card grid, not a table)

```vue
<script setup lang="ts">
const { data } = useApi<{ data: { id: number, name: string, users_count: number }[] }>('/admin/roles') // shape from Step A
</script>

<template>
  <div>
    <PageHeader title="Roles" subtitle="Manage roles and their permissions" />

    <VRow>
      <VCol v-for="r in data?.data ?? []" :key="r.id" cols="12" sm="6" md="4">
        <VCard :title="r.name" :subtitle="`${r.users_count} users`">
          <template #prepend>
            <CardIcon icon="ri-shield-user-line" />
          </template>
          <template #append>
            <ActionIconBtn icon="ri-edit-box-line" tooltip="Edit role" :to="`/admin/roles/${r.id}`" />
          </template>
        </VCard>
      </VCol>
    </VRow>
  </div>
</template>
```

**Notes:**
- Field names, roles and status values are placeholders — take them from the API (`data-patterns.md` Step A).
- Editing a role's permissions (a checkbox matrix) is too big for a dialog → `roles/[id].vue` page with Save in the `PageHeader` (`page-header.md` → "Form pages").
- A Permission management page reuses the table shape of (A); add/edit a permission with a form dialog. Materio's demo `AddEditRoleDialog` / `AddEditPermissionDialog` / `AddNewUserDrawer` are not in the starter kit — never import them.
- Don't hardcode role checks in pages; see `data-patterns.md` → "Auth and permissions".
