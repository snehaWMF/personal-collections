<script setup lang="ts">
definePage({
  meta: {
    title: 'Personal collections (old address)',
    platform: 'web',
    hidden: true,
  },
})

/**
 * Old address → `/article`. A full-page `location.replace` (not a router push)
 * so the new page load applies its usual starting-point rules (bare URL resets
 * to sample collections, `?reset` to none). `?article=` becomes `?title=`.
 */
import { useRoute, useRouter } from 'vue-router'

import { ARTICLE_PATH } from './useCollections'

const route = useRoute()
const router = useRouter()

const query = { ...route.query }
if (query.article !== undefined) {
  query.title = query.article
  delete query.article
}
window.location.replace(router.resolve({ path: ARTICLE_PATH, query }).href)
</script>

<template>
  <span />
</template>
