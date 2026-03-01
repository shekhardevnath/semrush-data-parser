<script setup>
import { computed, reactive, ref } from 'vue'

const SAMPLE_CSV = `Keyword,Intent,Volume,Trend,Keyword Difficulty,CPC (AUD),Competitive Density,SERP Features,Number of Results
fridge freezer,Commercial,12100,"1.00,1.00,0.81,0.81,0.67,0.67,0.67,0.81,0.81,0.81,1.00,1.00",28,0.60,1.00,"Sitelinks, Image, Video, People also ask, Related searches, Popular products",370000000
kings fridge,Transactional,12100,"1.00,1.00,0.66,0.81,0.81,0.54,0.54,0.54,0.66,0.81,0.81,1.00",18,0.79,1.00,"Sitelinks, Image, Image pack, Video, Video carousel, People also ask, Related searches, Popular products",266000000`

const rows = ref([])
const fileName = ref('Sample data')
const parseError = ref('')
const sortKey = ref('volume')
const sortDir = ref('desc')
const selectedIds = reactive(new Set())

const filters = reactive({
  keyword: '',
  intent: '',
  volumeMin: '',
  volumeMax: '',
  trendMin: '',
  trendMax: '',
  kdMin: '',
  kdMax: '',
  cpcMin: '',
  cpcMax: '',
  comMin: '',
  comMax: '',
  sf: '',
  resultsMin: '',
  resultsMax: ''
})

const intentOptions = computed(() => {
  return [...new Set(rows.value.flatMap((row) => parseIntentCodes(row.intent).map((item) => item.label)))].sort(
    (a, b) => a.localeCompare(b)
  )
})

const filteredRows = computed(() => {
  const keywordQuery = filters.keyword.trim().toLowerCase()
  const sfQuery = filters.sf.trim().toLowerCase()

  return rows.value.filter((row) => {
    const intentCodes = parseIntentCodes(row.intent)
    const intentLabels = intentCodes.map((item) => item.label)

    if (keywordQuery && !row.keyword.toLowerCase().includes(keywordQuery)) return false
    if (filters.intent && !intentLabels.includes(filters.intent)) return false
    if (!inRange(row.volume, filters.volumeMin, filters.volumeMax)) return false
    if (!inRange(row.trendAvg, filters.trendMin, filters.trendMax)) return false
    if (!inRange(row.keywordDifficulty, filters.kdMin, filters.kdMax)) return false
    if (!inRange(row.cpcAud, filters.cpcMin, filters.cpcMax)) return false
    if (!inRange(row.competitiveDensity, filters.comMin, filters.comMax)) return false
    if (!inRange(row.numberOfResults, filters.resultsMin, filters.resultsMax)) return false
    if (sfQuery && !row.serpFeatures.toLowerCase().includes(sfQuery)) return false
    return true
  })
})

const sortedRows = computed(() => {
  return filteredRows.value
    .map((row, index) => ({ row, index }))
    .sort((left, right) => {
      const a = left.row[sortKey.value]
      const b = right.row[sortKey.value]
      let cmp = 0
      if (typeof a === 'string' || typeof b === 'string') {
        cmp = String(a ?? '').localeCompare(String(b ?? ''), undefined, { sensitivity: 'base' })
      } else {
        cmp = compareNumbers(a, b)
      }
      if (cmp === 0) cmp = left.index - right.index
      return sortDir.value === 'asc' ? cmp : -cmp
    })
    .map((item) => item.row)
})

const allVisibleSelected = computed(() => {
  if (!sortedRows.value.length) return false
  return sortedRows.value.every((row) => selectedIds.has(row.id))
})

function compareNumbers(a, b) {
  const aMissing = a == null || Number.isNaN(a)
  const bMissing = b == null || Number.isNaN(b)
  if (aMissing && bMissing) return 0
  if (aMissing) return 1
  if (bMissing) return -1
  return a - b
}

function inRange(value, minRaw, maxRaw) {
  const hasMin = String(minRaw).trim() !== ''
  const hasMax = String(maxRaw).trim() !== ''
  if (!hasMin && !hasMax) return true
  if (value == null || Number.isNaN(value)) return false
  const min = hasMin ? Number(minRaw) : null
  const max = hasMax ? Number(maxRaw) : null
  if (min != null && value < min) return false
  if (max != null && value > max) return false
  return true
}

function setSort(key) {
  if (sortKey.value === key) {
    sortDir.value = sortDir.value === 'asc' ? 'desc' : 'asc'
    return
  }
  sortKey.value = key
  sortDir.value = key === 'volume' ? 'desc' : 'asc'
}

function sortIndicator(key) {
  if (sortKey.value !== key) return ''
  return sortDir.value === 'asc' ? '^' : 'v'
}

function normalizeHeader(value) {
  return value.toLowerCase().replace(/[^a-z0-9]+/g, '')
}

function parseCsv(text) {
  const parsed = []
  let row = []
  let field = ''
  let inQuotes = false

  for (let i = 0; i < text.length; i += 1) {
    const char = text[i]
    const next = text[i + 1]

    if (char === '"') {
      if (inQuotes && next === '"') {
        field += '"'
        i += 1
      } else {
        inQuotes = !inQuotes
      }
      continue
    }

    if (char === ',' && !inQuotes) {
      row.push(field)
      field = ''
      continue
    }

    if ((char === '\n' || char === '\r') && !inQuotes) {
      if (char === '\r' && next === '\n') i += 1
      row.push(field)
      field = ''
      if (row.some((cell) => cell.trim() !== '')) parsed.push(row)
      row = []
      continue
    }

    field += char
  }

  row.push(field)
  if (row.some((cell) => cell.trim() !== '')) parsed.push(row)
  return parsed
}

function parseNumber(raw) {
  if (raw == null) return null
  const cleaned = String(raw).trim().replace(/,/g, '')
  if (!cleaned) return null
  const value = Number(cleaned)
  return Number.isFinite(value) ? value : null
}

function parseTrend(raw) {
  if (!raw) return []
  return String(raw)
    .split(',')
    .map((part) => Number(part.trim()))
    .filter((value) => Number.isFinite(value))
}

function trendPath(values) {
  if (!values.length) return ''
  const width = 60
  const height = 18
  const min = Math.min(...values)
  const max = Math.max(...values)
  const span = max - min || 1
  return values
    .map((value, index) => {
      const x = (index / (values.length - 1 || 1)) * width
      const y = height - ((value - min) / span) * height
      return `${x.toFixed(2)},${y.toFixed(2)}`
    })
    .join(' ')
}

function trendFillPath(values) {
  const points = trendPath(values)
  if (!points) return ''
  return `0,18 ${points} 60,18`
}

function parseIntentCodes(intentRaw) {
  const intent = String(intentRaw || '').trim()
  if (!intent) return []
  return intent
    .split(/[,|;/]+/)
    .map((item) => item.trim().toLowerCase())
    .filter(Boolean)
    .map((item) => {
      if (item.startsWith('i')) return { code: 'I', label: 'Informational', className: 'intent-i' }
      if (item.startsWith('t')) return { code: 'T', label: 'Transactional', className: 'intent-t' }
      if (item.startsWith('c')) return { code: 'C', label: 'Commercial', className: 'intent-c' }
      if (item.startsWith('n')) return { code: 'N', label: 'Navigational', className: 'intent-n' }
      return null
    })
    .filter(Boolean)
}

function getSfCount(value) {
  return String(value || '')
    .split(',')
    .map((item) => item.trim())
    .filter(Boolean).length
}

function kdDotClass(value) {
  if (value == null || Number.isNaN(value)) return 'kd-dot kd-0'
  if (value < 30) return 'kd-dot kd-1'
  if (value < 50) return 'kd-dot kd-2'
  if (value < 70) return 'kd-dot kd-3'
  return 'kd-dot kd-4'
}

function loadCsv(text, name = 'Uploaded file') {
  parseError.value = ''
  try {
    const records = parseCsv(text)
    if (!records.length) throw new Error('The file is empty.')
    const headers = records[0]
    const headerMap = new Map(headers.map((header, index) => [normalizeHeader(header), index]))

    const findIndex = (aliases) => {
      for (const alias of aliases) {
        const index = headerMap.get(normalizeHeader(alias))
        if (index != null) return index
      }
      return -1
    }

    const idxKeyword = findIndex(['Keyword'])
    const idxIntent = findIndex(['Intent'])
    const idxVolume = findIndex(['Volume'])
    const idxTrend = findIndex(['Trend'])
    const idxDifficulty = findIndex(['Keyword Difficulty', 'KD', 'Keyword Difficulty Index'])
    const idxCpc = findIndex(['CPC (AUD)', 'CPC'])
    const idxDensity = findIndex(['Competitive Density', 'Competition'])
    const idxSerpFeatures = findIndex(['SERP Features', 'Keywords SERP Features'])
    const idxResults = findIndex(['Number of Results', 'Results'])

    if (idxKeyword < 0) throw new Error('Missing required "Keyword" column.')

    rows.value = records
      .slice(1)
      .map((record, index) => {
        const trend = parseTrend(record[idxTrend] ?? '')
        const trendAvg = trend.length ? trend.reduce((sum, value) => sum + value, 0) / trend.length : null
        const serpFeatures = String(record[idxSerpFeatures] ?? '').trim()
        return {
          id: index + 1,
          keyword: String(record[idxKeyword] ?? '').trim(),
          intent: String(record[idxIntent] ?? '').trim(),
          volume: parseNumber(record[idxVolume]),
          trend,
          trendAvg,
          keywordDifficulty: parseNumber(record[idxDifficulty]),
          cpcAud: parseNumber(record[idxCpc]),
          competitiveDensity: parseNumber(record[idxDensity]),
          serpFeatures,
          sfCount: getSfCount(serpFeatures),
          numberOfResults: parseNumber(record[idxResults])
        }
      })
      .filter((row) => row.keyword)
    fileName.value = name
    selectedIds.clear()
  } catch (error) {
    rows.value = []
    parseError.value = error instanceof Error ? error.message : 'Failed to parse CSV file.'
  }
}

function handleFileUpload(event) {
  const file = event.target.files?.[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = () => loadCsv(String(reader.result || ''), file.name)
  reader.onerror = () => {
    rows.value = []
    parseError.value = 'Unable to read the selected file.'
  }
  reader.readAsText(file)
}

function toggleAllVisible(checked) {
  if (checked) {
    for (const row of sortedRows.value) selectedIds.add(row.id)
  } else {
    for (const row of sortedRows.value) selectedIds.delete(row.id)
  }
}

function toggleRow(id, checked) {
  if (checked) selectedIds.add(id)
  else selectedIds.delete(id)
}

function clearFilters() {
  filters.keyword = ''
  filters.intent = ''
  filters.volumeMin = ''
  filters.volumeMax = ''
  filters.trendMin = ''
  filters.trendMax = ''
  filters.kdMin = ''
  filters.kdMax = ''
  filters.cpcMin = ''
  filters.cpcMax = ''
  filters.comMin = ''
  filters.comMax = ''
  filters.sf = ''
  filters.resultsMin = ''
  filters.resultsMax = ''
}

function formatInteger(value) {
  if (value == null || Number.isNaN(value)) return '-'
  return new Intl.NumberFormat('en-US', { maximumFractionDigits: 0 }).format(value)
}

function formatCompact(value) {
  if (value == null || Number.isNaN(value)) return '-'
  const abs = Math.abs(value)
  if (abs < 10000) return formatInteger(value)
  if (abs < 1000000) {
    const compactK = value / 1000
    const digits = abs >= 100000 ? 0 : 1
    return `${trimFixed(compactK, digits)}K`
  }
  return `${trimFixed(value / 1000000, 1)}M`
}

function trimFixed(value, digits) {
  return Number(value).toFixed(digits).replace(/\.0$/, '')
}

function formatDecimal(value, digits = 2) {
  if (value == null || Number.isNaN(value)) return '-'
  return Number(value).toFixed(digits)
}

loadCsv(SAMPLE_CSV, 'Sample data')
</script>

<template>
  <section class="semrush-table-wrap">
    <div class="topbar">
      <label class="top-btn primary">
        <input class="hidden-input" type="file" accept=".csv,text/csv" @change="handleFileUpload" />
        Upload CSV
      </label>
      <button class="top-btn" type="button" @click="loadCsv(SAMPLE_CSV, 'Sample data')">Load sample</button>
      <button class="top-btn" type="button" @click="clearFilters">Clear filters</button>
      <span class="filename">{{ fileName }}</span>
    </div>

    <div class="filters">
      <input v-model="filters.keyword" type="text" placeholder="Keyword contains..." />
      <select v-model="filters.intent">
        <option value="">All intents</option>
        <option v-for="intent in intentOptions" :key="intent" :value="intent">{{ intent }}</option>
      </select>
      <input v-model="filters.volumeMin" type="number" placeholder="Volume min" />
      <input v-model="filters.volumeMax" type="number" placeholder="Volume max" />
      <input v-model="filters.trendMin" type="number" step="0.01" placeholder="Trend avg min" />
      <input v-model="filters.trendMax" type="number" step="0.01" placeholder="Trend avg max" />
      <input v-model="filters.kdMin" type="number" placeholder="KD min" />
      <input v-model="filters.kdMax" type="number" placeholder="KD max" />
      <input v-model="filters.cpcMin" type="number" step="0.01" placeholder="CPC min" />
      <input v-model="filters.cpcMax" type="number" step="0.01" placeholder="CPC max" />
      <input v-model="filters.comMin" type="number" step="0.01" placeholder="Com. min" />
      <input v-model="filters.comMax" type="number" step="0.01" placeholder="Com. max" />
      <input v-model="filters.sf" type="text" placeholder="SERP feature contains..." />
      <input v-model="filters.resultsMin" type="number" placeholder="Results min" />
      <input v-model="filters.resultsMax" type="number" placeholder="Results max" />
    </div>

    <p v-if="parseError" class="error">{{ parseError }}</p>

    <div class="table-shell">
      <table class="semrush-table">
        <thead>
          <tr>
            <th class="check-col">
              <input type="checkbox" :checked="allVisibleSelected" @change="toggleAllVisible($event.target.checked)" />
            </th>
            <th class="keyword-col">
              <button class="sort-btn" type="button" @click="setSort('keyword')">
                Keyword
                <span v-if="sortIndicator('keyword')" class="sort-icon">{{ sortIndicator('keyword') }}</span>
              </button>
            </th>
            <th>
              <button class="sort-btn" type="button" @click="setSort('intent')">
                Intent
                <span v-if="sortIndicator('intent')" class="sort-icon">{{ sortIndicator('intent') }}</span>
              </button>
            </th>
            <th class="num-col">
              <button class="sort-btn" type="button" @click="setSort('volume')">
                Volume
                <span v-if="sortIndicator('volume')" class="sort-icon">{{ sortIndicator('volume') }}</span>
              </button>
            </th>
            <th>
              <button class="sort-btn" type="button" @click="setSort('trendAvg')">
                Trend
                <span v-if="sortIndicator('trendAvg')" class="sort-icon">{{ sortIndicator('trendAvg') }}</span>
              </button>
            </th>
            <th class="num-col">
              <button class="sort-btn" type="button" @click="setSort('keywordDifficulty')">
                KD %
                <span v-if="sortIndicator('keywordDifficulty')" class="sort-icon">
                  {{ sortIndicator('keywordDifficulty') }}
                </span>
              </button>
            </th>
            <th class="num-col">
              <button class="sort-btn" type="button" @click="setSort('cpcAud')">
                CPC (AUD)
                <span v-if="sortIndicator('cpcAud')" class="sort-icon">{{ sortIndicator('cpcAud') }}</span>
              </button>
            </th>
            <th class="num-col">
              <button class="sort-btn" type="button" @click="setSort('competitiveDensity')">
                Com.
                <span v-if="sortIndicator('competitiveDensity')" class="sort-icon">
                  {{ sortIndicator('competitiveDensity') }}
                </span>
              </button>
            </th>
            <th class="sf-col">
              <button class="sort-btn" type="button" @click="setSort('sfCount')">
                SF
                <span v-if="sortIndicator('sfCount')" class="sort-icon">{{ sortIndicator('sfCount') }}</span>
              </button>
            </th>
            <th class="num-col">
              <button class="sort-btn" type="button" @click="setSort('numberOfResults')">
                Results
                <span v-if="sortIndicator('numberOfResults')" class="sort-icon">
                  {{ sortIndicator('numberOfResults') }}
                </span>
              </button>
            </th>
          </tr>
        </thead>

        <tbody>
          <tr v-for="row in sortedRows" :key="row.id">
            <td class="check-col">
              <input type="checkbox" :checked="selectedIds.has(row.id)" @change="toggleRow(row.id, $event.target.checked)" />
            </td>
            <td class="keyword-col">
              <span class="expand-icon">+</span>
              <a href="#" class="keyword-link" @click.prevent>{{ row.keyword }}</a>
              <span class="keyword-mini-icon" />
            </td>
            <td>
              <div class="intent-list">
                <span
                  v-for="intent in parseIntentCodes(row.intent)"
                  :key="`${row.id}-${intent.code}`"
                  :class="['intent-badge', intent.className]"
                  :title="intent.label"
                >
                  {{ intent.code }}
                </span>
                <span v-if="!parseIntentCodes(row.intent).length" class="intent-empty">-</span>
              </div>
            </td>
            <td class="num-col">{{ formatCompact(row.volume) }}</td>
            <td class="trend-cell">
              <svg v-if="row.trend.length" class="sparkline" viewBox="0 0 60 18" width="60" height="18" aria-hidden="true">
                <polygon :points="trendFillPath(row.trend)" />
                <polyline :points="trendPath(row.trend)" />
              </svg>
              <span v-else>-</span>
            </td>
            <td class="num-col kd-col">
              <span>{{ formatDecimal(row.keywordDifficulty, 0) }}</span>
              <span :class="kdDotClass(row.keywordDifficulty)" />
            </td>
            <td class="num-col">{{ formatDecimal(row.cpcAud, 2) }}</td>
            <td class="num-col">{{ formatDecimal(row.competitiveDensity, 2) }}</td>
            <td class="sf-col">
              <span class="sf-icon" />
              <span class="sf-value">{{ row.sfCount }}</span>
            </td>
            <td class="num-col">{{ formatCompact(row.numberOfResults) }}</td>
          </tr>

          <tr v-if="!sortedRows.length">
            <td colspan="10" class="empty">No rows match the current filters.</td>
          </tr>
        </tbody>
      </table>
    </div>
  </section>
</template>

<style scoped>
.semrush-table-wrap {
  padding: 10px 0;
  color: #1f2937;
  font-family: Arial, Helvetica, sans-serif;
}

.topbar {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  align-items: center;
  margin: 0 10px 10px;
}

.top-btn {
  border: 1px solid #cfd5df;
  background: #ffffff;
  border-radius: 6px;
  padding: 6px 10px;
  font-size: 12px;
  cursor: pointer;
}

.top-btn.primary {
  border-color: #b6cef9;
  color: #0b57d0;
}

.hidden-input {
  display: none;
}

.filename {
  margin-left: auto;
  color: #7a8596;
  font-size: 12px;
}

.filters {
  margin: 0 10px 10px;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(130px, 1fr));
  gap: 6px;
}

.filters input,
.filters select {
  border: 1px solid #cfd5df;
  border-radius: 6px;
  padding: 6px 8px;
  font-size: 12px;
  background: #ffffff;
}

.error {
  margin: 0 10px 10px;
  border: 1px solid #f1b8bf;
  border-radius: 6px;
  background: #fef2f4;
  color: #b02339;
  padding: 8px 10px;
  font-size: 12px;
}

.table-shell {
  overflow: auto;
  border-top: 1px solid #d4d9e1;
  border-bottom: 1px solid #d4d9e1;
}

.semrush-table {
  width: 100%;
  min-width: 1100px;
  border-collapse: collapse;
  font-size: 13px;
}

.semrush-table th,
.semrush-table td {
  border-bottom: 1px solid #d7dce4;
  padding: 8px 10px;
  white-space: nowrap;
  line-height: 1.2;
  font-size: 13px;
}

.semrush-table thead th {
  background: #eceff4;
  font-weight: 500;
  color: #202a38;
}

.semrush-table tbody tr:nth-child(even) {
  background: #f5f6f8;
}

.semrush-table tbody tr:hover {
  background: #ecf3ff;
}

.sort-btn {
  border: 0;
  background: transparent;
  color: inherit;
  font: inherit;
  padding: 0;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.sort-icon {
  color: #5e6b7d;
  font-size: 10px;
}

.check-col {
  width: 36px;
  text-align: center;
}

.check-col input {
  width: 18px;
  height: 18px;
  border: 1px solid #b9c2cf;
  border-radius: 4px;
  appearance: none;
  background: #ffffff;
  display: inline-block;
}

.check-col input:checked {
  background: #0f75ff;
  border-color: #0f75ff;
}

.keyword-col {
  min-width: 300px;
}

.keyword-link {
  color: #0969da;
  text-decoration: none;
  font-weight: 500;
  margin-right: 6px;
}

.keyword-link:hover {
  text-decoration: underline;
}

.expand-icon {
  width: 15px;
  height: 15px;
  border: 2px solid #aab2bf;
  color: #98a2b3;
  border-radius: 50%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  margin-right: 7px;
}

.keyword-mini-icon {
  display: inline-block;
  width: 13px;
  height: 13px;
  border: 2px solid #a4aab6;
  border-radius: 2px;
  position: relative;
  top: 2px;
}

.keyword-mini-icon::before,
.keyword-mini-icon::after {
  content: '';
  position: absolute;
  background: #a4aab6;
}

.keyword-mini-icon::before {
  width: 7px;
  height: 2px;
  top: 2px;
  left: 2px;
}

.keyword-mini-icon::after {
  width: 2px;
  height: 2px;
  top: 6px;
  left: 2px;
}

.num-col {
  text-align: right;
}

.trend-cell {
  text-align: left;
}

.sparkline polygon {
  fill: #dceefe;
}

.sparkline polyline {
  fill: none;
  stroke: #41a0f7;
  stroke-width: 1.5;
  stroke-linejoin: round;
  stroke-linecap: round;
}

.intent-list {
  display: inline-flex;
  gap: 4px;
}

.intent-badge {
  width: 20px;
  height: 20px;
  border-radius: 4px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  font-weight: 700;
}

.intent-i {
  background: #bfe0ff;
  color: #0f72ce;
}

.intent-t {
  background: #a8ecbf;
  color: #0d7b46;
}

.intent-c {
  background: #f7e3a3;
  color: #996500;
}

.intent-n {
  background: #dbcffd;
  color: #6d45c4;
}

.intent-empty {
  color: #9aa3b2;
}

.kd-col {
  display: inline-flex;
  align-items: center;
  justify-content: flex-end;
  gap: 8px;
  width: 100%;
}

.kd-dot {
  width: 15px;
  height: 15px;
  border-radius: 50%;
  display: inline-block;
}

.kd-0 {
  background: #d0d7e2;
}

.kd-1 {
  background: #3dcc95;
}

.kd-2 {
  background: #24b98f;
}

.kd-3 {
  background: #10a383;
}

.kd-4 {
  background: #09806c;
}

.sf-col {
  text-align: left;
  width: 64px;
}

.sf-icon {
  width: 16px;
  height: 16px;
  display: inline-block;
  border: 2px solid #a4aab6;
  border-radius: 2px;
  position: relative;
  margin-right: 5px;
  top: 2px;
}

.sf-icon::before {
  content: '';
  width: 6px;
  height: 6px;
  border: 2px solid #a4aab6;
  border-radius: 50%;
  position: absolute;
  left: 2px;
  top: 2px;
}

.sf-icon::after {
  content: '';
  width: 5px;
  height: 2px;
  background: #a4aab6;
  position: absolute;
  right: 1px;
  bottom: 1px;
  transform: rotate(40deg);
}

.sf-value {
  display: inline-block;
  border-bottom: 1px dotted #808999;
  color: #4b5565;
  line-height: 1;
}

.empty {
  text-align: center;
  color: #7a8596;
  padding: 20px;
}

@media (max-width: 900px) {
  .filename {
    margin-left: 0;
    width: 100%;
  }

  .filters {
    grid-template-columns: 1fr 1fr;
  }
}
</style>
