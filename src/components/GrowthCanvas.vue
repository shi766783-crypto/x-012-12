<script setup>
import { onBeforeUnmount, onMounted, ref, watch } from 'vue'

const METRICS = [
  { key: 'height', label: '高度', unit: 'cm' },
  { key: 'leafCount', label: '叶片数', unit: '片' },
]

const props = defineProps({
  records: {
    type: Array,
    required: true,
  },
})

const canvasRef = ref(null)
const activeMetric = ref('height')

let frameId = 0

onMounted(() => {
  scheduleDraw()
  window.addEventListener('resize', scheduleDraw)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', scheduleDraw)
  cancelAnimationFrame(frameId)
})

watch(() => props.records, scheduleDraw, { deep: true })
watch(activeMetric, scheduleDraw)

function scheduleDraw() {
  cancelAnimationFrame(frameId)
  frameId = requestAnimationFrame(draw)
}

function parseDate(iso) {
  if (!iso || Number.isNaN(Date.parse(`${iso}T00:00:00`))) return null
  return new Date(`${iso}T00:00:00`).getTime()
}

function formatNumber(value) {
  return Number.isInteger(value) ? String(value) : value.toFixed(1)
}

function draw() {
  const canvas = canvasRef.value
  if (!canvas) return

  const ratio = window.devicePixelRatio || 1
  const width = canvas.clientWidth
  const height = canvas.clientHeight
  if (!width || !height) return

  canvas.width = Math.round(width * ratio)
  canvas.height = Math.round(height * ratio)

  const ctx = canvas.getContext('2d')
  ctx.setTransform(ratio, 0, 0, ratio, 0, 0)
  ctx.clearRect(0, 0, width, height)

  const padding = { top: 22, right: 24, bottom: 34, left: 44 }
  const plot = {
    x: padding.left,
    y: padding.top,
    width: width - padding.left - padding.right,
    height: height - padding.top - padding.bottom,
  }

  ctx.fillStyle = '#fbf8ef'
  ctx.fillRect(0, 0, width, height)

  if (!props.records.length) {
    drawGrid(ctx, plot, width, height)
    drawEmpty(ctx, width, height, null, false)
    return
  }

  const metric = METRICS.find((item) => item.key === activeMetric.value) || METRICS[0]

  // 日期合法的记录都参与横坐标定位；缺测的指标在该点断开折线
  const dated = props.records
    .map((record) => {
      const raw = record[metric.key]
      const missing = raw === null || raw === undefined || raw === ''
      const value = Number(raw)
      return {
        time: parseDate(record.date),
        value: !missing && Number.isFinite(value) ? value : null,
        record,
      }
    })
    .filter((entry) => entry.time !== null)
    .sort((a, b) => a.time - b.time)

  const entries = dated.filter((entry) => entry.value !== null)

  drawGrid(ctx, plot, width, height)

  if (!dated.length || !entries.length) {
    drawEmpty(ctx, width, height, metric, dated.length)
    return
  }

  // 横坐标范围按全部录入日期计算，缺测记录造成的时间间隔同样体现在曲线上
  const minTime = dated[0].time
  const maxTime = dated[dated.length - 1].time
  const values = entries.map((entry) => entry.value)
  const rawMin = Math.min(...values)
  const rawMax = Math.max(...values)
  let yMin = rawMin
  let yMax = rawMax
  if (yMin === yMax) {
    // 所有数值相同：上下各留出一定余量，避免点贴在边框上
    const pad = Math.max(Math.abs(yMin) * 0.1, 1)
    yMin -= pad
    yMax += pad
  } else {
    const pad = (yMax - yMin) * 0.12
    yMin -= pad
    yMax += pad
  }
  // 高度、叶片数均为非负指标，下边界不低于 0
  if (yMin < 0 && rawMin >= 0) yMin = 0
  const ySpan = Math.max(yMax - yMin, 1e-9)

  const xOf = (time) => {
    if (maxTime === minTime) return plot.x + plot.width / 2
    return plot.x + ((time - minTime) / (maxTime - minTime)) * plot.width
  }
  const yOf = (value) => plot.y + plot.height - ((value - yMin) / ySpan) * plot.height

  drawYTicks(ctx, plot, yMin, yMax)
  drawXTicks(ctx, plot, minTime, maxTime, width, height)

  // 折线：遇到缺测记录断开，不跨缺口连线；同日多条会在同一横坐标形成竖直线段
  ctx.strokeStyle = '#2f7d4f'
  ctx.lineWidth = 2
  ctx.lineJoin = 'round'
  ctx.beginPath()
  let penDown = false
  dated.forEach((entry) => {
    if (entry.value === null) {
      penDown = false
      return
    }
    const x = xOf(entry.time)
    const y = yOf(entry.value)
    if (!penDown) {
      ctx.moveTo(x, y)
      penDown = true
    } else {
      ctx.lineTo(x, y)
    }
  })
  ctx.stroke()

  // 数据点与数值标签
  const sameDateOffset = new Map()
  entries.forEach((entry) => {
    const x = xOf(entry.time)
    const y = yOf(entry.value)

    ctx.fillStyle = '#f5b947'
    ctx.strokeStyle = '#2f7d4f'
    ctx.lineWidth = 1.5
    ctx.beginPath()
    ctx.arc(x, y, 4, 0, Math.PI * 2)
    ctx.fill()
    ctx.stroke()

    const label = `${formatNumber(entry.value)}${metric.unit}`
    ctx.font = '11px system-ui'
    const labelWidth = ctx.measureText(label).width

    // 同一天录入多条时，标签上下交替，避免互相覆盖
    const occurrence = sameDateOffset.get(entry.time) ?? 0
    sameDateOffset.set(entry.time, occurrence + 1)
    let labelY = y - 9
    if (occurrence % 2 === 1 || labelY - 11 < plot.y) labelY = y + 15

    let labelX = x - labelWidth / 2
    labelX = Math.min(Math.max(labelX, 4), width - labelWidth - 4)
    labelY = Math.min(Math.max(labelY, plot.y + 4), height - padding.bottom - 6)

    ctx.fillStyle = '#314238'
    ctx.fillText(label, labelX, labelY)
  })
}

function drawEmpty(ctx, width, height, metric, hasDated) {
  ctx.fillStyle = '#667568'
  ctx.font = '14px system-ui'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  const message = !metric
    ? '暂无生长指标'
    : hasDated
      ? `暂无有效的${metric.label}数据`
      : '存在生长记录，但日期无效'
  ctx.fillText(message, width / 2, height / 2)
  ctx.textAlign = 'start'
  ctx.textBaseline = 'alphabetic'
}

function drawGrid(ctx, plot, width, height) {
  ctx.strokeStyle = '#e2ead9'
  ctx.lineWidth = 1
  for (let i = 0; i <= 4; i += 1) {
    const y = plot.y + (plot.height / 4) * i
    ctx.beginPath()
    ctx.moveTo(plot.x, y)
    ctx.lineTo(width - 24, y)
    ctx.stroke()
  }
}

function niceStep(rawStep) {
  const magnitude = 10 ** Math.floor(Math.log10(rawStep))
  const normalized = rawStep / magnitude
  let nice
  if (normalized <= 1) nice = 1
  else if (normalized <= 2) nice = 2
  else if (normalized <= 5) nice = 5
  else nice = 10
  return nice * magnitude
}

function drawYTicks(ctx, plot, yMin, yMax) {
  const step = niceStep((yMax - yMin) / 4)
  const first = Math.ceil(yMin / step) * step

  ctx.fillStyle = '#7a856f'
  ctx.font = '10px system-ui'
  ctx.textAlign = 'end'
  ctx.textBaseline = 'middle'
  for (let value = first; value <= yMax + step * 1e-6; value += step) {
    const y = plot.y + plot.height - ((value - yMin) / (yMax - yMin)) * plot.height
    if (y < plot.y - 1 || y > plot.y + plot.height + 1) continue
    ctx.fillText(formatNumber(Number(value.toFixed(6))), plot.x - 6, y)
  }
  ctx.textAlign = 'start'
  ctx.textBaseline = 'alphabetic'
}

function formatDateTick(time, daySpan) {
  const date = new Date(time)
  const month = date.getMonth() + 1
  const day = date.getDate()
  if (daySpan > 365) return `${date.getFullYear()}/${month}`
  return `${month}/${day}`
}

function drawXTicks(ctx, plot, minTime, maxTime, width, height) {
  const DAY = 24 * 60 * 60 * 1000
  const daySpan = (maxTime - minTime) / DAY
  const tickCount = Math.min(5, Math.max(2, Math.round(plot.width / 96) + 1))

  const ticks = []
  if (maxTime === minTime) {
    ticks.push(minTime)
  } else {
    for (let i = 0; i < tickCount; i += 1) {
      ticks.push(minTime + ((maxTime - minTime) * i) / (tickCount - 1))
    }
  }

  ctx.strokeStyle = '#e2ead9'
  ctx.fillStyle = '#7a856f'
  ctx.font = '10px system-ui'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'top'

  ticks.forEach((time) => {
    const x = maxTime === minTime
      ? plot.x + plot.width / 2
      : plot.x + ((time - minTime) / (maxTime - minTime)) * plot.width

    ctx.beginPath()
    ctx.moveTo(x, plot.y)
    ctx.lineTo(x, plot.y + plot.height)
    ctx.stroke()

    const label = formatDateTick(time, daySpan)
    const labelWidth = ctx.measureText(label).width
    const labelX = Math.min(Math.max(x, plot.x + labelWidth / 2), width - 24 - labelWidth / 2)
    ctx.fillText(label, labelX, height - 22)
  })

  ctx.textAlign = 'start'
  ctx.textBaseline = 'alphabetic'
}
</script>

<template>
  <div class="growth-chart">
    <div class="metric-switch" role="tablist" aria-label="切换生长指标">
      <button
        v-for="metric in METRICS"
        :key="metric.key"
        :class="{ active: activeMetric === metric.key }"
        type="button"
        @click="activeMetric = metric.key"
      >
        {{ metric.label }}
      </button>
    </div>
    <canvas ref="canvasRef" class="growth-canvas" aria-label="植物生长曲线"></canvas>
  </div>
</template>

<style scoped>
.growth-chart {
  display: grid;
  gap: 8px;
}

.metric-switch {
  display: inline-flex;
  gap: 4px;
  justify-self: end;
  padding: 3px;
  border: 1px solid #d8e2cf;
  border-radius: 999px;
  background: #f3f0e4;
}

.metric-switch button {
  min-height: 26px;
  padding: 2px 14px;
  color: #5d6b5e;
  border: 0;
  border-radius: 999px;
  background: transparent;
  font-size: 0.82rem;
}

.metric-switch button.active {
  color: #f8fbef;
  background: #2f7d4f;
  font-weight: 700;
}
</style>
