<script setup lang="ts">
import { computed, ref } from 'vue'
import MarkdownDescription from './MarkdownDescription.vue'

type Schema = Record<string, any>

const props = defineProps<{
  schema: Schema
  rootSchema: Schema
  modelValue: any
  label?: string
  depth?: number
}>()

const emit = defineEmits<{ 'update:modelValue': [value: any] }>()

const resolved = computed<Schema>(() => resolve(props.schema))
const level = computed(() => props.depth ?? 0)
const isObject = computed(() => resolved.value.type === 'object' || Boolean(resolved.value.properties))
const isArray = computed(() => resolved.value.type === 'array')
const enumOptions = computed(() => getOptions(resolved.value))
const collapsed = ref(false)
const collapsedProperties = ref(new Set<string>())
const variantOptions = computed(() => (resolved.value.oneOf ?? resolved.value.anyOf ?? [])
  .filter((branch: Schema) => !Object.hasOwn(branch, 'const')))
const selectedVariant = computed(() => {
  const index = variantOptions.value.findIndex((branch) => matchesValue(resolve(branch), props.modelValue))
  return index < 0 ? 0 : index
})
const visibleProperties = computed(() => Object.entries(resolved.value.properties ?? {}).filter(([key]) => key in (props.modelValue ?? {})))
const availableProperties = computed(() => Object.entries(resolved.value.properties ?? {}).filter(([key]) => !(key in (props.modelValue ?? {}))))

function resolve(schema: Schema): Schema {
  if (!schema?.$ref) return schema ?? {}
  const path = String(schema.$ref).replace('#/', '').split('/')
  return path.reduce((value, key) => value?.[key], props.rootSchema) ?? schema
}

function getOptions(schema: Schema) {
  if (schema.enum) return schema.enum.map((value: unknown) => ({ value, label: String(value) }))
  const branches = schema.oneOf ?? schema.anyOf
  if (!branches) return []
  return branches
    .filter((branch: Schema) => Object.hasOwn(branch, 'const'))
    .map((branch: Schema) => ({ value: branch.const, label: branch.description ?? String(branch.const) }))
}

function matchesValue(schema: Schema, value: unknown) {
  if (value === null) return schema.type === 'null' || schema.const === null
  if (Array.isArray(value)) return schema.type === 'array'
  if (typeof value === 'object') return schema.type === 'object' || Boolean(schema.properties)
  return schema.type === typeof value || (!schema.type && !schema.properties)
}

function isObjectSchema(schema: Schema) {
  const value = resolve(schema)
  return value.type === 'object' || Boolean(value.properties)
}

function isPropertyCollapsed(key: string) {
  return collapsedProperties.value.has(key)
}

function toggleProperty(key: string) {
  const next = new Set(collapsedProperties.value)
  if (next.has(key)) next.delete(key)
  else next.add(key)
  collapsedProperties.value = next
}

function defaultValue(schema: Schema): any {
  const value = resolve(schema)
  if (Object.hasOwn(value, 'default')) return structuredClone(value.default)
  const options = getOptions(value)
  if (options.length) return options[0].value
  if (value.type === 'boolean') return false
  if (value.type === 'number' || value.type === 'integer') return value.minimum ?? 0
  if (value.type === 'array') return []
  if (value.type === 'object' || value.properties) return {}
  return ''
}

function updatePrimitive(event: Event) {
  const target = event.target as HTMLInputElement | HTMLSelectElement
  let value: any = target.value
  if (resolved.value.type === 'number' || resolved.value.type === 'integer') value = value === '' ? undefined : Number(value)
  emit('update:modelValue', value)
}

function changeVariant(event: Event) {
  const branch = variantOptions.value[Number((event.target as HTMLSelectElement).value)]
  emit('update:modelValue', defaultValue(branch))
}

function updateBoolean(event: Event) {
  emit('update:modelValue', (event.target as HTMLInputElement).checked)
}

function updateChild(key: string, value: any) {
  emit('update:modelValue', { ...(props.modelValue ?? {}), [key]: value })
}

function removeChild(key: string) {
  const next = { ...(props.modelValue ?? {}) }
  delete next[key]
  emit('update:modelValue', next)
}

function addChild(event: Event) {
  const key = (event.target as HTMLSelectElement).value
  if (!key) return
  const childSchema = resolved.value.properties[key]
  updateChild(key, defaultValue(childSchema))
  ;(event.target as HTMLSelectElement).value = ''
}

function updateArrayItem(index: number, value: any) {
  const next = [...(props.modelValue ?? [])]
  next[index] = value
  emit('update:modelValue', next)
}

function removeArrayItem(index: number) {
  emit('update:modelValue', (props.modelValue ?? []).filter((_: unknown, itemIndex: number) => itemIndex !== index))
}

function addArrayItem() {
  emit('update:modelValue', [...(props.modelValue ?? []), defaultValue(resolved.value.items ?? {})])
}
</script>

<template>
  <div class="schema-field" :class="{ 'is-nested': level > 0 }">
    <template v-if="variantOptions.length">
      <div class="variant-picker">
        <label>Value type</label>
        <select class="form-control" :value="selectedVariant" @change="changeVariant">
          <option v-for="(variant, index) in variantOptions" :key="index" :value="index">
            {{ resolve(variant).title ?? resolve(variant).type ?? 'Custom value' }}{{ resolve(variant).description ? ` — ${resolve(variant).description}` : '' }}
          </option>
        </select>
      </div>
      <SchemaField
        :schema="variantOptions[selectedVariant]"
        :root-schema="rootSchema"
        :model-value="modelValue"
        :depth="level + 1"
        @update:model-value="emit('update:modelValue', $event)"
      />
    </template>

    <template v-else-if="isObject">
      <div v-if="label" class="object-heading">
        <div>
          <button class="object-toggle" type="button" :aria-expanded="!collapsed" @click="collapsed = !collapsed">
            <strong>{{ label }}</strong>
          </button>
          <div v-if="resolved.description && !collapsed" class="description"><MarkdownDescription :source="resolved.description" /></div>
        </div>
      </div>

      <div v-show="!collapsed" v-if="visibleProperties.length" class="object-properties">
        <div v-for="[key, childSchema] in visibleProperties" :key="key" class="property-row">
          <div class="property-meta">
            <button v-if="isObjectSchema(childSchema)" class="object-toggle" type="button" :aria-expanded="!isPropertyCollapsed(key)" @click="toggleProperty(key)">
              <span>{{ key }}</span>
            </button>
            <label v-else>{{ key }}</label>
            <div v-if="resolve(childSchema).description && !isPropertyCollapsed(key)" class="description"><MarkdownDescription :source="resolve(childSchema).description" /></div>
          </div>
          <div v-show="!isPropertyCollapsed(key)" class="property-control">
            <SchemaField
              :schema="resolve(childSchema)"
              :root-schema="rootSchema"
              :model-value="modelValue[key]"
              :depth="level + 1"
              @update:model-value="updateChild(key, $event)"
            />
          </div>
          <button class="icon-button remove" type="button" :aria-label="`Remove ${key}`" @click="removeChild(key)">×</button>
        </div>
      </div>

      <label v-show="!collapsed" v-if="availableProperties.length" class="add-property">
        <span>＋</span>
        <select aria-label="Add setting" @change="addChild">
          <option value="">Add setting…</option>
          <option v-for="[key, childSchema] in availableProperties" :key="key" :value="key">
            {{ key }}{{ resolve(childSchema).description ? ` — ${resolve(childSchema).description}` : '' }}
          </option>
        </select>
      </label>
    </template>

    <template v-else-if="isArray">
      <div class="array-editor">
        <div v-for="(item, index) in (modelValue ?? [])" :key="index" class="array-item">
          <SchemaField
            :schema="resolved.items ?? {}"
            :root-schema="rootSchema"
            :model-value="item"
            :depth="level + 1"
            @update:model-value="updateArrayItem(index, $event)"
          />
          <button class="icon-button remove" type="button" aria-label="Remove item" @click="removeArrayItem(index)">×</button>
        </div>
        <button class="secondary-button" type="button" @click="addArrayItem">Add item</button>
      </div>
    </template>

    <select v-else-if="enumOptions.length" class="form-control" :value="modelValue" @change="updatePrimitive">
      <option v-for="option in enumOptions" :key="String(option.value)" :value="option.value">{{ option.label }}</option>
    </select>

    <label v-else-if="resolved.type === 'boolean'" class="switch">
      <input type="checkbox" :checked="Boolean(modelValue)" @change="updateBoolean" />
      <span aria-hidden="true"></span>
      <b>{{ modelValue ? 'On' : 'Off' }}</b>
    </label>

    <input
      v-else
      class="form-control"
      :class="{ 'text-control': resolved.type !== 'number' && resolved.type !== 'integer' }"
      :type="resolved.type === 'number' || resolved.type === 'integer' ? 'number' : 'text'"
      :min="resolved.minimum"
      :max="resolved.maximum"
      :step="resolved.type === 'integer' ? 1 : 'any'"
      :value="modelValue ?? ''"
      @input="updatePrimitive"
    />
  </div>
</template>

<style scoped lang="scss">
$mobile: 720px;
$mono: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;

.schema-field { min-width: 0; }
.object-heading { margin: 18px 0 10px;
  strong { color: var(--text); font-size: 15px; font-weight: 500; }
  .description { margin: 4px 0 0; color: var(--muted); font-size: 12px; line-height: 1.5; }
}
.object-toggle { display: inline-flex; align-items: center; min-width: 0; padding: 0; color: inherit; font: inherit; text-align: left; background: transparent; border: 0;
  &:hover { color: var(--accent); }
}
.object-properties { border-top: 1px solid var(--border); }
.property-row { display: grid; grid-template-columns: minmax(150px, .9fr) minmax(220px, 1.1fr) 26px; align-items: start; gap: 18px; padding: 4px 8px; border-bottom: 1px solid var(--border); }
.property-meta {
  label, .object-toggle { display: block; color: var(--text); font-family: $mono; font-size: 12px; }
  .object-toggle { display: inline-flex; }
  .description { margin: 4px 0 0; color: var(--muted); font-size: 11px; line-height: 1.45; }
}
.property-control > .schema-field.is-nested {
  .object-properties { margin-top: 8px; padding: 0 10px; border: 1px solid var(--border); border-radius: 3px; }
  .property-row { grid-template-columns: 118px minmax(120px, 1fr) 22px; gap: 10px; padding: 14px 0; }
  .object-heading { display: none; }
}
.form-control { width: 100%; height: 28px; padding: 3px 7px; color: var(--text); font-size: 12px; background: var(--surface); border: 1px solid var(--input-border); border-radius: 2px; }
.text-control { font-family: $mono; }
.variant-picker { display: grid; grid-template-columns: 86px minmax(0, 1fr); align-items: center; gap: 8px; margin-bottom: 10px; color: var(--muted); font-size: 11px; }
.icon-button { width: 25px; height: 25px; padding: 0; color: var(--subtle); font-size: 20px; line-height: 1; background: transparent; border: 0;
  &:hover { color: var(--danger); background: color-mix(in srgb, var(--danger) 10%, transparent); }
}
.add-property { display: inline-flex; align-items: center; gap: 5px; margin-top: 12px; color: var(--accent); font-size: 12px;
  span { font-size: 16px; }
  select { max-width: 440px; color: var(--accent); font-size: 12px; background: transparent; border: 0; }
}
.switch { display: inline-flex; align-items: center; gap: 7px; min-height: 28px; color: var(--muted); font-size: 11px;
  input { position: absolute; opacity: 0;
    &:checked + span { background: var(--accent);
      &::after { transform: translateX(11px); }
    }
  }
  span { display: block; width: 25px; height: 14px; padding: 2px; background: var(--subtle); border-radius: 9px; transition: background .15s;
    &::after { display: block; width: 10px; height: 10px; background: #fff; border-radius: 50%; content: ''; transition: transform .15s; }
  }
  b { min-width: 18px; font-weight: 400; }
}
.array-editor { display: grid; gap: 8px; }
.array-item { display: grid; grid-template-columns: 1fr 25px; align-items: center; gap: 8px; }
.secondary-button { padding: 5px 11px; color: var(--text); font-size: 12px; line-height: 18px; white-space: nowrap; background: var(--surface-raised); border: 1px solid var(--input-border); border-radius: 3px;
  &:hover { background: color-mix(in srgb, var(--text) 10%, var(--surface-raised)); }
}
@media (max-width: $mobile) {
  .property-row { grid-template-columns: 1fr 23px; gap: 8px; }
  .property-meta { grid-column: 1 / -1; }
  .variant-picker { grid-template-columns: 1fr; }
  .property-control > .schema-field.is-nested .property-row { grid-template-columns: 1fr 20px; }
}
</style>
