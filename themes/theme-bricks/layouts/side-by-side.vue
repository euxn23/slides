<template>
  <div class="slidev-layout side-by-side flex flex-col w-full h-full">
    <slot name="title" />
    <div class="flex text-2xl my-auto pb-24">
      <div class="w-1/2">
        <slot name="left" />
      </div>
      <div class="w-1/2">
        <slot/>
      </div>
    </div>
  </div>
</template>
<style lang="postcss">
.slidev-layout.side-by-side {
  background-image: url("../assets/page-5.svg");
  background-repeat: no-repeat;
  background-position: center;
  background-size: cover;
}
</style>
