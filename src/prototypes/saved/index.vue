<script setup lang="ts">
definePage({
  meta: {
    title: 'Saved',
    platform: 'web',
    // Reached from the prototype's header, not listed on its own in the gallery.
    hidden: true,
  },
})

import { computed, nextTick, onBeforeUnmount, ref, watch } from 'vue'
import { RouterLink, useRoute, useRouter } from 'vue-router'
import { CdxButton, CdxIcon } from '@wikimedia/codex'
import {
  cdxIconAdd,
  cdxIconCheck,
  cdxIconDownTriangle,
  cdxIconEllipsis,
  cdxIconImageGallery,
  cdxIconListBullet,
  cdxIconLock,
  cdxIconSettings,
} from '@wikimedia/codex-icons'

import SpecialPageWrapper from '@/components/SpecialPageWrapper.vue'
import CollectionThumb from '../personal-collections/CollectionThumb.vue'
import CreateCollectionDialog from '../personal-collections/CreateCollectionDialog.vue'
import SavedCard from '../personal-collections/SavedCard.vue'
import SavedChrome from '../personal-collections/SavedChrome.vue'
import { SAVED_PATH, useCollections, type Collection } from '../personal-collections/useCollections'
import { globalSkin } from '@/theme'

const SAVED_PAGE = SAVED_PATH

const route = useRoute()
const router = useRouter()
const { collections, ui, findCollection, itemsIn } = useCollections()

const activeCollection = computed(() => findCollection(String(route.query.collection ?? '')))
const visibleItems = computed(() => itemsIn(activeCollection.value?.id))

/** Phones (Minerva) get their own layout: All items / Collections tabs, collections as a list. */
const isMobile = computed(() => globalSkin.value === 'mobile')
const mobileTab = computed(() =>
  activeCollection.value || route.query.view === 'collections' ? 'collections' : 'all',
)
function countLabel(collectionId: string) {
  const n = itemsIn(collectionId).length
  return `${n} ${n === 1 ? 'item' : 'items'}`
}

const collectionsTabTo = { path: SAVED_PAGE, query: { view: 'collections' } }

/** Mobile settings menu (gear): switch the item layout between list and grid. */
const settingsOpen = ref(false)
const settingsRef = ref<HTMLElement | null>(null)
const VIEW_OPTIONS = [
  { value: 'list', label: 'List', icon: cdxIconListBullet },
  { value: 'grid', label: 'Grid', icon: cdxIconImageGallery },
] as const

function chooseView(view: 'list' | 'grid') {
  ui.mobileView = view
  settingsOpen.value = false
}

function onSettingsPointerDown(event: PointerEvent) {
  if (!settingsRef.value?.contains(event.target as Node)) settingsOpen.value = false
}

function onSettingsKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape') settingsOpen.value = false
}

watch(settingsOpen, (isOpen) => {
  const method = isOpen ? 'addEventListener' : 'removeEventListener'
  document[method]('pointerdown', onSettingsPointerDown as EventListener)
  document[method]('keydown', onSettingsKeydown as EventListener)
})

const dropdownOpen = ref(false)
const dropdownRef = ref<HTMLElement | null>(null)
const dialogOpen = ref(false)

function collectionTo(collection: Collection) {
  return { path: SAVED_PAGE, query: { collection: collection.id } }
}

function onDocPointerDown(event: PointerEvent) {
  if (!dropdownRef.value?.contains(event.target as Node)) dropdownOpen.value = false
}

function onKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape') dropdownOpen.value = false
}

watch(dropdownOpen, (isOpen) => {
  if (isOpen) {
    document.addEventListener('pointerdown', onDocPointerDown)
    document.addEventListener('keydown', onKeydown)
  } else {
    document.removeEventListener('pointerdown', onDocPointerDown)
    document.removeEventListener('keydown', onKeydown)
  }
})

onBeforeUnmount(() => {
  document.removeEventListener('pointerdown', onDocPointerDown)
  document.removeEventListener('keydown', onKeydown)
  document.removeEventListener('pointerdown', onSettingsPointerDown)
  document.removeEventListener('keydown', onSettingsKeydown)
})

watch(
  () => route.query.collection,
  () => {
    dropdownOpen.value = false
  },
)

function onCreateClick() {
  dropdownOpen.value = false
  dialogOpen.value = true
}

async function pinToSide() {
  dropdownOpen.value = false
  ui.pinned = true
  await nextTick()
  document.querySelector<HTMLElement>('.saved-sidebar .saved-chip')?.focus()
}

function onCollectionCreated(collection: Collection) {
  void router.push(collectionTo(collection))
}
</script>

<template>
  <SavedChrome :last-edited-notice="false">
    <SpecialPageWrapper class="saved-page" help>
      <template #title>
        <template v-if="activeCollection"> Saved / {{ activeCollection.name }} </template>
        <template v-else>Saved</template>
        <CdxIcon
          v-if="!isMobile"
          class="saved-page__lock"
          :icon="cdxIconLock"
          size="small"
          title="Only you can see your saved items"
          aria-label="Private"
        />
      </template>

      <!-- ---------- Mobile (Minerva) ---------- -->
      <template v-if="isMobile">
        <nav
          class="saved-m-tabs"
          :class="{ 'saved-m-tabs--in-collection': activeCollection }"
          aria-label="Saved"
        >
          <RouterLink
            class="saved-m-tabs__tab"
            :class="{ 'saved-m-tabs__tab--active': mobileTab === 'all' }"
            :to="SAVED_PAGE"
          >
            All items
          </RouterLink>
          <RouterLink
            class="saved-m-tabs__tab"
            :class="{
              'saved-m-tabs__tab--active': mobileTab === 'collections' && !activeCollection,
            }"
            :to="collectionsTabTo"
          >
            Collections
          </RouterLink>
          <span ref="settingsRef" class="saved-m-settings">
            <CdxButton
              class="saved-m-tabs__settings"
              weight="quiet"
              aria-label="Settings"
              aria-haspopup="true"
              :aria-expanded="settingsOpen"
              @click="settingsOpen = !settingsOpen"
            >
              <CdxIcon :icon="cdxIconSettings" />
            </CdxButton>
            <div v-if="settingsOpen" class="saved-m-settings__panel" role="menu">
              <p class="saved-m-settings__label">View</p>
              <button
                v-for="option in VIEW_OPTIONS"
                :key="option.value"
                type="button"
                class="saved-m-settings__option"
                role="menuitemradio"
                :aria-checked="ui.mobileView === option.value"
                @click="chooseView(option.value)"
              >
                <CdxIcon :icon="option.icon" size="small" />
                <span>{{ option.label }}</span>
                <CdxIcon
                  v-if="ui.mobileView === option.value"
                  class="saved-m-settings__check"
                  :icon="cdxIconCheck"
                  size="small"
                />
              </button>
            </div>
          </span>
        </nav>

        <!-- Collections tab: a plain list with item counts. -->
        <ul v-if="mobileTab === 'collections' && !activeCollection" class="saved-m-list">
          <li>
            <button type="button" class="saved-m-list__create" @click="dialogOpen = true">
              Create collection
            </button>
          </li>
          <li v-for="collection in collections" :key="collection.id">
            <RouterLink class="saved-m-list__row" :to="collectionTo(collection)">
              <CollectionThumb :collection-id="collection.id" />
              <span class="saved-m-list__text">
                <span class="saved-m-list__name">{{ collection.name }}</span>
                <span class="saved-m-list__count">{{ countLabel(collection.id) }}</span>
              </span>
            </RouterLink>
          </li>
        </ul>

        <!-- All items, or one collection's items: two-column card grid. -->
        <template v-else>
          <template v-if="visibleItems.length">
            <p class="saved-page__sort saved-page__sort--mobile">Sorted by most recent</p>
            <ul v-if="ui.mobileView === 'grid'" class="saved-grid saved-grid--mobile">
              <li v-for="item in visibleItems" :key="item.key">
                <SavedCard :item="item" />
              </li>
            </ul>
            <ul v-else class="saved-m-rows">
              <li v-for="item in visibleItems" :key="item.key">
                <SavedCard :item="item" list />
              </li>
            </ul>
          </template>
          <div v-else class="saved-page__empty">
            <p class="saved-page__empty-title">
              {{ activeCollection ? 'This collection is empty' : 'No saved items yet' }}
            </p>
            <p>Use the bookmark on any article to save it here.</p>
          </div>
        </template>
      </template>

      <!-- ---------- Desktop ---------- -->
      <template v-else>
        <div class="saved-page__more-row">
          <CdxButton weight="quiet" aria-label="More options">
            <CdxIcon class="saved-page__kebab" :icon="cdxIconEllipsis" />
          </CdxButton>
        </div>

        <div class="saved-page__layout" :class="{ 'saved-page__layout--pinned': ui.pinned }">
          <!--
          Pinned: one "Collections" group, like Wikipedia's pinned Contents/Tools
          menus. "All items" is the first row — a smart collection of everything —
          then the create action, then the reader's collections (newest first, so a
          new one appears right under the button that made it).
        -->
          <nav v-if="ui.pinned" class="saved-sidebar" aria-label="Collections">
            <div class="saved-sidebar__heading">
              <span>Collections</span>
              <button type="button" class="saved-chip" @click="ui.pinned = false">hide</button>
            </div>

            <ul class="saved-sidebar__list">
              <li>
                <RouterLink
                  class="saved-sidebar__link"
                  :class="{ 'saved-sidebar__link--active': !activeCollection }"
                  :to="SAVED_PAGE"
                >
                  <span>All items</span>
                </RouterLink>
              </li>
              <li>
                <button type="button" class="saved-sidebar__create" @click="dialogOpen = true">
                  <CdxIcon :icon="cdxIconAdd" size="x-small" /> Create collection
                </button>
              </li>
              <li v-for="collection in collections" :key="collection.id">
                <RouterLink
                  class="saved-sidebar__link"
                  :class="{ 'saved-sidebar__link--active': activeCollection?.id === collection.id }"
                  :to="collectionTo(collection)"
                >
                  <span>{{ collection.name }}</span>
                </RouterLink>
              </li>
            </ul>
          </nav>

          <div class="saved-page__main">
            <div class="saved-page__toolbar">
              <!-- Unpinned: All items + Collections dropdown -->
              <div v-if="!ui.pinned" class="saved-tabs">
                <RouterLink
                  class="saved-tabs__tab"
                  :class="{ 'saved-tabs__tab--active': !activeCollection }"
                  :to="SAVED_PAGE"
                >
                  All items
                </RouterLink>

                <div ref="dropdownRef" class="saved-dropdown">
                  <button
                    type="button"
                    class="saved-tabs__tab saved-dropdown__trigger"
                    :aria-expanded="dropdownOpen"
                    aria-haspopup="true"
                    @click="dropdownOpen = !dropdownOpen"
                  >
                    Collections
                    <CdxIcon :icon="cdxIconDownTriangle" size="x-small" />
                  </button>

                  <div v-if="dropdownOpen" class="saved-dropdown__panel">
                    <div class="saved-dropdown__head">
                      <button type="button" class="saved-dropdown__create" @click="onCreateClick">
                        <CdxIcon :icon="cdxIconAdd" size="x-small" /> Create collection
                      </button>
                      <button type="button" class="saved-chip" @click="pinToSide">
                        move to side
                      </button>
                    </div>
                    <ul class="saved-dropdown__list">
                      <li v-for="collection in collections" :key="collection.id">
                        <RouterLink
                          class="saved-dropdown__item"
                          :class="{
                            'saved-dropdown__item--active': activeCollection?.id === collection.id,
                          }"
                          :to="collectionTo(collection)"
                        >
                          {{ collection.name }}
                        </RouterLink>
                      </li>
                      <li v-if="!collections.length" class="saved-dropdown__empty">
                        No collections yet
                      </li>
                    </ul>
                  </div>
                </div>
              </div>
              <p v-else-if="visibleItems.length" class="saved-page__sort">Sorted by most recent</p>
              <span v-else />

              <CdxButton weight="quiet" aria-label="Settings">
                <CdxIcon :icon="cdxIconSettings" />
              </CdxButton>
            </div>

            <template v-if="visibleItems.length">
              <p v-if="!ui.pinned" class="saved-page__sort">Sorted by most recent</p>
              <ul class="saved-grid">
                <li v-for="item in visibleItems" :key="item.key">
                  <SavedCard :item="item" />
                </li>
              </ul>
            </template>

            <div v-else class="saved-page__empty">
              <template v-if="activeCollection">
                <p class="saved-page__empty-title">This collection is empty</p>
                <p>Use the bookmark on any article to add it to {{ activeCollection.name }}.</p>
              </template>
              <template v-else>
                <p class="saved-page__empty-title">No saved items yet</p>
                <p>Use the bookmark on any article to save it here.</p>
              </template>
            </div>
          </div>
        </div>
      </template>
    </SpecialPageWrapper>
  </SavedChrome>

  <CreateCollectionDialog v-model:open="dialogOpen" @created="onCollectionCreated" />
</template>

<style scoped>
.saved-page {
  padding-bottom: var(--spacing-300, 48px);
}

.saved-page__lock {
  margin-inline-start: var(--spacing-50, 8px);
  color: var(--color-subtle, #54595d);
  vertical-align: middle;
}

/* Codex only ships a horizontal ellipsis — rotate it into a vertical "⋮". */
.saved-page__kebab {
  transform: rotate(90deg);
}

.saved-page__more-row {
  display: flex;
  justify-content: flex-end;
  margin-top: calc(-1 * var(--spacing-75, 12px));
  padding-bottom: 2px;
  border-bottom: 1px solid var(--border-color-subtle, #c8ccd1);
}

.saved-page__layout {
  display: block;
}

.saved-page__layout--pinned {
  display: grid;
  grid-template-columns: 212px minmax(0, 1fr);
  gap: var(--spacing-250, 40px);
}

.saved-page__toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  min-height: 44px;
  margin-top: var(--spacing-50, 8px);
}

.saved-page__sort {
  margin: 0 0 var(--spacing-25, 4px);
  font-size: var(--font-size-x-small, 12px);
  line-height: var(--line-height-x-small, 1.5);
}

.saved-page__empty {
  max-width: 480px;
  margin-top: var(--spacing-150, 24px);
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-small, 14px);
}

.saved-page__empty p {
  margin: 0 0 var(--spacing-25, 4px);
}

.saved-page__empty-title {
  color: var(--color-base, #202122);
  font-weight: var(--font-weight-bold, 700);
}

.saved-page__layout--pinned .saved-page__toolbar {
  align-items: flex-end;
  margin-bottom: var(--spacing-25, 4px);
}

.saved-page__sort--mobile {
  margin: var(--spacing-100, 16px) 0 var(--spacing-50, 8px);
}

.saved-page__layout--pinned .saved-page__toolbar .saved-page__sort {
  margin: 0;
}

/* ---- Grid ---- */
.saved-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(176px, 194px));
  gap: var(--spacing-75, 14px);
  margin: 0;
  padding: 0;
  list-style: none;
}

.saved-grid > li {
  display: flex;
  margin: 0;
}

.saved-grid > li > * {
  flex: 1;
}

/* ---- Tabs + dropdown ---- */
.saved-tabs {
  display: flex;
  align-items: center;
  gap: var(--spacing-100, 16px);
}

.saved-tabs__tab {
  display: inline-flex;
  align-items: center;
  gap: var(--spacing-50, 8px);
  margin: 0;
  padding: var(--spacing-25, 4px);
  border: 0;
  background: none;
  font: inherit;
  font-size: var(--font-size-small, 14px);
  /* Tabs are always bold; colour alone marks the current one. */
  font-weight: var(--font-weight-bold, 700);
  color: var(--color-progressive, #36c);
  text-decoration: none;
  cursor: pointer;
}

.saved-tabs__tab:hover {
  text-decoration: underline;
}

.saved-tabs__tab--active {
  color: var(--color-base, #202122);
  font-weight: var(--font-weight-bold, 700);
}

.saved-tabs__tab--active:hover {
  text-decoration: none;
}

.saved-dropdown {
  position: relative;
}

.saved-dropdown__trigger :deep(.cdx-icon) {
  color: var(--color-progressive, #36c);
}

.saved-dropdown__panel {
  position: absolute;
  top: calc(100% + 2px);
  left: 0;
  z-index: 40;
  box-sizing: border-box;
  min-width: 288px;
  border: 1px solid var(--border-color-base, #a2a9b1);
  border-radius: var(--border-radius-base, 2px);
  background-color: var(--background-color-base, #fff);
  box-shadow: var(--box-shadow-medium, 0 2px 8px rgba(0, 0, 0, 0.15));
  font-size: var(--font-size-small, 14px);
}

.saved-dropdown__head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--spacing-100, 16px);
  padding: var(--spacing-50, 8px) var(--spacing-50, 8px) var(--spacing-50, 8px)
    var(--spacing-75, 12px);
  border-bottom: 1px solid var(--border-color-subtle, #c8ccd1);
}

.saved-dropdown__create {
  display: inline-flex;
  align-items: center;
  gap: var(--spacing-25, 4px);
  padding: 0;
  border: 0;
  background: none;
  font: inherit;
  color: var(--color-progressive, #36c);
  cursor: pointer;
}

.saved-dropdown__create :deep(.cdx-icon) {
  color: var(--color-progressive, #36c);
}

.saved-dropdown__create:hover {
  text-decoration: underline;
}

.saved-dropdown__list {
  margin: 0;
  padding: var(--spacing-25, 4px) 0;
  list-style: none;
}

.saved-dropdown__list li {
  margin: 0;
}

.saved-dropdown__item {
  display: block;
  padding: var(--spacing-50, 8px) var(--spacing-75, 12px);
  color: var(--color-base, #202122);
  text-decoration: none;
}

.saved-dropdown__item:hover {
  background-color: var(--background-color-interactive-subtle, #f8f9fa);
}

.saved-dropdown__item--active {
  font-weight: var(--font-weight-bold, 700);
}

.saved-dropdown__empty {
  padding: var(--spacing-50, 8px) var(--spacing-75, 12px);
  color: var(--color-subtle, #54595d);
}

/* Small neutral chip — matches the "hide" chip on Wikipedia's pinnable menus. */
.saved-chip {
  padding: 1px var(--spacing-50, 8px);
  border: 0;
  border-radius: var(--border-radius-base, 2px);
  background-color: var(--background-color-interactive, #eaecf0);
  font: inherit;
  font-size: var(--font-size-small, 14px);
  color: var(--color-base, #202122);
  cursor: pointer;
}

.saved-chip:hover {
  background-color: var(--background-color-interactive--hover, #dadde3);
}

/* ---- Pinned sidebar ---- */
.saved-sidebar {
  padding-top: var(--spacing-100, 16px);
  font-size: var(--font-size-small, 14px);
}

.saved-sidebar__heading {
  display: flex;
  align-items: center;
  gap: var(--spacing-50, 8px);
  padding-bottom: var(--spacing-50, 8px);
  border-bottom: 1px solid var(--border-color-subtle, #c8ccd1);
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-x-small, 12px);
}

.saved-sidebar__heading .saved-chip {
  font-size: var(--font-size-x-small, 12px);
}

.saved-sidebar__list {
  margin: 0;
  padding: 0;
  list-style: none;
}

/* A line under every row so each reads as its own button. */
.saved-sidebar__list li {
  margin: 0;
  border-bottom: 1px solid var(--border-color-subtle, #c8ccd1);
}

.saved-sidebar__link,
.saved-sidebar__create {
  display: flex;
  align-items: center;
  gap: var(--spacing-50, 8px);
  box-sizing: border-box;
  width: 100%;
  min-height: 44px;
  padding: var(--spacing-50, 8px) var(--spacing-50, 8px) var(--spacing-50, 8px) 0;
  border: 0;
  background: none;
  font: inherit;
  text-align: start;
  text-decoration: none;
}

.saved-sidebar__link {
  justify-content: space-between;
  color: var(--color-base, #202122);
}

.saved-sidebar__link:hover {
  color: var(--color-progressive, #36c);
}

.saved-sidebar__link--active {
  font-weight: var(--font-weight-bold, 700);
}

.saved-sidebar__create {
  gap: var(--spacing-25, 4px);
  color: var(--color-progressive, #36c);
  cursor: pointer;
}

.saved-sidebar__create :deep(.cdx-icon) {
  color: var(--color-progressive, #36c);
}

.saved-sidebar__create:hover {
  text-decoration: underline;
}

/* ---- Mobile (Minerva) ---- */
.saved-m-tabs {
  display: flex;
  align-items: center;
  gap: var(--spacing-150, 24px);
  padding-bottom: var(--spacing-50, 8px);
  border-bottom: 1px solid var(--border-color-subtle, #c8ccd1);
  font-size: var(--font-size-small, 14px);
}

.saved-m-settings {
  position: relative;
  margin-inline: auto -6px;
}

.saved-m-tabs__settings.cdx-button {
  color: var(--color-base, #202122);
}

.saved-m-settings__panel {
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

.saved-m-settings__label {
  margin: 0;
  padding: var(--spacing-25, 4px) var(--spacing-100, 16px);
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-x-small, 12px);
  font-weight: var(--font-weight-bold, 700);
}

.saved-m-settings__option {
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

.saved-m-settings__option:hover {
  background-color: var(--background-color-interactive-subtle, #f8f9fa);
}

.saved-m-settings__option :deep(.cdx-icon) {
  color: var(--color-base, #202122);
}

.saved-m-settings__option .saved-m-settings__check {
  margin-inline-start: auto;
  color: var(--color-progressive, #36c);
}

.saved-grid--mobile {
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: var(--spacing-75, 12px);
}

.saved-m-tabs--in-collection {
  border-bottom: 0;
}

.saved-m-tabs__tab {
  font-weight: var(--font-weight-bold, 700);
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.saved-m-tabs__tab--active {
  color: var(--color-base, #202122);
  font-weight: var(--font-weight-bold, 700);
}

.saved-m-list {
  margin: 0;
  padding: 0;
  list-style: none;
}

.saved-m-list li {
  margin: 0;
  border-bottom: 1px solid var(--border-color-subtle, #c8ccd1);
}

.saved-m-list__create {
  display: block;
  width: 100%;
  padding: var(--spacing-100, 16px) 0;
  border: 0;
  background: none;
  font: inherit;
  font-size: var(--font-size-small, 14px);
  color: var(--color-progressive, #36c);
  text-align: start;
  cursor: pointer;
}

.saved-m-list__row {
  display: flex;
  align-items: flex-start;
  gap: var(--spacing-75, 12px);
  padding: var(--spacing-100, 16px) 0;
  color: var(--color-base, #202122);
  text-decoration: none;
}

.saved-m-list__text {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.saved-m-list__count {
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-small, 14px);
  line-height: var(--line-height-small, 1.571);
}

.saved-m-list__name {
  font-size: var(--font-size-large, 18px);
  line-height: var(--line-height-small, 1.375);
}

.saved-m-rows {
  margin: 0;
  padding: 0;
  list-style: none;
}

.saved-m-rows li {
  margin: 0;
  border-bottom: 1px solid var(--border-color-subtle, #c8ccd1);
}
</style>
