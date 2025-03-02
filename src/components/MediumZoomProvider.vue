<script setup lang="ts">
import mediumZoom from 'medium-zoom'
import { onBeforeUnmount, onMounted } from 'vue'

let zoom: ReturnType<typeof mediumZoom>

onMounted(() => {
  zoom = mediumZoom('.medium-zoom-image', {
    margin: 24,
    background: 'rgba(0, 0, 0, 0.9)',
    scrollOffset: 40,
  })
})

onBeforeUnmount(() => {
  if (zoom)
    // 清理 zoom 实例
    zoom.detach()
})
</script>

<template>
  <slot />
</template>

<style>
.medium-zoom-overlay {
  z-index: 100;
}

.medium-zoom-image--opened {
  z-index: 101;
}
</style>
