<script setup lang="ts">
/**
 * One card on the Saved page. Three looks, one per saved kind:
 *  - article: image, title, short description
 *  - section: article image, section heading
 *  - passage: the excerpt
 * Sections and passages add "from Article" as the description and their kind
 * (icon + "Section" / "Passage") as supporting text underneath.
 *
 * `list` (mobile): a row — square thumbnail on the left, the same text on the right.
 */
import { computed } from 'vue'
import { RouterLink } from 'vue-router'
import { CdxIcon } from '@wikimedia/codex'
import { cdxIconImage, cdxIconQuotes, cdxIconTextSummary } from '@wikimedia/codex-icons'

import { linkOf, useCollections, type SavedItem } from './useCollections'

const props = defineProps<{ item: SavedItem; list?: boolean }>()

const { summary } = useCollections()
const data = computed(() => summary(props.item.title))
const articleTitle = computed(() => data.value?.title ?? props.item.title)
/** A section's own image when it had one, otherwise the article's lead image. */
const imageUrl = computed(() =>
  props.item.kind === 'section' && props.item.imageUrl
    ? props.item.imageUrl
    : (data.value?.thumbnailUrl ?? null),
)
const hasImage = computed(() => !!imageUrl.value)
</script>

<template>
  <!-- Mobile list row -->
  <RouterLink v-if="list" class="saved-row" :class="`saved-row--${item.kind}`" :to="linkOf(item)">
    <!-- Every row keeps the thumbnail slot so rows line up; no image → placeholder. -->
    <img
      v-if="item.kind !== 'passage' && hasImage"
      class="saved-row__thumb"
      :src="imageUrl!"
      alt=""
      loading="lazy"
    />
    <span
      v-else-if="item.kind !== 'passage' && data === undefined"
      class="saved-row__thumb saved-row__thumb--loading"
    />
    <span v-else class="saved-row__thumb saved-row__thumb--placeholder" aria-hidden="true">
      <CdxIcon :icon="item.kind === 'passage' ? cdxIconQuotes : cdxIconImage" />
    </span>

    <span class="saved-row__body">
      <blockquote v-if="item.kind === 'passage'" class="saved-row__quote">
        {{ item.excerpt }}
      </blockquote>
      <span v-else class="saved-row__title">{{
        item.kind === 'section' ? item.heading : articleTitle
      }}</span>

      <span v-if="item.kind === 'article'" class="saved-row__description">
        {{ data?.description }}
      </span>
      <span v-else class="saved-row__description">
        From: {{ articleTitle
        }}<template v-if="item.kind === 'passage' && item.heading"> › {{ item.heading }}</template>
      </span>

      <span v-if="item.kind !== 'article'" class="saved-card__kind saved-row__kind">
        <CdxIcon
          :icon="item.kind === 'section' ? cdxIconTextSummary : cdxIconQuotes"
          size="x-small"
        />
        {{ item.kind === 'section' ? 'Section' : 'Passage' }}
      </span>
    </span>
  </RouterLink>

  <!-- Grid card -->
  <RouterLink v-else class="saved-card" :class="`saved-card--${item.kind}`" :to="linkOf(item)">
    <!-- Content first: the passage's words, or the image. -->
    <blockquote v-if="item.kind === 'passage'" class="saved-card__quote">
      {{ item.excerpt }}
    </blockquote>
    <template v-else>
      <img v-if="hasImage" class="saved-card__image" :src="imageUrl!" alt="" loading="lazy" />
      <span v-else-if="data === undefined" class="saved-card__image saved-card__image--loading" />
      <!-- No image on desktop: text only (mobile rows use a small placeholder instead). -->
    </template>

    <!--
      Title + description share one 4-line budget (a long title leaves fewer
      lines). Sections: the heading is the title, its article the description.
      Passages: the excerpt above is the content, its source the description.
    -->
    <span v-if="item.kind !== 'passage'" class="saved-card__text">
      <span class="saved-card__title">{{
        item.kind === 'section' ? item.heading : articleTitle
      }}</span>
      <template v-if="item.kind === 'section'">
        <br />
        <span class="saved-card__description">From: {{ articleTitle }}</span>
      </template>
      <template v-else-if="data?.description">
        <br />
        <span class="saved-card__description">{{ data.description }}</span>
      </template>
    </span>
    <span v-else class="saved-card__text saved-card__text--source">
      <span class="saved-card__description">
        From: {{ articleTitle }}<template v-if="item.heading"> › {{ item.heading }}</template>
      </span>
    </span>

    <!-- Supporting text: what kind of save this is. -->
    <span v-if="item.kind !== 'article'" class="saved-card__kind">
      <CdxIcon
        :icon="item.kind === 'section' ? cdxIconTextSummary : cdxIconQuotes"
        size="x-small"
      />
      {{ item.kind === 'section' ? 'Section' : 'Passage' }}
    </span>
  </RouterLink>
</template>

<style scoped>
.saved-card {
  display: flex;
  flex-direction: column;
  box-sizing: border-box;
  min-height: 280px;
  padding: var(--spacing-75, 12px);
  border: 1px solid var(--border-color-base, #a2a9b1);
  border-radius: var(--border-radius-base, 2px);
  background-color: var(--background-color-base, #fff);
  color: var(--color-base, #202122);
  text-decoration: none;
}

.saved-card:hover {
  border-color: var(--border-color-progressive--hover, #3056a9);
}


.saved-card:focus-visible {
  outline: 2px solid var(--outline-color-progressive--focus, #36c);
  outline-offset: 1px;
}

.saved-card__image {
  display: block;
  width: 100%;
  aspect-ratio: 166 / 158;
  margin-bottom: var(--spacing-100, 16px);
  border: 1px solid var(--border-color-subtle, #c8ccd1);
  object-fit: cover;
  box-sizing: border-box;
}

.saved-card__image--loading {
  background-color: var(--background-color-interactive-subtle, #f8f9fa);
}

/* Title and description flow as one block so they can share a 4-line clamp. */
.saved-card__text {
  display: -webkit-box;
  overflow: hidden;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 4;
  font-size: var(--font-size-small, 14px);
  line-height: var(--line-height-small, 1.571);
}

.saved-card__title {
  font-weight: var(--font-weight-bold, 700);
}

.saved-card__description {
  color: var(--color-subtle, #54595d);
}

/* Passage cards: the excerpt, in serif, fills the space an image would. */
.saved-card__quote {
  display: -webkit-box;
  margin: 0;
  /* Reset the global blockquote rule's left border and indent. */
  padding: 0;
  border: 0;
  overflow: hidden;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 8;
  font-family: var(--font-family-serif, 'Linux Libertine', Georgia, serif);
  font-size: var(--font-size-medium, 16px);
  line-height: var(--line-height-small, 1.375);
  color: var(--color-base, #202122);
}

/* Passage source line sits under the excerpt. */
.saved-card__text--source {
  margin-top: var(--spacing-75, 12px);
}

/* Supporting text: icon + kind, small and quiet, anchored to the card's bottom
   so it lines up across every card in a row. */
.saved-card__kind {
  display: inline-flex;
  align-items: center;
  gap: var(--spacing-25, 4px);
  margin-top: auto;
  padding-top: var(--spacing-50, 8px);
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-x-small, 12px);
  line-height: var(--line-height-x-small, 1.5);
}

.saved-card__kind :deep(.cdx-icon) {
  color: var(--color-subtle, #54595d);
}

/* ---- Mobile list row ---- */
.saved-row {
  display: flex;
  align-items: flex-start;
  gap: var(--spacing-75, 12px);
  padding: var(--spacing-100, 16px) 0;
  color: var(--color-base, #202122);
  text-decoration: none;
}

.saved-row__thumb {
  flex-shrink: 0;
  box-sizing: border-box;
  width: 72px;
  height: 72px;
  border: 1px solid var(--border-color-subtle, #c8ccd1);
  object-fit: cover;
}

.saved-row__thumb--loading {
  background-color: var(--background-color-interactive-subtle, #f8f9fa);
}

.saved-row__body {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.saved-row__title {
  font-size: var(--font-size-medium, 16px);
  font-weight: var(--font-weight-bold, 700);
  line-height: var(--line-height-small, 1.375);
}


.saved-row__description {
  display: -webkit-box;
  overflow: hidden;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 2;
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-small, 14px);
  line-height: var(--line-height-small, 1.571);
}

.saved-row__quote {
  display: -webkit-box;
  margin: 0;
  padding: 0;
  border: 0;
  overflow: hidden;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 3;
  font-family: var(--font-family-serif, 'Linux Libertine', Georgia, serif);
  font-size: var(--font-size-medium, 16px);
  line-height: var(--line-height-small, 1.375);
}

.saved-row__kind {
  margin-top: var(--spacing-25, 4px);
  padding-top: 0;
}

/* Mobile rows: no-image placeholder with the same footprint as a real thumbnail. */
.saved-row__thumb--placeholder {
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: var(--background-color-interactive-subtle, #f8f9fa);
}

.saved-row__thumb--placeholder :deep(.cdx-icon) {
  color: var(--color-placeholder, #72777d);
}
</style>
