<template>
  <LineChart
    aria-label="Context loaded over a task: loading everything upfront starts high and stays high; phase-based loading steps up only as each phase needs more"
    :series="series"
    :x-max="15"
    :y-max="70000"
    :x-ticks="[
      { value: 1.5, label: 'Orientation' },
      { value: 5.5, label: 'Task' },
      { value: 11.5, label: 'Implementation' },
    ]"
    :y-ticks="[0, 20000, 40000, 60000]"
    x-label="Phase"
    y-label="Context loaded (tokens)"
  />
</template>

<script setup>
const series = [
  { label: 'Load everything', dashed: true, points: [[0, 60000], [15, 60000]] },
  {
    label: 'Phase-based',
    accent: true,
    points: [[0, 3000], [3, 3000], [3, 12000], [8, 12000], [8, 22000], [15, 22000]],
  },
]
</script>
