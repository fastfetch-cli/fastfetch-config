<script setup lang="ts">
import { computed, onMounted, reactive, ref, watch } from 'vue'
import { parse, printParseErrorCode, type ParseError } from 'jsonc-parser'
import SidebarPanel from './components/SidebarPanel.vue'
import SettingsPane from './components/SettingsPane.vue'
import PreviewPane from './components/PreviewPane.vue'

type Config = Record<string, any>

const STORAGE_KEY = 'fastfetch-config-editor-config'

const schemaUrl = 'https://raw.githubusercontent.com/fastfetch-cli/fastfetch/refs/heads/dev/doc/json_schema.json'
const schema = ref<Record<string, any> | null>(null)
const config = reactive<Config>(loadSavedConfig())
const activeSection = ref('general')
const search = ref('')
const sidebarCollapsed = ref(false)
const previewCollapsed = ref(false)
const status = ref('Loading the latest Fastfetch schema…')
const error = ref('')
const fileInput = ref<HTMLInputElement | null>(null)

const sections = [
  { id: 'general', label: 'General', icon: '⚙' },
  { id: 'display', label: 'Display', icon: '◫' },
  { id: 'logo', label: 'Logo', icon: '◇' },
  { id: 'modules', label: 'Modules', icon: '☷' },
]

const preview = computed(() => JSON.stringify(config, null, 2))

function loadSavedConfig(): Config {
  try {
    const saved = localStorage.getItem(STORAGE_KEY)
    if (saved) {
      const parsed = JSON.parse(saved)
      if (parsed && typeof parsed === 'object' && !Array.isArray(parsed)) {
        if (!parsed.$schema) parsed.$schema = schemaUrl
        return parsed
      }
    }
  } catch { /* ignore corrupt data */ }
  return { $schema: schemaUrl }
}

function saveConfig() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(config))
}

watch(config, saveConfig, { deep: true })

function importConfig() {
  fileInput.value?.click()
}

async function readConfig(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (!file) return
  const errors: ParseError[] = []
  const parsed = parse(await file.text(), errors, { allowTrailingComma: true, disallowComments: false })
  if (errors.length || !parsed || typeof parsed !== 'object' || Array.isArray(parsed)) {
    error.value = errors.length
      ? `Could not parse ${file.name}: ${printParseErrorCode(errors[0].error)}.`
      : `Could not parse ${file.name}: the root value must be an object.`
    return
  }
  Object.keys(config).forEach((key) => delete config[key])
  Object.assign(config, parsed)
  if (!config.$schema) config.$schema = schemaUrl
  error.value = ''
  status.value = `Imported ${file.name}`
  ;(event.target as HTMLInputElement).value = ''
}

function newConfig() {
  Object.keys(config).forEach((key) => delete config[key])
  config.$schema = schemaUrl
  localStorage.removeItem(STORAGE_KEY)
  error.value = ''
  status.value = 'Started a new configuration'
}

function downloadConfig() {
  const contents = `// Generated with Fastfetch Config Editor\n${preview.value}\n`
  const blob = new Blob([contents], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const anchor = document.createElement('a')
  anchor.href = url
  anchor.download = 'config.jsonc'
  anchor.click()
  URL.revokeObjectURL(url)
  status.value = 'Downloaded config.jsonc'
}

function onUpdateConfigSection(key: string, value: unknown) {
  config[key] = value
}

function onRemoveConfigSection(key: string) {
  delete config[key]
}

onMounted(async () => {
  try {
    const response = await fetch(schemaUrl)
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    schema.value = await response.json()
    status.value = 'Schema loaded from fastfetch-cli/fastfetch · dev'
  } catch (reason) {
    error.value = `Unable to load the remote schema (${reason instanceof Error ? reason.message : 'unknown error'}). Check your network connection and try again.`
  }
})
</script>

<template>
  <main class="settings-shell">
    <header class="topbar">
      <div class="brand"><span class="brand-mark">›_</span><span>Fastfetch Config Editor</span></div>
      <div class="topbar-actions">
        <button class="secondary-button" type="button" @click="newConfig">New</button>
        <button class="secondary-button" type="button" @click="importConfig">Import JSONC</button>
        <button class="primary-button" type="button" @click="downloadConfig">Download config.jsonc</button>
        <button
          v-if="previewCollapsed"
          class="secondary-button"
          type="button"
          title="Show preview"
          @click="previewCollapsed = false"
        >◀ JSON</button>
        <input ref="fileInput" class="sr-only" type="file" accept=".json,.jsonc,application/json" @change="readConfig" />
      </div>
    </header>

    <div class="workspace" :class="{ 'sidebar-collapsed': sidebarCollapsed, 'preview-collapsed': previewCollapsed }">
      <SidebarPanel
        :active-section="activeSection"
        :search="search"
        :sections="sections"
        :schema-url="schemaUrl"
        :collapsed="sidebarCollapsed"
        @update:active-section="activeSection = $event"
        @update:search="search = $event"
        @toggle-collapse="sidebarCollapsed = !sidebarCollapsed"
      />
      <SettingsPane
        :schema="schema"
        :config="config"
        :active-section="activeSection"
        :sections="sections"
        :search="search"
        :error="error"
        :status="status"
        @update:config-section="onUpdateConfigSection"
        @remove:config-section="onRemoveConfigSection"
      />
      <PreviewPane
        v-show="!previewCollapsed"
        :config-json="preview"
        :error="error"
        :status="status"
        @toggle-collapse="previewCollapsed = !previewCollapsed"
      />
    </div>
  </main>
</template>

<style scoped lang="scss">
$tablet: 1060px;
$mobile: 720px;
:global(html),
:global(body),
:global(#app) { height: 100%; overflow: hidden; }

.settings-shell { display: flex; flex-direction: column; height: 100vh; height: 100dvh; min-height: 0; background: var(--canvas); overflow: hidden; }
.topbar { flex: 0 0 auto; display: flex; align-items: center; justify-content: space-between; gap: 20px; height: 54px; padding: 0 22px; color: var(--text); background: var(--surface); border-bottom: 1px solid var(--border);
  &-actions { display: flex; align-items: center; gap: 8px; }
}
.brand { display: flex; align-items: center; gap: 10px; font-size: 14px; font-weight: 600; white-space: nowrap;
  &-mark { color: var(--accent); font: 700 20px/1 var(--mono); letter-spacing: -4px; }
}
.workspace { flex: 1; display: grid; grid-template-columns: 238px minmax(450px, 1fr) minmax(320px, .75fr); min-height: 0; overflow: hidden; transition: grid-template-columns .2s; }
.workspace.sidebar-collapsed { grid-template-columns: 52px minmax(450px, 1fr) minmax(320px, .75fr); }
.workspace.preview-collapsed { grid-template-columns: 238px 1fr; }
.workspace.sidebar-collapsed.preview-collapsed { grid-template-columns: 52px 1fr; }

.sr-only { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; }

@media (max-width: $tablet) {
  .workspace { grid-template-columns: 208px minmax(0, 1fr); grid-template-rows: minmax(0, 1fr) 330px; }
  .workspace.sidebar-collapsed { grid-template-columns: 52px minmax(0, 1fr); grid-template-rows: minmax(0, 1fr) 330px; }
  .workspace.preview-collapsed { grid-template-columns: 208px 1fr; grid-template-rows: minmax(0, 1fr); }
  .workspace.sidebar-collapsed.preview-collapsed { grid-template-columns: 52px 1fr; }
}
@media (max-width: $mobile) {
  .topbar { flex-direction: column; align-items: flex-start; height: auto; min-height: 54px; padding: 10px 14px;
    &-actions { width: 100%; padding-bottom: 1px; overflow-x: auto;
      button { white-space: nowrap; }
    }
  }
  .workspace { display: grid; grid-template-columns: minmax(0, 1fr); grid-template-rows: auto minmax(0, 1fr) 280px; min-height: 0; }
  .workspace.preview-collapsed { grid-template-columns: 1fr; grid-template-rows: auto minmax(0, 1fr); }
}
</style>
