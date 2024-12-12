<template>
  <span class="opacity">
    <slot />
  </span>
</template>

<script setup lang="ts">
defineProps<{ opacity: number }>()
</script>

<style scoped lang="postcss">
.opacity, :slotted(*) {
  opacity: v-bind(opacity / 100);
}
</style>
