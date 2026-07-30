<script vapor lang="ts">
import { computed, ref } from 'vue';
import MarkdownDescription from './MarkdownDescription.vue';

type Schema = Record<string, any>;

const props = defineProps<{
  schema: Schema;
  rootSchema: Schema;
  modelValue: any;
  label?: string;
  depth?: number;
}>();

const emit = defineEmits<{ 'update:modelValue': [value: any] }>();

const resolved = computed<Schema>(() => resolve(props.schema));
const level = computed(() => props.depth ?? 0);
const isObject = computed(
  () => resolved.value.type === 'object' || Boolean(resolved.value.properties),
);
const isArray = computed(() => resolved.value.type === 'array');
const enumOptions = computed(() => getOptions(resolved.value));
const collapsed = ref(false);
const collapsedProperties = ref(new Set<string>());
const variantOptions = computed(() =>
  (resolved.value.oneOf ?? resolved.value.anyOf ?? []).filter(
    (branch: Schema) =>
      !Object.hasOwn(branch, 'const') || branch.const === null,
  ),
);
const selectedVariant = computed(() => {
  const index = variantOptions.value.findIndex((branch) =>
    matchesValue(resolve(branch), props.modelValue),
  );
  return index < 0 ? 0 : index;
});
const visibleProperties = computed(() =>
  (
    Object.entries(resolved.value.properties ?? {}) as [string, Schema][]
  ).filter(([key]) => key in (props.modelValue ?? {})),
);
const availableProperties = computed(() =>
  (
    Object.entries(resolved.value.properties ?? {}) as [string, Schema][]
  ).filter(([key]) => !(key in (props.modelValue ?? {}))),
);

function resolve(schema: Schema): Schema {
  if (!schema?.$ref) return schema ?? {};
  const path = String(schema.$ref).replace('#/', '').split('/');
  return path.reduce((value, key) => value?.[key], props.rootSchema) ?? schema;
}

function getOptions(schema: Schema) {
  if (schema.enum)
    return schema.enum.map((value: unknown) => ({
      value,
      label: String(value),
    }));
  const branches = schema.oneOf ?? schema.anyOf;
  if (!branches) return [];
  return branches
    .filter(
      (branch: Schema) =>
        Object.hasOwn(branch, 'const') && branch.const !== null,
    )
    .map((branch: Schema) => ({
      value: branch.const,
      label: branch.description ?? String(branch.const),
    }));
}

function matchesValue(schema: Schema, value: unknown) {
  if (value === null) return schema.type === 'null' || schema.const === null;
  if (Array.isArray(value)) return schema.type === 'array';
  if (typeof value === 'object')
    return schema.type === 'object' || Boolean(schema.properties);
  return schema.type === typeof value || (!schema.type && !schema.properties);
}

function isObjectSchema(schema: Schema) {
  const value = resolve(schema);
  return value.type === 'object' || Boolean(value.properties);
}

function isPropertyCollapsed(key: string) {
  return collapsedProperties.value.has(key);
}

function toggleProperty(key: string) {
  const next = new Set(collapsedProperties.value);
  if (next.has(key)) next.delete(key);
  else next.add(key);
  collapsedProperties.value = next;
}

function defaultValue(schema: Schema): any {
  const value = resolve(schema);
  if (Object.hasOwn(value, 'default'))
    return JSON.parse(JSON.stringify(value.default));
  if (Object.hasOwn(value, 'const'))
    return JSON.parse(JSON.stringify(value.const));
  const options = getOptions(value);
  if (options.length) return options[0].value;
  if (value.type === 'boolean') return false;
  if (value.type === 'null') return null;
  if (value.type === 'number' || value.type === 'integer')
    return value.minimum ?? 0;
  if (value.type === 'array') return [];
  if (value.type === 'object' || value.properties) return {};
  return '';
}

function updatePrimitive(event: Event) {
  const target = event.target as HTMLInputElement | HTMLSelectElement;
  let value: any = target.value;
  if (resolved.value.type === 'number' || resolved.value.type === 'integer')
    value = value === '' ? undefined : Number(value);
  emit('update:modelValue', value);
}

function changeVariant(event: Event) {
  const branch =
    variantOptions.value[Number((event.target as HTMLSelectElement).value)];
  emit('update:modelValue', defaultValue(branch));
}

function updateBoolean(event: Event) {
  emit('update:modelValue', (event.target as HTMLInputElement).checked);
}

function updateChild(key: string, value: any) {
  emit('update:modelValue', { ...(props.modelValue ?? {}), [key]: value });
}

function removeChild(key: string) {
  const next = { ...(props.modelValue ?? {}) };
  delete next[key];
  emit('update:modelValue', next);
}

function addChild(event: Event) {
  const key = (event.target as HTMLSelectElement).value;
  if (!key) return;
  const childSchema = resolved.value.properties[key];
  updateChild(key, defaultValue(childSchema));
  (event.target as HTMLSelectElement).value = '';
}

function updateArrayItem(index: number, value: any) {
  const next = [...(props.modelValue ?? [])];
  next[index] = value;
  emit('update:modelValue', next);
}

function removeArrayItem(index: number) {
  emit(
    'update:modelValue',
    (props.modelValue ?? []).filter(
      (_: unknown, itemIndex: number) => itemIndex !== index,
    ),
  );
}

function addArrayItem() {
  emit('update:modelValue', [
    ...(props.modelValue ?? []),
    defaultValue(resolved.value.items ?? {}),
  ]);
}
</script>

<template>
  <div class="schema-field" :class="{ 'is-nested': level > 0 }">
    <template v-if="variantOptions.length">
      <div class="variant-picker">
        <label>Value type</label>
        <select
          class="form-control"
          :value="selectedVariant"
          @change="changeVariant"
        >
          <option
            v-for="(variant, index) in variantOptions"
            :key="index"
            :value="index"
          >
            {{
              resolve(variant).title ?? resolve(variant).type ?? 'Custom value'
            }}{{
              resolve(variant).description
                ? ` — ${resolve(variant).description}`
                : ''
            }}
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
          <button
            class="object-toggle"
            type="button"
            :aria-expanded="!collapsed"
            @click="collapsed = !collapsed"
          >
            <strong>{{ label }}</strong>
          </button>
          <div v-if="resolved.description" class="description">
            <MarkdownDescription :source="resolved.description" />
          </div>
        </div>
      </div>

      <div
        v-if="visibleProperties.length && !collapsed"
        class="object-properties"
      >
        <div
          v-for="[key, childSchema] in visibleProperties"
          :key="key"
          class="property-row"
        >
          <div class="property-meta">
            <button
              v-if="isObjectSchema(childSchema)"
              class="object-toggle"
              type="button"
              :aria-expanded="!isPropertyCollapsed(key)"
              @click="toggleProperty(key)"
            >
              <span class="property-key">{{ key }}</span>
            </button>
            <label v-else class="property-key">{{ key }}</label>
            <div
              v-if="
                resolve(childSchema).description && !isPropertyCollapsed(key)
              "
              class="description"
            >
              <MarkdownDescription :source="resolve(childSchema).description" />
            </div>
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
          <button
            class="icon-button remove"
            type="button"
            :aria-label="`Remove ${key}`"
            @click="removeChild(key)"
          >
            ×
          </button>
        </div>
      </div>

      <label
        v-show="!collapsed"
        v-if="availableProperties.length"
        class="add-property"
      >
        <span>＋</span>
        <select aria-label="Add setting" @change="addChild">
          <option value="">Add setting…</option>
          <option
            v-for="[key, childSchema] in availableProperties"
            :key="key"
            :value="key"
          >
            {{ key
            }}{{
              resolve(childSchema).description
                ? ` — ${resolve(childSchema).description}`
                : ''
            }}
          </option>
        </select>
      </label>
    </template>

    <template v-else-if="isArray">
      <div class="array-editor">
        <div
          v-for="(item, index) in (modelValue as any[]) ?? []"
          :key="index"
          class="array-item"
        >
          <SchemaField
            :schema="resolved.items ?? {}"
            :root-schema="rootSchema"
            :model-value="item"
            :depth="level + 1"
            @update:model-value="updateArrayItem(index, $event)"
          />
          <button
            class="icon-button remove"
            type="button"
            aria-label="Remove item"
            @click="removeArrayItem(index)"
          >
            ×
          </button>
        </div>
        <button class="secondary-button" type="button" @click="addArrayItem">
          Add item
        </button>
      </div>
    </template>

    <select
      v-else-if="enumOptions.length"
      class="form-control"
      :value="modelValue"
      @change="updatePrimitive"
    >
      <option
        v-for="option in enumOptions"
        :key="String(option.value)"
        :value="option.value"
      >
        {{ option.label }}
      </option>
    </select>

    <label v-else-if="resolved.type === 'boolean'" class="switch">
      <input
        type="checkbox"
        :checked="Boolean(modelValue)"
        @change="updateBoolean"
      />
      <span aria-hidden="true"></span>
      <b>{{ modelValue ? 'On' : 'Off' }}</b>
    </label>

    <span
      v-else-if="resolved.type === 'null' || resolved.const === null"
      class="null-indicator"
      >null</span
    >

    <input
      v-else
      class="form-control"
      :class="{
        'text-control':
          resolved.type !== 'number' && resolved.type !== 'integer',
      }"
      :type="
        resolved.type === 'number' || resolved.type === 'integer'
          ? 'number'
          : 'text'
      "
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

.schema-field {
  min-width: 0;
}

.object-heading {
  display: flex;
  align-items: center;
  min-height: 36px;
  margin: 0 0 10px;

  .object-toggle {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    min-width: 0;
    padding: 0;
    color: inherit;
    font: inherit;
    text-align: left;
    background: transparent;
    border: 0;
    cursor: pointer;
    &::before {
      display: inline-block;
      width: 16px;
      color: var(--subtle);
      font-size: 10px;
      content: attr(aria-expanded);
    }
    &[aria-expanded='false']::before {
      content: '▶';
    }
    &[aria-expanded='true']::before {
      content: '▼';
    }
    &:hover {
      color: var(--accent);
    }
  }
  strong {
    color: var(--text);
    font-size: 13px;
    font-weight: 600;
  }
  .description {
    margin: 3px 0 0;
    color: var(--muted);
    font-size: 12px;
    line-height: 1.5;
  }
}

.object-properties {
  padding: 6px 0;
  background: var(--surface-raised);
  border: 1px solid var(--border);
  border-radius: 4px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
}

.property-row {
  display: grid;
  grid-template-columns: minmax(140px, 0.85fr) minmax(200px, 1.15fr) 28px;
  align-items: start;
  gap: 12px;
  padding: 8px 12px;
  border-bottom: 1px solid var(--border);

  &:last-child {
    border-bottom: 0;
  }
}

.property-meta {
  padding: 3px 0;

  .property-key {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    color: var(--text);
    font-family: var(--mono);
    font-size: 12px;
    font-weight: 500;
    line-height: 1.35;
  }

  .object-toggle {
    padding: 0;
    background: transparent;
    border: 0;
    font: inherit;
    cursor: pointer;
    text-align: left;

    &::before {
      display: inline-block;
      width: 14px;
      color: var(--subtle);
      font-size: 9px;
      content: '▶';
    }

    &[aria-expanded='true']::before {
      content: '▼';
    }

    &:hover {
      color: var(--accent);
    }
  }

  .description {
    margin: 3px 0 0;
    color: var(--muted);
    font-size: 11px;
    line-height: 1.45;
  }
}

.property-control > .schema-field.is-nested {
  .object-heading {
    display: none;
  }
  .object-properties {
    margin-top: 4px;
    padding: 4px 0;
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 3px;
  }
  .property-row {
    grid-template-columns: 110px minmax(100px, 1fr) 22px;
    gap: 8px;
    padding: 6px 10px;
  }
}

.form-control {
  width: 100%;
  height: 30px;
  padding: 3px 8px;
  color: var(--text);
  font-size: 12px;
  background: var(--surface);
  border: 1px solid var(--input-border);
  border-radius: 3px;
  transition:
    border-color 0.15s,
    box-shadow 0.15s;

  &:focus {
    border-color: var(--accent);
    box-shadow: 0 0 0 2px var(--accent-soft);
    outline: none;
  }
}

.text-control {
  font-family: var(--mono);
}

.variant-picker {
  display: grid;
  grid-template-columns: 86px minmax(0, 1fr);
  align-items: center;
  gap: 8px;
  margin-bottom: 10px;
  padding: 10px 12px;
  color: var(--muted);
  font-size: 11px;
  background: var(--surface-raised);
  border: 1px solid var(--border);
  border-radius: 4px;
}

.icon-button {
  width: 26px;
  height: 26px;
  padding: 0;
  color: var(--subtle);
  font-size: 20px;
  line-height: 1;
  background: transparent;
  border: 0;
  border-radius: 3px;
  cursor: pointer;

  &:hover {
    color: var(--danger);
    background: color-mix(in srgb, var(--danger) 10%, transparent);
  }
}

.add-property {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  margin-top: 10px;
  padding: 3px 12px;
  color: var(--accent);
  font-size: 12px;
  cursor: pointer;

  span {
    font-size: 16px;
  }
  select {
    max-width: 440px;
    color: var(--accent);
    font-size: 12px;
    background: transparent;
    border: 0;
    cursor: pointer;
  }
}

.switch {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  min-height: 30px;
  padding: 0 8px;
  color: var(--muted);
  font-size: 11px;
  cursor: pointer;

  input {
    position: absolute;
    opacity: 0;
    &:checked + span {
      background: var(--accent);
      &::after {
        transform: translateX(11px);
      }
    }
  }
  span {
    display: block;
    width: 25px;
    height: 14px;
    padding: 2px;
    background: var(--subtle);
    border-radius: 9px;
    transition: background 0.15s;

    &::after {
      display: block;
      width: 10px;
      height: 10px;
      background: #fff;
      border-radius: 50%;
      content: '';
      transition: transform 0.15s;
    }
  }
  b {
    min-width: 18px;
    font-weight: 400;
  }
}

.null-indicator {
  display: inline-block;
  padding: 2px 8px;
  color: var(--subtle);
  font-family: var(--mono);
  font-size: 12px;
  font-weight: 500;
  background: var(--surface);
  border: 1px dashed var(--border);
  border-radius: 3px;
}

.array-editor {
  display: grid;
  gap: 8px;
}

.array-item {
  display: grid;
  grid-template-columns: 1fr 26px;
  align-items: center;
  gap: 8px;
}

.secondary-button {
  padding: 5px 11px;
  color: var(--text);
  font-size: 12px;
  white-space: nowrap;
  background: var(--surface-raised);
  border: 1px solid var(--input-border);
  border-radius: 3px;
  cursor: pointer;

  &:hover {
    background: color-mix(in srgb, var(--text) 10%, var(--surface-raised));
  }
}

@media (max-width: $mobile) {
  .property-row {
    grid-template-columns: 1fr 24px;
    gap: 6px;
  }
  .property-meta {
    grid-column: 1 / -1;
  }
  .variant-picker {
    grid-template-columns: 1fr;
  }
  .property-control > .schema-field.is-nested .property-row {
    grid-template-columns: 1fr 20px;
  }
}
</style>
