<template>
  <div class="slidev-layout intro w-full h-full">
    <slot name="title" />
    <div class="flex gap-8 mt-16">
      <div class="flex">
        <slot name="left" />
      </div>
      <div class="text-2xl">
        <slot/>
      </div>
    </div>
  </div>
</template>
<style lang="postcss">
.slidev-layout.intro {
  background-image: url("../assets/page-5.svg");
  background-repeat: no-repeat;
  background-position: center;
  background-size: cover;
}
</style>
<script setup lang="ts">
</script>
