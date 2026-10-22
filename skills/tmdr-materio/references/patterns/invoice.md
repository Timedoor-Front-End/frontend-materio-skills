# Pattern: Invoice / Billing List

**When to use:** any billing/invoice/order list page — summary stats on top, filterable server-side data table below.

**Layout:** `default` (behind login, normal app chrome).

**Source (full-version):** `pages/apps/invoice/list/index.vue`, adapted to the starter-kit rules
(`PageHeader`, `useServerTable`, `ActionIconBtn`, `formatRupiah`).

**Key components:** `PageHeader`, `CardStatisticsHorizontal` (catalog), `VDataTableServer` + `useServerTable`,
`VChip` for status, `ActionIconBtn` for row actions, `ConfirmActionDialog` for delete.

```vue
<script setup lang="ts">
import { formatDate } from '@/utils/format'
import type { Invoice } from '@/utils/domain'

const search = ref('')
const status = ref<string | null>(null)

const headers = [
  { title: '#', key: 'number' },
  { title: 'Client', key: 'client_name' },
  { title: 'Issued', key: 'issued_at' },
  { title: 'Total', key: 'total', align: 'end' as const },
  { title: 'Status', key: 'status' },
  { title: 'Actions', key: 'actions', sortable: false, width: 0 },
]

const { page, itemsPerPage, sortBy, items, total, isLoading, refresh } = useServerTable<Invoice>(
  '/admin/invoices',
  () => ({ q: search.value, status: status.value }),
)

// summary numbers come from the API (Step A) — placeholders here
const stats = [
  { title: 'Clients', stats: '24', icon: 'ri-user-line', color: 'primary', change: 0 },
  { title: 'Invoices', stats: '165', icon: 'ri-pages-line', color: 'info', change: 0 },
]

const statusColor = (s: string) => ({ paid: 'success', sent: 'primary', overdue: 'error' }[s] ?? 'secondary')
</script>

<template>
  <div>
    <PageHeader title="Invoices" subtitle="Bills sent to clients">
      <template #actions>
        <!-- line items = dynamic rows → a page, not a dialog -->
        <VBtn prepend-icon="ri-add-line" to="/admin/invoices/create">Create Invoice</VBtn>
      </template>
    </PageHeader>

    <VRow class="mb-6">
      <VCol v-for="s in stats" :key="s.title" cols="12" sm="6" md="3">
        <CardStatisticsHorizontal v-bind="s" />
      </VCol>
    </VRow>

    <VCard> <!-- table card: no title, no CardIcon -->
      <VCardText class="d-flex flex-wrap gap-4">
        <VTextField v-model="search" placeholder="Search invoice" prepend-inner-icon="ri-search-line" density="compact" clearable />
        <VSelect v-model="status" placeholder="Status" :items="['draft', 'sent', 'paid', 'overdue']" density="compact" clearable />
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
        <template #item.issued_at="{ item }">{{ formatDate(item.issued_at) }}</template>
        <template #item.total="{ item }">{{ formatRupiah(item.total) }}</template>
        <template #item.status="{ item }">
          <VChip :color="statusColor(item.status)" size="small" class="text-capitalize">{{ item.status }}</VChip>
        </template>
        <template #item.actions="{ item }">
          <div class="d-flex flex-nowrap gap-2">
            <ActionIconBtn icon="ri-eye-line" tooltip="View" :to="`/admin/invoices/${item.id}`" />
            <ActionIconBtn icon="ri-delete-bin-7-line" tooltip="Delete" @click="openDelete(item)" />
          </div>
        </template>
      </VDataTableServer>
    </VCard>

    <!-- delete: ConfirmActionDialog as in patterns/dialogs.md, then refresh() -->
  </div>
</template>
```

**Notes:**
- Field names (`number`, `client_name`, `issued_at`, `total`, `status`) and status values are placeholders — take the real ones from the API (`data-patterns.md` Step A).
- `total` is money in sen → always `formatRupiah`, never divide in the page.
- `list` / `create` / `[id]` / `[id]/edit` are separate routes — generate only the sub-page(s) the PRD actually asks for. The create/edit form (client, dates, line items) is a page per `page-header.md` → "Form pages", with `DateTimeField` for dates and `NumberField currency` for amounts.
