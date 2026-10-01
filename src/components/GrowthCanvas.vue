<script setup>
import { onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { BLOOM_STATES } from '../constants/garden'

const props = defineProps({
  records: {
    type: Array,
    required: true,
  },
})

const METRICS = [
  { key: 'height', label: '高度', unit: 'cm', type: 'numeric', fallbackMax: 10 },
  { key: 'leafCount', label: '叶片数', unit: '片', type: 'numeric', integer: true, fallbackMax: 10 },
  { key: 'bloomState', label: '开花状态', type: 'category', categories: BLOOM_STATES },
]

const DAY_MS = 24 * 60 * 60 * 1000
const canvasRef = ref(null)
const selectedMetric = ref('height')
let resizeObserver = null

onMounted(() => {
  resizeObserver = new ResizeObserver(draw)
  resizeObserver.observe(canvasRef.value)
  draw()
})

onBeforeUnmount(() => {
  resizeObserver?.disconnect()
})

watch(
  [() => props.records, selectedMetric],
  draw,
  { deep: true },
)

function draw() {
  const canvas = canvasRef.value
  if (!canvas) return

  const width = canvas.clientWidth
  const height = canvas.clientHeight
  if (width <= 0 || height <= 0) return

  const ratio = window.devicePixelRatio || 1
  canvas.width = Math.round(width * ratio)
  canvas.height = Math.round(height * ratio)

  const ctx = canvas.getContext('2d')
  ctx.setTransform(ratio, 0, 0, ratio, 0, 0)
  ctx.clearRect(0, 0, width, height)

  drawBackground(ctx, width, height)

  const datedRecords = props.records
    .map((record) => ({ record, date: parseRecordDate(record.date) }))
    .filter((item) => item.date !== null)

  if (!datedRecords.length) {
    drawMessage(ctx, width, height, '暂无可按日期绘制的生长记录')
    return
  }

  const config = METRICS.find((metric) => metric.key === selectedMetric.value) || METRICS[0]
  const points = datedRecords
    .map((item) => ({ ...item, value: getMetricValue(item.record, config) }))
    .filter((item) => item.value !== null)
    .sort((a, b) => a.date.dayIndex - b.date.dayIndex)
  const chartPoints = groupPointsByDate(points, config)

  const layout = {
    top: 24,
    right: 18,
    bottom: 38,
    left: 56,
    width,
    height,
  }
  const plot = getPlotLayout(layout)
  const xSource = chartPoints.length ? chartPoints : datedRecords
  const minDay = Math.min(...xSource.map((item) => item.date.dayIndex))
  const maxDay = Math.max(...xSource.map((item) => item.date.dayIndex))
  const xAxis = buildXAxis(minDay, maxDay, plot.width)
  const yAxis = config.type === 'category' ? buildCategoryAxis(config) : buildNumericAxis(chartPoints, config)

  drawGrid(ctx, plot, xAxis, yAxis, config, minDay, maxDay)
  drawRecordCount(ctx, chartPoints.length, points.length, props.records.length, width)

  if (!chartPoints.length) {
    drawMessage(ctx, width, height, '当前指标暂无可绘制的有效记录')
    return
  }

  const xFor = (dayIndex) => getXPosition(dayIndex, minDay, maxDay, plot)
  const yFor = (value) => getYPosition(value, yAxis, plot)
  drawLine(ctx, chartPoints, xFor, yFor, plot)
  drawPoints(ctx, chartPoints, config, xFor, yFor, plot)
}

function parseRecordDate(dateISO) {
  if (typeof dateISO !== 'string') return null
  const match = /^(\d{4})-(\d{2})-(\d{2})$/.exec(dateISO)
  if (!match) return null

  const year = Number(match[1])
  const month = Number(match[2])
  const day = Number(match[3])
  const date = new Date(year, month - 1, day)

  if (
    date.getFullYear() !== year
    || date.getMonth() !== month - 1
    || date.getDate() !== day
  ) {
    return null
  }

  return {
    dayIndex: Math.floor(Date.UTC(year, month - 1, day) / DAY_MS),
  }
}

function getMetricValue(record, config) {
  if (config.type === 'category') {
    const index = config.categories.indexOf(record[config.key])
    return index === -1 ? null : index
  }

  const raw = record[config.key]
  if (raw === null || raw === undefined || String(raw).trim() === '') return null
  const value = Number(raw)
  return Number.isFinite(value) && value >= 0 ? value : null
}

function groupPointsByDate(points, config) {
  const groups = new Map()

  points.forEach((point) => {
    if (!groups.has(point.date.dayIndex)) groups.set(point.date.dayIndex, [])
    groups.get(point.date.dayIndex).push(point)
  })

  return [...groups.entries()].map(([dayIndex, items]) => {
    const first = items[0]
    const value = config.type === 'category'
      ? items.at(-1).value
      : items.reduce((sum, item) => sum + item.value, 0) / items.length

    return {
      ...first,
      date: { ...first.date, dayIndex },
      value,
      records: items.map((item) => item.record),
    }
  })
}

function getPlotLayout(layout) {
  const left = layout.width < 320 ? 48 : layout.left
  const right = Math.max(left + 24, layout.width - layout.right)
  const bottom = Math.max(layout.top + 48, layout.height - layout.bottom)
  return {
    left,
    right,
    top: layout.top,
    bottom,
    width: Math.max(1, right - left),
    height: Math.max(1, bottom - layout.top),
  }
}

function buildXAxis(minDay, maxDay, plotWidth) {
  if (minDay === maxDay) {
    return [{ dayIndex: minDay, label: formatDayIndex(minDay, minDay) }]
  }

  const maxLabels = Math.max(2, Math.floor(plotWidth / 72) + 1)
  const interval = chooseDayInterval((maxDay - minDay) / (maxLabels - 1))
  const ticks = []

  for (let dayIndex = minDay; dayIndex <= maxDay; dayIndex += interval) {
    ticks.push({ dayIndex, label: formatDayIndex(dayIndex, minDay) })
  }

  const lastTick = ticks[ticks.length - 1]
  if (maxDay - lastTick.dayIndex >= Math.ceil(interval * 0.7)) {
    ticks.push({ dayIndex: maxDay, label: formatDayIndex(maxDay, minDay) })
  }

  return ticks
}

function chooseDayInterval(rawInterval) {
  const intervals = [1, 2, 7, 14, 30, 60, 90, 180, 365]
  return intervals.find((interval) => interval >= rawInterval) || 365
}

function formatDayIndex(dayIndex, firstDayIndex) {
  const date = new Date(dayIndex * DAY_MS)
  const year = date.getUTCFullYear()
  const month = date.getUTCMonth() + 1
  const day = date.getUTCDate()
  const firstDate = new Date(firstDayIndex * DAY_MS)

  return year === firstDate.getUTCFullYear()
    ? `${month}/${day}`
    : `${year}/${month}/${day}`
}

function buildNumericAxis(points, config) {
  const values = points.map((point) => point.value)
  const maxValue = Math.max(config.fallbackMax, ...values, 0)
  const step = niceStep((maxValue * 1.08) / 4, config.integer)
  const tickCount = Math.max(1, Math.ceil(maxValue / step))
  const max = tickCount * step

  return {
    min: 0,
    max,
    ticks: Array.from({ length: tickCount + 1 }, (_, index) => index * step),
  }
}

function niceStep(rawStep, integer = false) {
  if (rawStep <= 0) return integer ? 1 : 1
  if (integer && rawStep < 1) return 1

  const power = 10 ** Math.floor(Math.log10(rawStep))
  const normalized = rawStep / power
  const multiplier = normalized <= 1 ? 1 : normalized <= 2 ? 2 : normalized <= 5 ? 5 : 10
  const step = multiplier * power

  return integer ? Math.max(1, Math.round(step)) : step
}

function buildCategoryAxis(config) {
  return {
    min: 0,
    max: config.categories.length - 1,
    ticks: config.categories.map((label, index) => ({ value: index, label })),
  }
}

function getXPosition(dayIndex, minDay, maxDay, plot) {
  if (minDay === maxDay) return plot.left + plot.width / 2
  return plot.left + ((dayIndex - minDay) / (maxDay - minDay)) * plot.width
}

function getYPosition(value, axis, plot) {
  if (axis.max === axis.min) return plot.top + plot.height / 2
  return plot.top + (1 - (value - axis.min) / (axis.max - axis.min)) * plot.height
}

function drawBackground(ctx, width, height) {
  ctx.fillStyle = '#fbf8ef'
  ctx.fillRect(0, 0, width, height)
}

function drawGrid(ctx, plot, xAxis, yAxis, config, minDay, maxDay) {
  ctx.font = '12px system-ui'
  ctx.textBaseline = 'middle'

  yAxis.ticks.forEach((tick) => {
    const rawValue = typeof tick === 'object' ? tick.value : tick
    const label = typeof tick === 'object' ? tick.label : formatAxisNumber(rawValue, config)
    const y = getYPosition(rawValue, yAxis, plot)

    ctx.strokeStyle = rawValue === 0 ? '#c8d5c0' : '#e2e9da'
    ctx.lineWidth = 1
    ctx.beginPath()
    ctx.moveTo(plot.left, y)
    ctx.lineTo(plot.right, y)
    ctx.stroke()

    ctx.fillStyle = '#657266'
    ctx.textAlign = 'right'
    ctx.fillText(label, plot.left - 8, y)
  })

  ctx.textBaseline = 'top'
  xAxis.forEach((tick, index) => {
    const x = getXPosition(tick.dayIndex, minDay, maxDay, plot)
    const isFirst = index === 0
    const isLast = index === xAxis.length - 1
    const align = isFirst ? 'left' : isLast ? 'right' : 'center'

    ctx.strokeStyle = '#edf1e8'
    ctx.beginPath()
    ctx.moveTo(x, plot.top)
    ctx.lineTo(x, plot.bottom)
    ctx.stroke()

    ctx.fillStyle = '#657266'
    ctx.textAlign = align
    ctx.fillText(tick.label, x, plot.bottom + 8)
  })

  ctx.strokeStyle = '#c8d5c0'
  ctx.beginPath()
  ctx.moveTo(plot.left, plot.top)
  ctx.lineTo(plot.left, plot.bottom)
  ctx.lineTo(plot.right, plot.bottom)
  ctx.stroke()
}

function formatAxisNumber(value, config) {
  if (config.integer) return String(Math.round(value))
  return String(Math.round(value * 10) / 10)
}

function drawLine(ctx, points, xFor, yFor, plot) {
  if (points.length < 2) return

  ctx.save()
  ctx.beginPath()
  ctx.rect(plot.left, plot.top, plot.width, plot.height)
  ctx.clip()

  ctx.strokeStyle = '#2f7d4f'
  ctx.lineWidth = 3
  ctx.lineJoin = 'round'
  ctx.lineCap = 'round'
  ctx.beginPath()

  points.forEach((point, index) => {
    const x = xFor(point.date.dayIndex)
    const y = yFor(point.value)
    if (index === 0) ctx.moveTo(x, y)
    else ctx.lineTo(x, y)
  })

  ctx.stroke()
  ctx.restore()
}

function drawPoints(ctx, points, config, xFor, yFor, plot) {
  const drawnLabels = new Set()

  points.forEach((point) => {
    const x = xFor(point.date.dayIndex)
    const y = yFor(point.value)

    ctx.fillStyle = '#f5b947'
    ctx.strokeStyle = '#fff8e7'
    ctx.lineWidth = 2
    ctx.beginPath()
    ctx.arc(x, y, 5, 0, Math.PI * 2)
    ctx.fill()
    ctx.stroke()

    const label = getPointLabel(point.value, config)
    const labelKey = `${Math.round(x)}-${Math.round(y)}-${label}`
    if (drawnLabels.has(labelKey)) return
    drawnLabels.add(labelKey)

    const labelWidth = ctx.measureText(label).width
    const horizontalPadding = Math.min(labelWidth / 2 + 4, plot.width / 2)
    const labelX = clamp(x, plot.left + horizontalPadding, plot.right - horizontalPadding)
    const labelY = clamp(y - 10, 12, plot.height + plot.top - 4)

    ctx.fillStyle = '#314238'
    ctx.font = '12px system-ui'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'bottom'
    ctx.fillText(label, labelX, labelY)
  })
}

function getPointLabel(value, config) {
  if (config.type === 'category') return config.categories[value]
  return `${formatAxisNumber(value, config)}${config.unit}`
}

function drawRecordCount(ctx, chartCount, validCount, totalCount, width) {
  ctx.fillStyle = '#7a856f'
  ctx.font = '12px system-ui'
  ctx.textAlign = 'right'
  ctx.textBaseline = 'top'
  ctx.fillText(`${chartCount} 天 · 有效 ${validCount}/${totalCount} 条`, width - 18, 6)
}

function drawMessage(ctx, width, height, text) {
  ctx.fillStyle = '#667568'
  ctx.font = '14px system-ui'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText(text, width / 2, height / 2)
}

function clamp(value, min, max) {
  if (min > max) return (min + max) / 2
  return Math.min(max, Math.max(min, value))
}
</script>

<template>
  <div class="growth-chart">
    <div class="growth-toolbar" aria-label="选择生长指标">
      <button
        v-for="metric in METRICS"
        :key="metric.key"
        :aria-pressed="selectedMetric === metric.key"
        :class="['metric-toggle', { active: selectedMetric === metric.key }]"
        type="button"
        @click="selectedMetric = metric.key"
      >
        {{ metric.label }}
      </button>
    </div>
    <canvas ref="canvasRef" class="growth-canvas" aria-label="按真实日期绘制的植物生长曲线"></canvas>
  </div>
</template>
