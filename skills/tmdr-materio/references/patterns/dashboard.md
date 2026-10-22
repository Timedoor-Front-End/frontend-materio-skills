# Pattern: Dashboard

**When to use:** any overview/summary/analytics landing page — grid of stat cards + charts + a table.

**Layout:** `default`.

**Source (full-version):** `pages/dashboards/analytics.vue`, `crm.vue`, `ecommerce.vue`

**Shape:** `PageHeader`, then a `VRow class="match-height"` grid of `VCol`s, each holding either a stat-card
component from the catalog (`CardStatisticsVertical`, `CardStatisticsWithIcon`, …) or a titled chart/table card.
Column widths vary (`md="4"`, `md="8"`) to create visual hierarchy — bigger cols for the "headline" widget.

```vue
<script setup lang="ts">
// numbers come from the API (data-patterns.md Step A) — placeholders here
const totalSales = { title: 'Total Sales', color: 'primary', icon: 'ri-money-dollar-circle-line', stats: formatRupiah(2560000000), change: 42, subtitle: 'This month' }
const newCustomers = { title: 'New Customers', color: 'success', icon: 'ri-user-add-line', stats: '862', change: -18, subtitle: 'This month' }
</script>

<template>
  <div>
    <PageHeader title="Dashboard" subtitle="How the business is doing this month" />

    <VRow class="match-height">
      <VCol cols="12" sm="6" md="4">
        <CardStatisticsVertical v-bind="totalSales" />
      </VCol>

      <VCol cols="12" sm="6" md="4">
        <CardStatisticsVertical v-bind="newCustomers" />
      </VCol>

      <VCol cols="12" md="8">
        <VCard title="Sales Overview">
          <template #prepend>
            <CardIcon icon="ri-line-chart-line" />
          </template>
          <VCardText>
            <!-- chart / table content -->
          </VCardText>
        </VCard>
      </VCol>
    </VRow>
  </div>
</template>
```

**Notes:**
- Full-version's dashboards import bespoke chart components (`AnalyticsAward`, `AnalyticsWeeklyOverview`, …) from `@/views/dashboards/...` — these are NOT in the starter kit. For a chart, use the project's existing chart lib (`@core/libs/apex-chart` or `@core/libs/chartjs` — check it exists first); on Nuxt, wrap the chart in `<ClientOnly>`.
- Several independent widgets fetching on load → fetch in parallel (`data-patterns.md`).
- Money values go through `formatRupiah`; a recent-records table inside a card is a client-side `VDataTable` of what the API returns.
