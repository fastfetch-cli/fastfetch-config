<script vapor lang="ts">
import { computed } from 'vue';
import { marked } from 'marked';

const props = defineProps<{ source: string }>();

const formatted = computed(() => marked.parse(props.source, { async: false }));
</script>

<template>
  <div class="markdown-description" v-html="formatted" />
</template>

<style lang="scss">
.markdown-description {
  > :first-child {
    margin-top: 0;
  }
  > :last-child {
    margin-bottom: 0;
  }
  p {
    margin: 0 0 6px;
    white-space: pre-wrap;
  }
  ul,
  ol {
    padding-left: 20px;
    margin: 4px 0 6px;
  }
  li + li {
    margin-top: 2px;
  }
  pre {
    padding: 8px;
    margin: 6px 0;
    overflow-x: auto;
    background: color-mix(in srgb, var(--text) 8%, transparent);
    border-radius: 3px;
    code {
      padding: 0;
      background: transparent;
    }
  }
  code {
    padding: 1px 3px;
    color: var(--text);
    font: 11px var(--mono);
    background: color-mix(in srgb, var(--text) 8%, transparent);
    border-radius: 2px;
  }
  table {
    width: 100%;
    margin: 6px 0;
    border-collapse: collapse;
  }
  th,
  td {
    padding: 4px 6px;
    text-align: left;
    border: 1px solid var(--border);
  }
  blockquote {
    padding-left: 8px;
    margin: 6px 0;
    border-left: 3px solid var(--accent);
  }
  a {
    color: var(--accent);
    text-decoration: none;
    &:hover {
      text-decoration: underline;
    }
  }
}
</style>
