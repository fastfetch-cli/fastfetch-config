<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{
  activeSection: string
  search: string
  sections: { id: string; label: string; icon: string }[]
  schemaUrl: string
}>()

const emit = defineEmits<{
  'update:activeSection': [id: string]
  'update:search': [value: string]
}>()

const searchModel = computed({
  get: () => props.search,
  set: (val) => emit('update:search', val),
})
</script>

<template>
  <aside class="sidebar">
    <div class="search-box">
      <span>⌕</span>
      <input v-model="searchModel" type="search" placeholder="Search settings" />
    </div>
    <nav aria-label="Settings sections">
      <button
        v-for="section in sections"
        :key="section.id"
        class="nav-item"
        :class="{ active: activeSection === section.id }"
        type="button"
        @click="emit('update:activeSection', section.id)"
      >
        <span>{{ section.icon }}</span>{{ section.label }}
      </button>
    </nav>
    <p class="source-note">
      Schema source<br />
      <a :href="schemaUrl" target="_blank" rel="noreferrer">fastfetch-cli/fastfetch</a>
    </p>
  </aside>
</template>

<style scoped lang="scss">
.sidebar {
  min-height: 0;
  padding: 16px 10px;
  overflow-y: auto;
  background: var(--sidebar);
  border-right: 1px solid var(--border);

  nav {
    display: grid;
    gap: 2px;
  }
}

.search-box {
  display: flex;
  align-items: center;
  gap: 8px;
  height: 30px;
  margin: 0 2px 14px;
  padding: 0 8px;
  color: var(--subtle);
  background: var(--surface);
  border: 1px solid var(--input-border);
  border-radius: 3px;

  input {
    width: 100%;
    min-width: 0;
    color: var(--text);
    font-size: 12px;
    background: transparent;
    border: 0;
    outline: none;
  }
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 10px;
  min-height: 30px;
  padding: 4px 10px;
  color: var(--text);
  font-size: 13px;
  text-align: left;
  background: transparent;
  border: 0;
  border-radius: 4px;

  &:hover {
    background: color-mix(in srgb, var(--text) 8%, transparent);
  }

  &.active {
    background: var(--accent-soft);
  }

  span {
    width: 16px;
    color: var(--muted);
    text-align: center;
  }
}

.source-note {
  position: sticky;
  top: calc(100vh - 95px);
  margin: 28px 10px 0;
  color: var(--subtle);
  font-size: 11px;
  line-height: 1.55;

  a {
    color: var(--accent);
    text-decoration: none;
  }
}
</style>
