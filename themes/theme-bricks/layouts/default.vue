<template>
  <div class="slidev-layout default w-full h-full">
    <slot/>
  </div>
</template>

<style lang="postcss">
.slidev-layout.default {
  background-image: url("../assets/page-5.svg");
  background-repeat: no-repeat;
  background-position: center;
  background-size: cover;

  h3 {
    @apply text-xl tracking-widest;
  }
}
</style>
