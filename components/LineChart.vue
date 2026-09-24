<template>
  <figure class="line-chart">
    <svg :viewBox="`0 0 ${W} ${H}`" role="img" :aria-label="ariaLabel">
      <!-- y gridlines + tick labels -->
      <g v-for="t in yTicks" :key="`y${t}`">
        <line :x1="M.l" :x2="W - M.r" :y1="y(t)" :y2="y(t)" class="grid" />
        <text :x="M.l - 6" :y="y(t)" class="tick" text-anchor="end" dominant-baseline="middle">
          {{ formatY(t) }}
        </text>
      </g>

      <!-- axes -->
      <line :x1="M.l" :x2="W - M.r" :y1="y(0)" :y2="y(0)" class="axis" />
      <line :x1="M.l" :x2="M.l" :y1="M.t" :y2="y(0)" class="axis" />

      <!-- x tick labels -->
      <text
        v-for="t in xTicks"
        :key="`x${t.value}`"
        :x="x(t.value)"
        :y="H - M.b + 14"
        class="tick"
        text-anchor="middle"
      >
        {{ t.label }}
      </text>

      <!-- axis titles -->
      <text :x="(M.l + W - M.r) / 2" :y="H - 4" class="axis-title" text-anchor="middle">{{ xLabel }}</text>
      <text
        :x="12"
        :y="(M.t + y(0)) / 2"
        class="axis-title"
        text-anchor="middle"
        :transform="`rotate(-90 12 ${(M.t + y(0)) / 2})`"
      >
        {{ yLabel }}
      </text>

      <!-- series -->
      <g v-for="s in series" :key="s.label">
        <polyline
          :points="s.points.map(([px, py]) => `${x(px)},${y(py)}`).join(' ')"
          :class="['series', s.accent ? 'accent-stroke' : 'muted-stroke', { dashed: s.dashed }]"
        />
        <text
          :x="x(last(s)[0]) + 6"
          :y="y(last(s)[1])"
          :class="['series-label', s.accent ? 'accent-fill' : 'muted-fill']"
          dominant-baseline="middle"
        >
          {{ s.label }}
        </text>
      </g>

      <!-- annotations -->
      <g v-for="a in annotations" :key="a.text">
        <circle :cx="x(a.x)" :cy="y(a.y)" r="2.5" class="muted-fill" />
        <text :x="x(a.x) + (a.dx ?? 6)" :y="y(a.y) + (a.dy ?? -6)" class="note" :text-anchor="a.anchor ?? 'start'">
          {{ a.text }}
        </text>
      </g>
    </svg>
    <figcaption>Illustrative — not measured data</figcaption>
  </figure>
</template>

<script setup>
const props = defineProps({
  series: { type: Array, required: true }, // [{ label, points: [[x, y]], accent?, dashed? }]
  xMax: { type: Number, required: true },
  yMax: { type: Number, required: true },
  xTicks: { type: Array, default: () => [] }, // [{ value, label }]
  yTicks: { type: Array, default: () => [] },
  xLabel: { type: String, default: '' },
  yLabel: { type: String, default: '' },
  annotations: { type: Array, default: () => [] },
  ariaLabel: { type: String, default: '' },
})

const W = 440
const H = 210
const M = { l: 48, r: 104, t: 10, b: 34 }

const x = v => M.l + (v / props.xMax) * (W - M.l - M.r)
const y = v => H - M.b - (v / props.yMax) * (H - M.t - M.b)
const last = s => s.points[s.points.length - 1]
const formatY = v => (v >= 1000 ? `${v / 1000}k` : `${v}`)
</script>

<style scoped>
.line-chart {
  margin: 0;
  width: 100%;
}
svg {
  width: 100%;
  height: auto;
  overflow: visible;
  font-family: inherit;
}
.grid {
  stroke: currentColor;
  stroke-opacity: 0.08;
}
.axis {
  stroke: currentColor;
  stroke-opacity: 0.35;
}
.tick {
  font-size: 9px;
  fill: currentColor;
  fill-opacity: 0.55;
}
.axis-title {
  font-size: 9.5px;
  fill: currentColor;
  fill-opacity: 0.7;
}
.series {
  fill: none;
  stroke-width: 2.25;
  stroke-linejoin: round;
  stroke-linecap: round;
}
.dashed {
  stroke-dasharray: 5 4;
}
.accent-stroke {
  stroke: #c2603a;
}
.muted-stroke {
  stroke: currentColor;
  stroke-opacity: 0.45;
}
.series-label {
  font-size: 10px;
  font-weight: 600;
}
.accent-fill {
  fill: #c2603a;
}
.muted-fill {
  fill: currentColor;
  fill-opacity: 0.55;
}
.note {
  font-size: 9px;
  fill: currentColor;
  fill-opacity: 0.7;
}
figcaption {
  font-size: 0.6rem;
  opacity: 0.45;
  text-align: right;
  margin-top: 0.1rem;
}
</style>
