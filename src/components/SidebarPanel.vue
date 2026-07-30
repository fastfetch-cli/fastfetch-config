<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{
  activeSection: string
  search: string
  sections: { id: string; label: string; icon: string }[]
  schemaUrl: string
  collapsed: boolean
}>()

const emit = defineEmits<{
  'update:activeSection': [id: string]
  'update:search': [value: string]
  'toggleCollapse': []
}>()

const searchModel = computed({
  get: () => props.search,
  set: (val) => emit('update:search', val),
})
</script>

<template>
  <aside class="sidebar" :class="{ collapsed }">
    <div v-if="!collapsed" class="search-box">
      <span>⌕</span>
      <input v-model="searchModel" type="search" placeholder="Search settings" />
    </div>
    <button
      class="collapse-toggle"
      type="button"
      :title="collapsed ? 'Expand sidebar' : 'Collapse sidebar'"
      @click="emit('toggleCollapse')"
    >{{ collapsed ? '▸' : '◂' }}</button>
    <nav aria-label="Settings sections">
      <button
        v-for="section in sections"
        :key="section.id"
        class="nav-item"
        :class="{ active: activeSection === section.id }"
        type="button"
        :title="collapsed ? section.label : undefined"
        @click="emit('update:activeSection', section.id)"
      >
        <span>{{ section.icon }}</span>
        <span v-if="!collapsed" class="nav-label">{{ section.label }}</span>
      </button>
    </nav>
    <p v-if="!collapsed" class="source-note">
      Schema source<br />
      <a :href="schemaUrl" target="_blank" rel="noreferrer">fastfetch-cli/fastfetch</a>
    </p>
  </aside>
</template>

<style scoped lang="scss">
.sidebar {
  display: flex;
  flex-direction: column;
  padding: 16px 10px;
  overflow-y: auto;
  background: var(--sidebar);
  border-right: 1px solid var(--border);

  &.collapsed {
    padding: 16px 4px;
    align-items: center;

    nav { width: 100%; }
  }

  nav {
    display: grid;
    gap: 2px;
  }
}

.collapse-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 24px;
  margin-bottom: 8px;
  padding: 0;
  color: var(--subtle);
  font-size: 14px;
  line-height: 1;
  background: transparent;
  border: 0;
  border-radius: 3px;
  cursor: pointer;

  &:hover {
    color: var(--text);
    background: color-mix(in srgb, var(--text) 8%, transparent);
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

  .sidebar.collapsed & {
    justify-content: center;
    padding: 4px 0;
  }

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
    flex-shrink: 0;
  }
}

.nav-label {
  width: auto !important;
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
