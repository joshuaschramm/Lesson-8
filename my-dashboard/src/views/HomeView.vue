<script setup lang="ts">
import { ref, computed, nextTick } from 'vue'
import { Bar, Line } from 'vue-chartjs'
import logoUrl from '@/assets/ff-logistics-logo.svg'
import * as XLSX from 'xlsx'
import { jsPDF } from 'jspdf'
import autoTable from 'jspdf-autotable'
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

// --- Expand / collapse ---
type ChartId = 'shipments' | 'onTime' | 'exceptions'
const expandedChart = ref<ChartId | null>(null)
const expandKey = ref(0) // force re-render on expand/collapse

function expandChart(id: ChartId) {
  expandedChart.value = id
  expandKey.value++
}
function collapseChart() {
  expandedChart.value = null
  expandKey.value++
}

const chartTitles: Record<ChartId, string> = {
  shipments: 'Monthly Shipment Volume',
  onTime: 'On-Time Delivery Rate',
  exceptions: 'Open Exceptions Trend',
}

// --- Data table for expanded view ---
const expandedTableHeaders = computed(() => {
  const id = expandedChart.value
  if (!id) return []
  const fieldMap: Record<ChartId, { text: string; value: string }> = {
    shipments: { text: 'Shipment Volume', value: 'shipmentVolume' },
    onTime: { text: 'On-Time Rate (%)', value: 'onTimeDeliveryRate' },
    exceptions: { text: 'Open Exceptions', value: 'openExceptions' },
  }
  return [
    { title: 'Month', key: 'month', align: 'start' as const },
    { title: fieldMap[id].text, key: fieldMap[id].value, align: 'end' as const },
  ]
})

const expandedTableItems = computed(() => {
  return filteredData.value.map((d) => ({ ...d }))
})

// --- Helpers ---
function formatNumber(n: number) {
  return n.toLocaleString()
}

// --- Export ---
function getExportRows() {
  return filteredData.value.map((d) => ({
    Month: d.month,
    Year: d.year,
    'Shipment Volume': d.shipmentVolume,
    'On-Time Delivery Rate (%)': d.onTimeDeliveryRate,
    'Regional Performance (%)': d.regionalPerformance,
    'Open Exceptions': d.openExceptions,
  }))
}

function downloadExcel() {
  const rows = getExportRows()
  const ws = XLSX.utils.json_to_sheet(rows)
  const wb = XLSX.utils.book_new()
  XLSX.utils.book_append_sheet(wb, ws, 'Metrics')
  XLSX.writeFile(wb, `FastForward_Logistics_${selectedMonth.value}.xlsx`)
}

function downloadPdf() {
  const rows = getExportRows()
  const doc = new jsPDF({ orientation: 'landscape' })
  doc.setFontSize(16)
  doc.text('FastForward Logistics — Dashboard Report', 14, 18)
  doc.setFontSize(10)
  doc.text(`Period: ${selectedMonth.value === 'All' ? 'Full Year 2025' : selectedMonth.value + ' 2025'}`, 14, 26)

  const columns = Object.keys(rows[0] || {})
  const body = rows.map((r) => columns.map((c) => String((r as any)[c])))

  autoTable(doc, {
    startY: 32,
    head: [columns],
    body,
    theme: 'grid',
    headStyles: { fillColor: [6, 232, 181], textColor: [8, 28, 59], fontStyle: 'bold' },
    styles: { fontSize: 9 },
  })

  doc.save(`FastForward_Logistics_${selectedMonth.value}.pdf`)
}
</script>

<template>
  <!-- App Bar -->
  <v-app-bar flat color="surface" :elevation="0" class="border-b px-6">
    <v-app-bar-title>
      <img :src="logoUrl" alt="FastForward Logistics" style="height: 36px; vertical-align: middle" />
    </v-app-bar-title>
    <template #append>
      <v-btn
        icon
        variant="text"
        size="small"
        color="#06e8b5"
        @click="downloadExcel"
        aria-label="Download Excel"
      >
        <v-icon>mdi-file-excel-outline</v-icon>
        <v-tooltip activator="parent" location="bottom">Download Excel</v-tooltip>
      </v-btn>
      <v-btn
        icon
        variant="text"
        size="small"
        color="#06e8b5"
        class="mr-2"
        @click="downloadPdf"
        aria-label="Download PDF"
      >
        <v-icon>mdi-file-pdf-box</v-icon>
        <v-tooltip activator="parent" location="bottom">Download PDF</v-tooltip>
      </v-btn>
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
    <!-- =========== EXPANDED CHART VIEW =========== -->
    <transition name="page-fade">
      <v-container v-if="expandedChart" key="expanded" fluid class="expanded-container">
        <div class="expanded-header">
          <h2 class="text-h5 font-weight-bold">{{ chartTitles[expandedChart] }}</h2>
          <v-btn
            icon
            variant="text"
            size="small"
            @click="collapseChart"
            aria-label="Back to dashboard"
          >
            <v-icon>mdi-arrow-collapse</v-icon>
            <v-tooltip activator="parent" location="bottom">Back to dashboard</v-tooltip>
          </v-btn>
        </div>

        <v-card variant="tonal" rounded="lg" class="expanded-chart-card">
          <div class="expanded-chart-inner">
            <Bar v-if="expandedChart === 'shipments'" :key="expandKey" :data="shipmentChartData" :options="barOptions as any" />
            <Line v-else-if="expandedChart === 'onTime'" :key="expandKey" :data="onTimeChartData" :options="lineOptions as any" />
            <Line v-else-if="expandedChart === 'exceptions'" :key="expandKey" :data="exceptionsChartData" :options="areaOptions as any" />
          </div>
        </v-card>

        <v-card variant="tonal" rounded="lg" class="mt-6 expanded-table-card">
          <v-data-table
            :headers="expandedTableHeaders as any"
            :items="expandedTableItems"
            density="compact"
            items-per-page="-1"
            class="bg-transparent"
            hide-default-footer
          />
        </v-card>
      </v-container>
    </transition>

    <!-- =========== DASHBOARD VIEW =========== -->
    <transition name="page-fade">
      <v-container v-if="!expandedChart" key="dashboard" fluid class="dashboard-container">
        <!-- Summary Cards -->
        <div class="card-grid">
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

        <!-- Charts Row -->
        <div class="chart-grid-2 chart-row-flex">
          <v-card variant="tonal" rounded="lg" class="pa-10 chart-card">
            <v-btn
              icon
              variant="text"
              size="x-small"
              class="expand-btn"
              @click="expandChart('shipments')"
              aria-label="Expand chart"
            >
              <v-icon size="small">mdi-arrow-expand</v-icon>
              <v-tooltip activator="parent" location="bottom">Expand</v-tooltip>
            </v-btn>
            <div class="chart-inner">
              <Bar :data="shipmentChartData" :options="barOptions as any" />
            </div>
          </v-card>
          <v-card variant="tonal" rounded="lg" class="pa-10 chart-card">
            <v-btn
              icon
              variant="text"
              size="x-small"
              class="expand-btn"
              @click="expandChart('onTime')"
              aria-label="Expand chart"
            >
              <v-icon size="small">mdi-arrow-expand</v-icon>
              <v-tooltip activator="parent" location="bottom">Expand</v-tooltip>
            </v-btn>
            <div class="chart-inner">
              <Line :data="onTimeChartData" :options="lineOptions as any" />
            </div>
          </v-card>
        </div>

        <!-- Full-width Exceptions Area Chart -->
        <v-card variant="tonal" rounded="lg" class="pa-10 chart-card conversions-card">
          <v-btn
            icon
            variant="text"
            size="x-small"
            class="expand-btn"
            @click="expandChart('exceptions')"
            aria-label="Expand chart"
          >
            <v-icon size="small">mdi-arrow-expand</v-icon>
            <v-tooltip activator="parent" location="bottom">Expand</v-tooltip>
          </v-btn>
          <div class="chart-inner">
            <Line :data="exceptionsChartData" :options="areaOptions as any" />
          </div>
        </v-card>
      </v-container>
    </transition>
  </v-main>
</template>

<style scoped>
/* ===== Page transition ===== */
.page-fade-enter-active,
.page-fade-leave-active {
  transition: opacity 0.3s ease, transform 0.3s ease;
}
.page-fade-enter-from {
  opacity: 0;
  transform: scale(0.97);
}
.page-fade-leave-to {
  opacity: 0;
  transform: scale(0.97);
}

/* ===== Dashboard view ===== */
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
  position: relative;
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

/* ===== Expand button ===== */
.expand-btn {
  position: absolute;
  top: 8px;
  right: 8px;
  z-index: 2;
  opacity: 0.5;
  transition: opacity 0.2s ease;
}
.chart-card:hover .expand-btn {
  opacity: 1;
}

/* ===== Expanded view ===== */
.expanded-container {
  padding: 32px;
  display: flex;
  flex-direction: column;
  min-height: calc(100vh - 64px);
  box-sizing: border-box;
}

.expanded-container :deep(.v-card) {
  padding: 8px !important;
}

.expanded-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 20px;
}

.expanded-chart-card {
  height: 50vh;
  min-height: 320px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.expanded-chart-inner {
  flex: 1;
  position: relative;
  padding: 16px;
}

.expanded-chart-inner canvas {
  position: absolute;
  top: 16px;
  left: 16px;
  width: calc(100% - 32px) !important;
  height: calc(100% - 32px) !important;
}

.expanded-table-card {
  flex: 1;
}

.expanded-table-card :deep(.v-table) {
  background: transparent !important;
}

.expanded-table-card :deep(th) {
  color: rgba(255,255,255,0.7) !important;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.expanded-table-card :deep(td) {
  border-bottom-color: rgba(255,255,255,0.06) !important;
}

/* ===== Misc ===== */
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
