<script setup lang="ts">
import { computed, reactive, ref } from 'vue'

type SortKey = 'keyword' | 'intent' | 'volume' | 'kd' | 'cpc' | 'competition' | 'results'
type SortDir = 'asc' | 'desc'

interface KeywordRow {
  id: number
  keyword: string
  searchVolume: number
  cpc: number
  competition: number
  numberOfResults: number
  trends: number[]
  relatedRelevance: number
  serpFeatures: number[]
  intentCodes: number[]
  kd: number
}

const EXPECTED_HEADERS = [
  'Keyword',
  'Search Volume',
  'CPC',
  'Competition',
  'Number of Results',
  'Trends',
  'Related Relevance',
  'Keywords SERP Features',
  'Intent',
  'Keyword Difficulty Index'
]

const DEMO_DATA = `Keyword;Search Volume;CPC;Competition;Number of Results;Trends;Related Relevance;Keywords SERP Features;Intent;Keyword Difficulty Index
digital marketing tools;27100;4.80;0.73;53400000;0.44,0.49,0.52,0.55,0.58,0.63,0.61,0.67,0.70,0.74,0.76,0.79;0.92;2,4,7;1,3;58
best seo software;12100;6.25;0.82;184000;0.31,0.35,0.40,0.37,0.45,0.48,0.52,0.54,0.56,0.59,0.61,0.63;0.87;1,2,5,9;1;64
what is semrush;5400;1.20;0.24;6900000;0.50,0.52,0.48,0.47,0.49,0.53,0.58,0.62,0.60,0.61,0.64,0.66;0.80;3,6;0;22
local seo checklist;3600;3.05;0.55;980000;0.27,0.29,0.34,0.33,0.39,0.43,0.47,0.45,0.49,0.53,0.57,0.60;0.74;4,8;0,1;39
buy seo audit;1900;9.40;0.88;124000;0.22,0.25,0.28,0.30,0.35,0.37,0.41,0.46,0.50,0.56,0.60,0.64;0.68;1,5,7;3;71
semrush login;14800;0.18;0.05;23000000;0.66,0.64,0.65,0.68,0.71,0.69,0.72,0.74,0.75,0.77,0.79,0.82;0.95;0,7;2;16`

function parseNumber(raw: string, label: string, lineNumber: number, isInt = false): number {
  const normalized = raw.trim().replace(/,/g, '')
  const parsed = isInt ? Number.parseInt(normalized, 10) : Number.parseFloat(normalized)
  if (!Number.isFinite(parsed)) {
    throw new Error(`Line ${lineNumber}: invalid ${label} value "${raw}"`)
  }
  return parsed
}

function parseIntList(raw: string, label: string, lineNumber: number): number[] {
  if (!raw.trim()) return []
  return raw.split(',').map((piece) => {
    const value = Number.parseInt(piece.trim(), 10)
    if (!Number.isFinite(value)) {
      throw new Error(`Line ${lineNumber}: invalid ${label} entry "${piece}"`)
    }
    return value
  })
}

function parseFloatList(raw: string, label: string, lineNumber: number): number[] {
  if (!raw.trim()) return []
  return raw.split(',').map((piece) => {
    const value = Number.parseFloat(piece.trim())
    if (!Number.isFinite(value)) {
      throw new Error(`Line ${lineNumber}: invalid ${label} entry "${piece}"`)
    }
    return value
  })
}

function parseSemrushText(text: string): KeywordRow[] {
  const lines = text
    .split(/\r?\n/)
    .map((line) => line.trim())
    .filter(Boolean)

  if (lines.length < 2) {
    throw new Error('No data rows found. Provide a semicolon-separated file with a header.')
  }

  const headers = lines[0].split(';').map((item) => item.trim())
  const headerValid =
    headers.length === EXPECTED_HEADERS.length &&
    EXPECTED_HEADERS.every((header, index) => headers[index] === header)
  if (!headerValid) {
    throw new Error(`Invalid header. Expected: ${EXPECTED_HEADERS.join(';')}`)
  }

  return lines.slice(1).map((line, rowIndex) => {
    const lineNumber = rowIndex + 2
    const cols = line.split(';')
    if (cols.length !== EXPECTED_HEADERS.length) {
      throw new Error(`Line ${lineNumber}: expected ${EXPECTED_HEADERS.length} columns, got ${cols.length}`)
    }

    const trends = parseFloatList(cols[5], 'Trends', lineNumber)
    if (trends.length !== 12) {
      throw new Error(`Line ${lineNumber}: Trends must have exactly 12 values`)
    }

    return {
      id: rowIndex + 1,
      keyword: cols[0].trim(),
      searchVolume: parseNumber(cols[1], 'Search Volume', lineNumber, true),
      cpc: parseNumber(cols[2], 'CPC', lineNumber),
      competition: parseNumber(cols[3], 'Competition', lineNumber),
      numberOfResults: parseNumber(cols[4], 'Number of Results', lineNumber, true),
      trends: trends.map((value) => Math.max(0, Math.min(1, value))),
      relatedRelevance: parseNumber(cols[6], 'Related Relevance', lineNumber),
      serpFeatures: parseIntList(cols[7], 'Keywords SERP Features', lineNumber).sort((a, b) => a - b),
      intentCodes: parseIntList(cols[8], 'Intent', lineNumber).sort((a, b) => a - b),
      kd: parseNumber(cols[9], 'Keyword Difficulty Index', lineNumber, true)
    }
  })
}

const rows = ref<KeywordRow[]>(parseSemrushText(DEMO_DATA))
const selectedIds = reactive(new Set<number>())
const selectedOnly = ref(false)
const keywordFilter = ref('')
const parseError = ref('')
const loadedFileName = ref('Demo data')
const sortKey = ref<SortKey>('volume')
const sortDir = ref<SortDir>('desc')

const intentMeta: Record<number, { short: string; title: string; cls: string }> = {
  0: { short: 'I', title: 'Informational', cls: 'intent intent-i' },
  1: { short: 'C', title: 'Commercial', cls: 'intent intent-c' },
  2: { short: 'N', title: 'Navigational', cls: 'intent intent-n' },
  3: { short: 'T', title: 'Transactional', cls: 'intent intent-t' }
}

const totalCount = computed(() => rows.value.length)

const filteredRows = computed(() => {
  const query = keywordFilter.value.trim().toLowerCase()
  return rows.value.filter((row) => {
    const matchesKeyword = !query || row.keyword.toLowerCase().includes(query)
    const matchesSelected = !selectedOnly.value || selectedIds.has(row.id)
    return matchesKeyword && matchesSelected
  })
})

const filteredCount = computed(() => filteredRows.value.length)

const allFilteredSelected = computed(() => {
  if (!filteredRows.value.length) return false
  return filteredRows.value.every((row) => selectedIds.has(row.id))
})

function compareIntent(a: number[], b: number[]): number {
  const limit = Math.max(a.length, b.length)
  for (let i = 0; i < limit; i += 1) {
    const av = a[i] ?? -1
    const bv = b[i] ?? -1
    if (av !== bv) return av - bv
  }
  return 0
}

function compareRows(a: KeywordRow, b: KeywordRow): number {
  switch (sortKey.value) {
    case 'keyword':
      return a.keyword.localeCompare(b.keyword)
    case 'intent':
      return compareIntent(a.intentCodes, b.intentCodes)
    case 'volume':
      return a.searchVolume - b.searchVolume
    case 'kd':
      return a.kd - b.kd
    case 'cpc':
      return a.cpc - b.cpc
    case 'competition':
      return a.competition - b.competition
    case 'results':
      return a.numberOfResults - b.numberOfResults
    default:
      return 0
  }
}

const sortedRows = computed(() => {
  return filteredRows.value
    .map((row, index) => ({ row, index }))
    .sort((a, b) => {
      const cmp = compareRows(a.row, b.row)
      if (cmp === 0) return a.index - b.index
      return sortDir.value === 'asc' ? cmp : -cmp
    })
    .map((entry) => entry.row)
})

function setSort(next: SortKey): void {
  if (sortKey.value === next) {
    sortDir.value = sortDir.value === 'asc' ? 'desc' : 'asc'
    return
  }
  sortKey.value = next
  sortDir.value = next === 'keyword' || next === 'intent' ? 'asc' : 'desc'
}

function onFileChange(event: Event): void {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return

  parseError.value = ''
  file
    .text()
    .then((text) => {
      const parsed = parseSemrushText(text)
      rows.value = parsed
      selectedIds.clear()
      loadedFileName.value = file.name
    })
    .catch((error: unknown) => {
      parseError.value = error instanceof Error ? error.message : 'Failed to parse file'
    })
    .finally(() => {
      input.value = ''
    })
}

function toggleAllFiltered(checked: boolean): void {
  if (checked) {
    filteredRows.value.forEach((row) => selectedIds.add(row.id))
  } else {
    filteredRows.value.forEach((row) => selectedIds.delete(row.id))
  }
}

function formatCompact(value: number): string {
  const abs = Math.abs(value)
  const fmt = (num: number) => (num % 1 === 0 ? num.toFixed(0) : num.toFixed(1))
  if (abs >= 1_000_000_000) return `${fmt(value / 1_000_000_000)}B`
  if (abs >= 1_000_000) return `${fmt(value / 1_000_000)}M`
  if (abs >= 1_000) return `${fmt(value / 1_000)}K`
  return new Intl.NumberFormat('en-US').format(value)
}

const intFormat = new Intl.NumberFormat('en-US')
function formatInt(value: number): string {
  return intFormat.format(value)
}

function sparklinePoints(points: number[]): string {
  const width = 60
  const height = 18
  const pad = 1
  const innerW = width - pad * 2
  const innerH = height - pad * 2
  return points
    .map((point, index) => {
      const x = pad + (innerW * index) / Math.max(points.length - 1, 1)
      const y = pad + (1 - point) * innerH
      return `${x.toFixed(2)},${y.toFixed(2)}`
    })
    .join(' ')
}

function kdClass(kd: number): string {
  if (kd <= 14) return 'kd-dot kd-green'
  if (kd <= 29) return 'kd-dot kd-light-green'
  if (kd <= 49) return 'kd-dot kd-orange'
  return 'kd-dot kd-red'
}

function intentBadge(code: number): { short: string; title: string; cls: string } {
  return intentMeta[code] ?? { short: String(code), title: 'Unknown Intent', cls: 'intent intent-unknown' }
}

function featureLabel(featureId: number): string {
  const labels: Record<number, string> = {
    0: 'FS',
    1: 'AD',
    2: 'IMG',
    3: 'VID',
    4: 'FAQ',
    5: 'MAP',
    6: 'REV',
    7: 'SIT',
    8: 'TOP',
    9: 'NWS'
  }
  return labels[featureId] ?? '•'
}
</script>

<template>
  <section class="semrush-table-wrap">
    <div class="controls">
      <label class="upload-btn">
        <input type="file" accept=".txt,text/plain" class="hidden-input" @change="onFileChange" />
        Upload Semrush file
      </label>
      <span class="file-label">{{ loadedFileName }}</span>
      <input
        v-model="keywordFilter"
        type="text"
        class="search-input"
        placeholder="Filter by keyword..."
      />
      <label class="toggle-label">
        <input v-model="selectedOnly" type="checkbox" />
        Selected only
      </label>
    </div>

    <div class="meta">
      <span>Total: {{ totalCount }}</span>
      <span>Filtered: {{ filteredCount }}</span>
      <span>Selected: {{ selectedIds.size }}</span>
    </div>

    <p v-if="parseError" class="error">{{ parseError }}</p>

    <div class="table-shell">
      <table class="semrush-table">
        <thead>
          <tr>
            <th class="col-check">
              <input
                type="checkbox"
                :checked="allFilteredSelected"
                @change="toggleAllFiltered(($event.target as HTMLInputElement).checked)"
              />
            </th>
            <th>
              <button class="th-btn" @click="setSort('keyword')">Keyword</button>
            </th>
            <th>
              <button class="th-btn" @click="setSort('intent')">Intent</button>
            </th>
            <th>
              <button class="th-btn" @click="setSort('volume')">Volume</button>
            </th>
            <th>Trend</th>
            <th>
              <button class="th-btn" @click="setSort('kd')">KD %</button>
            </th>
            <th>
              <button class="th-btn" @click="setSort('cpc')">CPC (AUD)</button>
            </th>
            <th>
              <button class="th-btn" @click="setSort('competition')">Com.</button>
            </th>
            <th>SERP Features</th>
            <th>
              <button class="th-btn" @click="setSort('results')">Results</button>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="row in sortedRows" :key="row.id">
            <td class="col-check">
              <input
                type="checkbox"
                :checked="selectedIds.has(row.id)"
                @change="
                  ($event.target as HTMLInputElement).checked
                    ? selectedIds.add(row.id)
                    : selectedIds.delete(row.id)
                "
              />
            </td>
            <td class="keyword-cell">
              <span class="expand-icon">+</span>
              <a href="#" class="keyword-link" @click.prevent>{{ row.keyword }}</a>
            </td>
            <td>
              <div class="intent-list">
                <span
                  v-for="code in row.intentCodes"
                  :key="`${row.id}-${code}`"
                  :class="intentBadge(code).cls"
                  :title="intentBadge(code).title"
                >
                  {{ intentBadge(code).short }}
                </span>
              </div>
            </td>
            <td>{{ formatInt(row.searchVolume) }}</td>
            <td>
              <svg width="60" height="18" viewBox="0 0 60 18" class="sparkline" aria-hidden="true">
                <polyline :points="sparklinePoints(row.trends)" />
              </svg>
            </td>
            <td>
              <span :class="kdClass(row.kd)" />
              {{ row.kd }}
            </td>
            <td>{{ row.cpc.toFixed(2) }}</td>
            <td>{{ row.competition.toFixed(2) }}</td>
            <td>
              <div class="serp-list">
                <span
                  v-for="featureId in row.serpFeatures"
                  :key="`${row.id}-serp-${featureId}`"
                  class="serp-item"
                  :title="`SERP Feature ${featureId}`"
                >
                  <svg width="18" height="18" viewBox="0 0 18 18" aria-hidden="true">
                    <rect x="1.2" y="1.2" width="15.6" height="15.6" rx="3" />
                    <text x="9" y="11" text-anchor="middle">{{ featureLabel(featureId) }}</text>
                  </svg>
                </span>
              </div>
            </td>
            <td>{{ formatCompact(row.numberOfResults) }}</td>
          </tr>
          <tr v-if="!sortedRows.length">
            <td colspan="10" class="empty-state">No rows match the current filters.</td>
          </tr>
        </tbody>
      </table>
    </div>
  </section>
</template>

<style scoped>
.semrush-table-wrap {
  border: 1px solid #d8dee9;
  border-radius: 10px;
  background: #fff;
  font-family: Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  color: #1f2d3d;
  padding: 14px;
}

.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  align-items: center;
  margin-bottom: 8px;
}

.upload-btn {
  border: 1px solid #c7d2fe;
  background: #eff6ff;
  color: #1d4ed8;
  border-radius: 8px;
  padding: 8px 12px;
  font-weight: 600;
  cursor: pointer;
}

.hidden-input {
  display: none;
}

.file-label {
  color: #64748b;
  font-size: 12px;
}

.search-input {
  min-width: 220px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  padding: 7px 10px;
  font-size: 13px;
}

.toggle-label {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 13px;
}

.meta {
  display: flex;
  gap: 16px;
  font-size: 12px;
  color: #475569;
  margin-bottom: 8px;
}

.error {
  margin: 0 0 10px;
  color: #b91c1c;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  padding: 8px 10px;
  font-size: 13px;
}

.table-shell {
  overflow-x: auto;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
}

.semrush-table {
  width: 100%;
  border-collapse: collapse;
  min-width: 980px;
  font-size: 13px;
}

.semrush-table th,
.semrush-table td {
  padding: 9px 10px;
  border-bottom: 1px solid #eef2f7;
  text-align: left;
  vertical-align: middle;
  white-space: nowrap;
}

.semrush-table thead th {
  background: #f8fafc;
  color: #334155;
  font-weight: 700;
}

.th-btn {
  border: 0;
  background: transparent;
  color: inherit;
  font: inherit;
  font-weight: 700;
  cursor: pointer;
  padding: 0;
}

.col-check {
  width: 36px;
}

.keyword-cell {
  min-width: 260px;
}

.expand-icon {
  display: inline-block;
  width: 18px;
  height: 18px;
  border: 1px solid #cbd5e1;
  border-radius: 50%;
  color: #64748b;
  line-height: 16px;
  text-align: center;
  margin-right: 8px;
  font-weight: 600;
}

.keyword-link {
  color: #2563eb;
  text-decoration: none;
}

.keyword-link:hover {
  text-decoration: underline;
}

.intent-list {
  display: flex;
  gap: 4px;
}

.intent {
  display: inline-flex;
  width: 20px;
  height: 20px;
  border-radius: 999px;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  font-weight: 700;
}

.intent-i {
  background: #dbeafe;
  color: #1d4ed8;
}

.intent-c {
  background: #ffedd5;
  color: #c2410c;
}

.intent-n {
  background: #ede9fe;
  color: #6d28d9;
}

.intent-t {
  background: #dcfce7;
  color: #15803d;
}

.intent-unknown {
  background: #e5e7eb;
  color: #334155;
}

.sparkline polyline {
  fill: none;
  stroke: #3b82f6;
  stroke-width: 1.7;
  stroke-linecap: round;
  stroke-linejoin: round;
}

.kd-dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 999px;
  margin-right: 6px;
  vertical-align: middle;
}

.kd-green {
  background: #10b981;
}

.kd-light-green {
  background: #84cc16;
}

.kd-orange {
  background: #f59e0b;
}

.kd-red {
  background: #ef4444;
}

.serp-list {
  display: flex;
  align-items: center;
  gap: 4px;
}

.serp-item svg {
  display: block;
}

.serp-item rect {
  fill: #f1f5f9;
  stroke: #cbd5e1;
}

.serp-item text {
  fill: #334155;
  font-size: 5.2px;
  font-weight: 700;
  font-family: Inter, sans-serif;
}

.empty-state {
  text-align: center;
  color: #64748b;
  padding: 16px 10px;
}
</style>
