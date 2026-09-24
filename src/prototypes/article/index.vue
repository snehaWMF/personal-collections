<script setup lang="ts">
definePage({
  meta: {
    // Placeholder copy — replace with the author's title and description.
    title: 'TODO: title',
    description: 'TODO: description',
    platform: 'web',
  },
})

import { computed, nextTick, ref, shallowRef, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { CdxButton, CdxIcon } from '@wikimedia/codex'
import {
  cdxIconBookmark,
  cdxIconBookmarkOutline,
  cdxIconEllipsis,
  cdxIconStar,
} from '@wikimedia/codex-icons'

import ArticleLive from '@/components/article/ArticleLive.vue'
import { globalSkin } from '@/theme'
import CreateCollectionDialog from '../personal-collections/CreateCollectionDialog.vue'
import InArticleSaves from '../personal-collections/InArticleSaves.vue'
import SavedChrome from '../personal-collections/SavedChrome.vue'
import SavePopover from '../personal-collections/SavePopover.vue'
import StickyHeader from '../personal-collections/StickyHeader.vue'
import {
  ARTICLE_PATH,
  keyOf,
  labelOf,
  normalizeTitle,
  SAVED_PATH,
  useCollections,
  type Collection,
  type SaveTarget,
} from '../personal-collections/useCollections'
import { useToast } from '../personal-collections/useToast'

const route = useRoute()
const router = useRouter()
const { isSaved, save, addToCollection, findPassage, startArticle } = useCollections()
const { show, dismiss } = useToast()

const article = computed(() => normalizeTitle(String(route.query.title || startArticle.value)))
const saved = computed(() => isSaved(article.value))

/** Where a Saved-page card asked to land: a section anchor or a saved passage's text. */
const focusSection = computed(() => (route.query.section ? String(route.query.section) : undefined))
const focusExcerpt = computed(() => {
  if (!route.query.passage) return undefined
  const item = findPassage(article.value, String(route.query.passage))
  return item?.kind === 'passage' ? item.excerpt : undefined
})

const parserRoot = shallowRef<HTMLElement | null>(null)

const popoverOpen = ref(false)
const popoverMode = ref<'added' | 'manage'>('added')
const popoverTarget = ref<SaveTarget>({ kind: 'article', title: article.value })
const dialogOpen = ref(false)

watch(article, () => {
  popoverOpen.value = false
  parserRoot.value = null
  window.scrollTo(0, 0)
})

/** Non-article namespaces — these open on Wikipedia instead of in the prototype. */
const NAMESPACE =
  /^(file|image|media|special|help|wikipedia|wp|template|category|portal|talk|user|draft|module|mediawiki|timedtext|book|wikt|wiktionary|commons|[a-z ]+ talk):/i

/**
 * Article bodies link with Parsoid-relative hrefs (`./Title`). Keep readers
 * inside the prototype: wiki links open here, same-page anchors scroll, and
 * everything else (files, help pages…) opens on Wikipedia in a new tab.
 */
function onArticleClick(event: MouseEvent) {
  const anchor = (event.target as HTMLElement).closest('a')
  const href = anchor?.getAttribute('href')
  if (!anchor || !href?.startsWith('./')) return
  event.preventDefault()

  const [path, fragment] = href.slice(2).split('#')
  const title = normalizeTitle(decodeURIComponent(path.split('?')[0]))

  if (fragment && title === article.value) {
    document.getElementById(decodeURIComponent(fragment))?.scrollIntoView()
    return
  }
  if (NAMESPACE.test(title) || !anchor.getAttribute('rel')?.includes('mw:WikiLink')) {
    window.open(`https://en.wikipedia.org/wiki/${href.slice(2)}`, '_blank', 'noopener')
    return
  }

  const to = { path: ARTICLE_PATH, query: { title } }
  if (event.metaKey || event.ctrlKey || event.shiftKey) {
    window.open(router.resolve(to).href, '_blank')
  } else {
    void router.push(to)
  }
}

/**
 * Every entry point (article bookmark, section bookmark, paragraph Save) lands
 * here: first press saves and offers collections; pressing a saved item manages it.
 */
async function requestSave(target: SaveTarget) {
  const key = keyOf(target)
  if (popoverOpen.value) {
    popoverOpen.value = false
    // Same item pressed again just closes; a different one reopens fresh.
    if (keyOf(popoverTarget.value) === key) return
    await nextTick()
  }
  dismiss()
  popoverTarget.value = target
  if (isSaved(key)) {
    popoverMode.value = 'manage'
  } else {
    save(target)
    popoverMode.value = 'added'
  }
  popoverOpen.value = true
}

function onBookmarkClick() {
  requestSave({ kind: 'article', title: article.value })
}

function onCollectionCreated(collection: Collection) {
  addToCollection(popoverTarget.value, collection.id)
  show({
    type: 'success',
    text: `${labelOf(popoverTarget.value)} has been added to`,
    link: { label: collection.name, to: `${SAVED_PATH}?collection=${collection.id}` },
  })
}
</script>

<template>
  <SavedChrome>
    <div class="pc-article" @click="onArticleClick">
      <ArticleLive :article="article" @parser-ready="parserRoot = $event">
        <!-- Mobile (Minerva) icon toolbar: the star becomes the save bookmark. -->
        <template #header-mobile-watch>
          <button
            type="button"
            class="pc-mobile-bookmark"
            data-save-trigger
            :aria-label="saved ? 'Saved — manage' : 'Save this article'"
            :aria-pressed="saved"
            :aria-expanded="popoverOpen"
            @click="onBookmarkClick"
          >
            <CdxIcon :icon="saved ? cdxIconBookmark : cdxIconBookmarkOutline" />
          </button>
        </template>
        <template #header-actions>
          <a href="#" class="pc-action pc-action--active" aria-current="true" @click.prevent
            >Read</a
          >
          <a href="#" class="pc-action" @click.prevent>Edit</a>
          <a href="#" class="pc-action" @click.prevent>Edit source</a>
          <a href="#" class="pc-action" @click.prevent>View history</a>
          <a href="#" class="pc-action pc-action--watch" @click.prevent>
            <CdxIcon :icon="cdxIconStar" size="x-small" />Watch
          </a>
          <CdxButton
            class="pc-bookmark"
            weight="quiet"
            data-save-trigger
            :aria-label="saved ? 'Saved — manage' : 'Save this article'"
            :aria-pressed="saved"
            :aria-expanded="popoverOpen"
            @click="onBookmarkClick"
          >
            <CdxIcon :icon="saved ? cdxIconBookmark : cdxIconBookmarkOutline" />
          </CdxButton>
          <CdxButton class="pc-more" weight="quiet" aria-label="More options">
            <CdxIcon class="pc-more__icon" :icon="cdxIconEllipsis" />
          </CdxButton>
        </template>
      </ArticleLive>
      <InArticleSaves
        :root="parserRoot"
        :title="article"
        :focus-section="focusSection"
        :focus-excerpt="focusExcerpt"
        @request="requestSave"
      />
    </div>

    <!-- Desktop only, like Vector 2022's own sticky header. -->
    <StickyHeader
      v-if="globalSkin === 'desktop'"
      :title="article"
      :saved="saved"
      :popover-open="popoverOpen"
      @bookmark-click="onBookmarkClick"
    />

    <template #panel>
      <SavePopover
        v-model:open="popoverOpen"
        :target="popoverTarget"
        :mode="popoverMode"
        :sheet="globalSkin === 'mobile'"
        @create-collection="dialogOpen = true"
      />
    </template>
  </SavedChrome>

  <CreateCollectionDialog v-model:open="dialogOpen" @created="onCollectionCreated" />
</template>

<style scoped>
/* Mirrors ArticleHeader's own action styles (slot content isn't covered by its scoped CSS). */
.pc-action {
  display: inline-flex;
  align-items: center;
  gap: 2px;
  margin: 0 0 -1px;
  padding: var(--spacing-50, 8px) 1px;
  border-bottom: 2px solid transparent;
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.pc-action:hover {
  text-decoration: underline;
}

.pc-action--active {
  border-bottom-color: var(--color-base, #202122);
  color: var(--color-base, #202122);
  font-weight: var(--font-weight-bold, 700);
}

.pc-action--active:hover {
  text-decoration: none;
}

.pc-action--watch :deep(.cdx-icon) {
  color: var(--color-progressive, #36c);
}

/*
 * Icon buttons are 32px wide around a 20px glyph (6px inset per side). Pull
 * them in so the visible gap matches the row's 12px gap between text links.
 */
.pc-bookmark,
.pc-more {
  color: var(--color-base, #202122);
}

.pc-bookmark {
  margin-inline-start: -6px;
}

.pc-more {
  margin-inline: -12px -6px;
}

/* Codex only ships a horizontal ellipsis — rotate it into a vertical "⋮". */
.pc-more__icon {
  transform: rotate(90deg);
}

/* Matches ArticleHeader's mobile `.article-header__icon-tool` buttons. */
.pc-mobile-bookmark {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  padding: 0;
  border: none;
  border-radius: var(--border-radius-base, 2px);
  background: transparent;
  cursor: pointer;
}

.pc-mobile-bookmark :deep(.cdx-icon) {
  color: var(--color-subtle, #54595d);
}

.pc-mobile-bookmark[aria-pressed='true'] :deep(.cdx-icon) {
  color: var(--color-base, #202122);
}
</style>
