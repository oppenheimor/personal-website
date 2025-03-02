<script setup lang="ts">
// https://github.com/vueuse/head
// you can use this to manipulate the document head in any components,
// they will be rendered correctly in the html results with vite-ssg
// import { SpeedInsights } from '@vercel/speed-insights/vue'
// import { Analytics } from '@vercel/analytics/nuxt'

useHead({
  title: "Vitesse",
  meta: [{ name: "description", content: "Opinionated Vite Starter Template" }],
});
</script>

<template>
  <!-- <SpeedInsights /> -->
  <!-- <Analytics /> -->
  <nav-bar />
  <router-view />
</template>
