<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{
  configJson: string
  error: string
  status: string
}>()

const emit = defineEmits<{
  'toggleCollapse': []
}>()

const highlightedPreviewLines = computed(() =>
  props.configJson
    .split('\n')
    .map((line) => highlightJsonc(line)),
)

function escapeHtml(value: string) {
  return value.replace(/[&<>"]/g, (character) =>
    ({
      '&': '&amp;',
      '<': '&lt;',
      '>': '&gt;',
      '"': '&quot;',
    })[character] ?? character,
  )
}

function highlightJsonc(line: string) {
  const escaped = escapeHtml(line)
  const tokenPattern =
    /("(?:\\.|[^"\\])*")(?=\s*:)|("(?:\\.|[^"\\])*")|\b(true|false|null)\b|(?<![\w."])(-?\d+(?:\.\d+)?(?:[eE][+-]?\d+)?)|([{}\[\],:])/g
  return escaped.replace(
    tokenPattern,
    (token, property, string, literal, number, punctuation) => {
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
    },
  )
}
</script>

<template>
  <aside class="preview-pane">
    <div class="preview-heading">
      <span>JSON PREVIEW</span>
      <div class="preview-heading-actions">
        <span class="saved-dot">● {{ error ? 'Error' : 'Ready' }}</span>
        <button class="preview-toggle" type="button" title="Close preview" @click="emit('toggleCollapse')">×</button>
      </div>
    </div>
    <ol class="code-preview" aria-label="Generated config.jsonc preview">
      <li v-for="(line, index) in highlightedPreviewLines" :key="index">
        <code v-html="line || ' '" />
      </li>
    </ol>
    <footer>{{ error || status }}</footer>
  </aside>
</template>

<style scoped lang="scss">
.preview-pane {
  position: relative;
  display: flex;
  flex-direction: column;
  min-width: 0;
  min-height: 0;
  color: var(--code);
  background: var(--surface-raised);
  border-left: 1px solid var(--border);

  footer {
    padding: 9px 14px;
    color: var(--muted);
    font-size: 11px;
    border-top: 1px solid var(--border);
  }
}

.preview-heading {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 42px;
  padding: 0 14px;
  color: var(--text);
  font: 12px var(--mono);
  border-bottom: 1px solid var(--border);
}

.preview-heading-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.preview-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  padding: 0;
  color: var(--subtle);
  font-size: 16px;
  line-height: 1;
  background: transparent;
  border: 0;
  border-radius: 3px;
  cursor: pointer;

  &:hover {
    color: var(--text);
    background: color-mix(in srgb, var(--text) 10%, transparent);
  }
}

.saved-dot {
  color: var(--muted);
  font-family: inherit;
  font-size: 10px;
}

.code-preview {
  flex: 1;
  min-height: 0;
  padding: 14px 14px 14px 48px;
  margin: 0;
  overflow: auto;
  color: var(--code);
  font: 12px/1.55 var(--mono);
  white-space: pre;
  tab-size: 2;

  li {
    padding-left: 10px;

    &::marker {
      color: var(--subtle);
      font-size: 10px;
      text-align: right;
    }
  }

  code {
    font: inherit;
  }
}

.token-property {
  color: #0451a5;
}

.token-string {
  color: #a31515;
}

.token-number {
  color: #098658;
}

.token-literal {
  color: #0000ff;
}

.token-punctuation {
  color: var(--code);
}

@media (prefers-color-scheme: dark) {
  .token-property {
    color: #9cdcfe;
  }

  .token-string {
    color: #ce9178;
  }

  .token-number {
    color: #b5cea8;
  }

  .token-literal {
    color: #569cd6;
  }
}
</style>
