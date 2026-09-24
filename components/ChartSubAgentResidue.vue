<template>
  <LineChart
    aria-label="Main context size over a session: a search done in the main context leaves about 40k tokens behind for good; a sub-agent leaves one line"
    :series="series"
    :x-max="20"
    :y-max="80000"
    :x-ticks="[0, 5, 10, 15, 20].map(v => ({ value: v, label: v }))"
    :y-ticks="[0, 20000, 40000, 60000, 80000]"
    :annotations="[{ x: 7, y: 55500, text: '20 files read, ~40k tokens stay', dx: 8, dy: 14 }]"
    x-label="Turn"
    y-label="Main context tokens"
  />
</template>

<script setup>
const base = t => 5000 + t * 1500
const turns = Array.from({ length: 21 }, (_, t) => t)

const series = [
  {
    label: 'Search inline',
    dashed: true,
    points: turns.map(t => [t, base(t) + (t < 5 ? 0 : t >= 7 ? 40000 : (t - 5) * 20000)]),
  },
  { label: 'Sub-agent', accent: true, points: turns.map(t => [t, base(t) + (t >= 7 ? 100 : 0)]) },
]
</script>
