<script setup lang="ts">
import { computed, onMounted, reactive, ref } from 'vue'
import { parse, printParseErrorCode, type ParseError } from 'jsonc-parser'
import SchemaField from './components/SchemaField.vue'
import MarkdownDescription from './components/MarkdownDescription.vue'

type Schema = Record<string, any>
type Config = Record<string, any>

const schemaUrl = 'https://raw.githubusercontent.com/fastfetch-cli/fastfetch/refs/heads/dev/doc/json_schema.json'
const schema = ref<Schema | null>(null)
const config = reactive<Config>({ $schema: schemaUrl })
const activeSection = ref('general')
const search = ref('')
const status = ref('Loading the latest Fastfetch schema…')
const error = ref('')
const fileInput = ref<HTMLInputElement | null>(null)
const selectedModuleType = ref('title')
const collapsedModules = ref(new Set<number>())

const sections = [
  { id: 'general', label: 'General', icon: '⚙' },
  { id: 'display', label: 'Display', icon: '◫' },
  { id: 'logo', label: 'Logo', icon: '◇' },
  { id: 'modules', label: 'Modules', icon: '☷' },
]

const sectionSchema = computed(() => schema.value?.properties?.[activeSection.value] ?? {})
const preview = computed(() => JSON.stringify(config, null, 2))
const highlightedPreviewLines = computed(() => preview.value
  .split('\n')
  .map((line) => highlightJsonc(line)))
const moduleOptions = computed<Schema[]>(() => {
  const itemSchema = schema.value?.properties?.modules?.items
  const objectChoice = itemSchema?.anyOf?.find((choice: Schema) => choice.type === 'object')
  return objectChoice?.oneOf ?? []
})
const filteredModuleOptions = computed(() => moduleOptions.value.filter((option) => {
  const label = String(option.title ?? option.properties?.type?.const ?? '')
  return label.toLowerCase().includes(search.value.toLowerCase())
}))

function setActiveSection(id: string) {
  activeSection.value = id
}

function setSection(value: unknown) {
  config[activeSection.value] = value
}

function removeSection() {
  delete config[activeSection.value]
}

function schemaForModule(module: Config): Schema {
  const matching = moduleOptions.value.find((option) => option.properties?.type?.const === module.type)
  if (!matching) return { type: 'object', properties: {} }
  const { type: _type, ...properties } = matching.properties ?? {}
  return { ...matching, properties }
}

function moduleTitle(module: unknown) {
  if (typeof module === 'string') return module
  const type = (module as Config).type
  return moduleOptions.value.find((option) => option.properties?.type?.const === type)?.title ?? type
}

function addModule() {
  if (!config.modules) config.modules = []
  config.modules.push({ type: selectedModuleType.value })
}

function moduleIndex(index: number | string) {
  return Number(index)
}

function isModuleCollapsed(index: number | string) {
  return collapsedModules.value.has(moduleIndex(index))
}

function toggleModule(index: number | string) {
  const position = moduleIndex(index)
  const next = new Set(collapsedModules.value)
  if (next.has(position)) next.delete(position)
  else next.add(position)
  collapsedModules.value = next
}

function updateModule(index: number | string, value: Config) {
  const position = moduleIndex(index)
  config.modules[position] = { type: config.modules[position].type, ...value }
}

function removeModule(index: number | string) {
  config.modules.splice(moduleIndex(index), 1)
}

function moveModule(index: number | string, direction: number) {
  const position = moduleIndex(index)
  const destination = position + direction
  if (destination < 0 || destination >= config.modules.length) return
  const [module] = config.modules.splice(position, 1)
  config.modules.splice(destination, 0, module)
}

function convertModule(index: number | string) {
  const position = moduleIndex(index)
  config.modules[position] = { type: config.modules[position] }
}

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

function escapeHtml(value: string) {
  return value.replace(/[&<>"]/g, (character) => ({
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
  })[character] ?? character)
}

function highlightJsonc(line: string) {
  const escaped = escapeHtml(line)
  const tokenPattern = /("(?:\\.|[^"\\])*")(?=\s*:)|("(?:\\.|[^"\\])*")|\b(true|false|null)\b|(?<![\w."])(-?\d+(?:\.\d+)?(?:[eE][+-]?\d+)?)|([{}\[\],:])/g
  return escaped.replace(tokenPattern, (token, property, string, literal, number, punctuation) => {
    const tokenClass = property
      ? 'token-property'
      : string
        ? 'token-string'
        : literal
          ? 'token-literal'
          : number
            ? 'token-number'
            : punctuation
              ? 'token-punctuation'
              : ''
    return `<span class="${tokenClass}">${token}</span>`
  })
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
        <input ref="fileInput" class="sr-only" type="file" accept=".json,.jsonc,application/json" @change="readConfig" />
      </div>
    </header>

    <div class="workspace">
      <aside class="sidebar">
        <div class="search-box"><span>⌕</span><input v-model="search" type="search" placeholder="Search settings" /></div>
        <nav aria-label="Settings sections">
          <button v-for="section in sections" :key="section.id" class="nav-item" :class="{ active: activeSection === section.id }" type="button" @click="setActiveSection(section.id)">
            <span>{{ section.icon }}</span>{{ section.label }}
          </button>
        </nav>
        <p class="source-note">Schema source<br /><a :href="schemaUrl" target="_blank" rel="noreferrer">fastfetch-cli/fastfetch</a></p>
      </aside>

      <section class="settings-pane">
        <template v-if="schema">
          <div class="pane-heading">
            <div>
              <p class="eyebrow">FASTFETCH CONFIGURATION</p>
              <h1>{{ sections.find((item) => item.id === activeSection)?.label }}</h1>
              <div class="section-description"><MarkdownDescription v-if="sectionSchema.description" :source="sectionSchema.description" /></div>
            </div>
            <button v-if="config[activeSection] !== undefined && activeSection !== 'modules'" class="text-button danger" type="button" @click="removeSection">Reset section</button>
          </div>

          <template v-if="activeSection === 'modules'">
            <div class="module-toolbar">
              <select v-model="selectedModuleType" class="form-control">
                <option v-for="option in filteredModuleOptions" :key="option.properties?.type?.const" :value="option.properties?.type?.const">{{ option.title }}</option>
              </select>
              <button class="primary-button" type="button" @click="addModule">Add module</button>
            </div>
            <p v-if="!config.modules?.length" class="empty-state">Choose a module to start composing the output order.</p>
            <article v-for="(module, index) in config.modules ?? []" :key="`${typeof module === 'string' ? module : module.type}-${index}`" class="module-card" :class="{ collapsed: typeof module !== 'string' && isModuleCollapsed(index) }">
              <header :class="{ 'is-collapsible': typeof module !== 'string' }" :role="typeof module !== 'string' ? 'button' : undefined" :tabindex="typeof module !== 'string' ? 0 : undefined" :aria-expanded="typeof module !== 'string' ? !isModuleCollapsed(index) : undefined" :aria-controls="typeof module !== 'string' ? `module-details-${index}` : undefined" @click="typeof module !== 'string' && toggleModule(index)" @keydown.enter.prevent="typeof module !== 'string' && toggleModule(index)" @keydown.space.prevent="typeof module !== 'string' && toggleModule(index)">
                <div v-if="typeof module !== 'string'" class="module-toggle"><span class="module-index">{{ moduleIndex(index) + 1 }}</span><strong>{{ moduleTitle(module) }}</strong><code>{{ module.type }}</code></div>
                <div v-else><span class="module-index">{{ moduleIndex(index) + 1 }}</span><strong>{{ moduleTitle(module) }}</strong><code>{{ module }}</code></div>
                <div class="module-actions" @click.stop @keydown.stop><button type="button" title="Move up" @click="moveModule(index, -1)">↑</button><button type="button" title="Move down" @click="moveModule(index, 1)">↓</button><button type="button" title="Remove module" class="danger" @click="removeModule(index)">×</button></div>
              </header>
              <div v-if="typeof module === 'string'" class="simple-module"><span>Runs with Fastfetch defaults.</span><button class="secondary-button" type="button" @click="convertModule(index)">Customize</button></div>
              <div v-else :id="`module-details-${index}`" v-show="!isModuleCollapsed(index)"><SchemaField :schema="schemaForModule(module)" :root-schema="schema" :model-value="module" @update:model-value="updateModule(index, $event)" /></div>
            </article>
          </template>

          <template v-else>
            <p v-if="config[activeSection] === undefined" class="empty-state">This section is not configured yet. Add it to expose its available settings.</p>
            <button v-if="config[activeSection] === undefined" class="primary-button" type="button" @click="setSection({})">Add {{ sections.find((item) => item.id === activeSection)?.label }} section</button>
            <SchemaField v-else :schema="sectionSchema" :root-schema="schema" :model-value="config[activeSection]" :label="sections.find((item) => item.id === activeSection)?.label" @update:model-value="setSection" />
          </template>
        </template>
        <div v-else class="loading-state"><span class="spinner"></span><p>{{ error || status }}</p></div>
      </section>

      <aside class="preview-pane">
        <div class="preview-heading"><span>config.jsonc</span><span class="saved-dot">● {{ error ? 'Error' : 'Ready' }}</span></div>
        <ol class="code-preview" aria-label="Generated config.jsonc preview">
          <li v-for="(line, index) in highlightedPreviewLines" :key="index"><code v-html="line || ' '" /></li>
        </ol>
        <footer>{{ error || status }}</footer>
      </aside>
    </div>
  </main>
</template>

<style scoped lang="scss">
$tablet: 1060px;
$mobile: 720px;
$mono: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;

:global(html),
:global(body),
:global(#app) { height: 100%; overflow: hidden; }

.settings-shell { display: flex; flex-direction: column; height: 100vh; height: 100dvh; min-height: 0; background: var(--canvas); overflow: hidden; }
.topbar { flex: 0 0 auto; display: flex; align-items: center; justify-content: space-between; gap: 20px; height: 54px; padding: 0 22px; color: var(--text); background: var(--surface); border-bottom: 1px solid var(--border);
  &-actions { display: flex; align-items: center; gap: 8px; }
}
.brand { display: flex; align-items: center; gap: 10px; font-size: 14px; font-weight: 600; white-space: nowrap;
  &-mark { color: var(--accent); font: 700 20px/1 $mono; letter-spacing: -4px; }
}
.workspace { flex: 1; display: grid; grid-template-columns: 238px minmax(450px, 1fr) minmax(320px, .75fr); min-height: 0; overflow: hidden; }
.sidebar { min-height: 0; padding: 16px 10px; overflow-y: auto; background: var(--sidebar); border-right: 1px solid var(--border);
  nav { display: grid; gap: 2px; }
}
.search-box { display: flex; align-items: center; gap: 8px; height: 30px; margin: 0 2px 14px; padding: 0 8px; color: var(--subtle); background: var(--surface); border: 1px solid var(--input-border); border-radius: 3px;
  input { width: 100%; min-width: 0; color: var(--text); font-size: 12px; background: transparent; border: 0; outline: none; }
}
.nav-item { display: flex; align-items: center; gap: 10px; min-height: 30px; padding: 4px 10px; color: var(--text); font-size: 13px; text-align: left; background: transparent; border: 0; border-radius: 4px;
  &:hover { background: color-mix(in srgb, var(--text) 8%, transparent); }
  &.active { background: var(--accent-soft); }
  span { width: 16px; color: var(--muted); text-align: center; }
}
.source-note { position: sticky; top: calc(100vh - 95px); margin: 28px 10px 0; color: var(--subtle); font-size: 11px; line-height: 1.55;
  a { color: var(--accent); text-decoration: none; }
}
.settings-pane { min-width: 0; min-height: 0; padding: 10px; overflow-y: auto; overscroll-behavior: contain; background: var(--surface); }
.pane-heading { display: flex; justify-content: space-between; gap: 20px; padding-bottom: 24px; margin-bottom: 20px; border-bottom: 1px solid var(--border);
  h1 { margin: 3px 0 6px; color: var(--text); font-size: 26px; font-weight: 500; }
}
.section-description { max-width: 680px; color: var(--muted); font-size: 13px; line-height: 1.5; }
.eyebrow { color: var(--accent) !important; font-size: 10px !important; font-weight: 700; letter-spacing: .8px; }
.empty-state { margin: 20px 0 12px; color: var(--muted); font-size: 13px; }
.primary-button, .secondary-button, .text-button { padding: 5px 11px; font-size: 12px; line-height: 18px; white-space: nowrap; border-radius: 3px; }
.primary-button { color: #fff; background: var(--accent); border: 1px solid var(--accent);
  &:hover { background: var(--accent-hover); border-color: var(--accent-hover); }
}
.secondary-button { color: var(--text); background: var(--surface-raised); border: 1px solid var(--input-border);
  &:hover { background: color-mix(in srgb, var(--text) 10%, var(--surface-raised)); }
}
.text-button { align-self: flex-start; color: var(--accent); background: transparent; border: 0; }
.danger { color: var(--danger) !important; }
.sr-only { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; }
.module-toolbar { display: flex; gap: 8px; max-width: 480px; margin: 8px 0 18px;
  :deep(.form-control) { width: 100%; }
}
.module-card { margin-top: 12px; overflow: hidden; background: var(--surface); border: 1px solid var(--border); border-radius: 4px;
  > header { display: flex; align-items: center; justify-content: space-between; min-height: 43px; padding: 0 10px 0 14px; background: var(--surface-raised); border-bottom: 1px solid var(--border);
    &.is-collapsible { cursor: pointer;
      &:hover { background: color-mix(in srgb, var(--text) 6%, var(--surface-raised)); }
    }
    > div:first-child, > .module-toggle { display: flex; align-items: center; gap: 8px; min-width: 0; color: var(--text); font-size: 13px; }
  }
  &.collapsed > header { border-bottom-color: transparent; }
  code { color: var(--muted); font: 11px $mono; }
  > div > :deep(.schema-field) { padding: 0 14px 16px; }
}
.module-index { display: inline-grid; width: 18px; height: 18px; place-items: center; color: var(--muted); font-size: 10px; border: 1px solid var(--input-border); border-radius: 50%; }
.module-actions { display: flex; gap: 2px;
  button { width: 25px; height: 25px; padding: 0; color: var(--muted); font-size: 14px; background: transparent; border: 0; border-radius: 3px;
    &:hover { background: color-mix(in srgb, var(--text) 10%, transparent); }
  }
}
.simple-module { display: flex; align-items: center; justify-content: space-between; padding: 14px; color: var(--muted); font-size: 12px; }
.preview-pane { position: relative; display: flex; flex-direction: column; min-width: 0; min-height: 0; color: var(--code); background: var(--surface-raised); border-left: 1px solid var(--border);
  footer { padding: 9px 14px; color: var(--muted); font-size: 11px; border-top: 1px solid var(--border); }
}
.preview-heading { display: flex; align-items: center; justify-content: space-between; height: 42px; padding: 0 14px; color: var(--text); font: 12px $mono; border-bottom: 1px solid var(--border); }
.saved-dot { color: var(--muted); font-family: inherit; font-size: 10px; }
.code-preview { flex: 1; min-height: 0; padding: 14px 14px 14px 48px; margin: 0; overflow: auto; color: var(--code); font: 12px/1.55 $mono; white-space: pre; tab-size: 2;
  li { padding-left: 10px;
    &::marker { color: var(--subtle); font-size: 10px; text-align: right; }
  }
  code { font: inherit; }
}
.token-property { color: #0451a5; }
.token-string { color: #a31515; }
.token-number { color: #098658; }
.token-literal { color: #0000ff; }
.token-punctuation { color: var(--code); }
.loading-state { display: grid; min-height: 260px; place-content: center; gap: 10px; color: var(--muted); font-size: 13px; text-align: center;
  p { max-width: 380px; margin: 0; }
}
.spinner { width: 22px; height: 22px; margin: auto; border: 2px solid var(--accent-soft); border-top-color: var(--accent); border-radius: 50%; animation: spin .8s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }

@media (prefers-color-scheme: dark) {
  .token-property { color: #9cdcfe; }
  .token-string { color: #ce9178; }
  .token-number { color: #b5cea8; }
  .token-literal { color: #569cd6; }
}
@media (max-width: $tablet) {
  .workspace { grid-template-columns: 208px minmax(0, 1fr); grid-template-rows: minmax(0, 1fr) 330px; }
  .sidebar { grid-row: 1 / span 2; }
  .preview-pane { grid-column: 2; grid-row: 2; height: auto; border-top: 1px solid var(--border); border-left: 0; }
}
@media (max-width: $mobile) {
  .topbar { flex-direction: column; align-items: flex-start; height: auto; min-height: 54px; padding: 10px 14px;
    &-actions { width: 100%; padding-bottom: 1px; overflow-x: auto;
      button { white-space: nowrap; }
    }
  }
  .workspace { display: grid; grid-template-columns: minmax(0, 1fr); grid-template-rows: auto minmax(0, 1fr) 280px; min-height: 0; }
  .sidebar { grid-row: 1; padding: 10px; overflow: visible; border-right: 0; border-bottom: 1px solid var(--border);
    nav { display: flex; overflow-x: auto; }
  }
  .nav-item { white-space: nowrap; }
  .source-note { display: none; }
  .settings-pane { grid-row: 2; padding: 10px; }
  .pane-heading { flex-direction: column; }
  .module-toolbar { max-width: none; }
  .preview-pane { grid-column: 1; grid-row: 3; height: auto; }
}
</style>
