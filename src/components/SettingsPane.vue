<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import SchemaField from './SchemaField.vue'
import MarkdownDescription from './MarkdownDescription.vue'

type Schema = Record<string, any>
type Config = Record<string, any>

const props = defineProps<{
  schema: Schema
  config: Config
  activeSection: string
  sections: { id: string; label: string; icon: string }[]
  search: string
}>()

const emit = defineEmits<{
  'update:configSection': [key: string, value: unknown]
  'remove:configSection': [key: string]
}>()

/* ── derived schema ─────────────────────────────────────────── */

const sectionSchema = computed(() => props.schema.properties?.[props.activeSection] ?? {})

/* ── module management ──────────────────────────────────────── */

const selectedModuleType = ref('title')
const collapsedModules = ref(new Set<number>())
const accordionMode = ref(false)

watch(accordionMode, (val) => {
  const modules = props.config.modules
  if (!modules?.length) return
  const next = new Set<number>()
  if (val) {
    // 开启手风琴 → 全部收起
    for (let i = 0; i < modules.length; i++) next.add(i)
  }
  // 关闭手风琴 → 全部展开 (Set 为空)
  collapsedModules.value = next
})

const moduleOptions = computed<Schema[]>(() => {
  const itemSchema = props.schema.properties?.modules?.items
  const objectChoice = itemSchema?.anyOf?.find((choice: Schema) => choice.type === 'object')
  return objectChoice?.oneOf ?? []
})

const filteredModuleOptions = computed(() =>
  moduleOptions.value.filter((option) => {
    const label = String(option.title ?? option.properties?.type?.const ?? '')
    return label.toLowerCase().includes(props.search.toLowerCase())
  }),
)

function schemaForModule(module: Config): Schema {
  const matching = moduleOptions.value.find(
    (option) => option.properties?.type?.const === module.type,
  )
  if (!matching) return { type: 'object', properties: {} }
  const { type: _type, ...properties } = matching.properties ?? {}
  return { ...matching, properties }
}

function moduleTitle(module: unknown) {
  if (typeof module === 'string') return module
  const type = (module as Config).type
  return (
    moduleOptions.value.find((option) => option.properties?.type?.const === type)?.title ?? type
  )
}

function addModule() {
  if (!props.config.modules) props.config.modules = []
  props.config.modules.push({ type: selectedModuleType.value })
}

function isModuleCollapsed(index: number) {
  return collapsedModules.value.has(index)
}

function toggleModule(index: number) {
  const next = new Set(collapsedModules.value)
  if (next.has(index)) {
    next.delete(index)
    if (accordionMode.value) {
      const modules = props.config.modules ?? []
      for (let i = 0; i < modules.length; i++) {
        if (i !== index) next.add(i)
      }
    }
  } else {
    next.add(index)
  }
  collapsedModules.value = next
}

function updateModule(index: number, value: Config) {
  props.config.modules[index] = { type: props.config.modules[index].type, ...value }
}

function removeModule(index: number) {
  props.config.modules.splice(index, 1)
}

function moveModule(index: number, direction: number) {
  const destination = index + direction
  if (destination < 0 || destination >= props.config.modules.length) return
  const [mod] = props.config.modules.splice(index, 1)
  props.config.modules.splice(destination, 0, mod)
}

function convertModule(index: number) {
  props.config.modules[index] = { type: props.config.modules[index] }
}

function simplifyModule(index: number) {
  props.config.modules[index] = props.config.modules[index].type
}

function isModuleSimple(module: Config) {
  const keys = Object.keys(module)
  return keys.length === 1 && keys[0] === 'type'
}

/* ── section helpers ────────────────────────────────────────── */

function setSection(value: unknown) {
  emit('update:configSection', props.activeSection, value)
}

function removeSection() {
  emit('remove:configSection', props.activeSection)
}
</script>

<template>
  <section class="settings-pane">
    <div class="pane-heading">
      <div>
        <p class="eyebrow">FASTFETCH CONFIGURATION</p>
        <h1>{{ sections.find((item) => item.id === activeSection)?.label }}</h1>
        <div class="section-description">
          <MarkdownDescription
            v-if="sectionSchema.description"
            :source="sectionSchema.description"
          />
        </div>
      </div>
      <button
        v-if="config[activeSection] !== undefined && activeSection !== 'modules'"
        class="text-button danger"
        type="button"
        @click="removeSection"
      >
        Reset section
      </button>
    </div>

    <template v-if="activeSection === 'modules'">
      <div class="module-toolbar">
        <select v-model="selectedModuleType" class="form-control">
          <option
            v-for="option in filteredModuleOptions"
            :key="option.properties?.type?.const"
            :value="option.properties?.type?.const"
          >
            {{ option.title }}
          </option>
        </select>
        <button class="primary-button" type="button" @click="addModule">Add module</button>
        <label class="accordion-toggle">
          <input type="checkbox" v-model="accordionMode" />
          <span class="toggle-track"><span class="toggle-thumb"></span></span>
          Accordion
        </label>
      </div>
      <p v-if="!config.modules?.length" class="empty-state">
        Choose a module to start composing the output order.
      </p>
      <article
        v-for="(module, index) in (config.modules as any[]) ?? []"
        :key="`${typeof module === 'string' ? module : module.type}-${index}`"
        class="module-card"
        :class="{ collapsed: isModuleCollapsed(index) }"
      >
        <header
          class="is-collapsible"
          role="button"
          tabindex="0"
          :aria-expanded="!isModuleCollapsed(index)"
          :aria-controls="`module-details-${index}`"
          @click="toggleModule(index)"
          @keydown.enter.prevent="toggleModule(index)"
          @keydown.space.prevent="toggleModule(index)"
        >
          <div v-if="typeof module !== 'string'" class="module-toggle">
            <span class="module-index">{{ index + 1 }}</span>
            <strong>{{ moduleTitle(module) }}</strong>
            <code>{{ module.type }}</code>
          </div>
          <div v-else>
            <span class="module-index">{{ index + 1 }}</span>
            <strong>{{ module }}</strong>
          </div>
          <div class="module-actions" @click.stop @keydown.stop>
            <button type="button" title="Move up" @click="moveModule(index, -1)">↑</button>
            <button type="button" title="Move down" @click="moveModule(index, 1)">↓</button>
            <button type="button" title="Remove module" class="danger" @click="removeModule(index)">×</button>
          </div>
        </header>
        <div
          :id="`module-details-${index}`"
          v-show="!isModuleCollapsed(index)"
        >
          <template v-if="typeof module === 'string'">
            <div class="simple-module">
              <span>Runs with Fastfetch defaults.</span>
              <button class="secondary-button" type="button" @click="convertModule(index)">
                Customize
              </button>
            </div>
          </template>
          <SchemaField
            v-else
            :schema="schemaForModule(module)"
            :root-schema="schema"
            :model-value="module"
            @update:model-value="updateModule(index, $event)"
          />
          <div v-if="isModuleSimple(module)" class="simplify-hint">
            <span>No custom settings — </span>
            <button class="text-button" type="button" @click="simplifyModule(index)">convert to string</button>
          </div>
        </div>
      </article>
    </template>

    <template v-else>
      <p v-if="config[activeSection] === undefined" class="empty-state">
        This section is not configured yet. Add it to expose its available settings.
      </p>
      <button
        v-if="config[activeSection] === undefined"
        class="primary-button"
        type="button"
        @click="setSection({})"
      >
        Add {{ sections.find((item) => item.id === activeSection)?.label }} section
      </button>
      <SchemaField
        v-else
        :schema="sectionSchema"
        :root-schema="schema"
        :model-value="config[activeSection]"
        :label="sections.find((item) => item.id === activeSection)?.label"
        @update:model-value="setSection"
      />
    </template>
  </section>
</template>

<style scoped lang="scss">
.settings-pane {
  min-width: 0;
  min-height: 0;
  padding: 10px;
  overflow-y: auto;
  overscroll-behavior: contain;
  background: var(--surface);
}

.pane-heading {
  display: flex;
  justify-content: space-between;
  gap: 20px;
  padding: 0 0 20px;
  margin-bottom: 16px;
  border-bottom: 1px solid var(--border);

  h1 {
    margin: 3px 0 6px;
    color: var(--text);
    font-size: 24px;
    font-weight: 600;
  }
}

.section-description {
  max-width: 680px;
  color: var(--muted);
  font-size: 13px;
  line-height: 1.5;
}

.eyebrow {
  color: var(--accent) !important;
  font-size: 10px !important;
  font-weight: 700;
  letter-spacing: 0.8px;
}

.empty-state {
  margin: 20px 0 12px;
  color: var(--muted);
  font-size: 13px;
}

.module-toolbar {
  display: flex;
  gap: 8px;
  margin: 8px 0 18px;
  align-items: center;

  :deep(.form-control) {
    flex: 1;
    max-width: 300px;
  }
}

.accordion-toggle {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  flex-shrink: 0;
  margin-left: auto;
  color: var(--muted);
  font-size: 12px;
  cursor: pointer;
  user-select: none;

  input {
    position: absolute;
    width: 0;
    height: 0;
    opacity: 0;
    pointer-events: none;
  }

  .toggle-track {
    position: relative;
    width: 32px;
    height: 18px;
    background: var(--input-border, #bbb);
    border-radius: 9px;
    transition: background 0.2s;
  }

  .toggle-thumb {
    position: absolute;
    top: 2px;
    left: 2px;
    width: 14px;
    height: 14px;
    background: #fff;
    border-radius: 50%;
    transition: transform 0.2s;
  }

  input:checked + .toggle-track {
    background: var(--accent);
  }

  input:checked + .toggle-track .toggle-thumb {
    transform: translateX(14px);
  }
}

.module-card {
  margin-top: 12px;
  overflow: hidden;
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 6px;
  box-shadow: 0 1px 3px rgba(0,0,0,.06);

  > header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    min-height: 43px;
    padding: 0 10px 0 14px;
    background: var(--surface-raised);
    border-bottom: 1px solid var(--border);
    user-select: none;

    &.is-collapsible {
      cursor: pointer;

      &:hover {
        background: color-mix(in srgb, var(--text) 6%, var(--surface-raised));
      }

      > div:first-child::before {
        display: inline-block;
        width: 14px;
        color: var(--subtle);
        font-size: 9px;
        content: "▼";
      }
    }

    &[aria-expanded="false"] > div:first-child::before {
      content: "▶";
    }

    > div:first-child,
    > .module-toggle {
      display: flex;
      align-items: center;
      gap: 8px;
      min-width: 0;
      color: var(--text);
      font-size: 13px;
    }
  }

  &.collapsed > header {
    border-bottom-color: transparent;
  }

  code {
    color: var(--muted);
    font: 11px var(--mono);
    padding: 1px 6px;
    border-radius: 3px;
    background: var(--surface);
    margin-left: 20px;
  }

  > div > :deep(.schema-field) {
    padding: 16px 14px;
  }
}

.module-index {
  display: inline-grid;
  width: 20px;
  height: 20px;
  line-height: 0;
  text-align: center;
  place-items: center;
  color: var(--muted);
  font-size: 10px;
  border: 1px solid var(--input-border);
  border-radius: 50%;
}

.module-actions {
  display: flex;
  gap: 2px;

  button {
    width: 25px;
    height: 25px;
    padding: 0;
    color: var(--muted);
    font-size: 14px;
    background: transparent;
    border: 0;
    border-radius: 3px;

    &:hover {
      background: color-mix(in srgb, var(--text) 10%, transparent);
    }
  }
}

.simple-module {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px;
  color: var(--muted);
  font-size: 12px;
}

.simplify-hint {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 6px 14px 12px;
  color: var(--subtle);
  font-size: 11px;

  .text-button {
    padding: 0;
    color: var(--accent);
    font-size: 11px;
    background: transparent;
    border: 0;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }
}

.spinner {
  width: 22px;
  height: 22px;
  margin: auto;
  border: 2px solid var(--accent-soft);
  border-top-color: var(--accent);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@media (max-width: 720px) {
  .pane-heading {
    flex-direction: column;
  }

  .settings-pane {
    padding: 10px;
  }
}
</style>
