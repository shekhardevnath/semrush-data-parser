<script setup>
import { computed, ref } from 'vue'
import SemrushKeywordTable from './components/SemrushKeywordTable.vue'
import SemrushKeywordMagicTable from './components/SemrushKeywordMagicTable.vue'

const activeView = ref('magic')

const currentComponent = computed(() => {
  return activeView.value === 'classic' ? SemrushKeywordTable : SemrushKeywordMagicTable
})
</script>

<template>
  <main class="app-shell">
    <div class="view-tabs">
      <button
        type="button"
        class="tab-btn"
        :class="{ active: activeView === 'classic' }"
        @click="activeView = 'classic'"
      >
        SEMrush Table
      </button>
      <button
        type="button"
        class="tab-btn"
        :class="{ active: activeView === 'magic' }"
        @click="activeView = 'magic'"
      >
        Keyword Magic Table
      </button>
    </div>

    <component :is="currentComponent" />
  </main>
</template>

<style scoped>
.app-shell {
  padding: 12px;
}

.view-tabs {
  display: flex;
  gap: 8px;
  margin-bottom: 10px;
}

.tab-btn {
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  background: #ffffff;
  color: #334155;
  padding: 7px 12px;
  font-size: 13px;
  cursor: pointer;
}

.tab-btn.active {
  background: #e8f0ff;
  border-color: #93c5fd;
  color: #1d4ed8;
  font-weight: 600;
}
</style>
