<script setup lang="ts">
import Tooltip from './Tooltip.vue'

defineProps<{
  title?: string
  disabled?: boolean
  active?: boolean
  noBg?: boolean
}>()

const emit = defineEmits(['click'])
</script>

<template>
  <Tooltip :title="title">
    <button
      type="button"
      v-bind="$attrs"
      @click="e => !disabled && emit('click', e)"
      class="icon-wrapper"
      :class="{ disabled, noBg, active }"
      :disabled="disabled"
      :aria-label="title"
    >
      <slot />
    </button>
  </Tooltip>
</template>

<style scoped lang="scss">
$w: 1.25rem;
.icon-wrapper {
  cursor: pointer;
  width: 2.75rem;
  height: 2.75rem;
  padding: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: 0;
  border-radius: var(--focus-radius-sm);
  color: var(--focus-ink-secondary);
  background: transparent;
  transition:
    color var(--focus-duration) var(--focus-ease),
    background-color var(--focus-duration) var(--focus-ease),
    opacity var(--focus-duration) var(--focus-ease);

  &:hover:not(.disabled, .noBg) {
    color: var(--focus-ink);
    background: var(--focus-surface-strong);
  }

  &.disabled {
    cursor: not-allowed;
    opacity: 0.3;
  }

  &.active {
    color: var(--focus-accent);
    background: var(--focus-accent-soft);
  }

  :deep(svg) {
    width: $w;
    height: $w;
  }
}
</style>
