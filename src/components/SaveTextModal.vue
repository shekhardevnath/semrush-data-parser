<template>
  <teleport to="body">
    <div
      v-if="modelValue"
      class="modal-overlay"
      role="dialog"
      aria-modal="true"
      aria-labelledby="save-text-modal-title"
      @click.self="handleCancel"
    >
      <div class="modal-card">
        <h2 id="save-text-modal-title" class="modal-title">{{ title }}</h2>
        <textarea
          ref="textareaRef"
          v-model="text"
          class="modal-textarea"
          :placeholder="placeholder"
          @keydown.esc.prevent="handleCancel"
        />
        <div class="modal-actions">
          <button type="button" class="btn btn-secondary" @click="handleCancel">Cancel</button>
          <button type="button" class="btn btn-primary" :disabled="isSaveDisabled || isSaving" @click="handleSave">
            {{ isSaving ? 'Saving...' : 'Save' }}
          </button>
        </div>
      </div>
    </div>
  </teleport>
</template>

<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, ref, watch } from 'vue'

interface SavedPayload {
  filename: string
  fileSize: number
}

const props = withDefaults(
  defineProps<{
    modelValue: boolean
    title?: string
    placeholder?: string
    defaultFilename?: string
  }>(),
  {
    title: 'Save notes',
    placeholder: 'Write your notes here...',
    defaultFilename: 'notes.txt'
  }
)

const emit = defineEmits<{
  (e: 'update:modelValue', value: boolean): void
  (e: 'saved', payload: SavedPayload): void
}>()

const text = ref('')
const textareaRef = ref<HTMLTextAreaElement | null>(null)
const isSaving = ref(false)

const isSaveDisabled = computed(() => !text.value.trim())

function onWindowKeydown(event: KeyboardEvent): void {
  if (event.key === 'Escape' && props.modelValue) {
    event.preventDefault()
    handleCancel()
  }
}

function hasFilePicker(): boolean {
  return typeof window !== 'undefined' && 'showSaveFilePicker' in window
}

function ensureTxtFilename(raw: string): string {
  const base = raw.trim() || 'notes.txt'
  return base.toLowerCase().endsWith('.txt') ? base : `${base}.txt`
}

async function saveWithFilePicker(content: string): Promise<string> {
  const picker = window.showSaveFilePicker as (options: {
    suggestedName: string
    types: { description: string; accept: Record<string, string[]> }[]
  }) => Promise<FileSystemFileHandle>

  const handle = await picker({
    suggestedName: ensureTxtFilename(props.defaultFilename),
    types: [{ description: 'Text file', accept: { 'text/plain': ['.txt'] } }]
  })

  const writable = await handle.createWritable()
  await writable.write(content)
  await writable.close()

  return ensureTxtFilename(handle.name)
}

function saveWithDownload(content: string): string {
  const filename = ensureTxtFilename(props.defaultFilename)
  const blob = new Blob([content], { type: 'text/plain;charset=utf-8' })
  const url = URL.createObjectURL(blob)

  const link = document.createElement('a')
  link.href = url
  link.download = filename
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
  URL.revokeObjectURL(url)

  return filename
}

async function handleSave(): Promise<void> {
  if (isSaveDisabled.value || isSaving.value) return

  const content = text.value
  const fileSize = new Blob([content], { type: 'text/plain;charset=utf-8' }).size

  isSaving.value = true
  try {
    let filename = ensureTxtFilename(props.defaultFilename)

    if (hasFilePicker()) {
      filename = await saveWithFilePicker(content)
    } else {
      filename = saveWithDownload(content)
    }

    emit('saved', { filename, fileSize })
    text.value = ''
    emit('update:modelValue', false)
  } catch (error) {
    if ((error as DOMException)?.name !== 'AbortError') {
      console.error('Failed to save text file:', error)
    }
  } finally {
    isSaving.value = false
  }
}

function closeModal(): void {
  emit('update:modelValue', false)
}

function handleCancel(): void {
  if (!text.value.trim()) {
    closeModal()
    return
  }

  const shouldClose = window.confirm('Discard unsaved notes?')
  if (shouldClose) {
    text.value = ''
    closeModal()
  }
}

watch(
  () => props.modelValue,
  async (isOpen) => {
    if (isOpen) {
      window.addEventListener('keydown', onWindowKeydown)
      await nextTick()
      textareaRef.value?.focus()
      return
    }
    window.removeEventListener('keydown', onWindowKeydown)
  }
)

onBeforeUnmount(() => {
  window.removeEventListener('keydown', onWindowKeydown)
})
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.48);
  display: grid;
  place-items: center;
  padding: 16px;
  z-index: 1000;
}

.modal-card {
  width: min(640px, 100%);
  background: #ffffff;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  box-shadow: 0 18px 40px rgba(15, 23, 42, 0.18);
  padding: 14px;
}

.modal-title {
  margin: 0 0 10px;
  font-size: 16px;
  color: #0f172a;
}

.modal-textarea {
  width: 100%;
  min-height: 200px;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  padding: 10px;
  font: inherit;
  resize: vertical;
}

.modal-textarea:focus {
  outline: 2px solid #bfdbfe;
  border-color: #60a5fa;
}

.modal-actions {
  margin-top: 10px;
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}

.btn {
  border-radius: 7px;
  border: 1px solid #cbd5e1;
  padding: 6px 12px;
  font-size: 13px;
  cursor: pointer;
}

.btn-secondary {
  background: #ffffff;
  color: #334155;
}

.btn-primary {
  background: #1d4ed8;
  border-color: #1d4ed8;
  color: #ffffff;
}

.btn:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

@media (max-width: 640px) {
  .modal-card {
    padding: 12px;
  }

  .modal-textarea {
    min-height: 160px;
  }
}
</style>
