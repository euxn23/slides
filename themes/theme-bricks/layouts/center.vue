<template>
  <div class="slidev-layout center w-full h-full flex flex-col justify-center items-center text-center">
    <slot />
  </div>
</template>

<style lang="postcss">
.slidev-layout.center {
  background-image: url("../assets/page-5.svg");
  background-repeat: no-repeat;
  background-position: center;
  background-size: cover;
  padding: 4em 5em;

  h1 {
    @apply text-5xl;
  }
  li {
    @apply text-2xl leading-10;
  }
}
</style>
