<script setup lang="ts">
import { ref, computed } from 'vue'
import { Bar, Line } from 'vue-chartjs'
import logoUrl from '@/assets/ff-logistics-logo.svg'
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  BarElement,
  PointElement,
  LineElement,
  Filler,
  Title,
  Tooltip,
  Legend,
} from 'chart.js'
import metricsData from '@/data/metrics.json'

ChartJS.register(
  CategoryScale,
  LinearScale,
  BarElement,
  PointElement,
  LineElement,
  Filler,
  Title,
  Tooltip,
  Legend
)

// --- Types ---
interface MonthData {
  month: string
  year: number
  shipmentVolume: number
  onTimeDeliveryRate: number
  regionalPerformance: number
  openExceptions: number
}

const data: MonthData[] = metricsData as MonthData[]

// --- State ---
const monthOptions = ['All', ...data.map((d) => d.month)]
const selectedMonth = ref('All')

// --- Computed: filtered data ---
const selectedIndex = computed(() =>
  data.findIndex((d) => d.month === selectedMonth.value)
)

const filteredData = computed(() =>
  selectedMonth.value === 'All'
    ? data
    : data.filter((d) => d.month === selectedMonth.value)
)

// --- Summary card values ---
const summaryShipments = computed(() =>
  selectedMonth.value === 'All'
    ? data.reduce((sum, d) => sum + d.shipmentVolume, 0)
    : filteredData.value[0]?.shipmentVolume ?? 0
)

const summaryOnTime = computed(() => {
  if (selectedMonth.value === 'All') {
    const avg = data.reduce((sum, d) => sum + d.onTimeDeliveryRate, 0) / data.length
    return +avg.toFixed(1)
  }
  return filteredData.value[0]?.onTimeDeliveryRate ?? 0
})

const summaryRegional = computed(() => {
  if (selectedMonth.value === 'All') {
    const avg = data.reduce((sum, d) => sum + d.regionalPerformance, 0) / data.length
    return +avg.toFixed(1)
  }
  return filteredData.value[0]?.regionalPerformance ?? 0
})

const summaryExceptions = computed(() =>
  selectedMonth.value === 'All'
    ? data.reduce((sum, d) => sum + d.openExceptions, 0)
    : filteredData.value[0]?.openExceptions ?? 0
)

// --- Change indicators (vs previous month) ---
function changeInfo(field: keyof MonthData) {
  if (selectedMonth.value === 'All') {
    const first = data[0]![field] as number
    const last = data[data.length - 1]![field] as number
    const pct = first === 0 ? 0 : (((last - first) / first) * 100)
    return { direction: pct >= 0 ? 'up' : 'down', pct: Math.abs(+pct.toFixed(1)) }
  }
  const idx = selectedIndex.value
  if (idx <= 0) return { direction: 'neutral' as const, pct: 0 }
  const prev = data[idx - 1]![field] as number
  const curr = data[idx]![field] as number
  const pct = prev === 0 ? 0 : (((curr - prev) / prev) * 100)
  return { direction: pct >= 0 ? 'up' : 'down', pct: Math.abs(+pct.toFixed(1)) }
}

const shipmentsChange = computed(() => changeInfo('shipmentVolume'))
const onTimeChange = computed(() => changeInfo('onTimeDeliveryRate'))
const regionalChange = computed(() => changeInfo('regionalPerformance'))
const exceptionsChange = computed(() => changeInfo('openExceptions'))

// --- Color palette (brand #06e8b5 tints & shades) ---
const COLORS = {
  brand: '#06e8b5',
  brandLight: 'rgba(6,232,181,0.25)',
  brandMid: '#04b38b',
  brandMidLight: 'rgba(4,179,139,0.25)',
  brandDark: '#038f6f',
  brandDarkLight: 'rgba(3,143,111,0.25)',
  grid: 'rgba(6,232,181,0.08)',
  tickText: 'rgba(255,255,255,0.5)',
}

// --- Charts ---
const chartLabels = computed(() => filteredData.value.map((d) => d.month))

const shipmentChartData = computed(() => ({
  labels: chartLabels.value,
  datasets: [
    {
      label: 'Shipment Volume',
      data: filteredData.value.map((d) => d.shipmentVolume),
      backgroundColor: COLORS.brand,
      borderRadius: 4,
      maxBarThickness: 48,
    },
  ],
}))

const onTimeChartData = computed(() => ({
  labels: chartLabels.value,
  datasets: [
    {
      label: 'On-Time Delivery %',
      data: filteredData.value.map((d) => d.onTimeDeliveryRate),
      borderColor: COLORS.brandMid,
      backgroundColor: COLORS.brandMidLight,
      pointBackgroundColor: COLORS.brandMid,
      tension: 0.35,
      fill: false,
      pointRadius: 4,
    },
  ],
}))

const exceptionsChartData = computed(() => ({
  labels: chartLabels.value,
  datasets: [
    {
      label: 'Open Exceptions',
      data: filteredData.value.map((d) => d.openExceptions),
      borderColor: COLORS.brandDark,
      backgroundColor: COLORS.brandDarkLight,
      pointBackgroundColor: COLORS.brandDark,
      tension: 0.35,
      fill: true,
      pointRadius: 4,
    },
  ],
}))

// --- Chart options ---
const baseScales = {
  x: {
    grid: { color: COLORS.grid },
    ticks: { color: COLORS.tickText },
  },
  y: {
    grid: { color: COLORS.grid },
    ticks: { color: COLORS.tickText },
  },
}

const barOptions = computed(() => ({
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
    title: { display: true, text: 'Monthly Shipment Volume', color: '#fff', font: { size: 14 } },
  },
  scales: {
    ...baseScales,
    y: {
      ...baseScales.y,
      ticks: {
        ...baseScales.y.ticks,
        callback: (v: number | string) => Number(v).toLocaleString(),
      },
    },
  },
}))

const lineOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
    title: { display: true, text: 'On-Time Delivery Rate', color: '#fff', font: { size: 14 } },
  },
  scales: {
    ...baseScales,
    y: {
      ...baseScales.y,
      min: 85,
      max: 100,
      ticks: {
        ...baseScales.y.ticks,
        callback: (v: number | string) => v + '%',
      },
    },
  },
}

const areaOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
    title: { display: true, text: 'Open Exceptions Trend', color: '#fff', font: { size: 14 } },
  },
  scales: baseScales,
}

// --- Helpers ---
function formatNumber(n: number) {
  return n.toLocaleString()
}
</script>

<template>
  <!-- App Bar -->
  <v-app-bar flat color="surface" :elevation="0" class="border-b px-6">
    <v-app-bar-title>
      <img :src="logoUrl" alt="FastForward Logistics" style="height: 36px; vertical-align: middle" />
    </v-app-bar-title>
    <template #append>
      <div style="width: 200px" class="mr-2">
        <v-select
          v-model="selectedMonth"
          :items="monthOptions"
          variant="outlined"
          density="compact"
          hide-details
          label="Month"
        />
      </div>
    </template>
  </v-app-bar>

  <v-main>
    <v-container fluid class="dashboard-container">
      <!-- Summary Cards -->
      <div class="card-grid">
        <!-- Shipment Volume -->
        <v-card variant="tonal" class="pa-10" rounded="lg">
          <div class="text-caption text-medium-emphasis mb-2">Shipment Volume</div>
          <div class="text-h5 font-weight-bold">{{ formatNumber(summaryShipments) }}</div>
          <div
            class="text-caption mt-2"
            :class="shipmentsChange.direction === 'up' ? 'text-success' : 'text-error'"
          >
            <v-icon size="x-small">
              {{ shipmentsChange.direction === 'up' ? 'mdi-arrow-up' : 'mdi-arrow-down' }}
            </v-icon>
            {{ shipmentsChange.pct }}% {{ selectedMonth === 'All' ? 'YTD' : 'vs prev' }}
          </div>
        </v-card>

        <!-- On-Time Delivery Rate -->
        <v-card variant="tonal" class="pa-10" rounded="lg">
          <div class="text-caption text-medium-emphasis mb-2">On-Time Delivery</div>
          <div class="text-h5 font-weight-bold">{{ summaryOnTime }}%</div>
          <div
            class="text-caption mt-2"
            :class="onTimeChange.direction === 'up' ? 'text-success' : 'text-error'"
          >
            <v-icon size="x-small">
              {{ onTimeChange.direction === 'up' ? 'mdi-arrow-up' : 'mdi-arrow-down' }}
            </v-icon>
            {{ onTimeChange.pct }}% {{ selectedMonth === 'All' ? 'YTD' : 'vs prev' }}
          </div>
        </v-card>

        <!-- Regional Performance -->
        <v-card variant="tonal" class="pa-10" rounded="lg">
          <div class="text-caption text-medium-emphasis mb-2">Regional Performance</div>
          <div class="text-h5 font-weight-bold">{{ summaryRegional }}%</div>
          <div
            class="text-caption mt-2"
            :class="regionalChange.direction === 'up' ? 'text-success' : 'text-error'"
          >
            <v-icon size="x-small">
              {{ regionalChange.direction === 'up' ? 'mdi-arrow-up' : 'mdi-arrow-down' }}
            </v-icon>
            {{ regionalChange.pct }}% {{ selectedMonth === 'All' ? 'YTD' : 'vs prev' }}
          </div>
        </v-card>

        <!-- Open Exceptions (lower is better) -->
        <v-card variant="tonal" class="pa-10" rounded="lg">
          <div class="text-caption text-medium-emphasis mb-2">Open Exceptions</div>
          <div class="text-h5 font-weight-bold">{{ formatNumber(summaryExceptions) }}</div>
          <div
            class="text-caption mt-2"
            :class="exceptionsChange.direction === 'down' ? 'text-success' : 'text-error'"
          >
            <v-icon size="x-small">
              {{ exceptionsChange.direction === 'up' ? 'mdi-arrow-up' : 'mdi-arrow-down' }}
            </v-icon>
            {{ exceptionsChange.pct }}% {{ selectedMonth === 'All' ? 'YTD' : 'vs prev' }}
          </div>
        </v-card>
      </div>

      <!-- Charts Row: Shipment Volume Bar + On-Time Delivery Line -->
      <div class="chart-grid-2 chart-row-flex">
        <v-card variant="tonal" rounded="lg" class="pa-10 chart-card">
          <div class="chart-inner">
            <Bar :data="shipmentChartData" :options="barOptions as any" />
          </div>
        </v-card>
        <v-card variant="tonal" rounded="lg" class="pa-10 chart-card">
          <div class="chart-inner">
            <Line :data="onTimeChartData" :options="lineOptions as any" />
          </div>
        </v-card>
      </div>

      <!-- Full-width Open Exceptions Area Chart -->
      <v-card variant="tonal" rounded="lg" class="pa-10 chart-card conversions-card">
        <div class="chart-inner">
          <Line :data="exceptionsChartData" :options="areaOptions as any" />
        </div>
      </v-card>
    </v-container>
  </v-main>
</template>

<style scoped>
.dashboard-container {
  display: flex;
  flex-direction: column;
  gap: 24px;
  padding: 32px;
  height: calc(100vh - 64px);
  box-sizing: border-box;
}

.dashboard-container :deep(.v-card) {
  padding: 8px !important;
}

.card-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 20px;
  flex-shrink: 0;
}

.chart-row-flex {
  flex: 3;
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 20px;
  min-height: 200px;
}

.chart-card {
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.chart-card :deep(.v-card__content) {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.chart-inner {
  flex: 1;
  position: relative;
  min-height: 100px;
}

.chart-inner canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100% !important;
  height: 100% !important;
}

/* Conversions card at bottom */
.conversions-card {
  flex: 2;
  min-height: 150px;
}

.chart-grid-2 {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 20px;
  flex: 3;
  min-height: 200px;
}

.v-app-bar :deep(.v-field__input) {
  padding-left: 12px;
}

:deep(.v-list-item) {
  padding-left: 16px !important;
}

@media (max-width: 1264px) {
  .card-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 600px) {
  .card-grid {
    grid-template-columns: 1fr;
  }
  .chart-grid-2 {
    grid-template-columns: 1fr;
  }
}
</style>

<style>
/* Global — dropdown menu renders outside scoped component */
.v-overlay .v-list-item {
  padding-left: 16px !important;
}
.v-overlay .v-list-item__content {
  padding-left: 8px;
}

/* Force app bar inner padding */
.v-app-bar > .v-toolbar__content {
  padding-left: 24px !important;
  padding-right: 24px !important;
}
</style>
