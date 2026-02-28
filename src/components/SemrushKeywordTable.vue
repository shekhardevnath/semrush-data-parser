<script setup lang="ts">
import { computed, onBeforeUnmount, reactive, ref, watch } from 'vue'
import SaveTextModal from './SaveTextModal.vue'

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

const SERP_FEATURE_LABELS: Record<number, string> = {
  0: 'Featured snippet',
  1: 'Sitelinks',
  2: 'Reviews',
  3: 'Video',
  4: 'People also ask',
  5: 'Images',
  6: 'Shopping',
  7: 'Top stories'
}

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
  if (!Number.isFinite(parsed)) throw new Error(`Line ${lineNumber}: invalid ${label} value "${raw}"`)
  return parsed
}

function parseIntList(raw: string, label: string, lineNumber: number): number[] {
  if (!raw.trim()) return []
  return raw.split(',').map((piece) => {
    const value = Number.parseInt(piece.trim(), 10)
    if (!Number.isFinite(value)) throw new Error(`Line ${lineNumber}: invalid ${label} entry "${piece}"`)
    return value
  })
}

function parseFloatList(raw: string, label: string, lineNumber: number): number[] {
  if (!raw.trim()) return []
  return raw.split(',').map((piece) => {
    const value = Number.parseFloat(piece.trim())
    if (!Number.isFinite(value)) throw new Error(`Line ${lineNumber}: invalid ${label} entry "${piece}"`)
    return value
  })
}

function parseSemrushText(text: string): KeywordRow[] {
  const lines = text
    .split(/\r?\n/)
    .map((line) => line.trim())
    .filter(Boolean)

  if (!lines.length) return []

  const headers = lines[0].split(';').map((item) => item.trim())
  const headerValid =
    headers.length === EXPECTED_HEADERS.length &&
    EXPECTED_HEADERS.every((header, index) => headers[index] === header)
  if (!headerValid) throw new Error(`Invalid header. Expected: ${EXPECTED_HEADERS.join(';')}`)

  if (lines.length === 1) return []

  return lines.slice(1).map((line, rowIndex) => {
    const lineNumber = rowIndex + 2
    const cols = line.split(';')
    if (cols.length !== EXPECTED_HEADERS.length) {
      throw new Error(`Line ${lineNumber}: expected ${EXPECTED_HEADERS.length} columns, got ${cols.length}`)
    }
    const trends = parseFloatList(cols[5], 'Trends', lineNumber)
    if (trends.length !== 12) throw new Error(`Line ${lineNumber}: Trends must have exactly 12 values`)

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
const parsedRows = computed(() => rows.value)
const selectedIds = reactive(new Set<number>())
const selectedOnly = ref(false)
const keywordInput = ref('')
const debouncedKeyword = ref('')
let searchDebounce: ReturnType<typeof setTimeout> | undefined

watch(keywordInput, (value) => {
  if (searchDebounce) clearTimeout(searchDebounce)
  searchDebounce = setTimeout(() => {
    debouncedKeyword.value = value
  }, 200)
})
onBeforeUnmount(() => {
  if (searchDebounce) clearTimeout(searchDebounce)
})

const selectedSerpFeatures = reactive(new Set<number>())
const parseError = ref('')
const loadedFileName = ref('Demo data')
const sortKey = ref<SortKey>('volume')
const sortDir = ref<SortDir>('desc')
const showSaveNotesModal = ref(false)

const intentMeta: Record<number, { short: string; title: string; cls: string }> = {
  0: { short: 'I', title: 'Informational', cls: 'intent intent-i' },
  1: { short: 'C', title: 'Commercial', cls: 'intent intent-c' },
  2: { short: 'N', title: 'Navigational', cls: 'intent intent-n' },
  3: { short: 'T', title: 'Transactional', cls: 'intent intent-t' }
}

const allKeywordCount = computed(() => parsedRows.value.length)
const totalVolume = computed(() => parsedRows.value.reduce((sum, row) => sum + row.searchVolume, 0))
const avgKd = computed(() => {
  if (!parsedRows.value.length) return 0
  return Math.round(parsedRows.value.reduce((sum, row) => sum + row.kd, 0) / parsedRows.value.length)
})

const featureStats = computed(() => {
  const counts = new Map<number, number>()
  for (const row of parsedRows.value) {
    const unique = new Set(row.serpFeatures)
    for (const feature of unique) counts.set(feature, (counts.get(feature) ?? 0) + 1)
  }
  return Array.from(counts.entries())
    .map(([id, count]) => ({ id, count, label: SERP_FEATURE_LABELS[id] ?? `SERP Feature ${id}` }))
    .sort((a, b) => b.count - a.count || a.id - b.id)
})

const uniqueFeatureCount = computed(() => featureStats.value.length)

const filteredRows = computed(() => {
  const query = debouncedKeyword.value.trim().toLowerCase()
  const activeFeatures = Array.from(selectedSerpFeatures)
  return parsedRows.value.filter((row) => {
    const matchesKeyword = !query || row.keyword.toLowerCase().includes(query)
    const matchesSelected = !selectedOnly.value || selectedIds.has(row.id)
    const matchesFeatures =
      !activeFeatures.length || activeFeatures.every((feature) => row.serpFeatures.includes(feature))
    return matchesKeyword && matchesSelected && matchesFeatures
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

const visibleRows = computed(() => sortedRows.value)

function setSort(next: SortKey): void {
  if (sortKey.value !== next) {
    sortKey.value = next
    sortDir.value = 'asc'
    return
  }
  sortDir.value = sortDir.value === 'asc' ? 'desc' : 'asc'
}

function isSortActive(key: SortKey): boolean {
  return sortKey.value === key
}

function onFileChange(event: Event): void {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return

  parseError.value = ''
  file
    .text()
    .then((text) => {
      rows.value = parseSemrushText(text)
      selectedIds.clear()
      selectedSerpFeatures.clear()
      keywordInput.value = ''
      debouncedKeyword.value = ''
      loadedFileName.value = file.name
    })
    .catch((error: unknown) => {
      parseError.value = error instanceof Error ? error.message : 'Failed to parse file'
    })
    .finally(() => {
      input.value = ''
    })
}

function toggleAllVisible(checked: boolean): void {
  if (checked) visibleRows.value.forEach((row) => selectedIds.add(row.id))
  else visibleRows.value.forEach((row) => selectedIds.delete(row.id))
}

function clearSelection(): void {
  selectedIds.clear()
}

const intFormat = new Intl.NumberFormat('en-US')
function formatNumber(value: number): string {
  return intFormat.format(value)
}

function formatCompact(value: number): string {
  const abs = Math.abs(value)
  const trim = (num: number) => (num % 1 === 0 ? num.toFixed(0) : num.toFixed(1))
  if (abs >= 1_000_000_000) return `${trim(value / 1_000_000_000)}B`
  if (abs >= 1_000_000) return `${trim(value / 1_000_000)}M`
  if (abs >= 1_000) return `${trim(value / 1_000)}K`
  return formatNumber(value)
}

function sparklinePoints(points: number[]): string {
  const width = 70
  const height = 20
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
    1: 'SL',
    2: 'RV',
    3: 'VD',
    4: 'PAA',
    5: 'IMG',
    6: 'SHP',
    7: 'TOP',
    8: 'MAP',
    9: 'AD'
  }
  return labels[featureId] ?? '*'
}

function featureName(featureId: number): string {
  return SERP_FEATURE_LABELS[featureId] ?? `SERP Feature ${featureId}`
}

function rowSerpVisible(featureIds: number[]): number[] {
  return featureIds.slice(0, 6)
}

function rowSerpOverflow(featureIds: number[]): number {
  return Math.max(0, featureIds.length - 6)
}

function toggleFeatureFilter(featureId: number): void {
  if (selectedSerpFeatures.has(featureId)) selectedSerpFeatures.delete(featureId)
  else selectedSerpFeatures.add(featureId)
}

function clearFilters(): void {
  keywordInput.value = ''
  debouncedKeyword.value = ''
  selectedOnly.value = false
  selectedSerpFeatures.clear()
}

function onNotesSaved(payload: { filename: string; fileSize: number }): void {
  console.log('Notes saved:', payload)
}
</script>

<template>
  <section class="semrush-wrap">
    <div class="controls">
      <label class="upload-btn">
        <input type="file" accept=".txt,text/plain" class="hidden-input" @change="onFileChange" />
        Upload Semrush file
      </label>
      <span class="file-label">{{ loadedFileName }}</span>
      <label class="toggle-label">
        <input v-model="selectedOnly" type="checkbox" />
        Selected only
      </label>
      <button class="clear-btn" @click="clearFilters">Clear filters</button>
      <button class="save-notes-btn" @click="showSaveNotesModal = true">Save notes</button>
    </div>

    <div v-if="selectedSerpFeatures.size" class="chips">
      <button
        v-for="featureId in Array.from(selectedSerpFeatures).sort((a, b) => a - b)"
        :key="`chip-${featureId}`"
        class="chip"
        @click="toggleFeatureFilter(featureId)"
      >
        SERP: {{ featureName(featureId) }} <span>&times;</span>
      </button>
    </div>

    <p v-if="parseError" class="error">{{ parseError }}</p>

    <div class="layout">
      <div class="main-panel">
        <div class="toolbar">
          <input
            v-model="keywordInput"
            type="text"
            class="search-input"
            placeholder="Search keyword..."
          />
          <div class="toolbar-stats">
            <span>All keywords: {{ formatNumber(allKeywordCount) }}</span>
            <span>|</span>
            <span>Total Volume: {{ formatCompact(totalVolume) }}</span>
            <span>|</span>
            <span>Avg KD: {{ avgKd }}%</span>
            <span>|</span>
            <span>Filtered: {{ formatNumber(filteredCount) }}</span>
            <template v-if="selectedIds.size">
              <span>|</span>
              <span>Selected: {{ selectedIds.size }}</span>
              <button class="link-btn" @click="clearSelection">Clear selection</button>
            </template>
          </div>
        </div>

        <div class="table-shell">
          <table class="table">
            <thead>
              <tr>
                <th class="w-check">
                  <input
                    type="checkbox"
                    :checked="allFilteredSelected"
                    @change="toggleAllVisible(($event.target as HTMLInputElement).checked)"
                  />
                </th>
                <th class="w-keyword">
                  <button class="th-btn" @click="setSort('keyword')">
                    Keyword
                    <span class="sort-icon" :class="{ active: isSortActive('keyword') }">
                      {{ isSortActive('keyword') ? (sortDir === 'asc' ? '▲' : '▼') : '↕' }}
                    </span>
                  </button>
                </th>
                <th class="w-intent center">
                  <button class="th-btn" @click="setSort('intent')">
                    Intent
                    <span class="sort-icon" :class="{ active: isSortActive('intent') }">
                      {{ isSortActive('intent') ? (sortDir === 'asc' ? '▲' : '▼') : '↕' }}
                    </span>
                  </button>
                </th>
                <th class="w-volume right">
                  <button class="th-btn" @click="setSort('volume')">
                    Volume
                    <span class="sort-icon" :class="{ active: isSortActive('volume') }">
                      {{ isSortActive('volume') ? (sortDir === 'asc' ? '▲' : '▼') : '↕' }}
                    </span>
                  </button>
                </th>
                <th class="w-trend center">Trend</th>
                <th class="w-kd right">
                  <button class="th-btn" @click="setSort('kd')">
                    KD %
                    <span class="sort-icon" :class="{ active: isSortActive('kd') }">
                      {{ isSortActive('kd') ? (sortDir === 'asc' ? '▲' : '▼') : '↕' }}
                    </span>
                  </button>
                </th>
                <th class="w-cpc right">
                  <button class="th-btn" @click="setSort('cpc')">
                    CPC (AUD)
                    <span class="sort-icon" :class="{ active: isSortActive('cpc') }">
                      {{ isSortActive('cpc') ? (sortDir === 'asc' ? '▲' : '▼') : '↕' }}
                    </span>
                  </button>
                </th>
                <th class="w-com right">
                  <button class="th-btn" @click="setSort('competition')">
                    Com.
                    <span class="sort-icon" :class="{ active: isSortActive('competition') }">
                      {{ isSortActive('competition') ? (sortDir === 'asc' ? '▲' : '▼') : '↕' }}
                    </span>
                  </button>
                </th>
                <th class="w-serp">SERP Features</th>
                <th class="w-results right">
                  <button class="th-btn" @click="setSort('results')">
                    Results
                    <span class="sort-icon" :class="{ active: isSortActive('results') }">
                      {{ isSortActive('results') ? (sortDir === 'asc' ? '▲' : '▼') : '↕' }}
                    </span>
                  </button>
                </th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in visibleRows" :key="row.id" class="row">
                <td class="w-check">
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
                <td class="w-keyword">
                  <span class="expand-icon">+</span>
                  <a href="#" class="keyword-link" @click.prevent>{{ row.keyword }}</a>
                </td>
                <td class="w-intent center">
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
                <td class="w-volume right">{{ formatNumber(row.searchVolume) }}</td>
                <td class="w-trend center">
                  <svg width="70" height="20" viewBox="0 0 70 20" class="sparkline" aria-hidden="true">
                    <line x1="0" y1="10" x2="70" y2="10" />
                    <polyline :points="sparklinePoints(row.trends)" />
                  </svg>
                </td>
                <td class="w-kd right"><span>{{ row.kd }}</span><span :class="kdClass(row.kd)" /></td>
                <td class="w-cpc right">{{ row.cpc.toFixed(2) }}</td>
                <td class="w-com right">{{ row.competition.toFixed(2) }}</td>
                <td class="w-serp">
                  <div class="serp-list">
                    <span
                      v-for="featureId in rowSerpVisible(row.serpFeatures)"
                      :key="`${row.id}-serp-${featureId}`"
                      :title="featureName(featureId)"
                    >
                      <svg width="16" height="16" viewBox="0 0 18 18" aria-hidden="true">
                        <rect x="1.2" y="1.2" width="15.6" height="15.6" rx="3" />
                        <text x="9" y="11" text-anchor="middle">{{ featureLabel(featureId) }}</text>
                      </svg>
                    </span>
                    <span
                      v-if="rowSerpOverflow(row.serpFeatures) > 0"
                      class="more-chip"
                      :title="`${rowSerpOverflow(row.serpFeatures)} more features`"
                    >
                      +{{ rowSerpOverflow(row.serpFeatures) }}
                    </span>
                  </div>
                </td>
                <td class="w-results right">{{ formatCompact(row.numberOfResults) }}</td>
              </tr>
              <tr v-if="!visibleRows.length && !parseError">
                <td colspan="10" class="empty-state">
                  {{ allKeywordCount === 0 ? 'No data loaded yet.' : 'No keywords match your filters.' }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <aside class="sidebar">
        <div class="sidebar-head">
          <h3>SERP Features</h3>
          <span>Unique features: {{ uniqueFeatureCount }}</span>
        </div>
        <div v-if="featureStats.length" class="legend-list">
          <button
            v-for="feature in featureStats"
            :key="`legend-${feature.id}`"
            class="legend-item"
            :class="{ active: selectedSerpFeatures.has(feature.id) }"
            @click="toggleFeatureFilter(feature.id)"
          >
            <svg width="16" height="16" viewBox="0 0 18 18" aria-hidden="true">
              <rect x="1.2" y="1.2" width="15.6" height="15.6" rx="3" />
              <text x="9" y="11" text-anchor="middle">{{ featureLabel(feature.id) }}</text>
            </svg>
            <span class="legend-label">{{ feature.label }}</span>
            <span class="legend-count">{{ feature.count }}</span>
          </button>
        </div>
        <p v-else class="legend-empty">No SERP features found in the current dataset.</p>
      </aside>
    </div>

    <SaveTextModal
      v-model="showSaveNotesModal"
      title="Save notes"
      default-filename="semrush-notes.txt"
      placeholder="Write notes for this keyword set..."
      @saved="onNotesSaved"
    />
  </section>
</template>

<style scoped>
.semrush-wrap {
  border: 1px solid #dbe3ee;
  border-radius: 10px;
  background: #f8fafc;
  font-family: Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  color: #1f2d3d;
  margin: 0;
  padding: 12px;
}

.controls,
.chips {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  align-items: center;
  margin-bottom: 8px;
}

.upload-btn {
  border: 1px solid #c7d2fe;
  background: #eff6ff;
  color: #1d4ed8;
  border-radius: 7px;
  padding: 6px 10px;
  font-weight: 600;
  font-size: 12px;
  cursor: pointer;
}

.hidden-input {
  display: none;
}

.file-label {
  color: #64748b;
  font-size: 12px;
}

.toggle-label {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 12px;
}

.clear-btn,
.link-btn {
  border: 1px solid #d1d5db;
  background: #fff;
  border-radius: 7px;
  padding: 5px 9px;
  font-size: 12px;
  cursor: pointer;
}

.clear-btn:hover,
.link-btn:hover {
  background: #f1f5f9;
}

.clear-btn {
  border-color: #f59e0b;
}

.save-notes-btn {
  margin-left: auto;
  border: 1px solid #059669;
  background: #10b981;
  color: #ffffff;
  border-radius: 7px;
  padding: 5px 10px;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
}

.save-notes-btn:hover {
  background: #059669;
}

.chip {
  border: 1px solid #bfdbfe;
  background: #eff6ff;
  color: #1d4ed8;
  border-radius: 999px;
  padding: 3px 9px;
  font-size: 11px;
  cursor: pointer;
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

.layout {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.main-panel {
  min-width: 0;
}

.toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
  margin-bottom: 8px;
}

.search-input {
  width: 210px;
  border: 1px solid #d1d5db;
  border-radius: 7px;
  padding: 6px 9px;
  font-size: 12px;
  background: #fff;
}

.toolbar-stats {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  font-size: 12px;
  color: #475569;
  align-items: center;
}

.table-shell {
  overflow-x: auto;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  background: #fff;
  max-height: 70vh;
}

.table {
  width: 100%;
  min-width: 1030px;
  border-collapse: separate;
  border-spacing: 0;
  font-size: 12px;
}

.table th,
.table td {
  padding: 6px 8px;
  border-bottom: 1px solid #eef2f7;
  white-space: nowrap;
  line-height: 1.25;
}

.table thead th {
  position: sticky;
  top: 0;
  z-index: 2;
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
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.sort-icon {
  font-size: 10px;
  color: #94a3b8;
}

.sort-icon.active {
  color: #334155;
}

.row:hover {
  background: #f8fbff;
}

.w-check {
  width: 36px;
  min-width: 36px;
  text-align: center;
}

.w-keyword {
  min-width: 260px;
}

.w-intent {
  width: 70px;
  min-width: 70px;
}

.w-volume {
  width: 90px;
  min-width: 90px;
}

.w-trend {
  width: 90px;
  min-width: 90px;
}

.w-kd {
  width: 80px;
  min-width: 80px;
}

.w-cpc {
  width: 95px;
  min-width: 95px;
}

.w-com {
  width: 80px;
  min-width: 80px;
}

.w-serp {
  width: 170px;
  min-width: 170px;
}

.w-results {
  width: 110px;
  min-width: 110px;
}

.right {
  text-align: right;
}

.center {
  text-align: center;
}

.expand-icon {
  display: inline-flex;
  width: 16px;
  height: 16px;
  border: 1px solid #cbd5e1;
  border-radius: 50%;
  color: #64748b;
  align-items: center;
  justify-content: center;
  margin-right: 6px;
  font-size: 11px;
  font-weight: 600;
  vertical-align: middle;
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
  justify-content: center;
  gap: 3px;
}

.intent {
  display: inline-flex;
  width: 18px;
  height: 18px;
  border-radius: 999px;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 700;
}

.intent-i { background: #dbeafe; color: #1d4ed8; }
.intent-c { background: #ffedd5; color: #c2410c; }
.intent-n { background: #ede9fe; color: #6d28d9; }
.intent-t { background: #dcfce7; color: #15803d; }
.intent-unknown { background: #e5e7eb; color: #334155; }

.sparkline line {
  stroke: #e5e7eb;
  stroke-width: 1;
}

.sparkline polyline {
  fill: none;
  stroke: #3b82f6;
  stroke-width: 1.6;
  stroke-linecap: round;
  stroke-linejoin: round;
}

.w-kd .kd-dot {
  display: inline-block;
  width: 7px;
  height: 7px;
  border-radius: 999px;
  margin-left: 6px;
  vertical-align: middle;
}

.kd-green { background: #10b981; }
.kd-light-green { background: #84cc16; }
.kd-orange { background: #f59e0b; }
.kd-red { background: #ef4444; }

.serp-list {
  display: flex;
  align-items: center;
  gap: 4px;
}

.serp-list rect,
.legend-item rect {
  fill: #f1f5f9;
  stroke: #cbd5e1;
}

.serp-list text,
.legend-item text {
  fill: #334155;
  font-size: 5.2px;
  font-weight: 700;
  font-family: Inter, sans-serif;
}

.more-chip {
  border-radius: 999px;
  background: #f1f5f9;
  color: #475569;
  font-size: 10px;
  padding: 1px 6px;
}

.empty-state {
  text-align: center;
  color: #64748b;
  padding: 16px 10px;
}

.sidebar {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  background: #fff;
  padding: 10px;
}

.sidebar-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid #f1f5f9;
  padding-bottom: 8px;
  margin-bottom: 8px;
}

.sidebar-head h3 {
  margin: 0;
  font-size: 13px;
}

.sidebar-head span {
  font-size: 11px;
  color: #64748b;
}

.legend-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.legend-item {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 8px;
  border: 1px solid #e2e8f0;
  border-radius: 7px;
  background: #fff;
  padding: 5px 7px;
  cursor: pointer;
  text-align: left;
}

.legend-item:hover {
  background: #f8fafc;
}

.legend-item.active {
  border-color: #bfdbfe;
  background: #eff6ff;
}

.legend-label {
  flex: 1;
  font-size: 12px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.legend-count {
  border-radius: 999px;
  background: #f1f5f9;
  padding: 1px 7px;
  font-size: 11px;
  color: #475569;
}

.legend-empty {
  font-size: 12px;
  color: #64748b;
}

@media (min-width: 1024px) {
  .layout {
    flex-direction: row;
    align-items: flex-start;
  }

  .main-panel {
    flex: 1;
  }

  .sidebar {
    width: 280px;
    position: sticky;
    top: 16px;
  }
}

@media (max-width: 820px) {
  .save-notes-btn {
    margin-left: 0;
  }

  .toolbar {
    flex-direction: column;
    align-items: flex-start;
  }

  .search-input {
    width: 100%;
  }
}
</style>
