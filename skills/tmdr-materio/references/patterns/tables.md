# Pattern: Data Tables

**When to use:** any list/index page centered on a table of records (with or without server pagination).

**Source:** the theme's own `pages/tables/data-table.vue` is a documentation gallery importing internal demo-only components — not copy-adaptable. Use the REAL usage instead, extracted from `pages/apps/invoice/list/index.vue` and `pages/apps/user/list/index.vue` (see `patterns/invoice.md` and `patterns/user-role-permission.md`), which already show the full `VDataTableServer` shape used across this template's actual business pages.

**Key component:** `VDataTableServer` (Vuetify built-in — server-side paginated table) or `VDataTable` (client-side, all rows loaded upfront) when the dataset is small and doesn't need server pagination.

```vue
<script setup lang="ts">
import type { Customer } from '@/utils/domain'

const search = ref('')
const status = ref('active')

const headers = [
  { title: 'Name', key: 'name' },
  { title: 'Balance', key: 'balance', align: 'end' as const },
  { title: 'Actions', key: 'actions', sortable: false, width: 0 }, // shrink to the buttons; header left-aligned
]

// Server-side: filters/search/sort are sent 500ms after the last change, paging immediately.
const { page, itemsPerPage, sortBy, items, total, isLoading, refresh } = useServerTable<Customer>(
  '/admin/customers',
  () => ({ q: search.value, status: status.value }),
)
</script>

<template>
  <VCard>
    <VCardText class="d-flex flex-wrap gap-4">
      <VTextField v-model="search" placeholder="Search name" prepend-inner-icon="ri-search-line" density="compact" clearable />
      <VSelect v-model="status" :items="statusOptions" density="compact" />
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
      :no-data-text="search || status !== 'active' ? 'No customers match your filters.' : 'No customers yet.'"
      item-value="id"
    >
      <template #item.actions="{ item }">
        <div class="d-flex flex-nowrap gap-2">
          <ActionIconBtn icon="ri-eye-line" tooltip="View" :to="`/admin/customers/${item.id}`" />
          <ActionIconBtn icon="ri-delete-bin-7-line" tooltip="Delete" @click="openDelete(item)" />
        </div>
      </template>
    </VDataTableServer>
  </VCard>
</template>
```

**Notes:**
- **List pages use `VDataTableServer` + `useServerTable` (`<src>/composables/useServerTable.ts`)**, never a client-side `VDataTable` that loads everything. The composable owns `page`, `itemsPerPage`, `sortBy`, `items`, `total`, `isLoading`, `refresh` — bind `:loading="isLoading"`, which is on from the first keystroke (debounce pending) until the response arrives; pass filters as a getter (`() => ({ q: search.value, status: status.value })`). Search, filters and sort are debounced 500ms; page / page size are immediate; a filter or sort change resets to page 1. Call `refresh()` after a mutation.
- Query param and response names are whatever `useServerTable` was written for (the kit default: `page`, `per_page`, `sort_by`, `sort_order` + filters, Laravel paginator JSON). If the project's API uses other names (data-patterns Step A), adapt the composable once — never per page. When the backend is in this repo: whitelist the sort column server-side and sort/filter computed values in the query, not after fetching.
- Small detail tables that the API returns whole (the lines of one order, one record's history) stay client-side `VDataTable`.
- Row actions use `ActionIconBtn` only (tooltip required), never `IconBtn`/`<VBtn icon>`.
- **Empty state:** every table sets `no-data-text`, naming the resource. It
  says "no matches" when search or filters are active, and "none yet" otherwise
  (Vuetify's default "No data available" is not accepted). Copy is in the
  project's UI language.
- **Image columns:** `<VImg :src="item.image_url ?? 'https://placehold.co/96x96?text=No+Image'" :alt="item.name" width="48" aspect-ratio="1" cover class="rounded" />`.
  People use initials instead (SKILL.md "Avatars").
- The actions column header is `{ key: 'actions', sortable: false, width: 0 }` (default left alignment, no `align: 'end'`) so the column shrinks to its content, and the buttons sit in `<div class="d-flex flex-nowrap gap-2">` so they always stay on one row.
- Controls above the table inside the same card get the card's 20px inset, no `VDivider` before the table. The page title and primary action live in `PageHeader`, not here.
  - Search/filter fields: one `<VCardText class="d-flex flex-wrap gap-4">` row.
  - Status tabs: no padding or margin around `VTabs`. Make the tabs themselves roomier: `<VTabs v-model="tab" height="56">` (a `VTab`'s height comes from `VTabs`) and `class="px-6"` on each `VTab`. The active underline sits directly on the table header.
- For the pagination footer, prefer the catalog's `TablePagination.vue` component instead of hand-rolling prev/next buttons every time.
- Use `VDataTable` (drop `Server` suffix, drop `items-length`) when the PRD implies a small, fully-client-side list with no need for server pagination — don't default to server pagination for a 20-row static list.
