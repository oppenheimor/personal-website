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

  // 添加打开和关闭事件监听器
  // 当 zoom 打开时，禁用页面滚动
  zoom.on('open', () => {
    document.body.style.overflow = 'hidden'
  })

  zoom.on('close', () => {
    document.body.style.overflow = ''
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

/* 确保 medium-zoom 的容器可以正常双指缩放 */
.medium-zoom-overlay,
.medium-zoom-image--opened {
  touch-action: none !important;
}
</style>
