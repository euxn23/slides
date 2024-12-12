<template>
  <div class="slidev-layout list w-full h-full flex flex-col justify-center items-center">
    <slot/>
  </div>
</template>
<style lang="postcss">
.slidev-layout.list {
  background-image: url("../assets/page-5.svg");
  background-repeat: no-repeat;
  background-position: center;
  background-size: cover;
  padding: 4em 5em;

  h2 {
    @apply leading-10 my-4;
  }
  li {
    @apply text-2xl leading-10 my-2;
  }
  p {
    @apply text-xl leading-10;
  }
}
</style>
