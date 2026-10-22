# Pattern: Statistic / Info Cards

**When to use:** any KPI row, summary tile, or metric grid — not a full dashboard, just the tiles themselves.

**Source (full-version):** `pages/pages/cards/card-statistics.vue`
**Components (already in starter-kit's `@core/components/cards/`):** `CardStatisticsHorizontal`, `CardStatisticsVertical`, `CardStatisticsWithIcon`, `CardStatisticsWithImages` — see `component-catalog.md` for their exact props.

```vue
<script setup lang="ts">
const statisticsWithIcon = [
  { title: 'Session', value: '21,459', change: 29, desc: 'Total Users', icon: 'ri-group-line', iconColor: 'primary' },
  { title: 'Paid Users', value: '4,567', change: 18, desc: 'Last Week', icon: 'ri-user-add-line', iconColor: 'error' },
]
</script>

<template>
  <VRow>
    <VCol
      v-for="(data, i) in statisticsWithIcon"
      :key="i"
      cols="12"
      sm="6"
      md="3"
    >
      <CardStatisticsWithIcon v-bind="data" />
    </VCol>
  </VRow>
</template>
```

**Notes:**
- Pick the variant by layout need: `Horizontal` for a wide single-row tile, `Vertical` for a taller tile with a subtitle, `WithIcon` for the most common icon+number+description tile, `WithImages` when an illustration fits better than an icon.
- Don't hand-roll a stat tile from raw `VCard`/`VIcon` — these four cover the vast majority of cases (layer 2 escalation should rarely be needed here).

**Card headers (project rule):** any card with a title shows a `CardIcon` in
its `#prepend` slot, also when the header is a `VCardItem` with custom
title/subtitle/append content:

```vue
<VCard title="Expenses">
  <template #prepend>
    <CardIcon icon="ri-money-dollar-circle-line" />
  </template>
  <template #append>
    <VBtn variant="text" prepend-icon="ri-add-line">Add Expense</VBtn>
  </template>
  ...
</VCard>
```

**Read-only details inside a card:** tint the block with the project utility
`bg-info-block` (table-header tone, dark-mode aware) on the element itself —
e.g. `<VList density="compact" class="bg-info-block rounded-lg">`. Don't wrap a
`VList` in a tinted `VSheet`: Materio paints `VList` white over it.

**Several actions in a narrow card:** a `VRow dense` grid of `VCol cols="6"`
with `block` buttons (two per row, 50% each), primary action first.
