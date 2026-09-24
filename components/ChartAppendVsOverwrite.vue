<template>
  <LineChart
    aria-label="Tokens loaded at session start: an appended log grows every session, an overwritten state document stays flat"
    :series="series"
    :x-max="10"
    :y-max="7000"
    :x-ticks="[1, 4, 7, 10].map(v => ({ value: v, label: v }))"
    :y-ticks="[0, 2000, 4000, 6000]"
    x-label="Session"
    y-label="Tokens loaded at start"
  />
</template>

<script setup>
const sessions = Array.from({ length: 10 }, (_, i) => i + 1)

const series = [
  { label: 'Append', dashed: true, points: sessions.map(s => [s, s * 600]) },
  { label: 'Overwrite', accent: true, points: sessions.map(s => [s, 600 + (s % 3) * 40]) },
]
</script>
