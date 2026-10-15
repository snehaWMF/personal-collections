<script setup lang="ts">
/** Mobile Saved page settings menu (gear): switch the item layout between list and grid. */
import { onBeforeUnmount, ref, watch } from 'vue'
import { CdxButton, CdxIcon } from '@wikimedia/codex'
import {
  cdxIconCheck,
  cdxIconImageGallery,
  cdxIconListBullet,
  cdxIconSettings,
} from '@wikimedia/codex-icons'

import { useCollections } from './useCollections'

const { ui } = useCollections()

const VIEW_OPTIONS = [
  { value: 'list', label: 'List', icon: cdxIconListBullet },
  { value: 'grid', label: 'Grid', icon: cdxIconImageGallery },
] as const

const open = ref(false)
const rootRef = ref<HTMLElement | null>(null)

function choose(view: 'list' | 'grid') {
  ui.mobileView = view
  open.value = false
}

function onDocPointerDown(event: PointerEvent) {
  if (!rootRef.value?.contains(event.target as Node)) open.value = false
}

function onKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape') open.value = false
}

watch(open, (isOpen) => {
  const method = isOpen ? 'addEventListener' : 'removeEventListener'
  document[method]('pointerdown', onDocPointerDown as EventListener)
  document[method]('keydown', onKeydown as EventListener)
})

onBeforeUnmount(() => {
  document.removeEventListener('pointerdown', onDocPointerDown)
  document.removeEventListener('keydown', onKeydown)
})
</script>

<template>
  <span ref="rootRef" class="view-settings">
    <CdxButton
      class="view-settings__button"
      weight="quiet"
      aria-label="Settings"
      aria-haspopup="true"
      :aria-expanded="open"
      @click="open = !open"
    >
      <CdxIcon :icon="cdxIconSettings" />
    </CdxButton>
    <div v-if="open" class="view-settings__panel" role="menu">
      <p class="view-settings__label">View</p>
      <button
        v-for="option in VIEW_OPTIONS"
        :key="option.value"
        type="button"
        class="view-settings__option"
        role="menuitemradio"
        :aria-checked="ui.mobileView === option.value"
        @click="choose(option.value)"
      >
        <CdxIcon :icon="option.icon" size="small" />
        <span>{{ option.label }}</span>
        <CdxIcon
          v-if="ui.mobileView === option.value"
          class="view-settings__check"
          :icon="cdxIconCheck"
          size="small"
        />
      </button>
    </div>
  </span>
</template>

<style scoped>
.view-settings {
  position: relative;
  margin-inline: auto -6px;
}

.view-settings__button.cdx-button {
  color: var(--color-base, #202122);
}

.view-settings__panel {
  position: absolute;
  top: calc(100% + 4px);
  right: 0;
  z-index: 40;
  box-sizing: border-box;
  min-width: 180px;
  padding: var(--spacing-50, 8px) 0;
  border: 1px solid var(--border-color-base, #a2a9b1);
  border-radius: var(--border-radius-base, 2px);
  background-color: var(--background-color-base, #fff);
  box-shadow: var(--box-shadow-medium, 0 2px 8px rgba(0, 0, 0, 0.15));
}

.view-settings__label {
  margin: 0;
  padding: var(--spacing-25, 4px) var(--spacing-100, 16px);
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-x-small, 12px);
  font-weight: var(--font-weight-bold, 700);
}

.view-settings__option {
  display: flex;
  align-items: center;
  gap: var(--spacing-75, 12px);
  width: 100%;
  padding: var(--spacing-50, 8px) var(--spacing-100, 16px);
  border: 0;
  background: none;
  font: inherit;
  font-size: var(--font-size-medium, 16px);
  color: var(--color-base, #202122);
  text-align: start;
  cursor: pointer;
}

.view-settings__option:hover {
  background-color: var(--background-color-interactive-subtle, #f8f9fa);
}

.view-settings__option :deep(.cdx-icon) {
  color: var(--color-base, #202122);
}

.view-settings__option .view-settings__check {
  margin-inline-start: auto;
  color: var(--color-progressive, #36c);
}
</style>
