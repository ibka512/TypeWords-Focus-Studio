<script setup lang="ts">

</script>

<template>
  <div class="focus-page-shell">
    <div class="page">
      <slot></slot>
    </div>
  </div>
</template>

<style scoped lang="scss">
.focus-page-shell {
  display: flex;
  justify-content: center;
  box-sizing: border-box;
  padding: 2rem;
}

.page {
  width: min(100%, 76rem);
  min-height: calc(100vh - 8.75rem);
}


@media (max-width: 768px) {
  .page {
    width: 100%;
    min-height: calc(100vh - 6rem);
  }

  .focus-page-shell {
    padding: 1rem 0.75rem;
  }
}

// 超小屏幕适配
@media (max-width: 480px) {
  .focus-page-shell {
    padding-inline: 0.625rem;
  }
}
</style>
